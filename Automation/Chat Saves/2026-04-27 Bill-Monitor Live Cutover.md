# bill-monitor — flipped to live

Session date: 2026-04-27 (~07:55–08:10 UTC)

## TL;DR

Flipped `bill-monitor` from `DRY_RUN=true` to `DRY_RUN=false`. First real LeadSimple stage advancement landed at **07:59:55 UTC** — the "1208 Mesquite Lane (Apr)" utility-bill process moved to **Tenant Paid** automatically when its Rentvine charge cleared. No errors, no alerts, monitor process healthy.

## What this means in practice

The manual step that's now gone: opening LeadSimple, finding the right Acct Utility Bills process, advancing the stage by hand.

The manual step that **stays**: recording the tenant payment in the Rentvine UI. That's the trigger event — bill-monitor just watches for the charge to leave `unpaidCharges` and then advances the LS stage.

Workflow:

1. Tenant pays you
2. **You** record the payment in Rentvine UI ← still manual
3. Rentvine charge moves out of unpaidCharges
4. bill-monitor detects → moves LS process to "Tenant Paid" (concierge) or "Paid" (owner) ← **THIS is what got automated**

If a process doesn't advance: first thing to check is whether the Rentvine charge was actually marked paid. That's the gating event, not anything bill-monitor controls.

## Current state

- **Process:** `node /root/dwc-mcp-servers/bill-monitor/index.js` (PID 246191, alive)
- **Mode:** `DRY_RUN=false`
- **Cadence:** `*/30 * * * *` — fires on the :00 and :30 of every hour
- **State file:** `/root/dwc-mcp-servers/bill-monitor/state.json`
- **Logs:** `dwc-ops` vault → `Automation/Utility Bill Pipeline/Bill Monitor Log.md`

## Tracked entries at cutover

**Concierge side (7 utility-bill processes registered):**

| Lease | Bill | Amount | Status |
|---|---|---|---|
| 40 | 3132 Burmese St | $231.80 | Closed in "Billing Complete" (pre-flip) |
| 40 | 1208 Mesquite Lane (Apr) | $115.31 | **Advanced live at 07:59:55** ← first live move |
| 40 | 1208 Mesquite Lane | $231.80 | Pending (charge unpaid) |
| 32 | 3132 Burmese St | $62.82 | Pending |
| 31 | 736 Sterling Drive | $94.57 | Pending |
| 33 | 2025 Belgian Drive (Aubrey) | $149.49 | Pending |
| 33 | 2025 Belgian Drive (TXU) | $201.50 | Pending |

**Owner side (3 outstanding-invoice processes registered):**

| Bill ID | Vendor / property | Amount | Status |
|---|---|---|---|
| 725 | Atmos Energy — 1517 Kinglet Place | $231.80 | Pending |
| 729 | City of Celina — 1517 Kinglet Place | $148.04 | Pending |
| 728 | Grayson County Electric Coop — 1517 Kinglet Place | $46.43 | Pending |

All 9 pending entries have `consecutive_errors: 0` and `last_amount_unpaid` matching the bill amount — system is correctly holding because the Rentvine charges haven't cleared.

## Config locked in

```json
{
  "polling_cron": "*/30 * * * *",
  "concierge": {
    "process_type_name": "Acct Utility Bills",
    "target_stage_name": "Tenant Paid"
  },
  "owner": {
    "process_type_name": "Acct Outstanding Invoices",
    "target_stage_name": "Paid"
  },
  "rentvine_payment_type_id": "8",
  "rentvine_call_delay_ms": 200,
  "alert_after_consecutive_errors": 3
}
```

## What to expect

Next time you record a tenant utility payment in Rentvine, within ≤30 minutes the matching LS process should auto-advance. Check `state.json` if you want to verify a specific entry — `advanced_at` going from `null` to a timestamp is the signal. Alerts fire after 3 consecutive errors on the same entry.
