# Architecture Research

**Domain:** Zero-cost static SPA + edge-proxy financial dashboard (Astro + Cloudflare Pages)
**Researched:** 2026-06-04
**Confidence:** HIGH — Routing decision based on official Cloudflare docs; data flow patterns based on verified Cloudflare Pages Functions behavior; Cache API constraints confirmed from official Workers docs.

---

## Routing Decision: Plain Pages Functions (No Astro Adapter)

**Decision: Use `output: "static"` in Astro with NO `@astrojs/cloudflare` adapter, and place the proxy in `/functions/api/quote.ts` (Pages Functions file-based routing).**

Rationale:

The Astro Cloudflare adapter in server or hybrid mode generates a `_worker.js` file at the build output root. Cloudflare Pages enforces a hard mutual exclusion: when `_worker.js` is present, the entire `/functions/` directory is **ignored** — its routes never run. You cannot use both simultaneously.

For this project:
- All four Astro pages are fully static (pre-rendered HTML — no per-request server logic needed in Astro itself)
- The only dynamic behavior is one proxy endpoint (`/api/quote`)
- Using the adapter adds `_worker.js` and kills Pages Functions routing with no benefit

The correct mode is:
1. `astro.config.mjs`: `output: "static"` — no adapter installed
2. Astro `npm run build` produces `./dist/` containing only static assets
3. `/functions/api/quote.ts` lives at the project root (NOT inside `./dist/`) — Cloudflare picks it up automatically during `wrangler pages deploy`
4. Cloudflare Pages serves `./dist/` as static assets and `/functions/api/*` as Functions routes side by side

This is confirmed by official Cloudflare documentation: "Create a `/functions` directory at the root of your Pages project (and not in the static root, such as `/dist`)." Pages Functions co-exists with static assets without any adapter involvement.

**Do NOT install `@astrojs/cloudflare`** — it generates `_worker.js` and suppresses Functions routing. The GitHub issue withastro/astro#15650 documents the exact breakage this causes with `output: "static"`.

---

## Standard Architecture

### System Overview

