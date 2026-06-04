# Pitfalls Research

**Domain:** Niche preferred-stock yield dashboard — Astro + Cloudflare Pages/Worker proxy + localStorage + free data APIs
**Researched:** 2026-06-04
**Confidence:** HIGH (pitfalls drawn from official Cloudflare docs, verified API behavior, securities compliance norms, and patterns confirmed across multiple community sources)

---

## Critical Pitfalls

### Pitfall 1: Free API Silently Returns No Data for Preferred Tickers (Not an Error — a Blank)

**What goes wrong:**
Most free stock APIs (Alpha Vantage, Finnhub, Twelve Data, Polygon free tier) cover common stock; preferred-stock tickers like STRK, STRC, STRD, and SATA are frequently absent. The API does not return a 4xx error — it returns a successful 200 with an empty array, a null field, or a `{"Note":"…"}` throttle message. Client code that does `data.price` on this quietly gets `undefined`, and yield silently displays as `NaN%` or `Infinity%` without any visible error state.

**Why it happens:**
Developers test against common-stock tickers (AAPL, MSFT) where coverage is universal, never verify preferred tickers before writing rendering logic, and assume a 200 response means valid data.

**How to avoid:**
- Treat any response that lacks the expected price or dividend field as a data-absent (not error) state — exactly like a network failure.
- Validate the payload: assert `typeof response.price === 'number' && isFinite(response.price)` before passing to yield computation.
- Test every data-source candidate explicitly against STRK, STRC, STRD, SATA before committing to it in the proxy.
- Design the UI so a missing-data row (muted, "enter manually") is the default render state, not something triggered only on thrown errors.
- Note: Strategy's own site uses Massive (formerly Polygon) as its data source — this is the highest-confidence free-tier lead for these tickers.

**Warning signs:**
- Yield column shows `NaN`, `Infinity`, or blank on first load even though the network call succeeded.
- Console shows `undefined` when accessing `.price` or `.dividend` on a parsed API response.
- Sparkline silently renders no points while returning no error.

**Phase to address:** Data-source research spike (earliest research phase) and proxy implementation phase.

---

### Pitfall 2: Ticker Symbol Format Mismatch Across Providers

**What goes wrong:**
Preferred stocks use provider-specific symbol conventions. Yahoo Finance uses `MSTR-PA` or `MSTR-PL` format. Polygon uses `MSTRP` or `MSTR%2FPR`. Alpha Vantage may not recognize any variant. STRK/STRC/STRD are recently issued with no historical naming convention — some providers ingest them under the NASDAQ symbol, others under an OTC variant, and some not at all. Sending the wrong symbol string returns empty data silently (see Pitfall 1).

**Why it happens:**
There is no universal preferred-stock ticker standard. Each exchange and data vendor independently decided on suffix conventions (-P, /PR, p, space, nothing).

**How to avoid:**
- Build a symbol-mapping table in the proxy: canonical internal symbol → provider-specific string. For example: `{ STRK: { polygon: "STRK", alphavantage: "STRK" } }`.
- Validate each provider's returned ticker in the response against the requested symbol before trusting the data.
- For each provider under consideration, manually confirm via their API or UI which exact string resolves STRK/STRC/STRD/SATA before writing proxy code.
- Store the canonical symbol in localStorage and UI, not the provider-specific string.

**Warning signs:**
- A provider that "covers preferred stocks" returns empty for STRK but non-empty for MSTR.
- Different providers return different prices for what should be the same instrument.
- Testing with hardcoded strings works but production fails after a symbol-mapping refactor.

**Phase to address:** Proxy implementation phase. The symbol table must be established in the data-source spike.

---

### Pitfall 3: Dividend Frequency Misidentification → Wrong Annualized Yield

**What goes wrong:**
STRC pays monthly (11.50% nominal, ~$0.958/month/share); STRK pays quarterly (8.00% nominal). If a data API returns the last single dividend payment without a frequency label, and the code multiplies it by 4 (quarterly assumption), STRC's yield is understated by 3x. Multiplying by 12 on STRK overstates by 3x. Some APIs return a `frequency` field; many do not. Special one-time dividends (e.g., dividend on liquidation or a return-of-capital distribution) are indistinguishable from regular dividends in the raw `amount` field.

**Why it happens:**
The yield formula looks trivially simple (`annual_div / price`), so developers hardcode the multiplier without verifying frequency, and copy the multiplier from common-stock examples (typically quarterly) without checking each preferred's actual schedule.

**How to avoid:**
- Store a per-ticker `dividendFrequency` constant (12 for monthly, 4 for quarterly) in the app config, initialized from the offering documents for STRK/STRC/STRD/SATA — not inferred from API data.
- If the API returns a `frequency` field, validate it matches the config constant; surface a warning if they diverge rather than silently trusting the API.
- Prefer using the stated annual rate from the prospectus (e.g., 8% × face value for STRK) as the primary dividend figure; use market-price-based computation only as a cross-check.
- Explicitly exclude `frequency === 0` (special/non-recurring) dividends from annualization — they must not be multiplied by 4 or 12.
- Add a unit test: `annualize(0.958, 'monthly') === 11.496`, `annualize(2.00, 'quarterly') === 8.00`.

