# Requirements: YieldGlance

**Defined:** 2026-06-04
**Core Value:** One glance gives an accurate, current dividend yield per ticker — even when the data feed doesn't cover these niche tickers, because manual override always lets the number be correct.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Foundation

- [ ] **FND-01**: Project scaffolds as an Astro static site (`output: "static"`, no Cloudflare adapter) that builds and serves locally
- [ ] **FND-02**: Per-ticker config (symbol, dividend frequency, face annual rate) for STRK/STRC/STRD/SATA lives in a single source-of-truth module, with frequency sourced from prospectus (STRC monthly, STRK/STRD quarterly, SATA verified)
- [ ] **FND-03**: Yield computation is a pure, unit-tested function: `yield = annual dividend ÷ current price`, annualized by frequency, returning `null` (never NaN/Infinity) on missing/zero/non-finite price
- [ ] **FND-04**: Live data coverage for the four tickers is verified by a real API call (Yahoo v8, Tiingo fallback), and the confirmed verdict (which tickers have price / dividend / history) is recorded in the config
- [ ] **FND-05**: Secret hygiene is established from the first commit: `.dev.vars` and any key files are gitignored; no API key appears in the repo or built `dist/`

### Yield Table

- [ ] **TBL-01**: An above-the-fold table is the hero, showing one row per watchlist ticker (STRK, STRC, STRD, SATA)
- [ ] **TBL-02**: Each row shows ticker, as-of date/time, and current dividend yield as a percentage (2 decimal places)
- [ ] **TBL-03**: Yield is computed live in the browser on page load from the effective annual dividend and current price
- [ ] **TBL-04**: All four rows always render — a ticker with no data is never hidden
- [ ] **TBL-05**: A no-data row renders in a muted state with a clear prompt to enter values manually; yield appears once a value is supplied
- [ ] **TBL-06**: A per-row indicator distinguishes manual values from fetched values

### Manual Override

- [ ] **OVR-01**: User can enter/correct the annual dividend amount per ticker
- [ ] **OVR-02**: User can enter/correct the current price per ticker (so yield computes with zero feed coverage)
- [ ] **OVR-03**: Manual values persist in localStorage across reloads, under a versioned schema key
- [ ] **OVR-04**: A manual value always takes precedence over a fetched value (`stored ?? fetched`)
- [ ] **OVR-05**: Override inputs validate entries (reject zero, negative, and non-numeric) before saving

### Live Data

- [ ] **LIV-01**: A Cloudflare Pages Function proxy fetches ticker data, hiding the data-API key (key read from env, never in client JS) and resolving CORS
- [ ] **LIV-02**: The proxy normalizes upstream responses into one canonical schema and validates that a price is finite before returning it
- [ ] **LIV-03**: The proxy edge-caches responses with a split TTL (price short, dividend long) to stay within free upstream rate limits
- [ ] **LIV-04**: On page load the client fetches live data once (no auto-poll) and updates the table in place
- [ ] **LIV-05**: A fetch failure, timeout, or missing ticker never white-screens — the app falls back to last-known cached or manually-entered values
- [ ] **LIV-06**: The proxy's CORS allowlist is restricted to the production origin

### Liveness & Trust

- [ ] **LVN-01**: A visible "as-of" timestamp shows when the displayed data was current (in a clear, unambiguous timezone)
- [ ] **LVN-02**: A subtle liveness cue indicates fetch state (loading → loaded/failed) on load
- [ ] **LVN-03**: Stale cached data (older than its TTL) is visibly indicated as such

### Sparkline (progressive enhancement)

- [ ] **SPK-01**: Where price history exists, each row shows a sparkline of the trailing window (as far back as the source allows, capped at 12 months)
- [ ] **SPK-02**: Hovering the sparkline reveals point detail (date · price, and yield where derivable)
- [ ] **SPK-03**: The sparkline is suppressed entirely (no empty chart, no layout shift) when no history is available

### Design & Compliance

- [ ] **DSN-01**: The UI uses the dark trading-desk identity: slate-900 background, brand-gold #fbbf24 accent, Inter / Merriweather / mono type scale, text-[10px/11px] font-mono uppercase tracking-widest labels
- [ ] **DSN-02**: Figures are rendered as large tabular/monospace numerals; the layout is one screen, table-first
- [ ] **DSN-03**: A risk / informational-data disclaimer is visible without scrolling (not footer-only)
- [ ] **DSN-04**: No buy/sell/accumulate language, no ISIN/WKN, and no green/red "good/bad" coloring of yield anywhere in the UI
- [ ] **DSN-05**: The interface is in English

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Watchlist

- **WL-01**: User can add/remove tickers in the UI (persisted to localStorage)
- **WL-02**: Tickers beyond the fixed four are supported

### Refresh

- **RFS-01**: Manual "refresh" control to re-fetch without reloading
- **RFS-02**: Optional auto-poll on an interval

### Localization

- **LOC-01**: German localization

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| User-editable watchlist (v1) | Fixed four-ticker personal dashboard; expansion is a code change for v1 |
| Auto-poll / interval refresh | Fetch on page open only — protects free Worker + upstream quotas |
| Buy/sell or trade-action language | Securities compliance |
| ISIN / WKN identifiers | Securities compliance |
| Green/red yield coloring | Avoids implying good/bad investment judgement (compliance) |
| Database / server-side state | Zero-cost constraint; state is localStorage + client fetch only |
| Chart in pure manual mode | Hand-entering a year of daily prices is unreasonable; sparkline is progressive enhancement only |
| `@astrojs/cloudflare` adapter | Generates `_worker.js`, which disables the `functions/` proxy directory |
| Any paid service or hosting tier | Hard zero-cost constraint |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| FND-01 | Phase 1 | Pending |
| FND-02 | Phase 1 | Pending |
| FND-03 | Phase 1 | Pending |
| FND-04 | Phase 1 | Pending |
| FND-05 | Phase 1 | Pending |
| TBL-01 | Phase 2 | Pending |
| TBL-02 | Phase 2 | Pending |
| TBL-03 | Phase 2 | Pending |
| TBL-04 | Phase 2 | Pending |
| TBL-05 | Phase 2 | Pending |
| TBL-06 | Phase 2 | Pending |
| OVR-01 | Phase 2 | Pending |
| OVR-02 | Phase 2 | Pending |
| OVR-03 | Phase 2 | Pending |
| OVR-04 | Phase 2 | Pending |
| OVR-05 | Phase 2 | Pending |
| DSN-01 | Phase 2 | Pending |
| DSN-02 | Phase 2 | Pending |
| DSN-03 | Phase 2 | Pending |
| DSN-04 | Phase 2 | Pending |
| DSN-05 | Phase 2 | Pending |
| LIV-01 | Phase 3 | Pending |
| LIV-02 | Phase 3 | Pending |
| LIV-03 | Phase 3 | Pending |
| LIV-04 | Phase 3 | Pending |
| LIV-05 | Phase 3 | Pending |
| LIV-06 | Phase 3 | Pending |
| LVN-01 | Phase 3 | Pending |
| LVN-02 | Phase 3 | Pending |
| LVN-03 | Phase 3 | Pending |
| SPK-01 | Phase 4 | Pending |
| SPK-02 | Phase 4 | Pending |
| SPK-03 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 33 total (FND×5, TBL×6, OVR×5, LIV×6, LVN×3, SPK×3, DSN×5 — header previously stated 31, actual count is 33)
- Mapped to phases: 33
- Unmapped: 0

---
*Requirements defined: 2026-06-04*
*Last updated: 2026-06-05 after roadmap creation*