```
┌──────────────────────────────────────────────────────────────────┐
│  BROWSER                                                         │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  Astro static HTML (index.html — served from CDN)        │     │
│  │  ┌───────────────────────────────────────────────────┐   │     │
│  │  │  TickerTable (vanilla TS island, client-side only) │   │     │
│  │  │  - Reads localStorage on mount                     │   │     │
│  │  │  - Renders rows immediately (muted / override)     │   │     │
│  │  │  - Fires fetch to /api/quote?tickers=...           │   │     │
│  │  │  - Merges fetched data (override always wins)      │   │     │
│  │  │  - Recomputes yield; updates DOM in-place          │   │     │
│  │  │  - Progressive: inits @fnando/sparkline if history  │   │     │
│  │  └──────────────────────┬────────────────────────────┘   │     │
│  │                         │  fetch /api/quote?tickers=...  │     │
│  └─────────────────────────┼───────────────────────────────┘     │
└────────────────────────────┼─────────────────────────────────────┘
                             │  Same-origin request (no CORS issue)
┌────────────────────────────┼─────────────────────────────────────┐
│  CLOUDFLARE EDGE (Pages Functions runtime)                       │
│  ┌─────────────────────────▼────────────────────────────────┐    │
│  │  /functions/api/quote.ts  (onRequest handler)             │    │
│  │                                                           │    │
│  │  1. Validate Origin header → 403 if not allowlisted       │    │
│  │  2. Parse ?tickers= query param                           │    │
│  │  3. Check Cache API (cache.match) — price 15min TTL       │    │
│  │     dividend 24h TTL (separate cache keys)                │    │
│  │  4. On cache miss: fetch Yahoo Finance v8 chart endpoint  │    │
│  │     – If Yahoo returns no data: fetch Tiingo (fallback)   │    │
│  │     – API key read from context.env.TIINGO_API_KEY        │    │
│  │       (secret — never in source, never in client)         │    │
│  │  5. Normalize response into canonical shape               │    │
│  │  6. cache.put() with TTL headers                          │    │
│  │  7. Return JSON with CORS header for allowlisted origin   │    │
│  └──────────┬────────────────────────┬───────────────────────┘    │
└─────────────┼────────────────────────┼──────────────────────────┘
              │ outbound fetch          │ outbound fetch (fallback)
              ▼                         ▼
┌─────────────────────────┐   ┌─────────────────────────────────┐
│  Yahoo Finance v8 API   │   │  Tiingo EOD + Dividends API     │
│  (no key — unofficial)  │   │  (key in context.env — secret)  │
│  query2.finance.yahoo   │   │  api.tiingo.com                 │
│  .com/v8/finance/chart  │   │  Authorization: Token <secret>  │
└─────────────────────────┘   └─────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Implementation |
|-----------|----------------|----------------|
| `src/pages/index.astro` | Static shell: layout, HTML structure, disclaimer, `<script>` tag for the island | Astro page, `output: "static"`, no server logic |
| `src/scripts/ticker-island.ts` | All client-side interactivity: read localStorage, fetch, merge, compute yield, update DOM, init sparkline | Vanilla TS, loaded as `<script type="module">` |
| `src/lib/store.ts` | Typed localStorage wrapper: read/write overrides, read/write last-known cache, version guard | Vanilla TS module, pure functions |
| `src/lib/yield.ts` | Pure yield computation: annualize dividend by frequency, guard divide-by-zero, return null on missing inputs | Vanilla TS, unit-testable pure functions |
| `src/lib/ticker-config.ts` | Per-ticker constants: canonical symbol, dividend frequency (monthly/quarterly), face-value annual rate | Static TypeScript config object |
| `functions/api/quote.ts` | Pages Function proxy: Origin validation, cache lookup, upstream fetch (Yahoo primary / Tiingo fallback), response normalization, cache write | Cloudflare Pages Function — `onRequest(context)` |
| `src/styles/global.css` | Tailwind v4 import, `@theme` block with brand tokens (slate-900, brand-gold #fbbf24) | PostCSS via `@tailwindcss/postcss` |

---

## Recommended Project Structure

```
yield-glance/
├── functions/
│   └── api/
│       └── quote.ts          # Pages Function proxy — /api/quote route
├── src/
│   ├── pages/
│   │   └── index.astro       # Static shell page
│   ├── scripts/
│   │   └── ticker-island.ts  # Client-side island: fetch + merge + render
│   ├── lib/
│   │   ├── store.ts          # localStorage typed wrapper
│   │   ├── yield.ts          # Pure yield computation
│   │   └── ticker-config.ts  # Per-ticker constants (frequency, face rate)
│   ├── styles/
│   │   └── global.css        # Tailwind @import + @theme tokens
│   └── components/
│       └── TickerRow.astro   # Optional: row HTML template (static markup)
├── public/
│   └── favicon.svg
├── .dev.vars                 # GITIGNORED — local secret: TIINGO_API_KEY=...
├── .env.example              # Committed placeholder — TIINGO_API_KEY=your_key_here
├── astro.config.mjs          # output: "static", @tailwindcss/postcss integration
├── postcss.config.mjs        # { "@tailwindcss/postcss": {} }
├── tsconfig.json             # Astro default strict config
└── wrangler.toml             # Pages config — NO [vars], NO secrets here
```

### Structure Rationale

- **`functions/` at root (not in src/ or dist/):** Cloudflare Pages picks up this directory independently of the Astro build. The `dist/` output holds static assets; `functions/` holds dynamic routes. They are deployed together but built separately.
- **`src/lib/` for pure modules:** `store.ts` and `yield.ts` are free of DOM references and can be unit-tested without a browser.
- **`src/scripts/` for the island:** The island script touches the DOM and is loaded client-side. Separating it from `lib/` keeps pure functions testable.
- **`ticker-config.ts` as single source of truth:** Dividend frequency and face-value annual rate come from prospectus documents — they must be a static config, not inferred from API responses (see Pitfall 3 in PITFALLS.md).

---

## Architectural Patterns

### Pattern 1: Pages Functions Proxy — Key Secrecy + CORS Resolution

**What:** The browser calls `/api/quote?tickers=STRK,STRC,STRD,SATA` on the same origin (no CORS at all from the browser's perspective). The Pages Function runs at the edge, reads `context.env.TIINGO_API_KEY` (a Wrangler secret — encrypted, never in source), makes the outbound call to Yahoo Finance (no key) or Tiingo (key in Authorization header), and returns normalized JSON. The API key never appears in client JS, in `wrangler.toml`, or in the git history.

**When to use:** Any time a client-side SPA needs to call an API that either (a) blocks CORS from browsers, (b) requires a secret key, or (c) both. This is the correct pattern for all three conditions that apply here.

**Trade-offs:** Adds one network hop (browser → edge function → upstream). In practice this is negligible: the edge function runs co-located with the CDN serving the static assets, so the round-trip is ~10ms extra. The benefit — key secrecy and CORS resolution — is non-negotiable.

**Key implementation note:** Validate `Origin` in the Function handler before doing any work. Return `403` for any origin not in the allowlist (`https://yield-glance.pages.dev` and localhost in dev). This prevents third-party sites from using your proxy to exhaust your quota.

