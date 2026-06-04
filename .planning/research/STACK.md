# Stack Research

**Domain:** Client-side dividend-yield dashboard (static SPA, thin proxy, niche preferred-share tickers)
**Researched:** 2026-06-04
**Confidence:** MEDIUM — Stack choices HIGH; data-source claims MEDIUM (preferred-ticker coverage unverifiable without live API testing; flags noted below)

---

## Critical Question 1: Data Source for STRK / STRC / STRD / SATA

### Background on the Tickers

All four are recent NASDAQ-listed preferred shares:

| Ticker | Full Name | IPO |
|--------|-----------|-----|
| STRK | Strategy Inc 8% Series A Perpetual Strike Preferred Stock | Jan 2025 |
| STRC | Strategy Inc Variable Rate Series A Perpetual Stretch Preferred Stock | Jul 2025 |
| STRD | Strategy Inc 10% Series A Perpetual Stride Preferred Stock | Jun 2025 |
| SATA | Strive Inc preferred stock | Nov 2025 |

Yahoo Finance shows live quote pages for all four. They trade on NASDAQ Global Select Market under straightforward ticker symbols (no dash-notation needed). The critical unknown is which APIs index them and return dividend data alongside price history.

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

**Rank 1 — Yahoo Finance v8 (unofficial) via Cloudflare Worker proxy**

Rationale: All four tickers have confirmed Yahoo Finance quote pages, which means the v8 chart endpoint (the API that powers those pages) should return data for them. No API key required means no secret rotation risk and no rate-limit account. The proxy is already required for CORS, so using Yahoo costs nothing extra in infrastructure. The `?events=div` parameter returns dividend events in the same response as price history — one request covers both needs per ticker. Historical range covers these tickers back to their respective IPO dates.

Risk: Yahoo never documented this API and can change or block it without notice. The cookie/crumb auth flow used by yfinance is unreliable without a proper session; the v8 chart endpoint (not v7 quote) has historically required fewer auth headers. The Worker proxy can add required headers. Rate limits are IP-based and undocumented — a personal dashboard with fetch-on-load-only will consume 4 requests per page view, which is far below any known threshold.

Confidence: MEDIUM. Data coverage for these tickers is inferred from Yahoo Finance UI existing for them — not directly tested against the v8 API endpoint. The app must treat every fetch result as potentially empty and fall back to manual entry.

**Rank 2 — Tiingo EOD + dividends via Cloudflare Worker proxy**

Rationale: Legitimate free tier with well-defined limits (1000/day, 50/hour — more than enough for a personal dashboard). Has a dedicated dividend endpoint. Covers 65,000+ US instruments. STRK/STRC/STRD/SATA use plain NASDAQ symbols with no dash formatting needed. The IPO-recency risk is that Tiingo may not have indexed instruments less than ~6 months old.

