# Design Doc: difference-ai-app

A phone spec database, search/comparison tool, and AI-assisted phone finder built on Next.js 14 (App Router) + Postgres/Prisma. This doc explains how the pieces fit together and why some non-obvious decisions were made. For local setup, env vars, and deployment steps, see [README.md](README.md) — this doc does not repeat that.

## 1. Goals and non-goals

- **Goal**: let a visitor search/filter a phone catalog, compare up to 4 phones side by side, and get AI-assisted recommendations or Q&A grounded in real spec data.
- **Goal**: tolerate an LLM provider being unavailable, slow, or unconfigured — every AI-backed feature has a deterministic non-LLM fallback so the site never hard-fails on a phone question.
- **Non-goal**: real-time pricing or retailer inventory. The schema has no price field; affiliate links are search-query URLs, not per-SKU deep links (see [§7](#7-affiliate-links--analytics)).
- **Non-goal**: user accounts, saved comparisons across sessions, or multi-instance rate limiting (see [§6](#6-rate-limiting--abuse-protection)).

## 2. Tech stack

| Layer | Choice | Notes |
|---|---|---|
| Framework | Next.js 14, App Router | `next.config.mjs` is empty — no custom webpack/experimental flags |
| Language | TypeScript, `strict: true` | one legacy CommonJS file, see [§3](#3-data-model--the-json-string-column-workaround) |
| DB | Postgres via Prisma 6 | migrated off SQLite; `DATABASE_URL`/`DIRECT_URL` split for pooled vs. direct connections |
| Styling | Tailwind CSS v4 | |
| Analytics | `@vercel/analytics` | thin event wrappers in `src/lib/analytics.ts` |
| AI | OpenAI / Hugging Face / Ollama, chained with a deterministic fallback | `src/lib/ai-providers.ts`, see [§5](#5-ai-provider-abstraction) |
| Tests | Vitest | added for the pure parsing/formatting logic in `src/lib`, see [§9](#9-testing) |
| Package manager | npm | `package-lock.json` is the only lockfile |

One version mismatch worth knowing about: `eslint-config-next` is pinned to `15.4.4` while `next` itself is `14.0.4` — this is why `next lint` currently drops into an interactive "how would you like to configure ESLint?" prompt instead of running directly (plain `npx eslint <files>` still works against `eslint.config.mjs`).

## 3. Data model & the JSON-string-column workaround

`prisma/schema.prisma` defines two models:

- **`Brand`**: `id`, `slug` (unique), `name`, timestamps, `devices` relation.
- **`Device`**: `id`, `brandId` (FK, cascade delete), `slug`, `model`, `name`, `imageUrl?`, plus:
  - `specBlob` / `rawPayload` — the normalized spec sections and the original scraped payload, both JSON-stringified.
  - Structured, individually-typed spec columns pulled out of `specBlob` at import time: `displaySizeInches`, `displayResolution`, `displayRefreshRate`, `displayType`, `performanceChipset`, `performanceChipsetNodeNm`, `cameraMainMp`, `cameraFrontMp`, `batteryCapacityMah`, `batteryWiredChargingW`, `weightG`, `releaseDate`, `isDiscontinued`.
  - `performanceRamOptions` / `storageOptions` — **`String?`, not a native array or JSON column** — each holds a JSON-stringified `number[]` (e.g. `"[8,12,16]"`).
  - `@@unique([brandId, slug])`, `@@index([brandId, name])`.

The RAM/storage-as-JSON-string design is a deliberate simplification (one row per phone, not a separate `DeviceMemoryOption` table) but it has a real cost: **Prisma cannot express "device has a RAM option ≥ X" as a database-level filter** against a string column. Every place that needs a RAM threshold works around this the same way — fetch a candidate set at the DB level, then filter in JS:

- `searchDevices` in [`src/lib/phone-catalog.ts:95-192`](src/lib/phone-catalog.ts) — when `minRam` is set, drops the normal `take: 60` cap so the JS-side RAM filter isn't working from an already-biased alphabetical slice, then re-slices to 60 after filtering.
- `getCandidatePool` in [`src/lib/phone-recommendation.ts:117-158`](src/lib/phone-recommendation.ts) — same pattern, with progressive relaxation (see [§5](#5-ai-provider-abstraction)) if the RAM-filtered result is empty.

Both call sites parse the column with `parseNumericArrayBlob` (`src/lib/phone-normalization.js:385-400`), which degrades to `[]` on missing/invalid JSON rather than throwing — every caller can treat "no RAM data" and "malformed RAM data" identically.

**If a future feature needs a real "≥X GB" filter to scale past a few thousand rows**, the fix is a proper `Int[]` column (Postgres supports native arrays) or a child table — not a bigger JS-side scan. This doc flags it; it hasn't been a performance problem yet.

The one non-TypeScript file in `src/lib` is `phone-normalization.js` (CommonJS, `module.exports`) — `allowJs: true` and the `@/*` path alias let it be imported from `.ts` files (e.g. `src/lib/device-detail.ts:3-7`) with a manual type cast on the destructured functions. It was left as JS because it was the original scraping-import script's normalization logic; there's no technical reason it couldn't be converted to TS if it's touched significantly again.

## 4. Data ingestion

`normalizePhoneRecord` (`phone-normalization.js:319-371`) is the single entry point that turns a raw scraped/imported phone object into a `Device` row: it accepts either a `specs` object-of-objects shape or a `specifications` array shape (some source data used one, some the other), maps section/field names through an alias table (`SECTION_ALIASES`, lines 1-18, e.g. `memory`→`performance`, `body`→`design`), and runs a battery of regex parsers over the normalized sections — RAM options, storage options, refresh rate, release date, chipset name/node size, camera MP, battery mAh, wired charging watts, weight. These regexes are the highest-risk code in the project (see [§9](#9-testing) — commit `f6a7233` fixed exactly this class of bug, where a storage-GB value was being misread as a RAM value).

Two scripts drive ingestion (`scripts/`):
- `import-phone-data.mjs` — reads JSON files from `data/imports/`, runs them through `normalizePhoneRecord`, upserts `Brand`/`Device` by `brandId_slug`. This is `npm run import:phones`.
- `import-phone-backfill.mjs` — same normalization pipeline but only inserts records missing from the catalog (no upsert-overwrite of existing rows).

The GSMArena scraper itself was removed (commit `58c891c`, "Remove dead GSMArena scraping/backfill tooling") — the project no longer scrapes live; new data arrives as JSON files fed into the importer above. Two housekeeping items fell out of the codebase exploration for this doc and are safe to clean up whenever convenient: a stray `scripts/__pycache__/scrape-gsmarena.cpython-38.pyc` (bytecode cache for the deleted scraper) and a root `requirements.txt` (`beautifulsoup4`, `requests`) that only existed to support it.

The SQLite→Postgres migration path (`export-sqlite-catalog.mjs` / `import-catalog-dump.mjs`) is now a one-way historical artifact rather than an active workflow — kept in case the dump needs re-importing, but not part of the normal ingestion loop.

## 5. Core request flows

### 5.1 Search & filter (home page)

`src/app/page.tsx` holds controlled inputs for text query, brand, release year, size range, min battery, min RAM, and an "available only" checkbox, debounced 200ms before calling `GET /api/phones/search`. Critically, `hasActiveSearch` gates whether any request fires at all — with zero active filters the page shows an empty-state placeholder rather than dumping the full catalog. `GET /api/phones/search` → `searchDevices()` (`phone-catalog.ts:95-192`) builds an `AND` of optional Prisma conditions plus the JS-side RAM workaround from [§3](#3-data-model--the-json-string-column-workaround).

`GET /api/phones` (brand list + release years, for filter dropdowns) and `GET /api/phones/[brand]` (devices in one brand) are simpler read-only endpoints with no rate limiting — they're cheap, cacheable-shape reads with no LLM or write cost, unlike the two POST endpoints below.

### 5.2 Compare workspace

State lives in `sessionStorage` under the key `differenceai:selectedDevices` (`page.tsx:41`), capped at `MAX_COMPARE = 4` (`page.tsx:39`):
- On mount, one effect hydrates `selectedDevices` from `sessionStorage` directly (bypassing `addToComparison`, so restoring a previous session doesn't re-fire "phone selected" analytics).
- A separate effect persists `selectedDevices` back to `sessionStorage` on every change.
- A third mount-only effect reads a `?devices=id1,id2,...` query param (used by "Compare this phone" deep links from device pages), resolves each ID via `GET /api/phones/device/[id]`, and adds them up to the cap.

`addToComparison` no-ops if the device is already selected or the cap is reached; `submitComparison` requires ≥2 devices and navigates to `/compare?devices=...`.

`src/app/compare/page.tsx` fetches each device's detail in parallel and renders a comparison table. **The table's row/category structure is taken entirely from the first selected device's `detailSpec`** (`buildDeviceDetail` output, [§3](#3-data-model--the-json-string-column-workaround) is the shape source) — other devices' values are looked up by matching `categoryIndex`/`specIndex` position, not by category/spec name. A device with different category ordering or a missing category would silently misalign rather than error. This is fine today because `buildDeviceDetail` always emits the same fixed category list (Display, Performance, Camera, Battery & Charging, Design & Build, Connectivity, Software) for every device — but if that function's output shape ever becomes conditional per-device, the compare table needs to switch to name-based lookup.

### 5.3 AI phone finder (`PhoneFinderChat` → `/api/recommend`)

`askPhoneRecommendation` (`src/lib/phone-recommendation.ts`) is a three-stage pipeline, deliberately split so the LLM never touches the database directly:

1. **`extractSoftFilters`** (`phone-recommendation.ts:65-94`) — a pure, LLM-free regex/keyword pass over the free-text description (e.g. "compact"/"small" → `sizeRange: 'compact'`, "gaming"/"flagship" → `minRam: 8`, a brand name mention → `brandSlug`). Fully inspectable and cheap.
2. **`getCandidatePool`** (`:117-158`) — runs `extractSoftFilters`'s output through Prisma to fetch a bounded candidate set (`CANDIDATE_POOL_SIZE`), always excluding discontinued and >10-year-old phones. If the soft filters were too narrow and matched nothing, it retries once with no soft filters rather than ever handing the LLM an empty pool.
3. An LLM (via `runProviderChain`, [§6](#6-ai-provider-abstraction)) picks from that bounded, pre-filtered JSON candidate list and returns structured picks — it never sees raw DB access, only the JSON payload it's handed.

`POST /api/recommend` is rate-limited (10 requests/60s per IP) and validates the description is present and ≤500 chars.

### 5.4 Comparison assistant (`ComparisonChat` → `/api/compare/chat`)

`askComparisonAssistant` (`src/lib/comparison-assistant.ts:201-235`) resolves 1–6 device IDs to summaries via `buildPhoneSummary` (`:83-140`), which — like `buildDeviceDetail` in [§3](#3-data-model--the-json-string-column-workaround) — prefers structured Prisma columns and only falls back to regex-parsing `specBlob` when a structured field is null. The entire summary is embedded as JSON directly in the LLM prompt with explicit "answer only from this data, do not invent prices/benchmarks" instructions (`buildComparisonPrompt:142-167`) — this is single-shot Q&A grounded in structured specs, not a free multi-turn chat; there's no conversation history sent back to the model.

Two response-shaping layers worth knowing about:
- An in-memory cache (`Map`, 5-minute TTL, 200-entry cap, oldest-inserted-key eviction — not true LRU) keyed by sorted device IDs + normalized question, so repeated identical questions don't re-hit the LLM.
- If `runProviderChain` returns `null` (all providers failed/unconfigured), `buildFallbackAnswer` (`:169-190`) builds a deterministic bullet list from the same summary fields — no LLM involved — and the response reports `source: 'fallback'`.

`POST /api/compare/chat` is rate-limited identically to `/api/recommend` (10/60s per IP), and validates 1–6 device IDs and a ≤2000-char prompt.

## 6. AI provider abstraction

`src/lib/ai-providers.ts` chains three real providers plus an implicit fallback sentinel:

```
COMPARE_ASSISTANT_PROVIDER (if set)
  → openai (if OPENAI_API_KEY set and COMPARE_ASSISTANT_DISABLE_OPENAI unset)
  → huggingface (if HUGGINGFACE_API_KEY or HF_TOKEN set)
  → ollama (no key required — assumes a local/self-hosted endpoint)
  → 'fallback' sentinel
```

`getProviderOrder()` (`:271-285`) is a pure function producing this ordering — easy to unit test in isolation (not yet done; see [§9](#9-testing)). `runProviderChain()` (`:293-315`) walks the order, tries each real provider, logs+swallows individual failures, and stops at the `'fallback'` sentinel **without calling anything itself** — by design, every caller (`comparison-assistant.ts`, `phone-recommendation.ts`) owns its own deterministic fallback rather than `ai-providers.ts` having one baked in. This keeps the fallback behavior specific to what each feature can reconstruct from structured data (a bullet-point spec summary for comparisons, a filtered device list for recommendations) instead of a generic "AI unavailable" message.

Each provider has its own request-shape quirks handled by dedicated parsers: `getTextFromOpenAIResponse` (OpenAI's Responses API — `output_text` or walking `output[].content[]`) vs. `getTextFromChatCompletionResponse` (standard chat-completions `choices[].message.content`, used by both Hugging Face and Ollama). Requests are wrapped in `fetchWithTimeout` — `20s` generically, `40s` for Hugging Face specifically (extended in commit `0f45e75` after observed slow cold-starts on some HF-hosted models).

## 7. Affiliate links & analytics

`getRetailerLinks` (`src/lib/affiliate-links.ts`) builds **search-query URLs**, not per-SKU deep links — the schema has no retailer product ID, so a link like the Amazon one is a tagged search for the device name rather than a direct product page. Each retailer is gated behind its own env var (`AFFILIATE_AMAZON_ID`, `AFFILIATE_EBAY_ID`, `AFFILIATE_BESTBUY_ID`) and is **omitted entirely**, not shown untracked, if that var is unset — so in dev/preview environments without affiliate IDs configured, expect fewer "Buy" options to render, which is correct behavior, not a bug.

`src/lib/analytics.ts` wraps `@vercel/analytics`'s `track()` with one named function per event (phone view, compare clicked, affiliate click, AI request, search CTA, finder CTA, search input focus, search performed, phone selected, compare started, recommendation requested) — keeping event names centralized in one file rather than scattered `track('...')` string literals through components.

## 8. Rate limiting & abuse protection

`checkRateLimit` (`src/lib/rate-limit.ts:26-49`) is an **in-memory, per-process, fixed-window** limiter — a `Map<key, {count, resetAt}>` with periodic sweeping of expired buckets (every 500th call). This is explicitly documented in the file's own header comment as fine for a single-instance deployment but **not correct if the app ever runs as multiple instances or serverless functions**, since each instance/invocation would have its own independent counter — a determined client could get up to `limit × instance-count` requests through. If the deployment target changes to a multi-instance or serverless model, this needs to move to a shared store (Redis, Postgres, etc.) before the rate limits mean anything.

Both `/api/recommend` and `/api/compare/chat` apply the identical pattern — `getClientIp(request)` → `checkRateLimit(prefix + ip, 10, 60_000)` → 429 with a `Retry-After` header on block — but the ~15 lines implementing it are duplicated between the two route files rather than factored into a shared helper. Low-risk duplication (small, unlikely to drift), but worth extracting into e.g. `src/lib/api-guard.ts` if a third rate-limited route is ever added.

`getClientIp` prefers the runtime-injected `request.ip` (available on some hosts), then `x-forwarded-for` (first entry), then `x-real-ip`, then `'unknown'` — `'unknown'` intentionally still gets its own rate-limit bucket rather than bypassing limiting entirely.

## 9. Testing

Vitest was added (this session) to cover the parsing/formatting logic identified above as the highest-risk, lowest-test-cost code: pure functions with no DB or network dependency, directly implicated in a real regression (`f6a7233`).

| File | Covers |
|---|---|
| `src/lib/phone-normalization.test.ts` | `slugify`/`normalizeKey`, RAM/storage/refresh-rate/date/chipset parsing (via `normalizePhoneRecord`), `parseNumericArrayBlob` |
| `src/lib/device-detail.test.ts` | `buildDeviceDetail`'s storage formatting and RAM/storage/wireless-charging/refresh-rate regex fallbacks |
| `src/lib/rate-limit.test.ts` | `checkRateLimit` windowing/reset behavior, `getClientIp` header precedence |

Run with `npm test` (`vitest run`) or `npm run test:watch`. Config lives in `vitest.config.ts` (Node environment, `@/*` alias mirroring `tsconfig.json`, and an inline empty `css.postcss` config to stop Vite from trying to load the project's Tailwind `postcss.config.mjs` during test runs).

**Not yet covered**, in rough priority order for a follow-up pass:
- `extractSoftFilters` / `getCandidatePool` (`phone-recommendation.ts`) — needs a mocked Prisma client.
- `getProviderOrder` (`ai-providers.ts`) — pure, no excuse not to cover it, just wasn't in this pass's scope.
- The response-shape parsers in `ai-providers.ts` (`getTextFromOpenAIResponse`, `getTextFromChatCompletionResponse`) — pure given a fixture payload, no network needed.
- API route tests (`/api/recommend`, `/api/compare/chat`, search) with mocked Prisma + mocked `fetch` for the LLM calls.
- No e2e coverage of the compare workspace or AI finder flows exists (would need Playwright).

The README's Scripts/Validation sections don't yet mention `npm test` — worth a follow-up doc fix so a future contributor knows the suite exists.

## 10. Known limitations & follow-ups

Collected from the analysis above, roughly in order of "worth doing soon" to "fine to leave":

1. **Rate limiting won't survive multi-instance/serverless deployment** ([§8](#8-rate-limiting--abuse-protection)) — revisit before scaling horizontally.
2. **Compare table alignment is positional, not name-based** ([§5.2](#52-compare-workspace)) — safe only as long as `buildDeviceDetail` emits a fixed category shape for every device.
3. **RAM/storage-as-JSON-string columns require JS-side filtering** ([§3](#3-data-model--the-json-string-column-workaround)) — fine at current catalog size; would need a real array column or child table to scale.
4. Minor duplicated logic: the rate-limit-check boilerplate between the two POST routes, and the provider-source `meta` string ternary between `PhoneFinderChat.tsx` and `ComparisonChat.tsx` — both harmless today, candidates for extraction if a third instance shows up.
5. `eslint-config-next@15.4.4` vs `next@14.0.4` version mismatch breaks `npx next lint`'s non-interactive mode.
6. Dead artifacts from the removed GSMArena scraper: `scripts/__pycache__/scrape-gsmarena.cpython-38.pyc`, root `requirements.txt`.
7. README's API route list omits `/api/recommend` and doesn't mention `npm test`.
