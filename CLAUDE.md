<!-- GSD:project-start source:PROJECT.md -->

## Project

**YieldGlance**

YieldGlance is a standalone single-page web app that shows **live dividend yield instead of price** for a small, fixed watchlist of securities — starting with the MicroStrategy/Strategy preferred shares STRK, STRC, STRD, and SATA. The hero is an above-the-fold table (ticker · as-of date · current dividend yield), where yield = annual dividend ÷ current price, computed live in the visitor's own browser on page load. It is a personal, trading-desk-style dashboard — informational only, not investment advice.

**Core Value:** **One glance gives an accurate, current dividend yield per ticker** — even when the data feed doesn't cover these niche tickers, because manual override always lets the number be correct.

### Constraints

- **Budget**: Strictly zero-cost — Cloudflare Pages free tier, free Worker/Function, free data API only. No paid tiers, no database.
- **Tech stack**: Astro static site; Tailwind-style utility classes implied by the design idiom; Cloudflare Pages + Pages Function/Worker proxy; localStorage for persistence; client-side fetch.
- **Security**: Data-API key must live only in the Worker/Function environment — never exposed in client JS or the repo.
- **Compatibility**: Must run as a static SPA in a modern browser; degrade gracefully when feeds are missing or rate-limited.
- **Compliance**: Securities context — no buy/sell language, no ISIN/WKN, mandatory informational disclaimer.
- **Scope**: One screen, table-first, fixed four-ticker watchlist, fetch-on-load only.

<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->

## Technology Stack

## Critical Question 1: Data Source for STRK / STRC / STRD / SATA

### Background on the Tickers

| Ticker | Full Name | IPO |
|--------|-----------|-----|
| STRK | Strategy Inc 8% Series A Perpetual Strike Preferred Stock | Jan 2025 |
| STRC | Strategy Inc Variable Rate Series A Perpetual Stretch Preferred Stock | Jul 2025 |
| STRD | Strategy Inc 10% Series A Perpetual Stride Preferred Stock | Jun 2025 |
| SATA | Strive Inc preferred stock | Nov 2025 |

### Candidate APIs: Evaluation Matrix

| API | Free-tier limits | API key needed | CORS (browser direct) | Preferred stock coverage | Dividend data | Price history | Verdict |
|-----|-----------------|----------------|----------------------|--------------------------|---------------|---------------|---------|
| **Yahoo Finance v8 (unofficial)** | Undocumented; IP-based throttle; community estimates ~360 req/hour | No | **Blocked** — CORS + cookie required; must run server-side | HIGH — Yahoo Finance UI pages exist for all four tickers; v8 chart API covers what the site covers | YES — `events=div` in query string returns dividend events array | YES — decades of history in v8 chart response | **#1 candidate via proxy** |
| **Tiingo EOD** | 1000 req/day, 50 req/hour, 500 unique symbols/month | YES (free registration) | NOT documented as browser-safe; API key in `Authorization` header; proxy required regardless | MEDIUM — 65,000+ US stocks/ETFs. Prefers dash notation (BRK-A). STRK/STRC/STRD are plain symbols and should work. Coverage of IPOs <1 year old is uncertain | YES — dedicated corporate-actions/dividends endpoint | YES — EOD data | **#2 candidate via proxy** |
| **Finnhub** | 60 req/min; free US stocks + basic fundamentals | YES | **Blocked** — open CORS GitHub issue unresolved; content-type preflight rejection | MEDIUM — covers NASDAQ stocks. Preferred share coverage for niche tickers unconfirmed | Dividend yield in metrics endpoint, not historical dividend calendar | YES — stock candles (25-year history for US stocks) | **#3 candidate via proxy, dividend data weak** |
| **Alpha Vantage** | 25 req/day (extremely restrictive) | YES | Not browser-safe; proxy required | LOW — preferred stocks use quirky symbol conventions (e.g. CLNY-PJ); STRK/STRC may or may not work as plain symbols | YES — TIME_SERIES_DAILY returns adjusted data | YES — 20+ years | **REJECT** — 25 req/day is unusable |
| **Twelve Data** | 800 req/day, 8 req/min; "internal non-display usage" restriction on free tier | YES | Not specified; proxy required regardless | MEDIUM — 1M+ instruments claimed; "global trial symbols" restriction on free plan is undocumented | YES — dividend calendar endpoint | YES | **RISKY** — "non-display usage" clause conflicts with a public dashboard; legal risk |
| **Financial Modeling Prep (FMP)** | 250 req/day; EOD only | YES | 403 blocked (pricing page inaccessible) | UNKNOWN — not tested against these specific tickers | YES — stable dividends endpoint | YES — 5-year EOD | **LOW PRIORITY** — 250 req/day tight for dashboard |
| **Polygon.io** | 5 req/min; free tier | YES | Not browser-safe; proxy required | MEDIUM — "all US equities markets" but dividend data noted as unreliable by reviewers | UNRELIABLE per multiple reviews | YES | **REJECT** — unreliable dividends, tight rate limit |

### Ranked Recommendation

