# Owners Dashboard — full build session recap

> **Session date:** 2026-05-20
> **What landed:** `owners.collincountyrent.com` is live as the owner-facing performance dashboard. Service at `/root/dwc-mcp-servers/owners-tracker/` on the Linode, port 3018, PM2 process `owners-tracker`, nginx vhost + Let's Encrypt cert in place, daily 4 AM Central cron syncing from Rentvine.

## What an owner sees when they log in

Email-only magic link auth (no passwords). Owner enters their email on `/login` → Gmail API sends a JWT-signed link from `info@dwcproperty.com` → click lands them in their dashboard with a 24-hour session.

Pages they have access to:

- **Portfolio** (`/portfolio`) — composite score across their book, monthly + annual cash flow, occupancy rate, per-property cards in a grid, "you vs DWC book average" comparison bars
- **Property detail** (`/property/:id`) — 9 sections: hero with composite, score radar (Chart.js), TTM financials (gross rent / opex / capex / net), Plano-area Realtor rent comps, comparable DWC homes (city-only, never addresses, viewer's home highlighted at its score rank), property value waterfall (4 sources), refinance flag (when rate delta ≥ 0.75%), capex 5-year outlook, tenant snapshot, work-order activity log, AI-generated "About this property" narrative (~350-500 words, refreshes monthly), and a floating AI chat box (Claude Sonnet 4)
- **Leaderboard** (`/leaderboard`) — top 10 anonymized "Property #N" + viewer's own rank highlighted in DWC orange. Opt-out per owner removes their properties entirely.
- **Reports** (`/reports`) — annual PDF (Playwright headless, 8.5×11 letter, ~100 pages for the master report or per-property single-home reports). Suitable for forwarding to a CPA.
- **Settings** (`/settings`) — leaderboard opt-in toggle, monthly email opt-in toggle (default off, Darrell enables per VIP later), per-property purchase price, mortgage details (unlocks refinance flag), owner value override, CAD assessed value (from owner's tax bill), capex item overrides (install year + custom life)

## The DWC Performance Score

7 weighted components, each 0-100, weighted-average to a composite. Weights locked in `src/config/score-weights.json`:

| Component | Weight | What it measures |
|---|---|---|
| Net yield | 25 | TTM net (gross rent − opex) / property value, normalized 4%=50 / 10%=100 |
| Occupancy | 18 | Days occupied last 365 / 365 |
| Rent vs market | 14 | Vs DWC portfolio median for same beds, ±0.5 baths, ±20% sqft. Fallback: DFW benchmark. Capped at 110. |
| Maintenance burden | 14 | Tiered: 24-month non-cancelled WO count + vendor-bill spend. Each scored 0-100, averaged. |
| Condition | 10 | Current-state penalty: -5 per WO 30-60d, -10 per 60d+, -10 per high-priority open |
| Rentability fit | 10 | Distance from 3/2/2500 ideal. -10/bed off, -8/bath off, -1/100sf off |
| Tenant quality | 9 | Late-fee-charge-based (not payment-date) with 5-day grace. Renewal probability from lease end date. |

### Hard-won classifier rules

The classifier ran through several iterations before the numbers were trustworthy. These edge cases now have explicit handling in `src/sync/rentvine-transactions.js`:

- **"Rent (MM-YYYY)" type-1 entries** are *tenant charges*, NOT vendor bills. Required because rent invoices were getting flagged capex (any type-1 ≥ $2k was capex; rent is $2k+).
- **"Affordable Housing (MM-YYYY)"** = government-assisted rent, GL account 4105 in Rentvine. Same treatment as standard rent. The classifier also recognizes Section 8, HAP, voucher, subsidized — all in one regex `RENT_CHARGE_DESC_PREFIXES`.
- **Type-7 ledger charges** that say "Resident Benefit Package", "Tenant Protection Program", "Pet Rent", "Utility Convenience", "Application Fee", "Move-in Fee" are tenant *allocations*, NOT owner expenses. These flow through DWC and don't come out of the owner's net.
- **Cancelled work orders** are flagged by `cancelledByUserID` (not `dateClosed`). 27 of the WOs in the DB had been mis-counted as open before this fix.
- **Bathroom data** lives in `unit.fullBaths` + `unit.halfBaths` (NOT `bathrooms` / `baths`). Total baths = full + 0.5 × half.
- **Square footage** is in `unit.size` (NOT `sqft` / `squareFeet`).
- **Rent payment** (type-2) descriptions are unreliable for "is this rent?" — many are blank or generic ("Resident Portal Payment"). For gross rent we use the CHARGES side (type-1 rent invoices), not the payments side.

## Suppressed-fee policy (privacy)

The dashboard never exposes:

- Management fees, leasing fees, vendor markups, owner-draw splits, "what DWC makes"
- Late fee dollar totals (count of late months OK; dollar amount NEVER — late fees go to DWC)
- Any other property's name, address, or owner identity (only aggregate medians or city as a comparison reference)

Enforcement is three layers:

1. **System prompt rules** in `src/ai/system-prompt.txt` — Claude is told these are non-negotiable
2. **Tool-layer scoping** in `src/ai/tools.js` — every tool validates the requested `property_id` belongs to the session's owner before returning data. Owner ID comes from the session cookie, never from the LLM input
3. **Pre/post regex filters** in `src/ai/sanitizer.js` — questions matching forbidden terms are short-circuited before the LLM call; responses are post-filtered as belt-and-suspenders

## AI features

Two distinct surfaces, both Sonnet 4, both using the existing `ANTHROPIC_API_KEY` from `invoice-watcher/.env`:

### In-app chat (`/api/ask`)

- Floating button bottom-right on property and portfolio pages
- Owner-scoped tool use: 8 read-only tools (get_property_summary, get_score_explanation, get_recent_work_orders, get_financials_summary, get_capex_outlook, get_market_context, get_comparable_dwc_homes, list_owner_portfolio)
- Rate-limited 20/day, 200/month per owner; audit log in `chat_messages` table
- Cost: ~$0.005 per question (with prompt caching)

### Monthly property narratives

- 350-500 word personalized report under "About this property"
- Replaces a generic static blurb. Reads gross rent, opex, score components, capex outlook, comparables — writes a paragraph-form report in DWC voice
- Cached in `property_narratives` table, refreshed monthly via `src/jobs/monthly-narratives.js` cron (1st of month, 5 AM Central)
- Cost: ~$0.012 per property per month → **~$7.12/year for all 53 properties**

## Data sources

- **Rentvine REST** — owners, properties, units, leases, work orders, accounting/transactions. Pattern copied from `insurance-audit/rentvine.js`. Same credentials (`RENTVINE_ACCESS_KEY` / `RENTVINE_SECRET`).
- **Realtor.com via RapidAPI** (`realtor-search.p.rapidapi.com`) — two uses:
  - City-level rent comps (median/p25/p75 for "Plano 4-bed" etc.) → `city_rent_samples` table, refreshed via daily-sync
  - Per-property valuations (Cotality™/Quantarium/Collateral Analytics estimates) → `properties.realtor_com_value`, refreshed via daily-sync, 30-day TTL. 51 of 53 properties currently have a value.
- **Freddie Mac PMMS** (FRED CSV mirror) — weekly 30-year mortgage rate for refinance flags. Endpoint currently slow/intermittent from this Linode; falls back to a static benchmark of 6.85% in `data/benchmarks-dfw-2026q2.json`.
- **CAD assessed value** — NOT auto-scraped. Owners enter from their annual property tax bill via Settings. Per-county scrapers (Collin / Denton / Dallas / Kaufman / Grayson / Tarrant / Fannin) deferred — too many distinct sites + anti-bot risk.
- **`finances-tracker/data/finances.db`** — referenced in plan but ultimately unused. finances.db is company-level only; everything per-property comes from Rentvine.

## Operational stuff

- **Auth allowlist:** `office@dwcproperty.com` is the only admin login. (`darrell@dwcproperty.com` doesn't exist — that email was a mistake in the original spec.) Owners use whatever email Rentvine has on file for them.
- **Email sending:** Gmail API (NOT SMTP — outbound SMTP is blocked at the Linode level). OAuth tokens for `info@dwcproperty.com` are symlinked into `auth/` from `gmail-resolve-monitor/auth/`. The SMTP app password Darrell handed me (`spfu gxxf qxml vmsr`) is documented in `.env.example` but unused unless the firewall opens up.
- **Cron entries** (root crontab):
  ```
  0 10 * * *   cd /root/dwc-mcp-servers/owners-tracker && node src/jobs/daily-sync.js
  0 14 * * 1   cd /root/dwc-mcp-servers/owners-tracker && node src/jobs/weekly-rate-pull.js
  0 11 1 * *   cd /root/dwc-mcp-servers/owners-tracker && node src/jobs/monthly-narratives.js
  ```
- **PM2:** `owners-tracker`, max_memory_restart 350M, logs at `/root/dwc-mcp-servers/owners-tracker/logs/{out,error}.log`
- **Logos:** copied from `/root/dwc-mcp-servers/inspections-portal/public/img/` — `dwc-logo.png` + `dwc-logo-2x.png`. White-filtered for the nav, native colors on the login card.

## Bugs hunted and fixed during this session

1. **WO 100021 false-open** — cancelled WOs were being counted as open because `cancelledByUserID` was set but `dateClosed` was null. Status mapper now treats `cancelledByUserID` as the cancellation signal. 27 WOs reclassified.
2. **Paula Keylor scored as a chronic-late tenant when she's the best on the book** — she pays before the 1st every month, but Rentvine `datePosted` reads the cleared date, which often falls 2-3 days into the month. Switched the tenant-quality signal from posting date to *late-fee charges* on her ledger. Real chronic-late tenants are now correctly the bottom of the list.
3. **5750 Salisbury (DHA tenant) scored composite 61.7 with $0 net yield** — its "Affordable Housing (MM-YYYY)" charges were being booked as operating expenses (because the classifier didn't know they're rent). Fixed; Salisbury jumped to composite 87.3 with proper net yield.
4. **Lottie had $32k in "capex"** — those were rent invoices over $2k. Classifier now requires capex descriptions to also match a maintenance-language regex.
5. **Gross rent was $176k overstated portfolio-wide** — counted all type-2 payments (which include TBP add-ons, security deposit transfers, money-order deposits). Switched to charges-side only (`Rent (` + `Affordable Housing (` prefixes).
6. **Multi-unit "2711 Kimsey Drive" duplicates** — Rentvine has 8 unit rows for the same physical 4-unit complex (units 38-41 AND duplicate property records 68-71). Darrell tried to deactivate the duplicates; the deactivation didn't take in Rentvine (`isActive: "1"`, `dateTimeDeactivated: null`). Documented; cleanup pending in Rentvine UI.

## Parked / future

- **Property Meld integration** — designed in plan file, parked until Darrell has API access. Full runbook lives at `/root/.claude/plans/dwc-owner-dashboard-keen-popcorn.md` (Appendix). Will replace the Rentvine-WO-based maintenance score with a richer 4-signal model when activated.
- **VIP invite cohort (M14)** — Darrell asked to test with a dummy account first before sending to any real owner. M13.5 dummy-account smoke test is implicitly done via office@ live testing.
- **Bulk invite (M15)** — gated on M14 feedback.

## Where to look when something's off

- **A score looks wrong**: read `score_notes` column on the property row — it has the JSON of inputs (`ttm_income`, `ttm_opex`, `ttm_capex`, `value_source`, `property_value`, `rent_market_source`). Cross-check against `transactions` table.
- **A narrative is stale or missing**: query `property_narratives` for the property_id; `generated_at` older than 30 days triggers async regen on next page view. Or force one with `node -e "require('./src/ai/narrative').generateNarrative(db.prepare('SELECT * FROM properties WHERE id=?').get(N))"`.
- **The chat refuses something it shouldn't**: check `src/ai/sanitizer.js` `FORBIDDEN_RE`. The post-filter is the most likely culprit.
- **Realtor value is missing for a property**: run `node -e "require('./src/sync/realtor-property-value').refreshProperty(db, prop)"` or just wait for the next daily-sync.
- **A new affordable-housing program label shows up**: extend `RENT_CHARGE_DESC_PREFIXES` in `src/sync/rentvine-transactions.js` and run the in-place reclassification SQL from the plan file (Appendix).