Risk: Free tier requires registration; API key must be stored as a Worker secret. Dividend endpoint is separate from the price endpoint — 2 requests per ticker for full data = 8 requests per page load. Coverage of tickers less than ~12 months old (STRD, SATA IPO'd mid-late 2025) is uncertain.

Confidence: MEDIUM.

**Recommended implementation strategy:**

Use Yahoo Finance v8 as the primary source (no API key, confirmed ticker pages exist). Fall back to Tiingo if Yahoo returns no data for a ticker. Both are proxied through the same Cloudflare Worker. The Worker response normalizes both sources into one schema. If neither source returns data, the client falls back to manual entry — this is the designed behavior per PROJECT.md.

### Explicit Manual-Entry Fallback

This is not a fallback — it is a first-class mode. The app must:

1. Render all four ticker rows on every page load, regardless of fetch result
2. Rows with no fetched data display in muted state with a "Enter values manually" prompt
3. Manual overrides for both annual dividend and current price persist in localStorage
4. Yield = annual dividend ÷ current price is computed from whatever values are present (fetched or manual), never blocked on live data
5. The sparkline column is suppressed (empty) in full manual mode — do not prompt for daily price entry

---

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

---

## Installation

```bash
# Scaffold
npm create astro@latest yield-glance -- --template minimal --typescript strict --no-git

# Tailwind CSS v4 (Astro 6 — use PostCSS to avoid rolldown incompatibility)
npm install tailwindcss @tailwindcss/postcss

# Tailwind CSS v4 (Astro 5 — Vite plugin works fine)
# npm install tailwindcss @tailwindcss/vite

# Sparkline
npm install @fnando/sparkline

# Dev tools
npm install -D wrangler
```

**PostCSS config for Astro 6 + Tailwind v4:**
```js
// postcss.config.mjs
export default {
  plugins: {
    "@tailwindcss/postcss": {},
  },
};
```

**Global CSS:**
```css
/* src/styles/global.css */
@import "tailwindcss";

@theme {
  --color-brand-gold: #fbbf24;
  --color-slate-900: #0f172a;
  /* extend as needed */
}
```

---

## Cloudflare Pages Function Proxy Pattern

File: `functions/api/quote.js` (maps to `/api/quote`)

```js
export async function onRequest(context) {
  const { request, env } = context;
  const url = new URL(request.url);
  const ticker = url.searchParams.get("ticker");

  if (!ticker) {
    return new Response(JSON.stringify({ error: "ticker required" }), {
      status: 400,
      headers: { "Content-Type": "application/json" },
    });
  }

  // Yahoo Finance v8 chart — no API key needed
  const yahooUrl =
    `https://query2.finance.yahoo.com/v8/finance/chart/${encodeURIComponent(ticker)}` +
    `?range=1y&interval=1d&events=div&corsDomain=finance.yahoo.com`;

  try {
    const upstream = await fetch(yahooUrl, {
      headers: {
        "User-Agent": "Mozilla/5.0",
        "Accept": "application/json",
      },
    });

    if (!upstream.ok) throw new Error(`upstream ${upstream.status}`);

    const data = await upstream.json();
    return new Response(JSON.stringify(data), {
      headers: {
        "Content-Type": "application/json",
        "Access-Control-Allow-Origin": "*",
        "Cache-Control": "no-store",
      },
    });
  } catch (err) {
    return new Response(JSON.stringify({ error: err.message }), {
      status: 502,
      headers: { "Content-Type": "application/json" },
    });
  }
}
```

Secrets: `wrangler secret put TIINGO_API_KEY` (add Tiingo fallback in the same function using `env.TIINGO_API_KEY`). Never in source code or `wrangler.toml` `[vars]`.

Local dev: create `.dev.vars`:
```
TIINGO_API_KEY=your_key_here
```

---

## localStorage Persistence Pattern

No framework, no store library. Typed wrapper pattern:

```ts
// src/lib/store.ts
const STORE_KEY = "yg_overrides_v1";

export interface TickerOverride {
  annualDividend?: number;  // user-entered override
  currentPrice?: number;    // user-entered override
  updatedAt?: string;       // ISO timestamp
}

type Store = Record<string, TickerOverride>;

export function loadStore(): Store {
  try {
    return JSON.parse(localStorage.getItem(STORE_KEY) ?? "{}") as Store;
  } catch {
    return {};
  }
}

export function saveOverride(ticker: string, patch: Partial<TickerOverride>): void {
  const store = loadStore();
  store[ticker] = { ...store[ticker], ...patch, updatedAt: new Date().toISOString() };
  localStorage.setItem(STORE_KEY, JSON.stringify(store));
}
```

Override takes precedence: `finalDividend = override?.annualDividend ?? fetchedDividend`. Yield computes only when both values are present (either source).

---

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

---

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

---

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

---

## Data-Source Confidence Summary

| Source | Likely covers STRK/STRC/STRD/SATA | Dividend data | Confidence |
|--------|-----------------------------------|---------------|------------|
| Yahoo Finance v8 | HIGH — UI pages confirmed for all four | Via `events=div` in chart response | MEDIUM (inferred from UI, not API-tested) |
| Tiingo EOD | MEDIUM — broad US coverage; IPO-recency risk for 2025 tickers | Separate endpoint | MEDIUM |
| Finnhub | MEDIUM — NASDAQ coverage; preferred-share indexing unconfirmed | Weak (yield metric only) | LOW |
| Alpha Vantage | LOW — 25/day limit; preferred-share symbol quirks | Available but moot given limit | LOW |
| Polygon.io | MEDIUM — broad coverage | UNRELIABLE per reviews | LOW |
| Twelve Data | MEDIUM — broad coverage | Available | LOW (terms risk) |

**Bottom line on data confidence:** No source has been live-tested against these specific tickers' API endpoints. The app's manual-first design is the correct engineering response to this uncertainty. Treat every successful API response as a bonus, not a guarantee.

---

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

---
*Stack research for: YieldGlance — client-side dividend-yield dashboard*
*Researched: 2026-06-04*