### Explicit Manual-Entry Fallback

## Critical Question 2: Build Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Astro | 6.4.x (latest stable) | Static site generator | Ships zero JS to client by default; `output: 'static'` for pure static build; Cloudflare acquisition means long-term backing; Node 22 required. The Tailwind v4 + rolldown-vite compatibility issue (see below) is the one friction point in v6. Alternatively pin to Astro 5.17.x (equally stable, no rolldown issue). |
| Tailwind CSS | 4.3.x | Utility CSS | Design system uses slate-900 / brand-gold / mono / tracking-widest — all expressible as Tailwind utilities. `@theme` block in v4 replaces tailwind.config.js for custom tokens. One `@import "tailwindcss"` in global.css. |
| @tailwindcss/postcss | 4.x | Tailwind integration for Astro 6 | Astro 6 ships Vite 7 with rolldown bundler. `@tailwindcss/vite` conflicts with rolldown's `tsconfigPaths` field — build fails. Use `@tailwindcss/postcss` + `postcss.config.mjs` instead. If on Astro 5 (pinned), use `@tailwindcss/vite` in `astro.config.mjs` which works correctly. |
| Cloudflare Pages | Free tier | Static hosting | 100k Function invocations/day (shared with Workers); unlimited static asset requests; git-connected CI; custom domains on free tier. Note: Cloudflare is pushing Cloudflare Workers as the future platform; Pages is in maintenance mode but fully stable. |
| Cloudflare Pages Functions | Free tier (100k req/day shared) | Thin proxy to hide data-API key and resolve CORS | File-based routing under `/functions/` directory; `env.SECRET_NAME` for secrets set via `wrangler secret put`; `onRequest(context)` handler returns a `Response`. No framework needed — plain JS/TS. |
| TypeScript | 5.x (via Astro built-in) | Type safety | Astro includes TS support out of the box; use `.ts` for Worker functions and client scripts; no separate tsconfig needed beyond Astro's default. |
| Inter / Merriweather | via @fontsource or Astro 6 Fonts API | Typography | Matches existing blog. Astro 6 has a built-in Fonts API for self-hosting and fallback generation. If on Astro 5, use `@fontsource/inter` and `@fontsource/merriweather`. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| @fnando/sparkline | latest (npm) | Inline SVG sparkline chart, zero dependencies | Use for the per-ticker price-history sparkline. Accepts array of numbers. `onmousemove` callback enables hover tooltip. Only load when price history data is available — do not render in full manual mode. |
| wrangler | 3.x (devDep) | Local Cloudflare Pages dev + secret management | `wrangler pages dev` for local testing of Functions. `wrangler secret put API_KEY` for production secrets. `.dev.vars` file for local secret injection (gitignored). |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| Node.js | Runtime | v22+ required for Astro 6. v18.17+ for Astro 5. |
| Wrangler CLI | Local Pages dev server, deployment, secret management | `npx wrangler pages dev ./dist --compatibility-date=2025-01-01` to test Functions locally against built output. |
| `.dev.vars` | Local secret injection | Place at project root, gitignored. Wrangler reads it as environment variables during local dev. Never commit. |

## Installation

# Scaffold

# Tailwind CSS v4 (Astro 6 — use PostCSS to avoid rolldown incompatibility)

# Tailwind CSS v4 (Astro 5 — Vite plugin works fine)

# npm install tailwindcss @tailwindcss/vite

# Sparkline

# Dev tools

## Cloudflare Pages Function Proxy Pattern

