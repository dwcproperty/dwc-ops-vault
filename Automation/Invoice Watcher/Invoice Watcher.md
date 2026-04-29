# Invoice Watcher

Self-contained Node service that turns labelled inbound emails into LeadSimple **Acct Outstanding Invoices** deals.

- Server: `mcp.collincountyrent.com` (96.126.118.135)
- Path: `/root/dwc-mcp-servers/invoice-watcher/`
- PM2 name: `invoice-watcher` (id 14)
- Port: 3005 (`GET /health`)
- Logs: `/var/log/dwc/invoice-watcher.log` and PM2 (`/root/.pm2/logs/invoice-watcher-*.log`)
- Poll interval: 120s
- Built: 2026-04-29

## How it flows

1. You forward a vendor invoice into `office@dwcproperty.com` and add the label **DWC/Invoice-Inbound**.
2. Within 2 minutes the poller picks it up.
3. `parser.js` ships the PDF (or email body if no PDF) to Anthropic Sonnet with a strict JSON-only schema.
4. `leadsimple.js` finds-or-creates the vendor contact, finds the property by address, looks up the owner from related processes, and creates a deal in the **New Invoice** stage.
5. Custom fields are populated, vendor + owner attached, property linked.
6. Gmail label flips: `Inbound` → `Processed` (or `Review` if the property was ambiguous; `Failed` if anything threw).
7. `processed.json` records the Gmail message id so a re-poll can never duplicate.

## LeadSimple custom field map (23 fields)

Discovered 2026-04-29 from `/rest/process_types/982dd2b8-6391-46f5-820d-764aad18ebc9/custom_fields`. Auto-populated by the watcher are marked ✅; left blank for downstream automations are ⏳.

| Key | Type | Auto? | Source |
|---|---|---|---|
| `vendor_name` | text | ✅ | parsed `vendor_company` |
| `invoice_number` | text | ✅ | parsed `invoice_number` |
| `bill_date` | date | ✅ | parsed `invoice_date` |
| `due_date` | date | ✅ | parsed `due_date` |
| `service_period_start` | date | ✅ | parsed `service_period_start` |
| `service_period_end` | date | ✅ | parsed `service_period_end` (or `service_date` fallback) |
| `service_address` | text | ✅ | parsed `property_address.line_1` |
| `invoice_amount_due` | currency | ✅ | parsed `amount_due` |
| `account_number_at_vendor` | text | ✅ | parsed `account_number` |
| `source_email` | url | ✅ | constructed Gmail link |
| `raw_parsed_data` | text | ✅ | full Sonnet JSON |
| `tenant_due_date` | date | ⏳ | manual |
| `tenant_name` | text | ⏳ | future: derive from active lease |
| `utility_type` | choices | ⏳ | future: derive from vendor name |
| `responsibility` | choices | ⏳ | manual or future |
| `qbo_bill_id` | text | ⏳ | future: QuickBooks bridge |
| `rentvine_property_id` | number | ⏳ | future: bill-monitor handoff |
| `rentvine_lease_id` | text | ⏳ | future |
| `rentvine_charge_id_not_late_fee_id` | text | ⏳ | future |
| `rentvine_unit_id` | text | ⏳ | future |
| `rentvine_ledger_id` | text | ⏳ | future |
| `rentvine_transaction_id` | text | ⏳ | future |
| `utility_charge_amount_unpaid` | currency | ⏳ | future |

Source of truth lives at `/root/dwc-mcp-servers/invoice-watcher/leadsimple-field-map.json`. Run `npm run ls-probe` to detect drift between the local map and LeadSimple.

## Gmail label setup

Created 2026-04-29 in `office@dwcproperty.com`.

| Label | ID |
|---|---|
| `DWC/Invoice-Inbound` | `Label_4` |
| `DWC/Invoice-Processed` | `Label_5` |
| `DWC/Invoice-Failed` | `Label_6` |
| `DWC/Invoice-Review` | `Label_7` |

## Re-process a message

If a deal needs retrying (parser was off, you tweaked the spec, etc):

1. Remove the `DWC/Invoice-Processed` (or `-Failed` / `-Review`) label in Gmail. Re-add `DWC/Invoice-Inbound`.
2. SSH into the box: `ssh root@mcp.collincountyrent.com`.
3. Edit `/root/dwc-mcp-servers/invoice-watcher/processed.json` and delete the entry keyed by the Gmail message id.
4. `pm2 restart invoice-watcher` (or just wait up to 120s).

## Common failure modes

| Symptom | What to grep | Fix |
|---|---|---|
| Email keeps getting `Failed` | `'parse failed'` and `ANTHROPIC_KEY_MISSING` | Set `ANTHROPIC_API_KEY` in `.env` and restart. |
| Email keeps getting `Failed` (parser variants) | `'parse failed'` + `PARSE_FAIL` | Inspect raw response — Sonnet went off-script. Open the email and check it's actually an invoice. |
| Deal created but property blank | `'no property match'` | LS address mismatch. Update LS property or the parser-supplied address. |
| Deal in Review | `'multiple property matches'` or `'deal enrich failed'` | Open in LS, attach the right property by hand. |
| Repeated Gmail 401 | `'Gmail auth missing'` or `'invalid_grant'` | Token expired. Re-run `setup-oauth` in the **gmail-resolve-monitor** service — symlinks here will pick up the refresh. |
| LS 429 | `'LeadSimple GET'` / `'PUT'` `→ 429` | Rate limit. The next poll will catch up automatically. |

## BLOCK status (2026-04-29)

**Anthropic API key not provisioned on this box.**  
The service is up and idle; it boots, polls, label-queries successfully — but the moment a real invoice email is labelled, parsing throws `ANTHROPIC_KEY_MISSING` and the email gets routed to `DWC/Invoice-Failed`.

To unblock: paste a key into `/root/dwc-mcp-servers/invoice-watcher/.env` and `pm2 restart invoice-watcher`.

## Probe deal in LS

`9e9c6c5b-e9ba-4674-a5bf-d61d317e7fe8` — created during Phase 0 to verify writes. Named `TEST DELETE ME — Claude Code probe`. Safe to delete in the LS UI.

## Future work

- Tenant notification when a new invoice posts.
- Hook bill-monitor's Rentvine charge ids back into the matching invoice deal.
- Auto-derive `utility_type` from vendor name (Atmos → gas, water company → water, etc).
- QuickBooks bill creation and `qbo_bill_id` round-trip.
- Webhook trigger (Zapier or Gmail push) to drop poll latency from up-to-2-min to seconds.

## Related

- [[../Utility Bill Pipeline/]] — sister automation that handles utility-bill processes once a deal exists
- [[../Gmail Resolve/]] — donates the Gmail OAuth credentials this service reuses