```typescript
// functions/api/quote.ts — core structure
const ALLOWED_ORIGINS = [
  "https://yield-glance.pages.dev",
  "http://localhost:4321",
];

export async function onRequest(context: EventContext<Env, string, unknown>) {
  const { request, env } = context;
  const origin = request.headers.get("Origin") ?? "";

  if (!ALLOWED_ORIGINS.includes(origin)) {
    return new Response(JSON.stringify({ error: "forbidden" }), { status: 403 });
  }

  const url = new URL(request.url);
  const tickers = (url.searchParams.get("tickers") ?? "").split(",").filter(Boolean);
  if (tickers.length === 0) {
    return new Response(JSON.stringify({ error: "tickers required" }), { status: 400 });
  }

  // env.TIINGO_API_KEY is the Wrangler secret — never logged, never returned to client
  const results = await fetchAllTickers(tickers, env.TIINGO_API_KEY);

  return new Response(JSON.stringify(results), {
    headers: {
      "Content-Type": "application/json",
      "Access-Control-Allow-Origin": origin,
    },
  });
}
```

### Pattern 2: Split-TTL Edge Caching (Price 15min / Dividend 24h)

**What:** The Cache API (`caches.default`) is used in the Pages Function to cache upstream responses. Price data is cached with a 15-minute TTL (`max-age=900`). Dividend data, which changes only on ex-dividend dates, is cached for 24h (`max-age=86400`). The cache key includes the ticker and data type. A custom `X-Data-Age` header is returned so the client can render an accurate "as-of" timestamp.

**When to use:** Any proxy that calls a rate-limited upstream API must cache at the edge. Without caching, 4 tickers × N page loads = N×4 upstream calls; with 15-minute caching, N page loads in a 15-minute window cost exactly 4 upstream calls total.

**Trade-offs:** The Cache API in Cloudflare Workers does NOT support `stale-while-revalidate` or `stale-if-error` directives (confirmed in official docs). Implement explicit fallback logic: on cache miss + upstream failure, return the last successful cached response (clone and re-put with the original TTL). `stale-while-revalidate` must be simulated with explicit code, not directives.

**Important constraint:** `cache.put()` only accepts `GET` requests as the cache key. The proxy endpoint is already a GET, so this is satisfied. Clone the upstream response before caching because Response bodies are single-use.

```typescript
// Separate cache keys for price vs dividend
const priceKey = new Request(`https://cache.internal/price/${ticker}`);
const dividendKey = new Request(`https://cache.internal/dividend/${ticker}`);

const cache = caches.default;

// Check price cache
const cachedPrice = await cache.match(priceKey);
if (cachedPrice) {
  // Cache hit — return immediately with age header
}

// Cache miss — fetch upstream
const upstream = await fetchYahoo(ticker);
const responseToCache = new Response(upstream.body, {
  headers: { "Cache-Control": "max-age=900" }, // 15 min for price
});
await cache.put(priceKey, responseToCache);
```

### Pattern 3: Progressive-Enhancement Render (localStorage-First)

**What:** The DOM is rendered immediately on page load using values from localStorage (manual overrides + last-known cached data). The async fetch to `/api/quote` runs in parallel; when it resolves, DOM cells are updated in-place. This means the user never sees a blank table — they always see their last-known or manually-entered values immediately, and live data replaces them when available.

**When to use:** Any dashboard where fetch latency or failure must not block meaningful display. This is the correct pattern for this project given the graceful-degradation requirement.

**Trade-offs:** The initial render may briefly show stale data before the live fetch resolves. Acceptable: this is a yield dashboard, not a live trading terminal. Mark stale values with a subtle indicator (e.g., "cached" badge) that disappears when live data arrives.

---

## Data Flow

### Complete Request Flow (Page Load)

```
1. Browser requests index.html
   ↓
   Cloudflare CDN → serves pre-built static HTML from dist/

