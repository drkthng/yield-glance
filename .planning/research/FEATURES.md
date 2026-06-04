# Feature Research

**Domain:** Single-page dividend-yield watchlist dashboard (personal, fixed-ticker, table-first)
**Researched:** 2026-06-04
**Confidence:** HIGH (core table/yield features), MEDIUM (sparkline implementation specifics), HIGH (compliance UX)

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features that, if missing, make the tool feel broken or untrustworthy.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Hero yield table above the fold | The entire value proposition is one glance = one accurate yield per ticker; if the table is not the first thing visible, the tool has failed | LOW | Four fixed rows: STRK, STRC, STRD, SATA. Columns: Ticker · Annual Dividend · Current Price · Yield % · As-Of Date |
| Yield computed correctly as annual-dividend ÷ current-price | This is the domain formula; any deviation makes the numbers meaningless to the user | LOW | Display as percentage, 2 decimal places minimum. Label clearly as "Current Yield" not vague "Yield" |
| As-of timestamp per ticker | Financial data without a time reference is dangerous — user must know how fresh the number is | LOW | Show date (and ideally HH:MM) of the price used. Can be fetched-time or last-manual-edit time |
| Subtle liveness cue on load | Confirms fetch completed (or failed); silently stale data is worse than no data | LOW | A small dot or label: "Updated just now" / "As of [timestamp]" / "Manual" — changes state on load completion |
| Graceful degradation on fetch failure | A white-screen or broken table on a 429/timeout destroys trust | MEDIUM | Show last-known cached value (from localStorage) with a "stale" indicator. Never remove a row. Never crash |
| Manual override of annual dividend | Niche preferred tickers (STRK/STRC/STRD/SATA) are frequently absent from free data feeds; tool must be usable with zero feed coverage | MEDIUM | Inline editable field, persisted to localStorage, takes precedence over fetched value |
| Manual override of current price | Same as above — price coverage for preferred shares on free APIs is sparse | MEDIUM | Inline editable field, persisted to localStorage, takes precedence over fetched value |
| No-data rows stay visible | Hiding a row when data is missing breaks the fixed-four watchlist mental model; the empty row itself communicates "this ticker has no data yet" | LOW | Show ticker label and muted placeholder cells with a "Enter manually" affordance |
| Prompt to manually enter when no data | Empty rows without a call-to-action are confusing; user needs to know what to do | LOW | Small inline label or icon in the empty price/dividend cell: e.g. "— add value" or a pencil icon |
| Persistent manual overrides across sessions | Users should not have to re-enter dividend/price after closing the browser | LOW | localStorage key per ticker, read on mount. Clear UX for "this is manually overridden" vs "fetched" |
| Visible risk / informational disclaimer | Securities context legally and ethically requires a disclaimer; its absence signals naivety | LOW | Footer placement minimum. Text: "For informational purposes only. Not investment advice. Data may be delayed or inaccurate." |
| No buy/sell language anywhere | Regulatory compliance in securities context; buy/sell language implies investment advice | LOW | Audit all button labels, tooltips, empty-state text. "Enter value" not "Buy signal." |

---

### Differentiators (What Makes This Tool Nice)

