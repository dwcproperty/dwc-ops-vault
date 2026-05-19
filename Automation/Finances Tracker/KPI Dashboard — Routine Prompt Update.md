# KPI Dashboard — Routine Prompt Update Needed

**Updated:** 2026-05-19
**Status:** Code deployed, dashboard live with backfilled data. **Action item:** update the remote routine prompt so future daily syncs populate `revenue_by_category` AND `owners_draw` per-month.

## What's live at finances.collincountyrent.com/kpi

- YTD totals: Gross Revenue, Total Expenses, Net Income, Owner Pay, Retained
- Monthly P&L table (trailing 12 + current): per-month gross / expenses / net / owner pay / retained, plus a TTM footer row
- RPD family: All-PM RPD, Mgmt-Fee-Only RPD, Profit/door, Cost/door
- Operations KPIs: Vacancy, delinquency, avg rent, doors/FTE, renewal rate, WO resolution time, expiring 60d
- Trend charts: doors, RPD monthly, revenue mix stacked, collection milestones

**Current TTM totals (12mo through 2026-05-19):**
- Gross: $83,644 · Expenses: $69,539 · Net: $13,743 · Owner Pay: $33,699 · Retained: −$20k
- All-PM RPD: $1,707/door (calibrating — see note below)

## Action: update the remote routine prompt

The daily 12:03 UTC routine at https://claude.ai/code/routines/trig_01BWxsrv83iGoiRSpnjGe84x currently POSTs basic snapshot fields. It needs to also pull (a) per-account income breakdown and (b) per-month owner's draw and include them in the POST body.

### Replace the routine prompt with this

```
Daily DWC finances sync. You will:

1. Call `mcp__claude_ai_Intuit_QuickBooks__profit-loss-quickbooks-account` with
   periodStart = first day of fiscal year (2026-01-01)
   periodEnd = today (YYYY-MM-DD)

2. Call `mcp__claude_ai_Intuit_QuickBooks__qbo_accounting_get_balance_sheet` with
   start_date = first day of trailing 13 months back from today (e.g. 2025-04-30)
   end_date = today
   split_by = "MONTH"
   This gives end-of-month cumulative balances on every equity account, including
   "Owner's Draw" — the row to find is exactly that name.

3. From the P&L response, for each entry in monthlyBreakdown, extract:
   - period (YYYY-MM derived from the range key like "2026-04-01 - 2026-04-30")
   - period_start, period_end
   - revenue = totalIncome
   - expenses = totalExpenses
   - net_income = netIncome
   - revenue_by_category = list of { account_name, amount } from incomeAccounts, EXCLUDING:
     * keys starting with "Total for "
     * group headers ("Income", "Residential PM Income", "Cost of Goods Sold", etc.)
     * accounts ending in "(deleted)"
     * accounts with amount = 0

4. From the balance sheet response, find the row where cells[0].value == "Owner's Draw".
   For each month M starting from the second column onward:
     owner_draw_for_month_M = abs( balance_at_M − balance_at_previous_month )
   Attach this as `owners_draw` to the corresponding monthly snapshot from step 3.

5. Also build a YTD snapshot:
   - period = "YTD"
   - period_start = 2026-01-01
   - period_end = today
   - revenue/expenses/net_income from summaryBreakdown
   - owners_draw = sum of monthly owner draws for periods >= 2026-01
   - revenue_by_category = aggregated across YTD months

6. POST to https://finances.collincountyrent.com/ingest:
   Headers: Authorization: Bearer <INGEST_TOKEN from secrets>
   Body:
   {
     "snapshots": [
       { period, period_start, period_end, revenue, expenses, net_income, owners_draw, revenue_by_category },
       ... one entry per month + one for YTD
     ]
   }

7. Confirm response is 200 with accepted = [list of periods]. Report back the count.
```

### Why this works

- `/ingest` classifies each `revenue_by_category` entry into a bucket via `src/config/revenue_buckets.json` and stores in `finances_revenue_categories`.
- `owners_draw` writes directly to `finances_snapshots.owners_draw` for that period.
- Both feed the new `/kpi` page directly.

## Backfill already done

Today (2026-05-19) the trailing 12 months were backfilled manually:

- **Per-account revenue** via `src/sync/backfill.js` — reconciles to the cent for all 13 months.
- **Monthly owner's draw** via `src/sync/backfill_owners_draw.js` — uses a hardcoded snapshot of the balance sheet pulled today. To refresh historicals later, re-run after updating the `cumulativeByMonth` map in that script with a fresh balance-sheet pull.

## Verification done

✓ Per-account income sums == P&L total income for all 13 months (after excluding double-posted "(deleted)" legacy accounts)
✓ Door count (49) matches your Rentvine UI manual count
✓ Owner pay YTD ($13,562) matches QBO balance-sheet diff of "Owner's Draw" from Dec 31 to today
✓ Rentvine→QBO sync: vendor bills, owner disbursements, and mgmt-fee charges all flow through QBO P&L. No separate Rentvine ledger pull needed.

## RPD calibration caveat (will self-correct)

Door snapshot history starts 2026-05-19. Until ~6 months of daily door snapshots accumulate, RPD uses today's door count (49) as the denominator. DWC grew from ~25 doors a year ago → 49 today, so dividing TTM revenue by 49 understates RPD. Steady-state estimate: ~$2,100/door for mgmt-only (matches 8% × $2,159 avg rent × 12). Dashboard shows a yellow "⚠ calibrating" note on the hero card until 6 months of snapshots accumulate.

## Optional v2 KPIs (not yet tracked)

- Owner retention rate
- Average days vacant (DOM)
- Application conversion rate
- Maintenance margin %
- NPS / tenant satisfaction
- Eviction rate (see processes.collincountyrent.com)
