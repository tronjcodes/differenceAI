# DifferenceAI

DifferenceAI is a smartphone comparison app built with Next.js, Prisma, and PostgreSQL. It hosts a self-owned catalog of thousands of devices with normalized specifications, and pairs it with AI features that help you find and compare phones in plain language.

## Features

- **Search and filter** a Postgres-backed catalog of thousands of phones by name, brand, release year, screen size, minimum battery, minimum RAM, and availability.
- **AI phone finder** — describe what you want in plain language (e.g. "a compact phone with a great camera") and get matching picks from the catalog.
- **Side-by-side comparison** of up to four phones at once, with specs normalized into consistent categories (display, performance, camera, battery, design, connectivity, software).
- **AI comparison assistant** — ask natural-language questions about the phones you've selected; answers are grounded in the actual stored specs, with a deterministic fallback if no AI provider is configured.
- **Device detail pages** with full spec breakdowns and related-phone suggestions.
- **Buy links** to retailers (Amazon, eBay, Best Buy) when affiliate IDs are configured.

## Tech Stack

- Next.js 14 (App Router) + React 18 + TypeScript
- Prisma + PostgreSQL
- Tailwind CSS
- Vitest for unit tests
- AI: OpenAI, Hugging Face, or a local Ollama model (optional — the app works without any of them)

## Running Locally

### 1. Install dependencies

```bash
npm install
```

This also runs `prisma generate` automatically.

### 2. Set up a local PostgreSQL database

Create a database, then set `DATABASE_URL` (and `DIRECT_URL`, which can be the same value for local dev) in `.env.local`:

```bash
DATABASE_URL=postgresql://localhost:5432/differenceai_dev
DIRECT_URL=postgresql://localhost:5432/differenceai_dev
```

Prisma's CLI reads `.env`, not `.env.local` — copy the same two variables into a `.env` file as well (or export them in your shell) before running Prisma commands.

Apply the schema:

```bash
npx prisma migrate deploy
```

### 3. Load phone data

The repo ships with a small sample dataset (`data/imports/sample-phones.json`) so the app runs locally without the full production catalog. Import it (and anything else you drop into `data/imports/`):

```bash
npm run import:phones
```

### 4. Run the app

```bash
npm run dev
```

Open `http://localhost:3000`.

### 5. (Optional) Configure the AI features

Without any AI provider configured, the phone finder and comparison assistant still work using deterministic, spec-based fallbacks. To enable real AI answers, set one of the provider configs documented in `.env.example` (OpenAI, Hugging Face, or a local Ollama server).

## Testing

```bash
npm test
```

Runs the Vitest suite covering the phone-spec parsing and normalization logic.