Features that match the yield-first, trading-desk personality and go beyond generic financial dashboards.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Per-ticker sparkline of trailing ≤12-month price history | Contextualizes the current price — user sees if yield is high because price dropped recently vs. secular trend | HIGH | Progressive enhancement only: render if price history exists in API response; skip entirely in pure manual mode. Hover shows date + price + computed yield at that point |
| Hover detail on sparkline (date · price · yield at that point) | Transforms a decorative mini-chart into an informational tool; yield-at-a-date is genuinely useful for preferred shares | MEDIUM | Requires crosshair/bisect interaction. Tooltip should show: Date, Price, Yield (computed from same annual dividend) |
| Dark "trading-desk" visual identity | Matches the user's existing blog aesthetic (slate-900, brand-gold #fbbf24, Inter/mono scale); signals seriousness and makes numbers easy to scan | LOW | Monospace numerics in the yield column. Uppercase tracking-widest labels. High-contrast accent only on the yield percentage |
| Manual vs. fetched source indicator per cell | User immediately knows which values are "live" vs. hand-entered without hunting | LOW | Small tag or color shift: "manual" label or muted italic for overridden cells vs. normal weight for fetched |
| Yield highlighted as the primary column | Reinforces the yield-first purpose; users should read yield before price | LOW | Larger font size or bold accent color (#fbbf24) on yield %. Price and dividend are supporting data |
| Fetch error displayed inline per ticker, not globally | A 429 on one ticker should not mask the other three that returned data | MEDIUM | Per-row error state: "Fetch failed — using cached" alongside the stale value |
| localStorage TTL awareness | Warns user when cached values are older than a configurable threshold (e.g. 24h) so stale manual overrides don't silently mislead | MEDIUM | Show age indicator: "Last updated 2 days ago" for cached data. Not a hard expiry — just a visual cue |
| No-reload, no auto-poll explicit messaging | User understands the app fetches once on load; "Refresh page to update" removes confusion | LOW | Single line below or in footer. Prevents confusion from users expecting live ticking |
| Yield formula shown on hover / tooltip | Builds trust; user can verify calculation | LOW | Tooltip on yield cell: "Annual dividend $X ÷ Price $Y = Z%" |

---

### Anti-Features (Deliberately NOT Building)

Features that seem useful but are out of scope, compliance-risky, or complexity traps for a v1 personal tool.

| Anti-Feature | Why Requested | Why Problematic | Alternative |
|--------------|---------------|-----------------|-------------|
| User-editable watchlist (add/remove tickers in UI) | Generic watchlist tools always have this | Expands scope dramatically; requires ticker search, validation, API coverage check, storage schema changes. Fixed four is a deliberate constraint | Watchlist changes are code changes in v1. Add UI management only after v1 validates the core concept |
| Auto-refresh / polling interval | Users expect real-time data | Burns free Cloudflare Worker quota and free API rate limits in forgotten-tab scenarios; adds complexity for a personal tool with one daily-glance use case | Explicit "Refresh" button (v1.x) or a manual page reload. State the fetch-on-load model visibly |
| Buy/sell signals or action language | Every trading tool has these | Securities compliance — implies investment advice without the legal framework | Informational-only framing throughout. No "buy" "sell" "accumulate" "target" language anywhere |
| ISIN / WKN identifiers | Standard in European financial tools | Compliance concern in EU context; adds no value to a four-ticker fixed watchlist where tickers are the identity | Use exchange ticker symbols only (STRK, STRC, STRD, SATA) |
| Price chart in pure manual mode | Sparkline looks great, user might want it | Hand-entering 365 daily prices is unreasonable; an empty sparkline is confusing; a partial manual sparkline is misleading | Suppress sparkline entirely when no API price history is available. Show a text note: "Price history unavailable" |
| Alerts / notifications / email | Every financial dashboard eventually adds these | Requires backend, auth, scheduling — incompatible with zero-cost static + Worker architecture | Glance-based UX: user checks the dashboard when they want an update |
| Multi-user / portfolio tracking | Portfolio tools are the natural evolution | Requires auth, per-user storage, backend — destroys the zero-cost constraint | Stay personal (per-browser localStorage). No accounts, no sharing |
| Historical yield chart (full-page, interactive) | Natural desire to see yield over time | History requires a price-history feed; preferred tickers likely lack coverage; a full chart is a different product | Sparkline (mini, progressive) covers the directional context need without full chart infrastructure |
| German localization / multi-language | User's blog may have German audience | Adds i18n complexity; financial/compliance terminology requires legal review per locale | English-only v1. Trivial to add one locale later if demand exists |
| Dark/light theme toggle | Standard in modern apps | Adds implementation cost for a fixed-identity tool; the dark trading-desk theme IS the brand | Commit to dark theme. Add toggle only if user explicitly requests it post-launch |
| CSV / export | Power users want data portability | Out of scope for a four-ticker glance tool; adds UI complexity | Manual data entry screen is already the "power user" escape hatch |
| Paid data API integration | Better coverage for preferred tickers | Hard zero-cost constraint; paid tiers are explicitly excluded | Free API + manual fallback is the architecture |

---

## Feature Dependencies

```
[Hero yield table]
    └──requires──> [Yield formula: annual dividend ÷ current price]
                       ├──requires──> [Annual dividend value]  ←── fetched OR manual override
                       └──requires──> [Current price value]    ←── fetched OR manual override

[Manual override — dividend]
    └──requires──> [localStorage persistence layer]
    └──enhances──> [Hero yield table] (yield computes even with no feed)

[Manual override — price]
    └──requires──> [localStorage persistence layer]
    └──enhances──> [Hero yield table]

[No-data rows (visible, muted)]
    └──requires──> [Hero yield table renders all 4 rows unconditionally]
    └──enhances──> [Manual override — dividend] (empty state prompts entry)
    └──enhances──> [Manual override — price]

[Graceful degradation]
    └──requires──> [localStorage persistence layer] (read last-known values)
    └──requires──> [Per-row error/stale state]
    └──enhances──> [Hero yield table]

[Sparkline (per-ticker price history)]
    └──requires──> [Price history data from API] (progressive enhancement — no manual equivalent)
    └──requires──> [Hero yield table rows exist]
    └──enhances──> [Hover tooltip: date · price · yield at that point]
                       └──requires──> [Annual dividend value] (to compute historical yield)

[As-of timestamp + liveness cue]
    └──requires──> [Fetch completion signal]
    └──enhances──> [Graceful degradation] (shows "stale" when using cached)

[Manual vs fetched source indicator]
    └──requires──> [localStorage persistence layer]
    └──enhances──> [Hero yield table]

[Risk / informational disclaimer]
    └──independent] (static content, no dependencies)

[localStorage persistence layer]
    └──independent] (foundational — everything that persists depends on it)
```

### Dependency Notes

- **Sparkline requires API price history:** If the data source does not return ≤12-month OHLC or close prices for these preferred tickers, sparklines are suppressed for that ticker. This is the highest-risk dependency — preferred share coverage on free APIs is uncertain.
- **All yield computation requires at least one value from each input:** If both fetch AND manual override are absent for a ticker's price, the yield cell shows "—" with a prompt. The row stays visible.
- **Hover tooltip on sparkline requires annual dividend:** The historical yield shown in the tooltip is computed as annual-dividend ÷ historical-price. If annual dividend is unknown (no fetch, no manual), the tooltip shows price only — not yield.
- **Graceful degradation requires localStorage cache to have been populated on a prior successful load:** On first load with a total fetch failure and no manual entries, the table shows all four rows in no-data state. This is acceptable — user is immediately prompted to enter values manually.
- **Disclaimer is independent:** Static HTML/text, no data dependency. Must render even if all fetches fail.

---

## MVP Definition

### Launch With (v1)

| Feature | Why Essential |
|---------|---------------|
| Hero yield table — all 4 rows always visible | Core value proposition |
| Yield = annual-dividend ÷ price, computed in browser | The formula is the product |
| As-of timestamp (fetched time or "Manual") | Financial data without a timestamp is unacceptable |
| Liveness cue (loading → loaded/failed state) | Confirms fetch completed; sets data freshness expectation |
| Manual override — annual dividend, persisted in localStorage | Required for niche tickers with no feed coverage |
| Manual override — current price, persisted in localStorage | Required for niche tickers with no feed coverage |
| No-data rows visible with manual-entry prompt | Watchlist must show all 4 tickers regardless of data availability |
| Graceful degradation (stale cached values on fetch failure) | Never white-screen; never lose manually entered data |
| Risk / informational disclaimer in footer | Securities compliance — non-negotiable |
| No buy/sell language anywhere in UI | Securities compliance — non-negotiable |
| Dark trading-desk visual identity | Brand continuity; reinforces seriousness |
| Manual vs. fetched source indicator per row | Trust-building; user must know what they're seeing |

### Add After Validation (v1.x)

| Feature | Trigger for Adding |
|---------|-------------------|
| Sparkline — per-ticker price history mini-chart | Only after confirming a free API returns ≤12-month price history for STRK/STRC/STRD/SATA. Add as progressive enhancement. |
| Hover tooltip on sparkline (date · price · yield) | Requires sparkline to exist first |
| Explicit "Refresh page to update" nudge | Add if user testing reveals confusion about data staleness |
| localStorage TTL warning (stale cache indicator) | Add if users report trusting outdated cached values without knowing |
| Yield formula shown on cell hover | Add if users ask "how is this calculated?" |
| Fetch error displayed inline per ticker | Add if multi-ticker partial failures are observed in practice |

### Future Consideration (v2+)

| Feature | Why Defer |
|---------|-----------|
| Additional tickers (beyond fixed four) | Requires watchlist management UI, ticker validation, API coverage check |
| Auto-refresh / polling | Requires quota analysis and architecture change |
| Dark/light theme toggle | Low priority; dark IS the identity |
| German localization | Only if audience data supports it |
| Export / share | Different product scope |

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Hero yield table (4 rows, always visible) | HIGH | LOW | P1 |
| Yield formula (annual dividend ÷ price) | HIGH | LOW | P1 |
| Manual override — dividend (localStorage) | HIGH | MEDIUM | P1 |
| Manual override — price (localStorage) | HIGH | MEDIUM | P1 |
| No-data rows with entry prompt | HIGH | LOW | P1 |
| Graceful degradation (stale cache fallback) | HIGH | MEDIUM | P1 |
| As-of timestamp + liveness cue | HIGH | LOW | P1 |
| Risk / informational disclaimer | HIGH | LOW | P1 |
| No buy/sell language audit | HIGH | LOW | P1 |
| Dark visual identity (slate-900, brand-gold) | MEDIUM | LOW | P1 |
| Manual vs. fetched source indicator | MEDIUM | LOW | P1 |
| Sparkline — price history mini-chart | MEDIUM | HIGH | P2 |
| Hover tooltip on sparkline | MEDIUM | MEDIUM | P2 |
| Yield formula tooltip on yield cell | LOW | LOW | P2 |
| localStorage TTL / stale-cache warning | MEDIUM | MEDIUM | P2 |
| Per-ticker inline fetch-error state | MEDIUM | MEDIUM | P2 |

---

## Compliance UX Requirements

Securities-specific requirements that are not optional and must be treated as table stakes regardless of complexity.

| Requirement | Standard | Implementation |
|-------------|----------|----------------|
| Visible disclaimer ("not investment advice") | Industry standard; mirrors Google Finance, Preferred Stock Channel, and all financial data tools | Footer text, always visible. Minimum: "For informational purposes only. Not investment advice. Data may be delayed or inaccurate." |
| No buy/sell/accumulate language | Securities compliance | Audit all labels, tooltips, empty states, error messages, button text |
| No ISIN / WKN identifiers | Avoids triggering securities registration obligations in EU/DE jurisdictions | Use exchange ticker symbols only |
| "Data may be delayed" caveat | Standard for any data-feed-backed tool; Preferred Stock Channel uses "Quote data delayed at least 20 minutes" | Include in disclaimer; reinforce with as-of timestamp |
| No price targets or recommendations | Informational-only posture | Do not add any language that implies action (e.g., "high yield opportunity", "undervalued") |

---

## Competitor Feature Reference

| Feature | Preferred Stock Channel | Plainzer / Simply Wall St | YieldGlance Approach |
|---------|------------------------|--------------------------|----------------------|
| Yield display | Yes, as screener column | Yes, with portfolio context | Yes — hero column, primary number |
| Manual entry | No | No (broker sync only) | Yes — core differentiator; manual-first |
| Fixed watchlist | No — screener/search model | No — portfolio model | Yes — four tickers, one screen |
| Sparkline | No | Yes (full price history) | Progressive enhancement only |
| Data delay disclaimer | "Delayed 20 min" | Not prominently stated | Footer + as-of timestamp |
| Dark theme | No | Partial | Yes — full dark trading-desk identity |
| Auto-refresh | Yes (page-level) | Yes | No — fetch on load only |
| Buy/sell language | Screener context implies action | Portfolio context implies management | None — informational only |

---

## Sources

- Preferred Stock Channel — preferred yield display and disclaimer patterns: https://www.preferredstockchannel.com/
- Capitally — dividend tracker feature comparison: https://www.mycapitally.com/blog/best-dividend-tracker
- GetGen AI — disclaimer placement best practices: https://www.getgen.ai/post/best-practices-for-disclaimer-placement-expert-tips-for-compliance-and-clarity
- AG Grid sparklines tooltips: https://www.ag-grid.com/javascript-data-grid/sparklines-tooltips/
- LogRocket — empty state UX: https://blog.logrocket.com/ux-design/empty-state-ux/
- Digital Fortress — localStorage with TTL: https://digitalfortress.tech/js/localstorage-with-ttl-time-to-expiry/
- NN/G — empty states in complex applications: https://www.nngroup.com/articles/empty-state-interface-design/
- Wildnet Edge — Fintech UX dashboard best practices: https://www.wildnetedge.com/blogs/fintech-ux-design-best-practices-for-financial-dashboards
- Ryan O'Connell CFA — preferred stock yield calculator (formula reference): https://ryanoconnellfinance.com/calculators/preferred-stock-yield-calculator/

---

*Feature research for: YieldGlance — single-page dividend-yield watchlist dashboard*
*Researched: 2026-06-04*
