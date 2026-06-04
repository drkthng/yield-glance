# YieldGlance

## What This Is

YieldGlance is a standalone single-page web app that shows **live dividend yield instead of price** for a small, fixed watchlist of securities — starting with the MicroStrategy/Strategy preferred shares STRK, STRC, STRD, and SATA. The hero is an above-the-fold table (ticker · as-of date · current dividend yield), where yield = annual dividend ÷ current price, computed live in the visitor's own browser on page load. It is a personal, trading-desk-style dashboard — informational only, not investment advice.

## Core Value

**One glance gives an accurate, current dividend yield per ticker** — even when the data feed doesn't cover these niche tickers, because manual override always lets the number be correct.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

(None yet — ship to validate)

### Active

<!-- Current scope. Building toward these. -->

- [ ] Hero table above the fold: ticker · as-of date · current dividend yield, for the fixed watchlist (STRK, STRC, STRD, SATA)
- [ ] Yield computed live in the browser on page load: annual dividend ÷ current price
- [ ] Live data fetched via a Cloudflare Pages Function/Worker proxy that hides the data-API key and resolves CORS — no key in client JS
- [ ] Per-ticker sparkline mini-chart of the trailing window (as far back as the source allows, capped at 12 months), with detail on hover — progressive enhancement where price history exists
- [ ] Per-ticker manual override of the annual dividend amount, persisted in localStorage, taking precedence over fetched values
- [ ] Manual override of current price (used when no feed covers a ticker), so yield always computes in full manual mode
- [ ] No-data rows stay visible in a muted state and prompt manual entry; yield appears once a value is supplied
- [ ] Visible "as-of" timestamp and a subtle liveness cue on load
- [ ] Graceful degradation: a failed/missing/rate-limited fetch never white-screens — fall back to last-known or manually-entered values
- [ ] Dark "trading-desk" visual identity matching the existing blog (slate-900 bg, brand-gold #fbbf24 accent, Inter / Merriweather / mono scale, text-[10px/11px] font-mono uppercase tracking-widest labels)
- [ ] Risk / informational-data disclaimer visible in the UI
- [ ] English (single language)

### Out of Scope

<!-- Explicit boundaries. Includes reasoning to prevent re-adding. -->

- User-editable watchlist (add/remove tickers in UI) — fixed at four tickers for v1; expanding is a code change
- Auto-polling / interval refresh — fetch on page open only, to stay safely inside free Cloudflare Worker and data-API quotas
- Buy/sell or any trade-action language — securities compliance
- ISIN / WKN identifiers — securities compliance
- Database / server-side state — state is localStorage + live client-side fetch only (zero-cost constraint)
- Chart in pure manual mode — hand-entering a year of daily prices is unreasonable; sparkline is progressive enhancement only
- German localization — English only for v1 (add later only if trivial)
- Any paid service or paid hosting tier — hard zero-cost constraint

## Context

- **Niche-ticker data coverage is the central unknown.** STRK/STRC/STRD are MicroStrategy/Strategy preferred shares; SATA is similarly niche. Free data APIs commonly cover common stock but not these preferreds. The app is therefore designed manual-first: it must be fully usable with hand-entered annual dividend (and price), with live fetch as progressive enhancement wherever coverage exists. Resolving which (if any) free source covers these tickers — for current price, dividend amount, and ≤1yr price history — is the first research priority.
- **Architecture is fully client-side + thin proxy.** Static Astro site on Cloudflare Pages; a free Cloudflare Pages Function/Worker acts as a key-hiding, CORS-resolving proxy to the data API. No secret ever reaches client JS. Each page load triggers a fetch from that visitor's own browser through the proxy — loads are independent, so concurrent visitors don't interfere; the only shared limits are the free Worker quota and the upstream API rate limit, both ample for a 4-ticker personal dashboard fetching once per load.
- **Design continuity with an existing blog.** The look should sit in the same visual family as the user's blog: dark slate-900 background, brand-gold (#fbbf24) accent, Inter / Merriweather / mono type scale, and the text-[10px]/text-[11px] font-mono uppercase tracking-widest label idiom. Modern, clean, one screen, table-first, with big tabular/mono figures. (Sourcing the exact blog tokens/repo would tighten fidelity in the UI phase — optional.)
- **Compliance posture.** These are securities. No buy/sell language anywhere, no ISIN/WKN, and a visible risk/informational-data disclaimer. The posture extends even to repo-level metadata.

## Constraints

- **Budget**: Strictly zero-cost — Cloudflare Pages free tier, free Worker/Function, free data API only. No paid tiers, no database.
- **Tech stack**: Astro static site; Tailwind-style utility classes implied by the design idiom; Cloudflare Pages + Pages Function/Worker proxy; localStorage for persistence; client-side fetch.
- **Security**: Data-API key must live only in the Worker/Function environment — never exposed in client JS or the repo.
- **Compatibility**: Must run as a static SPA in a modern browser; degrade gracefully when feeds are missing or rate-limited.
- **Compliance**: Securities context — no buy/sell language, no ISIN/WKN, mandatory informational disclaimer.
- **Scope**: One screen, table-first, fixed four-ticker watchlist, fetch-on-load only.

## Key Decisions

<!-- Decisions that constrain future work. Add throughout project lifecycle. -->

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Manual-first design (live fetch as progressive enhancement) | Niche preferred tickers likely lack free-feed coverage; app must work regardless | — Pending |
| Fixed watchlist (STRK/STRC/STRD/SATA) for v1 | Keeps it a focused personal dashboard; expansion is a code change | — Pending |
| Fetch on page load only (no auto-poll) | Safest for free Worker + API quotas; avoids forgotten-tab quota burn | — Pending |
| Manual override of both dividend and price | Guarantees yield computes even with zero feed coverage | — Pending |
| No-data rows stay visible, prompt manual entry | Preserves the watchlist as a complete view; nudges correction | — Pending |
| Cloudflare Worker/Function proxy hides API key | Zero-cost CORS + secret hiding without a backend or DB | — Pending |
| Astro static + localStorage, no database | Zero-cost hosting; state is per-browser | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-06-04 after initialization*