2. HTML parsed; <script type="module" src="/scripts/ticker-island.js"> evaluated
   ↓
   ticker-island.ts runs:
   a. loadStore() — reads localStorage (sync, ~1ms)
   b. Renders all 4 rows immediately using:
      - storedOverride.annualDividend ?? null
      - storedOverride.currentPrice ?? null
      - lastKnown.price ?? null, lastKnown.fetchedAt ?? null
   c. Rows with no values render in muted "—" state

3. fetch("/api/quote?tickers=STRK,STRC,STRD,SATA") fires
   ↓
   Same-origin GET → no preflight CORS needed

4. Pages Function: /functions/api/quote.ts (onRequest)
   a. Validate Origin header → pass (same domain) or 403
   b. For each ticker:
      - cache.match(priceKey) → hit (return) or miss (continue)
      - cache.match(dividendKey) → hit (return) or miss (continue)
   c. On cache miss: fetch Yahoo Finance v8 chart API
      URL: https://query2.finance.yahoo.com/v8/finance/chart/{ticker}
           ?range=1y&interval=1d&events=div&corsDomain=finance.yahoo.com
      Headers: User-Agent: Mozilla/5.0, Accept: application/json
   d. If Yahoo response.ok === false OR result lacks price field:
      - Fallback to Tiingo EOD price endpoint + dividends endpoint
      - Authorization: Token {context.env.TIINGO_API_KEY}
   e. Normalize to canonical schema (see below)
   f. cache.put(priceKey, response with Cache-Control: max-age=900)
   g. cache.put(dividendKey, response with Cache-Control: max-age=86400)
   h. Return JSON with Access-Control-Allow-Origin: {origin}

5. Client receives proxy response (or error)
   ↓
   ticker-island.ts:
   a. On success: parse JSON into TickerData[]
   b. For each ticker, compute effective values:
      effectiveDividend = store[ticker].annualDividend ?? fetched.annualDividend
      effectivePrice    = store[ticker].currentPrice   ?? fetched.currentPrice
   c. computeYield(effectiveDividend, effectivePrice)
      - Guard: if (!effectivePrice || !isFinite(effectivePrice) || effectivePrice <= 0) return null
      - yield = effectiveDividend / effectivePrice
   d. Update DOM cells in-place; mark "live" indicator
   e. Save fetched values to localStorage lastKnown[ticker]:
      { price, fetchedAt: ISO timestamp, dividendFetched, dividendAsOf }
   f. If history array present in response: init @fnando/sparkline on the row

6. On fetch failure (network error, 500, timeout):
   ↓
   ticker-island.ts:
   a. Catch all errors — never throw
   b. Display "Using cached data" indicator
   c. Yield continues to compute from localStorage values (step 2 values remain)
   d. No rows are hidden or blanked out
```

### Override-Precedence Rule

This is the single most important merge rule. It must be applied consistently everywhere:

```typescript
// CORRECT — manual override always wins, even if fetched data exists
const effectiveDividend = store[ticker]?.annualDividend ?? fetched?.annualDividend ?? null;
const effectivePrice    = store[ticker]?.currentPrice   ?? fetched?.currentPrice   ?? null;

// WRONG — fetched data silently overwrites the override the user carefully entered
// const effectiveDividend = fetched?.annualDividend ?? store[ticker]?.annualDividend;
```

Rule: `storedOverride ?? fetchedValue ?? null`. The nullish-coalescing chain reads left to right — stored always wins over fetched. Only fall through to fetched if the stored value is explicitly `undefined` or `null` (i.e., the user has not set an override).

### Graceful Degradation Ladder

```
Tier 1 (best): Live fetched price + live fetched dividend
               → yield = fetched / fetched   [show "live" badge]

Tier 2:        Manual override price + live fetched dividend (or vice versa)
               → yield computed from mixed sources  [show "partial manual" badge]

Tier 3:        Manual override price + manual override dividend (no live data)
               → yield = override / override  [show "manual" badge]