**Warning signs:**
- Yield for STRC shows ~3.8% instead of ~11.5%.
- Yield for STRK shows ~24% instead of ~8%.
- API returns `frequency: 0` for a record that matches the regular dividend date.

**Phase to address:** Yield-computation module (core logic phase).

---

### Pitfall 4: API Key Committed to the Git Repository

**What goes wrong:**
A developer stores the data API key in `wrangler.toml` under `[vars]`, in a `.env` file that was not gitignored before the first commit, or directly in the Astro config. The key appears in GitHub. Automated bots scan public repos for API key patterns within minutes of a push; the key is used and the account is suspended or billed.

**Why it happens:**
`wrangler.toml` is a config file that belongs in git, so developers add `API_KEY = "abc123"` to `[vars]` without realizing `[vars]` is plaintext and committed. First-time Cloudflare Workers developers confuse vars (config, committed) with secrets (encrypted, never committed).

**How to avoid:**
- Use `wrangler secret put DATA_API_KEY` to store the key as an encrypted Worker secret — it never appears in the file system or git history.
- For local development, use a `.dev.vars` file containing `DATA_API_KEY=...` and add `.dev.vars` and `.env` to `.gitignore` before writing any code.
- Never put any token, key, or password in `wrangler.toml`.
- Add a `gitleaks` or GitHub secret-scanning pre-commit hook from day one.
- Create a `.env.example` with placeholder values and commit that instead.
- The rule: if it's in a file that git tracks, it's not a secret.

**Warning signs:**
- `wrangler.toml` contains a string that looks like an API key (alphanumeric, 20+ chars).
- `git log --all -p | grep -i "api_key"` returns a result.
- GitHub shows a "Secret scanning alert" notification.

**Phase to address:** Project scaffolding phase (day zero — before the first commit).

---

### Pitfall 5: API Key Leaks into the Client JavaScript Bundle

**What goes wrong:**
In a static Astro site, any value accessed at build time or imported from a module that is not in a `src/pages/api/` server-side file gets bundled into client JS. If the proxy Worker is never built, or the dev falls back to calling the data API directly from a client-side fetch call to avoid CORS complexity, the key becomes visible in the browser's network panel or by reading the built JS file.

**Why it happens:**
CORS errors during development feel blocking; the easiest fix is to proxy from the client instead of from the Worker. Or the developer puts `import.meta.env.DATA_API_KEY` in a client component, not realizing that Astro only keeps values prefixed `PUBLIC_` private by convention.

**How to avoid:**
- The Worker/Pages Function is the only entity that ever calls the upstream data API. The client JS calls only the Worker endpoint (on your own domain, so no CORS needed).
- Never reference the API key anywhere in `src/` directories that produce client output.
- Verify at build time: `grep -r "API_KEY" dist/` must return nothing.
- In the Worker, validate the `Origin` header to restrict which domains can call it (defense in depth against key theft via a third-party site calling your proxy).

**Warning signs:**
- CORS errors when calling the data API directly from client JS (sign that client is trying to call the API directly).
- The API key string appears in `dist/_astro/*.js`.
- The browser network panel shows requests going to the data provider's domain from the page, not to your own worker.

**Phase to address:** Proxy architecture phase; verify in build/deploy phase.

---

### Pitfall 6: Overly Permissive CORS on the Worker Proxy

**What goes wrong:**
The Worker sets `Access-Control-Allow-Origin: *` so any website on the internet can call the proxy, effectively borrowing the API quota and potentially exhausting the 100k daily Workers requests for a personal dashboard. Combined with an exposed Worker URL, this turns a personal proxy into a free public data relay.

**Why it happens:**
`*` is the first CORS fix shown in tutorials; it makes the error go away; developers move on without tightening it. The Worker URL is public by default at `*.pages.dev`.

**How to avoid:**
- Set `Access-Control-Allow-Origin` to the specific production domain only (e.g., `https://yield.example.com`).
- During development, also allow `http://localhost:4321`.
- Check the `Origin` request header against an allowlist; if not in the list, return a 403.
- Do not publish the Worker endpoint URL in public documentation or the repo README.

**Warning signs:**
- Another site can fetch your Worker endpoint and get data back.
- Workers request count is higher than page view count.
- Rate limit errors on the upstream API despite low personal usage.

**Phase to address:** Proxy implementation phase.

---

### Pitfall 7: Stale Price with Fresh Dividend (or Vice Versa) Produces a Meaningless Yield

**What goes wrong:**
The proxy fetches price and dividend from two different endpoints with different cache ages or different data-freshness guarantees. Price is from a real-time endpoint (age: seconds); dividend is from a fundamentals endpoint that updates quarterly (age: days to weeks). The displayed yield appears precise (e.g., `11.42%`) but is computed from a mix of a today's price and a dividend figure that has not been updated since the last quarterly filing.

