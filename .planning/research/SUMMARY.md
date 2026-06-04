# Project Research Summary

**Project:** YieldGlance
**Domain:** Niche preferred-stock dividend-yield dashboard — static SPA + edge proxy
**Researched:** 2026-06-04
**Confidence:** MEDIUM (stack HIGH; data-source coverage for target tickers MEDIUM — unverified live)

## Executive Summary

YieldGlance is a personal, trading-desk-style single-page dashboard displaying dividend yield (annual dividend ÷ current price) for a fixed four-ticker watchlist of niche preferred shares: STRK, STRC, STRD, and SATA. The central engineering challenge is that these tickers are recent NASDAQ-listed preferred shares (IPO'd 2025) with **unverified free-API coverage**. No free data source has been confirmed to return price, dividend, and price-history data for all four via its API endpoint — Yahoo Finance UI pages exist for all four, which makes the v8 chart endpoint the highest-confidence path, but this has not been live-tested. The correct engineering response is a **manual-first design**: the app must be fully functional with hand-entered annual dividend and price, with live data fetch as progressive enhancement. Every phase must treat "no API data" as the default state, not an error.

The recommended architecture is a fully static Astro site (`output: "static"`) deployed to Cloudflare Pages, with a single Cloudflare Pages Function at `functions/api/quote.ts` acting as a CORS-resolving, key-hiding proxy. The proxy calls Yahoo Finance v8 (no key required) as primary and Tiingo (free tier, API key as Wrangler secret) as fallback, normalizes both into one canonical response schema, and caches at the edge (price: 15-min TTL; dividend: 24h TTL). Client-side TypeScript renders the table immediately from localStorage, then updates cells in-place when the proxy responds. Manual overrides stored in localStorage always win over fetched values — the override-wins precedence rule is the single most important merge invariant in the codebase.

Key risks: (1) preferred ticker API coverage may be zero for some or all tickers — early in Phase 1, make a live API call to confirm before writing any rendering logic; (2) installing the `@astrojs/cloudflare` adapter on a static site kills Pages Functions routing — the no-adapter decision is non-negotiable; (3) API keys committed to git or bundled into client JS are the most likely security failure mode; (4) wrong dividend-frequency multiplier (monthly vs. quarterly) produces a yield that is off by 3x — frequency constants must come from prospectus documents, not API inference.

## Key Findings

### Recommended Stack

The stack is well-established for zero-cost static dashboards with a thin proxy. Astro 6 (static mode) is the right generator — it ships zero JS by default, is Cloudflare-backed, and pairs naturally with Pages Functions. The one friction point is a Tailwind v4 + rolldown-vite incompatibility in Astro 6: use `@tailwindcss/postcss` via `postcss.config.mjs` instead of the Vite plugin. If this is a concern, pin to Astro 5.17.x where `@tailwindcss/vite` works correctly — migration to v6 is trivial later. Cloudflare Pages free tier is more than sufficient (100k Function requests/day, unlimited static assets). No database, no framework, no heavy chart library — all deliberate constraints.

**Core technologies:**
- **Astro 6 (static):** Static site generator — zero JS shipped by default; `output: "static"` is the only correct mode for this project; Node 22 required
- **Tailwind CSS 4.3 + @tailwindcss/postcss:** Utility CSS — PostCSS path avoids rolldown-vite incompatibility in Astro 6; `@theme` block replaces config file for brand tokens
- **Cloudflare Pages + Pages Functions:** Hosting + edge proxy — free tier sufficient; Pages Functions file-based routing at `/functions/api/quote.ts`; no adapter installed
- **Vanilla TypeScript (client island):** Interactivity — no React/Vue/Svelte overhead for a 4-row table; island self-initializes via `<script type="module">`
- **Yahoo Finance v8 (unofficial) via proxy:** Primary data source — no API key; confirmed UI pages for all four tickers; `?events=div` returns price + dividend in one request
- **Tiingo EOD + dividends via proxy:** Fallback data source — free tier (1000 req/day), legitimate ToS, API key stored as Wrangler secret; separate price and dividend endpoints (2 calls/ticker)
- **@fnando/sparkline:** Inline SVG sparklines — zero dependencies; progressive enhancement only; suppress entirely in manual-only mode
- **localStorage (typed wrapper):** Persistence — versioned schema key (`yg:store:v1`); no database; typed wrapper with try/catch on all operations

**Critical version constraint:** `@tailwindcss/vite` BREAKS with Astro 6 (Vite 7 / rolldown). Use `@tailwindcss/postcss` instead. Do not install `@astrojs/cloudflare` adapter — it generates `_worker.js` which causes Cloudflare to ignore the entire `/functions/` directory.

### Expected Features

The feature set is intentionally narrow. The entire value proposition is: one glance, one accurate yield per ticker, even when the data feed is absent.

**Must have (table stakes — v1):**
- Hero yield table above the fold, all 4 rows always visible (even with no data)
- Yield = annual dividend ÷ current price, computed in browser, displayed as percentage (2 decimal places)
- As-of timestamp per ticker (fetched time or "Manual") — financial data without a time reference is dangerous
- Manual override of annual dividend, persisted in localStorage, always takes precedence over fetched value
- Manual override of current price, same precedence rule
- No-data rows render in muted state with "Enter value" prompt — row never disappears
- Graceful degradation: fetch failure falls back to last-known localStorage values; table never white-screens
- Manual vs. fetched source indicator per row (pencil icon or "manual" badge)
- Visible risk / informational disclaimer — above the fold or adjacent to the table, not footer-only
- No buy/sell/accumulate language anywhere in the UI (compliance non-negotiable)
- Dark trading-desk visual identity (slate-900, brand-gold #fbbf24, monospace numerics, uppercase tracking-widest labels)

**Should have (v1.x — add after confirming API coverage):**
- Per-ticker sparkline: trailing ≤90-day price history mini-chart (progressive enhancement — only if API returns history)
- Hover tooltip on sparkline: date · price · yield at that point (requires annual dividend to compute historical yield)
- Per-ticker inline fetch-error state (vs. global error) — show per-row "Fetch failed — using cached"
- localStorage TTL awareness — age indicator when cached values are older than 24h

**Defer (v2+):**
- User-editable watchlist (add/remove tickers)
- Auto-refresh / polling interval
- Dark/light theme toggle
- Export / sharing
- Additional tickers beyond the fixed four

### Architecture Approach

The architecture is a clean three-layer system: static HTML shell (Astro) → client-side vanilla TS island (fetch + merge + render) → Cloudflare Pages Function proxy (CORS resolution + key secrecy + edge caching → Yahoo Finance / Tiingo). The critical routing decision is **no `@astrojs/cloudflare` adapter**: the adapter generates `_worker.js` which causes Cloudflare Pages to ignore the `/functions/` directory entirely. With `output: "static"` and no adapter, the static `dist/` and the `functions/` directory coexist correctly.

**Major components:**
1. `src/pages/index.astro` — static HTML shell; layout, disclaimer, `<script type="module">` tag; no server logic
2. `src/scripts/ticker-island.ts` — all client interactivity: read localStorage, fetch proxy, merge override-wins, compute yield, update DOM, init sparkline
3. `src/lib/store.ts` — typed localStorage wrapper: read/write overrides, read/write lastKnown cache, version guard (`_v: 1`)
4. `src/lib/yield.ts` — pure yield computation: annualize by frequency, guard divide-by-zero, return `null` on missing inputs (never `NaN`)
5. `src/lib/ticker-config.ts` — per-ticker constants from prospectus: symbol, dividend frequency (monthly/quarterly), face-value annual rate — single source of truth
6. `functions/api/quote.ts` — Pages Function proxy: Origin validation → Cache API lookup → Yahoo v8 fetch → Tiingo fallback → normalize → cache.put → return JSON

**Key patterns:**
- Override-wins precedence: `storedOverride ?? fetched ?? null` — never reversed
- Split-TTL edge caching: price 15-min TTL, dividend 24h TTL (separate cache keys)
- Progressive-enhancement render: table renders immediately from localStorage; live data updates cells in-place
- Graceful degradation ladder: Live → Mixed → Manual → Cached → No-data (table always renders at every tier)

### Critical Pitfalls

1. **Free API silently returns no data for preferred tickers (not a 4xx — a blank 200)** — Validate every proxy response for `typeof price === 'number' && isFinite(price)` before passing to yield computation; make a live API call against STRK/STRC/STRD/SATA before writing any rendering logic

2. **`@astrojs/cloudflare` adapter installed on a static site kills Pages Functions routing** — Never install the adapter; `output: "static"` only; test with `wrangler pages dev ./dist`, not `astro dev`

3. **Dividend frequency misidentification → yield off by 3x** — STRC pays monthly (×12), STRK/STRD pay quarterly (×4); store frequency constants in `ticker-config.ts` from prospectus documents; never infer from API `frequency` field; exclude `frequency === 0` (special dividends) from annualization

4. **Override-precedence bug: fetched value silently wins over manual correction** — Always write `storedOverride.annualDividend ?? fetched?.annualDividend`, never the reverse; unit-test this invariant explicitly

5. **API key in `wrangler.toml` [vars] or committed to git** — Use `wrangler secret put TIINGO_API_KEY` for production; `.dev.vars` (gitignored) for local dev; verify `grep -r "API_KEY" dist/` returns nothing after every build

6. **`stale-while-revalidate` directive not supported in Cloudflare Workers Cache API** — Implement explicit TTL + fallback logic; cannot use `stale-while-revalidate` or `stale-if-error` in `cache.put()`

7. **Division by zero / NaN propagation in yield column** — `computeYield()` must return `null` (not `NaN`/`Infinity`) when price is 0, null, or non-finite; render null as `—` in the table

8. **Compliance: missing disclaimer or implicit buy/sell framing** — Disclaimer must be visible without scrolling (not footer-only); no green/red coloring on yield; no comparative language; no ISIN/WKN anywhere

## Implications for Roadmap

### Phase 1: Foundation and Data-Source Verification

**Rationale:** The most critical unknown — whether any free API actually returns data for STRK/STRC/STRD/SATA — must be resolved before building rendering logic. A live API call to the Yahoo Finance v8 endpoint (and Tiingo if Yahoo fails) with the real ticker symbols is the single highest-value action in the entire project. If coverage is zero, the app ships as manual-only with no proxy at all; if partial, the proxy is built around confirmed tickers only. This phase also establishes the scaffolding and secret hygiene that is impossible to retrofit safely.

**Delivers:** Confirmed data-source path (or manual-only verdict); project scaffolding with `.gitignore`, `.dev.vars`, `.env.example`; `ticker-config.ts` with prospectus-sourced frequency constants; `yield.ts` pure functions with unit tests

**Addresses:** Yield formula correctness, dividend-frequency annualization correctness, secret hygiene from day zero
**Avoids:** Pitfalls 1 (silent blank API), 2 (ticker symbol format mismatch), 3 (frequency misidentification), 4 (API key committed to git), 12 (ToS violation)

**Research flag:** Needs live API verification — run a `curl` or Wrangler dev script against Yahoo Finance v8 and Tiingo with STRK/STRC/STRD/SATA before writing any proxy code. This is a 15-minute spike.

### Phase 2: Core UI Shell and localStorage Layer

**Rationale:** With frequency constants and yield computation validated in Phase 1, the static HTML shell and localStorage layer can be built. This phase makes the app usable in full manual mode — the minimum viable product is a table that computes yield from hand-entered values with no API dependency whatsoever.

**Delivers:** Static `index.astro` shell with full dark design system (slate-900, brand-gold, mono labels); `store.ts` typed localStorage wrapper with versioned schema (`_v: 1`); manual override input fields with validation (reject 0, negative, non-numeric); override-wins merge logic; graceful degradation to muted "no data" rows; compliance disclaimer visible adjacent to table

**Addresses:** All P1 table-stakes features; manual override persistence; no-data rows; compliance non-negotiables
**Avoids:** Pitfalls 8 (localStorage schema breakage), 9 (override-precedence bug), 10 (white-screen on failure), 15 (compliance), 16 (NaN/Infinity in UI)

**Research flag:** Standard patterns — no additional research needed

### Phase 3: Proxy and Edge Caching

**Rationale:** With the data source confirmed in Phase 1 and the UI shell operational in Phase 2, the proxy can be built and wired in. This phase handles key secrecy, CORS, and split-TTL caching. Cannot start until Phase 1 confirms the upstream API endpoints and response shapes.

**Delivers:** `functions/api/quote.ts` with Origin validation (allowlist), Yahoo v8 primary fetch, Tiingo fallback, response normalization to canonical `TickerResponse` schema, split-TTL Cache API (price 15min / dividend 24h), `X-Data-Age` header; `wrangler secret put` for Tiingo key; CORS locked to production domain

**Addresses:** Live data fetch; key secrecy; CORS resolution; rate-limit protection via edge cache
**Avoids:** Pitfalls 4 (key in git), 5 (key in client bundle), 6 (permissive CORS), 11 (rate-limit exhaustion), 13 (routing misconfiguration), 14 (over-caching stale data)

**Research flag:** Standard Cloudflare Pages Functions patterns — fully documented in ARCHITECTURE.md; no additional research needed

### Phase 4: Live Data Integration and Display Polish

**Rationale:** With proxy operational, wire up the client island to fetch from `/api/quote`, merge fetched data with the override-wins rule, display as-of timestamps in ET timezone, show liveness cues, and add per-ticker source indicator.

**Delivers:** `ticker-island.ts` full integration (localStorage read → immediate render → fetch → merge → in-place DOM update → cache lastKnown); as-of timestamp in ET (`Intl.DateTimeFormat` with `timeZone: 'America/New_York'`); liveness cue (loading → loaded/failed); manual vs. fetched source indicator; fetch timeout via `AbortController` (5s); per-ticker error state

**Addresses:** Liveness cue; as-of timestamp; source indicator; graceful degradation on fetch failure
**Avoids:** Pitfalls 7 (stale price/dividend mix), 10 (white-screen), 17 (timezone ambiguity)

**Research flag:** Standard patterns — no additional research needed

### Phase 5: Sparkline (Progressive Enhancement)

**Rationale:** Sparkline is deliberately last — it requires confirmed price-history data from the API (Phase 1 verdict), a working proxy (Phase 3), and a working render loop (Phase 4). If Phase 1 shows no history data available, this phase is skipped without affecting the core product.

**Delivers:** `@fnando/sparkline` integration rendering trailing ≤90-day closes per ticker (where history exists); sparkline hidden when no history; hover tooltip showing date · price · historical yield; SVG space reserved in table layout to prevent layout shift

**Addresses:** Per-ticker sparkline; sparkline hover detail
**Avoids:** Pitfall 1 variant — suppress sparkline entirely when no data; never render an empty sparkline

**Research flag:** Conditional on Phase 1 API coverage verdict for price-history fields in the Yahoo v8 response.

### Phase Ordering Rationale

- Phase 1 must be first: data-source coverage is the central unknown; all subsequent phases build on its verdict
- Phase 2 is intentionally independent of live API coverage: manual-only mode is the designed fallback and a shippable MVP on its own
- Phase 3 depends on Phase 1 (confirmed endpoint shapes) but not Phase 2
- Phase 4 depends on both Phase 2 (UI render loop) and Phase 3 (proxy operational)
- Phase 5 depends on Phase 4 and on Phase 1 confirming price history is available
- The app is shippable after Phase 2 (manual-only) and again after Phase 4 (live data + manual fallback)

### Research Flags

Phases needing live verification during planning:
- **Phase 1:** Live API call against Yahoo Finance v8 with each of STRK, STRC, STRD, SATA is mandatory before writing any proxy or rendering code. Check for: `meta.regularMarketPrice`, `events.dividends`, `timestamp[]`, `indicators.quote[0].close[]`. If Yahoo fails for any ticker, repeat with Tiingo. Document which tickers have confirmed coverage.
- **Phase 5:** Conditional — only execute if Phase 1 confirms price-history arrays are present.

Phases with standard patterns (no research phase needed):
- **Phase 2:** localStorage, Astro static, Tailwind v4 — all well-documented
- **Phase 3:** Cloudflare Pages Functions proxy — fully documented in ARCHITECTURE.md with working code
- **Phase 4:** Client island fetch + DOM update + `Intl.DateTimeFormat` — standard patterns

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All technology choices verified against official docs; Tailwind/rolldown issue documented with working workaround |
| Features | HIGH | Core table-stakes features clear; dependency graph fully mapped; sparkline explicitly conditional on API coverage |
| Architecture | HIGH | No-adapter routing decision verified against official Cloudflare Pages docs and GitHub issues; all patterns confirmed |
| Pitfalls | HIGH | Drawn from official docs, verified API behavior, and multiple community sources; 17 pitfalls with concrete prevention |
| Data-source coverage | MEDIUM | Yahoo Finance v8 coverage inferred from UI page existence — not live-API-tested; this is the one genuine unknown |

**Overall confidence:** MEDIUM — engineering approach is HIGH confidence; data-source coverage for the specific niche tickers is MEDIUM until a live API call confirms it.

### Gaps to Address

- **CRITICAL — Verify API coverage with a live call in Phase 1:** Before writing any proxy or rendering logic, test `https://query2.finance.yahoo.com/v8/finance/chart/STRK?range=1y&interval=1d&events=div` and each of the other three tickers. Inspect for `meta.regularMarketPrice`, `events.dividends`, `timestamp[]`, `indicators.quote[0].close[]`. If Yahoo returns no data for any ticker, test Tiingo EOD and dividends endpoints. Log the verdict in `ticker-config.ts` comments before proceeding.

- **SATA dividend frequency and face rate:** Listed as unknown in `ticker-config.ts` (placeholder). Check the SATA prospectus and confirm payment frequency and declared annual rate before Phase 1 is complete.

- **STRC variable rate:** STRC is a variable-rate preferred; the 11.50% nominal rate must be verified against the current declared rate at build time. Manual override exists precisely for this — document the update procedure in the project.

- **Wrangler compatibility:** Confirm `wrangler@3.x` `wrangler pages dev ./dist` works correctly with the chosen Astro version at scaffolding time.

## Sources

### Primary (HIGH confidence)
- Cloudflare Pages Functions docs — routing, file-based setup, Advanced Mode (`_worker.js` conflict), Cache API constraints, secrets management
- Astro 6 release blog + GitHub issue #16542 — Tailwind v4 + rolldown-vite incompatibility and PostCSS workaround
- GitHub withastro/astro#15650 — `@astrojs/cloudflare` adapter + `output: "static"` conflict with Pages Functions
- Tailwind CSS v4 official docs — `@theme` block, `@import "tailwindcss"`, PostCSS installation path

### Secondary (MEDIUM confidence)
- Yahoo Finance UI pages for STRK, STRC, STRD, SATA — confirms tickers exist in Yahoo's system; v8 API coverage inferred (not directly tested)
- Tiingo pricing/docs — 1000 req/day, 65,000+ US instruments, free tier confirmed; IPO-recency risk for 2025 tickers noted
- @fnando/sparkline GitHub — zero-dependency SVG sparkline, `onmousemove` callback confirmed
- Multiple community sources for Yahoo Finance v8 chart endpoint behavior (no auth key required, CORS blocked from browser)

### Tertiary (LOW confidence / needs live validation)
- Yahoo Finance v8 API coverage for STRK/STRC/STRD/SATA specifically — inferred from UI existence only; must be validated with a live call before Phase 3 work begins
- Tiingo coverage for 2025-IPO preferred tickers — broad US coverage claimed but recency risk for tickers less than 12 months old

---
*Research completed: 2026-06-04*
*Ready for roadmap: yes — pending live API coverage verification spike in Phase 1*