Tier 4:        last-known fetched price (from localStorage) + any dividend source
               → yield computed from cached price  [show "cached · as-of {date}" badge]

Tier 5 (worst): One or both values missing
               → yield = null  [render "—" in yield cell; show "Enter value" prompt]
```

The table always renders. No tier causes a blank row. No tier throws. The muted prompt in Tier 5 is the designed UX, not an error state.

---

## localStorage Schema

```typescript
// Key: "yg:store:v1"  (versioned — change key name on any structural change)
interface YieldGlanceStore {
  _v: 1;                          // schema version — check before reading
  overrides: {
    [ticker: string]: {
      annualDividend?: number;    // user-entered annual dividend (NOT per-payment)
      currentPrice?: number;      // user-entered current price
      updatedAt?: string;         // ISO 8601 timestamp of last manual edit
    };
  };
  lastKnown: {
    [ticker: string]: {
      price?: number;             // last successfully fetched price
      annualDividend?: number;    // last successfully fetched annual dividend (annualized)
      dividendAsOf?: string;      // ISO date of the dividend record from API
      fetchedAt?: string;         // ISO 8601 timestamp of the fetch
    };
  };
}
```

**Schema rules:**
- The key `"yg:store:v1"` is versioned in the key name itself — never reuse it for a structurally different schema.
- On read, check `_v === 1`. If absent or mismatched, discard with a one-time notice and start fresh.
- `overrides` holds user-entered values only. Never write API data here.
- `lastKnown` holds the most recent successful API fetch. Written after every successful proxy response.
- Manual price input validates on entry: reject `0`, negative, non-numeric. Do not store invalid values.
- Wrap all `localStorage.getItem` / `JSON.parse` in try/catch — corrupted storage must not crash the app.

---

## Sparkline Data Path (Progressive Enhancement)

The Yahoo Finance v8 chart endpoint returns a `timestamp[]` and `close[]` array in the same response as the current quote. This is the sparkline data source — no separate request is needed.

```
Proxy normalizes: { ..., history: Array<{ ts: number, close: number }> | null }

Client, after live fetch resolves:
  if (ticker.history && ticker.history.length > 1) {
    const values = ticker.history.map(p => p.close);
    sparkline(svgElement, values, { onmousemove: showTooltip });
  }
  // if history is null (manual-only mode or API gap): sparkline SVG stays hidden
```

The sparkline SVG element is rendered in the HTML at full opacity but with no data points — it becomes visible only when `sparkline()` is called with data. This avoids layout shift: the column space is reserved in the table regardless of data availability.

The sparkline is capped at 90 days of daily data (90 data points) to keep the JSON response compact on mobile. The Workers Cache API caches this history for 15 minutes alongside the price.

---

## Proxy Response Normalization Schema

The proxy normalizes both Yahoo Finance and Tiingo responses into one canonical shape that the client consumes. The client never knows which upstream was used.

```typescript
// Returned by /api/quote — one entry per requested ticker
interface TickerResponse {
  ticker: string;           // canonical symbol (STRK / STRC / STRD / SATA)
  price: number | null;     // current price (null if unavailable)
  priceAsOf: string | null; // ISO 8601 timestamp (in UTC, full timestamp — never date-only)
  annualDividend: number | null; // annualized dividend (null if unavailable)
  dividendAsOf: string | null;   // ISO date of last dividend record
  history: Array<{ ts: number; close: number }> | null; // ≤90 daily closes
  source: "yahoo" | "tiingo" | "none"; // which upstream returned data
  cached: boolean;          // true if served from Worker Cache API
  dataAge: number | null;   // seconds since data was fetched (from Cache-Control)
}
```

The client uses `source: "none"` to know no live data is available for a ticker and to trigger the "Enter values manually" prompt.

---

## Yield Computation (Pure Function)

```typescript
// src/lib/yield.ts

// Per-ticker config — sourced from prospectus, NOT from API
export const TICKER_CONFIG: Record<string, { freq: 12 | 4; faceAnnualRate: number }> = {
  STRK: { freq: 4,  faceAnnualRate: 0.08  },  // 8.00% quarterly
  STRC: { freq: 12, faceAnnualRate: 0.115 },  // 11.50% monthly (variable rate — update if changed)
  STRD: { freq: 4,  faceAnnualRate: 0.10  },  // 10.00% quarterly
  SATA: { freq: 4,  faceAnnualRate: null  },  // check prospectus — placeholder
};