Worse: after a dividend change (e.g., STRC adjusting its rate), the old cached dividend amount persists in the Worker cache or in localStorage while the new price is live, producing a silently wrong yield.

**Why it happens:**
Price and dividend are treated as symmetric real-time values, but dividend data has much coarser update cadence in financial APIs. Developers verify accuracy at the moment of building (when both happen to be current) and never test a scenario where one is stale.

**How to avoid:**
- Display separate `as-of` timestamps for price and for dividend: "Price as of 14:32 ET · Dividend as of 2026-04-15."
- Never cache dividend data longer than 24h in the Worker, and never mix it with a price that is live.
- Treat the dividend figure as "last known quarterly declared rate" — stable for STRK/STRC/STRD/SATA unless Strategy announces a change. Pre-seed the dividend constants from the prospectus; use API data only as a cross-check.
- The manual override for dividend exists precisely for this: when the API value diverges from reality, the user can correct it.

**Warning signs:**
- The app displays yield but one `as-of` label is weeks old.
- Dividend field in the API response matches an old quarterly value after a rate change.
- localStorage holds a dividend value from before a dividend adjustment, and no UI cue shows its age.

**Phase to address:** Yield-computation and data-freshness logic phase.

---

### Pitfall 8: localStorage Schema Breaking Existing User Data on Redeploy

**What goes wrong:**
In v1, overrides are stored as `{ ticker: { annualDividend: number, price: number } }`. In v2, the schema changes to `{ version: 2, overrides: { ticker: { annualDividend: number, manualPrice: number } } }` (field renamed). On first visit after deploy, the app reads the old schema, finds `price` instead of `manualPrice`, falls back to undefined, and the manually-entered override the user relied on silently disappears.

**Why it happens:**
localStorage is schemaless and never enforced at write time. Developers rename fields during refactoring without adding a migration path.

**How to avoid:**
- Version the localStorage key from day one: `yield-glance:overrides:v1`. On read, check the version; if absent or mismatched, either migrate or discard safely with a user-visible notice ("Your saved values were reset; please re-enter them.").
- Write a pure migration function `migrateV1toV2(stored)` before merging any schema change.
- Never reuse the same key name for a structurally different value.
- Store a schema version number inside the JSON value itself, e.g., `{ _v: 1, ... }`.
- Treat losing manually-entered dividend overrides as equivalent to data loss — migration is not optional.

**Warning signs:**
- After deploying a refactor, manually-entered values silently reset to empty.
- `JSON.parse(localStorage.getItem('yield-glance:overrides'))?.price` returns a value but the UI shows no override applied (because the code now reads `manualPrice`).
- App throws a TypeError on a property access that was present in old schema.

**Phase to address:** localStorage persistence design (first phase that touches localStorage); verify in any phase that changes the stored shape.

---

### Pitfall 9: localStorage Override Precedence Bug (Fetch Silently Wins)

**What goes wrong:**
The merge logic is `const dividend = fetchedData?.annualDividend ?? storedOverride.annualDividend`. If the API returns a value (even a stale or wrong one), the nullish coalescing operator uses it and the user's manual override is ignored. The user enters a corrected dividend; the page shows it once; on the next load the API value wins and the correction is gone.

**Why it happens:**
`??` correctly handles `null`/`undefined` but not "API returned a bad value." Developers write "API wins unless absent" when the correct rule is "manual override always wins."

**How to avoid:**
- The precedence rule must be explicit: **manual override always wins**, regardless of whether fetched data exists.
- Code pattern: `const dividend = storedOverride.annualDividend ?? fetchedData?.annualDividend`.
- Write a unit test asserting that when both an override and a fetched value exist, the override is used.
- In the UI, show a distinct indicator (e.g., a pencil icon or "manual" badge) when the override is active, so the user can see which value is in effect.

**Warning signs:**
- Manually-entered values appear correctly on the first render but revert after subsequent loads.
- Manually-entered dividend produces the correct yield once, then shows a slightly different value (the API value) on reload.
- Removing localStorage clears the issue, confirming it is a merge-precedence bug.

**Phase to address:** localStorage and state-management phase.

---

### Pitfall 10: White-Screen or Broken Layout on Fetch Failure

**What goes wrong:**
The Worker or upstream API is unavailable. The client fetch throws a network error. The component that renders the yield table waits for a resolved promise that never resolves (or rejects unhandled), and the page either white-screens, shows an infinite spinner, or throws an uncaught React/Astro island error that breaks the entire DOM below it.

**Why it happens:**
Happy-path development: the data-fetch is a prerequisite to rendering any row, so the component renders nothing until data arrives, with no fallback. Unhandled promise rejections in fetch chains crash the surrounding component.