## localStorage Persistence Pattern

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| Astro 6 (static) | Next.js / Nuxt | Never for this project — zero-cost constraint eliminates server-side rendering on paid tiers; Astro's static output is the right fit |
| Astro 6 (static) | Astro 5 (pinned) | Prefer Astro 5.17 if you want to avoid the Tailwind v4 + rolldown-vite issue until it's resolved upstream; migration to 6 is straightforward later |
| @tailwindcss/postcss | @tailwindcss/vite | Only if on Astro 5; do NOT use with Astro 6 (build fails due to rolldown incompatibility as of June 2026) |
| Yahoo Finance v8 via proxy | Tiingo via proxy | If Yahoo returns empty for any ticker; use as secondary source in the same Worker; costs an API key secret |
| Yahoo Finance v8 via proxy | Finnhub via proxy | If you want a legitimate free-tier account; 60 req/min is ample; but dividend history is weak (yield-metric only, no calendar) |
| @fnando/sparkline | Roll-your-own SVG | If you want zero extra dependencies; ~30 lines of SVG path math is sufficient for a simple line sparkline; reasonable given the tiny scope |
| @fnando/sparkline | Chart.js / D3 | Never — massively oversized for a 4-row sparkline; adds tens of KB |
| Cloudflare Pages Functions | Cloudflare Workers (standalone) | Same underlying runtime; Pages Functions are co-located with the static site in the same repo/deploy, which is simpler for this project. Standalone Workers are appropriate if the proxy grows complex or is shared across multiple apps. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| @astrojs/tailwind | Deprecated for Tailwind v4; only works with v3 | @tailwindcss/vite (Astro 5) or @tailwindcss/postcss (Astro 6) |
| @tailwindcss/vite with Astro 6 | Build fails: `Missing field 'tsconfigPaths'` in rolldown-vite as of June 2026 | @tailwindcss/postcss + postcss.config.mjs |
| Alpha Vantage free tier | 25 requests/day — exhausted by a single page load of 4 tickers with history | Yahoo Finance v8 or Tiingo |
| Polygon.io free tier | Dividend data documented as unreliable; 5 req/min is tight for 4 tickers | Yahoo Finance v8 or Tiingo |
| Twelve Data free tier | "Internal non-display usage" clause on free plan; publicly visible dashboard violates terms | Yahoo Finance v8 (no terms) or Tiingo |
| Chart.js / D3 / Recharts | Overkill for 4 inline sparklines; large bundle | @fnando/sparkline or raw SVG |
| React / Vue / Svelte | Framework overhead for a single-page static dashboard with minimal interactivity | Astro's island architecture + vanilla TS client scripts |
| Any database (SQLite, KV, D1) | Violates zero-cost and no-server constraints; localStorage is sufficient | localStorage via typed wrapper |
| API key in Wrangler [vars] | Plain-text in config file, committed to repo — exposes the secret | wrangler secret put → env.SECRET_NAME in Function code |
| yfinance Python library | Server-side only; not usable from a Worker | Direct fetch to Yahoo Finance v8 chart endpoint in Worker |

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| astro@6.4.x | node@22+ | Node 18/20 support dropped in Astro 6 |
| astro@5.17.x | node@18.17+ | Stable fallback if v6 causes friction |
| tailwindcss@4.3.x | @tailwindcss/vite (Astro 5) | Works via Vite plugin path |
| tailwindcss@4.3.x | @tailwindcss/postcss (Astro 6) | Required path; Vite plugin breaks with rolldown |
| @tailwindcss/vite@4.x | vite@6 / Astro 5 | DO NOT pair with Astro 6 (Vite 7 / rolldown) |
| wrangler@3.x | Astro 6 or 5 | Pages dev server independent of Astro version |
| @fontsource/* | Astro 5 | On Astro 6, prefer the built-in Fonts API instead |

## Data-Source Confidence Summary

| Source | Likely covers STRK/STRC/STRD/SATA | Dividend data | Confidence |
|--------|-----------------------------------|---------------|------------|
| Yahoo Finance v8 | HIGH — UI pages confirmed for all four | Via `events=div` in chart response | MEDIUM (inferred from UI, not API-tested) |
| Tiingo EOD | MEDIUM — broad US coverage; IPO-recency risk for 2025 tickers | Separate endpoint | MEDIUM |
| Finnhub | MEDIUM — NASDAQ coverage; preferred-share indexing unconfirmed | Weak (yield metric only) | LOW |
| Alpha Vantage | LOW — 25/day limit; preferred-share symbol quirks | Available but moot given limit | LOW |
| Polygon.io | MEDIUM — broad coverage | UNRELIABLE per reviews | LOW |
| Twelve Data | MEDIUM — broad coverage | Available | LOW (terms risk) |

## Sources

- Astro 6 release blog (astro.build/blog/astro-6/) — stable release date March 2026, Node 22 requirement, Vite 7
- GitHub issue withastro/astro#16542 — @tailwindcss/vite incompatibility with Astro 6 rolldown-vite; PostCSS workaround
- Tailwind CSS docs (tailwindcss.com/docs/installation/framework-guides/astro) — v4 setup steps verified
- Tailwind CSS @theme docs (tailwindcss.com/docs/theme) — custom color token syntax
- Cloudflare Pages Functions pricing (developers.cloudflare.com/pages/functions/pricing/) — 100k req/day shared limit
- Cloudflare Workers secrets docs — `wrangler secret put`, `.dev.vars`, `env.SECRET_NAME` access pattern
- Yahoo Finance v8 chart endpoint — multiple community sources confirm `query2.finance.yahoo.com/v8/finance/chart/{ticker}` works without auth key; CORS blocks browser-direct
- Yahoo Finance UI pages confirmed for STRK, STRC, STRD, SATA (search results)
- Tiingo pricing/docs (tiingo.com) — 1000 req/day, 50/hour, 500 symbols/month free; preferred share dash notation
- Tiingo coverage assessment (quantstart.com/articles/evaluating-data-coverage-with-tiingo/) — 82,468 global securities
- Finnhub pricing — 60 req/min free tier; CORS issue (GitHub finnhubio/Finnhub-API#286, open, unresolved)
- Alpha Vantage — 25 req/day free tier (alphavantage.co); preferred stock symbol quirks confirmed
- Polygon.io — 5 req/min; dividend data reliability issue per multiple reviews
- Twelve Data pricing (twelvedata.com/pricing) — "non-display usage" restriction on free tier
- @fnando/sparkline (github.com/fnando/sparkline) — zero-dependency SVG sparkline, onmousemove callback

<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->

## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->

## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->

## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->

## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:

- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->

## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