// annualize: multiply a single-payment amount by frequency
// Rejects frequency === 0 (special/non-recurring dividend — must not be annualized)
export function annualizeDividend(paymentAmount: number, freq: 4 | 12): number {
  return paymentAmount * freq;
}

// computeYield: returns null (not NaN/Infinity) on any missing or invalid input
export function computeYield(
  annualDividend: number | null | undefined,
  price: number | null | undefined
): number | null {
  if (annualDividend == null || price == null) return null;
  if (!isFinite(price) || price <= 0) return null;
  if (!isFinite(annualDividend) || annualDividend < 0) return null;
  return annualDividend / price;
}
```

---

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 1–5 users (current target) | No changes needed. 4 tickers × fetch-on-load-only × 15-min Worker cache = well within free tier limits (100k Functions req/day, Yahoo undocumented but generous, Tiingo 1000/day). |
| 50–500 users | Worker cache becomes critical — ensure it's implemented before any public sharing. With 15-min TTL, 100 users/15 min still costs only 4 upstream calls (1 cache miss) per ticker. Cloudflare free tier handles this comfortably. |
| 500+ users | Consider Cloudflare KV for persistent cross-datacenter caching (Cache API is per-datacenter). At this scale the Yahoo unofficial endpoint may throttle — evaluate Tiingo as primary. Still zero-infrastructure-cost on free tier up to ~100k Worker requests/day. |

---

## Anti-Patterns

### Anti-Pattern 1: Install `@astrojs/cloudflare` Adapter on a Static Site

**What people do:** Install the Cloudflare adapter because "it's the Cloudflare integration," then wonder why the `/functions/` directory routes return 404 in production.

**Why it's wrong:** The adapter generates `dist/_worker.js`. Cloudflare Pages sees `_worker.js` and ignores the entire `/functions/` directory. The proxy endpoint never runs. This is a hard platform constraint, not a bug.

**Do this instead:** No adapter. `output: "static"` only. Functions go in `/functions/` at the project root.

### Anti-Pattern 2: Fetching the Upstream API Directly from Client JS

**What people do:** Call `https://query2.finance.yahoo.com/v8/finance/chart/STRK` directly from `fetch()` in the browser to avoid dealing with the proxy.

**Why it's wrong:** (a) CORS blocks it — the browser will refuse the request. (b) Even if proxied by a browser extension, this pattern exposes the upstream URL and leaks any API key if Tiingo is later used directly. (c) It bypasses the edge cache, burning the upstream rate limit on every page load.

**Do this instead:** All upstream calls go through the Pages Function proxy exclusively. The client calls only `/api/quote` on its own origin.

### Anti-Pattern 3: `fetched ?? override` (Fetched Wins)

**What people do:** Write `const dividend = fetched?.annualDividend ?? storedOverride.annualDividend` because it "reads naturally" (try live data first).

**Why it's wrong:** If the API returns any value — even a wrong or stale one — the user's manual correction is silently discarded. The user enters a corrected dividend after a rate change; on the next reload, the old API value wins and the correction disappears. This is the exact bug documented in Pitfall 9.

**Do this instead:** `const dividend = storedOverride.annualDividend ?? fetched?.annualDividend`. Override always wins.

### Anti-Pattern 4: Render Only After Fetch Resolves

**What people do:** `fetch("/api/quote").then(data => renderTable(data))` — the table doesn't exist until the fetch completes.

**Why it's wrong:** During fetch (100–800ms depending on cache), the user sees a blank or spinner. If the fetch fails, the table never renders. Stored values are ignored entirely.

**Do this instead:** Render the table immediately from localStorage. Update cells in-place when the fetch resolves. The table always exists.