**How to avoid:**
- Fetch with a timeout (e.g., `AbortController` + 5-second timeout). If it fires, proceed with stored/manual values.
- Render the table rows immediately using localStorage or hardcoded defaults; fill in live values when they arrive. Progressive enhancement, not blocking.
- Every ticker row renders in a "no live data" muted state by default — it never waits for fetch before existing in the DOM.
- Catch all fetch and parse errors at the top level of the data-fetch function; return `null` values rather than throwing.
- Show a subtle indicator ("Using cached data" or "Could not connect") rather than hiding the entire table.

**Warning signs:**
- Blank page with no error when the Worker URL is temporarily wrong.
- Network tab shows a failed fetch, but the UI shows no row or error state.
- The TypeScript type system allows `yield` to be `undefined` at the point of rendering, meaning the no-data state is not explicitly handled.

**Phase to address:** Core UI and data-fetch phase; verify in integration testing.

---

### Pitfall 11: Rate Limit Exhaustion Consuming the Daily Quota by Mid-Morning

**What goes wrong:**
Alpha Vantage free tier: 25 requests/day, 5 requests/minute. With 4 tickers and no caching, a single user visiting 7 times exhausts the daily quota (4 × 7 = 28 > 25). A second user after that gets throttle messages — silent empty data — for the rest of the day. No error is surfaced; the page degrades to manual-only mode without telling the visitor why.

**Why it happens:**
Personal dashboard designed for one user, but the daily quota is shared across all visitors. Fetch-on-load with 4 tickers means each page load costs 4 API calls if there is no Worker-level caching.

