# Roadmap: YieldGlance

## Overview

YieldGlance ships as a manual-first dividend-yield dashboard for four niche preferred shares (STRK, STRC, STRD, SATA). Phase 1 establishes the scaffolding and resolves the single biggest unknown — whether any free API actually returns data for these tickers — before a line of UI code is written. Phase 2 delivers a fully usable, shippable product in pure manual-entry mode: all four rows render, yield computes, overrides persist, and compliance is satisfied without any live API dependency. Phase 3 layers in the Cloudflare proxy and live data integration, making the app production-grade with graceful degradation. Phase 4 adds sparklines as a progressive enhancement, conditional on the Phase 1 API coverage verdict — if no history data exists, this phase is skipped.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation + Verification** - Scaffold project, establish secret hygiene, verify API coverage for the four niche tickers
- [ ] **Phase 2: Manual-Only MVP** - Full yield table with manual override, localStorage persistence, design system, and compliance — shippable with zero API dependency
- [ ] **Phase 3: Live Data + Liveness** - Cloudflare Pages Function proxy, live fetch integration, as-of timestamps, and liveness cues
- [ ] **Phase 4: Sparkline** - Per-ticker sparkline mini-charts as progressive enhancement (conditional on Phase 1 coverage verdict)

## Phase Details

### Phase 1: Foundation + Verification
**Goal**: Project scaffolds correctly, secret hygiene is enforced from the first commit, dividend-frequency constants are sourced from prospectuses, yield computation is pure and tested, and live API coverage for STRK/STRC/STRD/SATA is verified with a real call (recording which tickers have price / dividend / history).
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: FND-01, FND-02, FND-03, FND-04, FND-05
**Success Criteria** (what must be TRUE):
  1. `npm run build` produces a valid static `dist/` with no API key in any output file (verified by grep)
  2. `computeYield()` unit tests pass, covering null inputs, zero price, negative price, and correct frequency annualization for both monthly (STRC) and quarterly (STRK/STRD) frequencies
  3. `.dev.vars` and any key files are absent from git history and the built output; `.env.example` is committed as a placeholder
  4. A recorded verdict exists (in `ticker-config.ts` comments or a spike notes file) stating which of the four tickers return a valid price, dividend amount, and price history from Yahoo Finance v8 (and Tiingo fallback if Yahoo fails)
  5. `ticker-config.ts` contains prospectus-sourced frequency and face-rate constants for all four tickers (SATA frequency confirmed, STRC variable rate noted)
**Plans**: TBD

### Phase 2: Manual-Only MVP
**Goal**: Users can open the app, see all four ticker rows, enter annual dividend and price per ticker, have yield computed and displayed as a percentage, have their values persist across reloads, and read a visible disclaimer — with zero dependency on any live API.
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: TBL-01, TBL-02, TBL-03, TBL-04, TBL-05, TBL-06, OVR-01, OVR-02, OVR-03, OVR-04, OVR-05, DSN-01, DSN-02, DSN-03, DSN-04, DSN-05
**Success Criteria** (what must be TRUE):
  1. All four ticker rows (STRK, STRC, STRD, SATA) are visible above the fold; a row with no data renders in a muted state with a clear "enter value" prompt rather than disappearing
  2. User can type an annual dividend and a current price into a row's input fields; yield appears as a percentage (2 decimal places) immediately upon entry, computed live in the browser
  3. After a page reload, previously entered values are still present and yield still displays — values persisted in localStorage under the versioned schema key
  4. A manual value entered by the user always wins over any placeholder or default; the row shows a visual indicator distinguishing manually-entered values from fetched/empty ones
  5. Override inputs reject zero, negative, and non-numeric entries without saving; a risk/informational disclaimer is visible without scrolling, and no buy/sell language or ISIN/WKN appears anywhere in the UI
**Plans**: TBD
**UI hint**: yes

### Phase 3: Live Data + Liveness
**Goal**: The app fetches live price and dividend data through a Cloudflare Pages Function proxy that hides the API key and resolves CORS, updates the yield table in place on page load, shows accurate as-of timestamps, and degrades gracefully to cached or manually-entered values on any fetch failure — never white-screening.
**Mode:** mvp
**Depends on**: Phase 2 (UI render loop), Phase 1 (confirmed endpoint shapes and ticker coverage)
**Requirements**: LIV-01, LIV-02, LIV-03, LIV-04, LIV-05, LIV-06, LVN-01, LVN-02, LVN-03
**Success Criteria** (what must be TRUE):
  1. The Tiingo API key is never present in client JS, the built `dist/`, or the git repo — it lives only in the Wrangler secret environment; CORS is restricted to the production origin allowlist
  2. On page load the table renders immediately from localStorage, then live data updates cells in-place (no full re-render, no blank intermediate state); fetching and loaded/failed states are visibly indicated
  3. Each row displays an "as-of" timestamp in an unambiguous timezone (ET) showing when the displayed value was current; stale cached data (older than TTL) is visibly indicated as such
  4. A fetch failure, timeout, or missing ticker for any or all rows never produces a white screen or blank row — the app falls back to last-known cached or manually-entered values and shows a per-row or global degraded-state indicator
  5. The proxy edge-caches responses (price 15-min TTL, dividend 24h TTL) so repeated page loads within the TTL window do not consume upstream API quota
**Plans**: TBD

### Phase 4: Sparkline
**Goal**: Where the Phase 1 API coverage verdict confirms price history is available, each ticker row shows an inline sparkline of trailing daily closes (capped at 12 months), with hover detail revealing date, price, and yield at that point — suppressed entirely when no history exists.
**Mode:** mvp
**Depends on**: Phase 3 (working render loop and proxy), Phase 1 (coverage verdict confirming history data)
**Requirements**: SPK-01, SPK-02, SPK-03
**Success Criteria** (what must be TRUE):
  1. For tickers with confirmed price history, a sparkline mini-chart appears in the row with no layout shift (space is reserved even when suppressed)
  2. Hovering the sparkline reveals a tooltip showing at minimum the date and price for that point; when annual dividend is available, yield at that point is also shown
  3. When no price history is available for a ticker (coverage verdict: none, or API returns empty), no sparkline element or empty chart placeholder is rendered for that ticker — the row looks identical to the no-sparkline baseline
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation + Verification | 0/TBD | Not started | - |
| 2. Manual-Only MVP | 0/TBD | Not started | - |
| 3. Live Data + Liveness | 0/TBD | Not started | - |
| 4. Sparkline | 0/TBD | Not started | - |