---

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Yahoo Finance v8 (unofficial) | Outbound `fetch()` in Pages Function, no key | Add `User-Agent` and `corsDomain` headers. Normalize response before returning. Treat any response missing a valid price as "no data." |
| Tiingo EOD + Dividends | Outbound `fetch()` in Pages Function with `Authorization: Token ${context.env.TIINGO_API_KEY}` | Separate price and dividend endpoints — 2 calls per ticker. Key read from Wrangler secret, never from source code. |
| Cloudflare Cache API (`caches.default`) | `cache.match()` / `cache.put()` inside Pages Function | Only functions in deployed Workers — does not work in `wrangler pages dev` preview. Use a dev-mode bypass: skip cache when a `?nocache=1` query param is present (dev only). |
| localStorage | Read/write from client-side island script only | Never accessed server-side. Typed wrapper with try/catch on all operations. Versioned schema key. |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| Astro page → client island | `<script type="module">` tag in `.astro` file; island self-initializes on DOMContentLoaded | No Astro-specific island framework needed (no React, no `client:load` directive) — vanilla TS script handles everything |
| Client island → proxy | `fetch("/api/quote?tickers=...")` — same-origin GET | No CORS headers needed on the request; proxy returns `Access-Control-Allow-Origin` for defense-in-depth |
| Proxy → localStorage | No direct connection — proxy never reads or writes localStorage | Client is the only actor that writes localStorage; proxy is stateless |
| `store.ts` → `yield.ts` | Direct function calls — both are pure TS modules | No events, no reactive store — YAGNI for a 4-ticker dashboard |
| `ticker-config.ts` → `yield.ts` | Import — config provides `freq` constant to annualization functions | Config is the single source of truth for frequency; API `frequency` field is a cross-check only |

---

## Suggested Build Order

Build in dependency order — each item can only start after its prerequisite is usable.

| Step | Component | Prerequisite | Reason |
|------|-----------|--------------|--------|
| 1 | `ticker-config.ts` — per-ticker frequency + face-rate constants | None | All other modules depend on it; verify prospectus data before any code |
| 2 | `yield.ts` — pure yield computation with NaN/Infinity guards | `ticker-config.ts` | Unit-testable in isolation; validates the core math before any UI |
| 3 | `store.ts` — versioned localStorage typed wrapper | None (parallel with step 2) | Required by the island; build and test its read/write/version-check in isolation |
| 4 | `functions/api/quote.ts` — Pages Function proxy | None (parallel with steps 2–3) | Can be developed and tested with `wrangler pages dev` independently; verify origin validation, cache TTL, and upstream normalization before connecting to client |
| 5 | Static HTML shell (`index.astro`) | Tailwind tokens, design tokens | Provides the table structure the island populates; no dynamic logic yet |
| 6 | `ticker-island.ts` — client integration | Steps 1–5 all done | Wires together `store.ts`, `yield.ts`, the proxy endpoint, and the DOM; last to build because it has the most dependencies |
| 7 | Sparkline integration | Step 6 working, step 4 returning history | Progressive enhancement — add only after core yield display is verified |
| 8 | Manual override UI (input fields + save handlers) | Step 6 working | Requires the island render loop to be correct before adding write paths |

---

## Sources

- Cloudflare Pages Functions — Advanced Mode (`_worker.js`): https://developers.cloudflare.com/pages/functions/advanced-mode/
- Cloudflare Pages Functions — File-based Routing: https://developers.cloudflare.com/pages/functions/routing/
- Cloudflare Pages Functions — Get Started: https://developers.cloudflare.com/pages/functions/get-started/
- Cloudflare Workers Cache API: https://developers.cloudflare.com/workers/runtime-apis/cache/
- Cloudflare Workers Cache — How It Works: https://developers.cloudflare.com/workers/reference/how-the-cache-works/
- GitHub withastro/astro#15650 — v6 + Cloudflare static output adapter conflict: https://github.com/withastro/astro/issues/15650
- GitHub withastro/astro#13582 — `_worker.js` directory uploaded as asset: https://github.com/withastro/astro/issues/13582
- Astro Deploy to Cloudflare — official guide: https://docs.astro.build/en/guides/deploy/cloudflare/
- FixDevs — `_worker.js` suppresses functions/ directory: https://fixdevs.com/blog/cloudflare-pages-not-working/
- Cloudflare — Build an API with Pages Functions (tutorial): https://developers.cloudflare.com/pages/tutorials/build-an-api-with-pages-functions/
- Cloudflare Workers Secrets: https://developers.cloudflare.com/workers/configuration/secrets/

---
*Architecture research for: YieldGlance — zero-cost static SPA + edge-proxy financial dashboard*
*Researched: 2026-06-04*