**How to avoid:**
- Cache API responses at the Worker level (Cloudflare Cache API or KV if needed) with a short TTL (e.g., 15–30 minutes for price, 24h for dividend). Multiple visitors within the TTL window cost one API call total.
- For a 4-ticker dashboard with 30-minute cache, a personal site can sustain 25/(4 calls per miss) = ~6 full cache misses per day — roughly every 4 hours, which is more than sufficient.
- Alternatively, batch all 4 tickers in a single API call if the provider supports it (Polygon's batch endpoint, for example).
- Surface a visible notice when the API response is a throttle message rather than silently showing no data.
- The fetch-on-load-only constraint (no auto-polling) already aligns with quota conservation — do not relax this constraint.

**Warning signs:**
- Alpha Vantage returns `{"Note":"Thank you for using Alpha Vantage! Our standard API rate limit is 25 requests per day..."}` — a 200 with JSON, not an HTTP error.
- Yield values disappear mid-day for all tickers simultaneously.
- Worker logs show more outbound requests per page load than expected.

**Phase to address:** Proxy implementation phase (Worker caching must be part of the proxy design, not added later).

---

### Pitfall 12: API Terms of Service Violation for Public Display

**What goes wrong:**
Free-tier stock data APIs (Alpha Vantage, Twelve Data, Finnhub) often prohibit redistribution or commercial display of data. A personal dashboard "informational only" site may still qualify as a public website redistributing market data. Violating ToS results in key suspension. Some require attribution or their logo in the UI.

**Why it happens:**
Developers read "free for personal use" and stop there, without reading the redistribution and display clauses that apply even to non-commercial sites.

**How to avoid:**
- Read the ToS of every data provider before integration, specifically the sections on: redistribution, public display, commercial use, attribution requirements.
- Twelve Data and Finnhub both allow personal/non-commercial display with attribution; confirm this for your chosen provider.
- If the site is ever made publicly accessible (not just localhost), ensure the ToS permits "display on a public website."
- Keep a note in the project of which provider was chosen and the ToS clause that authorizes the use.
- If any provider's ToS is ambiguous, prefer the one with clearer permissive language.

**Warning signs:**
- API key suddenly stops working without any rate-limit or error message — key may have been suspended.
- ToS section titled "Redistribution" or "Display" requires a separate license or written consent.
- The provider's FAQ distinguishes "personal use on your own device" from "a website others can visit."

**Phase to address:** Data-source selection spike (before any API key is obtained or used).

---

### Pitfall 13: Astro output Mode / Cloudflare Pages Functions Routing Misconfiguration

**What goes wrong:**
Using Astro's `output: "hybrid"` mode with the Cloudflare adapter causes API routes (the proxy endpoint) to be treated as static files — they return the same built response on every request, ignoring URL parameters and environment variables. The proxy "works" in local dev (`wrangler dev`) but always returns the same empty response in production. Or: using `output: "static"` with no adapter means Pages Functions are not built at all, and the proxy 404s in production.

A second variant: placing the proxy in `functions/` (the Cloudflare Pages directory mode) while also having `_worker.js` generated by Astro's adapter causes a conflict — Cloudflare ignores the `functions/` directory when `_worker.js` is present.

**Why it happens:**
Astro and Cloudflare each have two integration modes that interact. The combination is not obvious, and the local dev server (`astro dev`) does not replicate the Pages Functions runtime, so bugs are invisible until deploy.

**How to avoid:**
- For a primarily static site with one dynamic proxy endpoint: use `output: "static"` for the Astro build and place the proxy in `functions/api/quote.ts` (Pages Functions directory mode) separately from the Astro build output.
- Alternatively, use `output: "server"` with the `@astrojs/cloudflare` adapter and treat the entire site as server-rendered — simpler routing, higher CPU usage per request.
- Test the proxy endpoint with `wrangler pages dev ./dist` (not `astro dev`) before declaring it working.
- Do not mix `_worker.js` (advanced mode) and the `functions/` directory — pick one deployment mode and document it.

**Warning signs:**
- Proxy endpoint returns the same response regardless of the `?ticker=` query parameter.
- API route works in `astro dev` but returns 404 or a stale response after Pages deploy.
- The `_routes.json` file is missing or does not include the `/api/*` paths.

**Phase to address:** Project scaffolding and proxy architecture phase.

---

### Pitfall 14: Cloudflare Worker Caching Leaks Quota (Over-Caching the Proxy)

**What goes wrong:**
The Worker caches API responses aggressively (e.g., 24h TTL for price data). A visitor at 9:00 AM gets a yield computed from last night's price. Meanwhile, the Worker has consumed 4 API calls in 24 hours — the quota is fine — but the data is stale and the yield is wrong by the time markets open. The user sees a "precise-looking" yield (11.42%) that is computed from a price that is 16 hours old.

Conversely, with no Worker caching, each of 4 page loads (different users) makes 4 separate API calls to price + dividend endpoints — 16 calls total for 4 users — rapidly exhausting the 25-call daily limit.

**Why it happens:**
The dual constraint (fresh data vs. limited quota) is not analyzed explicitly. The developer picks either "no cache" (burns quota) or "long cache" (stale data) without finding the correct TTL.

**How to avoid:**
- Price data: cache for 15–30 minutes (markets update continuously; 15 min lag is acceptable for a yield dashboard).
- Dividend data: cache for 24h or longer (declared dividends do not change intra-day).
- Return a `CF-Cache-Age` or custom `X-Data-Age` header so the client can show a precise "as-of" timestamp rather than "live."
- Outside of US market hours (before 9:30 AM ET, after 4:00 PM ET, weekends): accept a longer price cache TTL since prices do not change.

**Warning signs:**
- The `as-of` timestamp on yield is hours behind the current time during market hours.
- Removing the Worker cache causes a 25-call/day exhaustion by noon.
- The Worker response headers show `cf-cache-status: HIT` with a very long age during open-market hours.

**Phase to address:** Proxy implementation phase.

---

### Pitfall 15: Compliance — Implicit Buy/Sell Framing or Missing Disclaimer

**What goes wrong:**
UI copy that reads "STRK is yielding 11.42% — great value" or a highlight that turns cells green when yield exceeds a threshold implies a recommendation. A missing or hard-to-find disclaimer leaves the site without legal protection. Mentioning ISIN, WKN, or CUSIP identifiers brings the site closer to regulated instrument identification territory in some jurisdictions.

These are preferred securities of a Nasdaq-listed company, and displaying market data about them to the public (even informally) sits in a gray zone under SEC Rule 10b-5 if any statement could be construed as material and misleading.

**Why it happens:**
UI design naturally reaches for positive/negative color coding and superlative copy to make dashboards feel useful. The disclaimer gets added as an afterthought at the bottom, in tiny gray text that no one reads.

**How to avoid:**
- The disclaimer must be visible above the fold or immediately adjacent to the table — not only in a footer.
- Required language: "For informational purposes only. Not investment advice. Consult a qualified financial advisor before making investment decisions. Data may be delayed or inaccurate."
- No green/red traffic-light yield coloring that implies "good" / "bad." Use neutral styling — brand gold for all active values, muted for missing values.
- No comparative language ("high yield," "attractive," "cheap"). Present numbers only.
- No ISIN, WKN, or CUSIP anywhere — not in code comments that are visible in source, not in the DOM.
- Review every string in the codebase for buy/sell/recommend/attractive/avoid language before each deploy.

**Warning signs:**
- UI mockup has green cells for "above-threshold" yield.
- Copy review finds words like "high yield," "strong," or "worth watching."
- The disclaimer is only in a `<footer>` below the scroll.
- Any element's `data-*` attribute or `aria-label` contains an ISIN.

**Phase to address:** UI phase (design tokens and copy review); verify in pre-launch checklist.

---

### Pitfall 16: Division by Zero / Missing Price Produces Infinity or NaN in the UI

**What goes wrong:**
`yield = annualDividend / currentPrice` — if `currentPrice` is `0`, `null`, `undefined`, or an empty string from an unfilled manual override, the result is `Infinity`, `-Infinity`, or `NaN`. These propagate through `toFixed()` and `toLocaleString()` and display as `Infinity%` or `NaN%` in the live table.

**Why it happens:**
The formula is simple enough that developers write it inline without guard clauses. The no-price state is not explicitly modeled as "yield is unknown" — it falls through to a numeric computation.

**How to avoid:**
- Before computing yield, check: `if (!price || !isFinite(price) || price <= 0) return null`.
- A `null` yield is rendered as `—` (em dash) in the table, not as a number.
- Do not rely on `toFixed()` to hide `NaN` — it does not; `NaN.toFixed(2)` returns `"NaN"`.
- The manual price input must validate on entry: non-numeric input and zero must be rejected with an inline error, not silently stored.

**Warning signs:**
- The yield cell shows `Infinity%` or `NaN%`.
- `typeof computedYield === 'number' && !isFinite(computedYield)` is true.
- A user reports that entering `0` in the price field makes the yield cell show something weird.

**Phase to address:** Yield-computation module and manual-entry validation phase.

---

### Pitfall 17: Timezone Ambiguity on "As-Of" Timestamp

**What goes wrong:**
The Worker returns a Unix timestamp from the API. The client renders it with `new Date(ts).toLocaleString()` — which uses the visitor's local timezone. A user in Germany sees "15:32" as the price time; the actual NYSE close was 16:00 ET (22:00 CET). The user concludes the price is from mid-afternoon US time, not the close. Or worse: the timestamp is a date-only string (`2026-06-03`) interpreted in UTC, displayed as `June 2` in UTC-6 timezones — a full day off.

**Why it happens:**
JavaScript's `Date` constructor is timezone-naive for date-only strings; `new Date("2026-06-03")` is treated as UTC midnight and displays as June 2 in UTC-6. Developers test in their own timezone and never notice.

**How to avoid:**
- Always render timestamps in ET (America/New_York) with the timezone label explicit: "as of 2026-06-03 16:00 ET."
- Use `Intl.DateTimeFormat` with `timeZone: 'America/New_York'` for all timestamp rendering.
- Pass full ISO-8601 timestamps with timezone offsets from the Worker (never date-only strings).
- Show the market status alongside the timestamp: "(market closed)" or "(market open — delayed)" so the user knows what the timestamp means.

**Warning signs:**
- Timestamp displayed is one day behind in certain timezones.
- `new Date("2026-06-03")` in the browser console returns midnight UTC, not midnight ET.
- Users in non-ET timezones report a wrong date on the "as-of" label.

**Phase to address:** Data display and timestamp rendering phase.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Hardcode dividend multiplier without a config constant | Fewer abstraction layers | A dividend-frequency change (monthly→quarterly) requires a code search instead of a one-line config edit | Never — the constants belong in a single config object |
| `Access-Control-Allow-Origin: *` on the Worker | No CORS errors in 5 minutes | Proxy becomes a free public data relay; quota exhausted by strangers | Never for a public proxy |
| Omit localStorage version from day one | Less initial boilerplate | First schema change requires a history-rewriting migration to avoid silent data loss | Never — add `_v: 1` on day one |
| Single `as-of` timestamp for both price and dividend | Simpler UI | Silently shows a false precision when price is live and dividend is weeks stale | Never — show separate ages |
| No Worker-level cache (direct-to-API on each request) | Simpler proxy code | Exhausts 25-call/day quota by mid-morning for even light traffic | Acceptable only in a locked, single-user localhost dev environment |
| Display `NaN` or `Infinity` yield inline | Zero extra code | Looks broken; implies recommendation (e.g., `Infinity%` yield is literally "infinitely good") | Never |
| Store API key in `wrangler.toml` `[vars]` | Easy local dev | Key committed to git, credential rotation required, potential account suspension | Never — use `.dev.vars` + `wrangler secret put` |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Alpha Vantage free tier | Treating `{"Note":"...rate limit..."}` (HTTP 200) as data, not an error | Parse response first; if `Note` key exists, treat as no-data / quota-exhausted |
| Cloudflare Pages Functions | Placing the proxy in `functions/` directory while Astro adapter generates `_worker.js` | Pick one: either use Astro adapter (server/hybrid mode with `_worker.js`) or pure Pages Functions (static Astro output + `functions/`) |
| Cloudflare `wrangler secret` | Setting secrets only in dashboard for production, forgetting `.dev.vars` for local dev | Maintain `.dev.vars` (gitignored) that mirrors production secrets for local `wrangler pages dev` |
| Preferred stock tickers | Sending canonical symbol (e.g., `STRK`) to a provider that requires a different format | Maintain a symbol-map object in the proxy; test each provider with each ticker explicitly |
| localStorage manual overrides | Calling `JSON.parse(localStorage.getItem(key))` without a try/catch | Wrap in try/catch; return `null` on parse error; corrupted storage should not crash the app |
| API dividend endpoint | Using last-payment-date dividend without checking `frequency` field | Cross-reference with per-ticker frequency constant; reject `frequency === 0` (special dividend) from annualization |
| Cloudflare Cache API | Using `stale-while-revalidate` or `stale-if-error` directives with `cache.put()` | These directives are not supported in Workers Cache API — use explicit TTL + fallback logic instead |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| No Worker-level cache for 4-ticker fetch | 4 upstream API calls per page load; 25/day quota gone after 6 visitors | Cache API responses in the Worker for 15–30 min | Immediately — even at 1-user personal scale with Alpha Vantage free tier |
| Synchronous localStorage read on paint | Main thread blocks, slow initial render on older mobile | Read localStorage asynchronously or in a `useEffect`-equivalent after hydration | At ~5ms read time for small data, rarely perceptible — acceptable risk |
| Unbatched ticker requests | 4 sequential fetch calls from client to Worker | Single Worker endpoint accepts all tickers, returns batch JSON | At 4 tickers with serial fetches: ~400ms overhead on slow connections |
| Sparkline with large date range | 365 daily data points × 4 tickers = 1460 JSON objects in one response | Cap history at 90 days for sparkline; use compact format | Noticeable on mobile at >200 data points per ticker |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| API key in `wrangler.toml` `[vars]` (plaintext, committed to git) | Key exposure → account suspension, billing fraud | Use `wrangler secret put`; local dev via `.dev.vars` (gitignored) |
| `Access-Control-Allow-Origin: *` on proxy Worker | Any site can exhaust your API quota through your proxy | Allowlist only your production domain and `localhost:4321` |
| No `Origin` validation in Worker | Bot can call Worker endpoint and consume quota | Check `request.headers.get('Origin')` against allowlist; return 403 if not matched |
| API key in Astro client component via `import.meta.env` | Key bundled into client JS and visible in browser | Keys must only be accessed in server-side code (Pages Function / Worker); never in client components |
| `.dev.vars` committed to git | Local dev secret exposed | Add `.dev.vars` and `.env` to `.gitignore` before `git init` |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Table rows disappear while fetching | User sees blank table; cannot read previously valid yield | Render rows immediately with stored/default values; update in place when fetch completes |
| "Error" as the only fallback state | User cannot tell if their manually-entered values are safe | Show "Using saved values" with a timestamp when live fetch fails |
| Green/red yield coloring | Implies a buy/sell recommendation — compliance risk | Use a single neutral accent color (brand gold) for all yield values; no traffic-light coding |
| Disclaimer only in footer | Scroll-away disclaimer is legally and UX-weak | Disclaimer visible adjacent to the table, not scroll-dependent |
| Timezone-naive "as-of" label | User in non-ET timezone sees wrong date or confusing time | Explicitly label "as of [time] ET" using `Intl.DateTimeFormat` with `timeZone: 'America/New_York'` |
| Manual-entry field accepts zero or non-numeric values | Silent NaN yield in the table | Validate on input: reject 0, negative, and non-numeric values inline; do not store invalid values |
| No visual distinction between live and manual values | User cannot tell which yield is computed from API data vs. their own entry | Show a "manual" badge / pencil icon on rows where overrides are active |

---

## "Looks Done But Isn't" Checklist

- [ ] **Yield computation:** Verify yield returns `null` (not `NaN`) when price is missing — check `isFinite(price) && price > 0` guard is present before division.
- [ ] **Frequency constants:** Verify the dividend multiplier for each ticker is sourced from the prospectus and stored as a per-ticker config constant, not inferred from a single API call.
- [ ] **API key absence from bundle:** Run `grep -r "API_KEY\|api_key" dist/` after build — result must be empty.
- [ ] **CORS lockdown:** Verify Worker returns 403 for requests with Origin headers not in the allowlist — test with `curl -H "Origin: https://evil.com"`.
- [ ] **localStorage versioning:** Confirm the stored JSON contains `_v: 1` (or equivalent) and that a read of version-missing data returns `null` without crashing.
- [ ] **Override precedence:** Confirm that when both a localStorage override and a fetched value exist for annualDividend, the override value wins — test by setting an override, then triggering a fetch with a different mocked API value.
- [ ] **Fetch failure graceful degradation:** Confirm the table renders rows in a muted "no data" state when the Worker endpoint returns a 500 or times out — test by pointing the fetch URL at a non-existent endpoint.
- [ ] **Disclaimer visibility:** Confirm the disclaimer is visible without scrolling on a 1080p viewport — not only in the footer.
- [ ] **Timestamp timezone:** Confirm `as-of` timestamps display in ET notation for a user whose browser is set to UTC+1 or UTC-8.
- [ ] **ToS compliance:** Confirm the chosen data provider's ToS has been read and permits public display of their data on a personal website.

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| API key committed to git | HIGH | 1. Rotate key immediately in provider dashboard. 2. Remove from wrangler.toml, add to `.gitignore`. 3. Use `git filter-repo` to scrub from history if repo is public. 4. Set new key via `wrangler secret put`. |
| Wrong dividend multiplier in production | LOW | Update per-ticker config constant; redeploy (Cloudflare Pages redeploy is <1 min). |
| localStorage schema breakage after deploy | MEDIUM | Ship a migration function that reads old schema key, converts, writes to new key, deletes old key. Show a one-time notice to users that saved values were migrated. |
| Rate limit exhaustion (daily quota) | LOW | Add Worker-level caching with 30-min TTL. Quota resets at midnight UTC — no data until then if no cache. |
| Compliance language in production | MEDIUM | Remove offending copy, add/fix disclaimer, redeploy. If the site has significant traffic, consider a brief take-down to prevent archival of the non-compliant version. |
| Proxy 404 in production (routing misconfiguration) | MEDIUM | Verify `_routes.json` includes `/api/*`; confirm Astro output mode matches deployment mode; test with `wrangler pages dev` before next deploy. |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| API silently returns no data for preferred tickers | Data-source spike (Phase 1) | Manually call each candidate API with STRK/STRC/STRD/SATA before any code is written |
| Ticker symbol format mismatch | Data-source spike + proxy implementation | Proxy symbol-map unit test; integration test with real API response |
| Dividend frequency misidentification | Yield-computation module | Unit test: `annualize(amount, frequency)` with known values from prospectus |
| API key committed to git | Project scaffolding (day zero) | `git log --all -p | grep -i api_key` returns nothing; `.gitignore` includes `.dev.vars` |
| API key leaks into client bundle | Proxy architecture phase | `grep -r "API_KEY" dist/` post-build returns nothing |
| Overly permissive CORS | Proxy implementation | `curl -H "Origin: https://evil.com" <worker-url>` returns 403 |
| Stale price / stale dividend mixed yield | Data-freshness design | Separate `as-of` timestamps visible in UI for price and dividend |
| localStorage schema breakage | First localStorage phase | Versioned key present in stored JSON from day one; migration test |
| Override precedence bug | State-management phase | Unit test: override wins when both override and fetched value present |
| White-screen on fetch failure | Core UI phase | Integration test: fetch endpoint replaced with 500; table rows remain visible in muted state |
| Rate limit exhaustion | Proxy implementation | Load test: 10 simulated page loads exhaust fewer than 10 API calls (Worker cache is working) |
| ToS redistribution violation | Data-source selection spike | ToS clause noted in project documentation before any key is obtained |
| Astro output / Pages Functions routing | Project scaffolding | `wrangler pages dev ./dist` produces correct API response before first commit of proxy code |
| Worker over-caching stale data | Proxy implementation | Verify Worker cache TTL is ≤30 min for price; confirm `X-Data-Age` header is returned |
| Compliance / missing disclaimer | UI phase | Disclaimer visible above the fold on 1080p without scrolling; no buy/sell language in any string |
| Division by zero on missing price | Yield-computation module | Unit test: `computeYield(100, 0)` returns `null`, not `Infinity` |
| Timezone ambiguity on as-of label | Data display phase | Test in a browser with timezone set to UTC-6: label shows correct ET date |

---

## Sources

- Cloudflare Workers Limits (official): https://developers.cloudflare.com/workers/platform/limits/
- Cloudflare Pages Limits (official): https://developers.cloudflare.com/pages/platform/limits/
- Cloudflare Workers Secrets (official): https://developers.cloudflare.com/workers/configuration/secrets/
- Cloudflare Workers CORS example (official): https://developers.cloudflare.com/workers/examples/cors-header-proxy/
- Alpha Vantage rate limits: https://www.macroption.com/alpha-vantage-api-limits/
- Astro Cloudflare adapter docs: https://docs.astro.build/en/guides/integrations-guide/cloudflare/
- Deploy Astro on Cloudflare Pages: https://docs.astro.build/en/guides/deploy/cloudflare/
- Cloudflare Pages Functions routing pitfall (community): https://community.cloudflare.com/t/cloudflare-page-functions-no-route-astro-build/615656
- Accidentally committed secrets guide: https://dev.to/safvantsy/accidentally-committed-secrets-a-simple-git-fix-is-not-enough-3gem
- Dividend frequency annualization: https://fastercapital.com/content/Dividend-Yield--Yield-Yearnings--The-Strategy-of-Annualizing-Dividend-Yields-for-Investors.html
- STRK/STRC instrument overview: https://www.theblock.co/learn/394455/what-are-mstr-strk-and-strc-strategys-stock-and-preferred-shares-explained
- localStorage schema migration patterns: https://janmonschke.com/simple-frontend-data-migration/
- Securities disclaimer requirements: https://accountinginsights.org/investment-disclaimer-key-points-you-need-to-know/
- Twelve Data ToS (commercial vs personal): https://support.twelvedata.com/en/articles/5332349-commercial-and-personal-usage
- Special dividend detection (frequency=0): https://massive.com/docs/rest/stocks/corporate-actions/dividends

---
*Pitfalls research for: Niche preferred-stock yield dashboard (YieldGlance)*
*Researched: 2026-06-04*
