# 2026-05-02 — invoice-watcher 48h audit

**Window:** 2026-04-30T04:00Z → 2026-05-02T~18:00Z (audit run)
**Auditor:** Claude Code (automated MCP audit)
**Systems checked:** LeadSimple (process MCP), Rentvine (transaction/bill MCP), Gmail (office@dwcproperty.com)

---

## Summary

| Metric | Count |
|---|---|
| Total new Acct Outstanding Invoices processes | **8** |
| Created by watcher (have watcher comments) | **5** |
| Created manually (no watcher comments) | **3** |
| Happy path (New Invoice + all 3 Rentvine fields populated) | **0** ⚠️ |
| Auto-charge confirmed in Rentvine | **0** ⚠️ |
| Now in "New Invoice" stage | **0** (all moved forward by human) |
| Now in "Property Verification Needed" | **0** |
| Now in "Awaiting DWC Payment" | **2** (both manually created) |
| Now in "Paid" | **6** (5 watcher + Lone Star manually) |
| Notification emails (Invoice Watcher) received | **1** (pre-shipment catch-up only) |

> **⚠️ AUDIT LIMITATION:** The LeadSimple MCP `get_process` endpoint does **not** expose custom fields. The three Rentvine fields (`rentvine_charge_id_not_late_fee_id`, `rentvine_property_id`, `rentvine_ledger_id`) cannot be read directly via MCP. The conclusion that auto-charge failed is inferred from: (a) zero Airview AC bills in Rentvine transactions for Apr 30, (b) `list_bills` search for "Airview" returns empty, (c) no Rentvine IDs mentioned in watcher comments.

---

## Happy-path deals

**None.** All 5 watcher-created processes lack confirmed Rentvine bills. See "Failures requiring manual posting" below.

---

## Failures requiring manual posting

All 5 watcher-created processes are Airview AC invoices. The Rentvine auto-charge did not fire for any of them. Darrell manually reviewed and moved all 5 to "Paid" on 2026-05-01T09:56Z.

| Vendor | Address | Inv # | Amount | LS Process ID | Gmail Src | What's Missing |
|---|---|---|---|---|---|---|
| Airview AC | 602 Carver St, Whitesboro TX | 56260 | $99.00 | `07c3f62b` | `19dd9bc8757daf4f` | No Rentvine bill; Company Owed role blank (API limit) |
| Airview AC | 105 Commerce St, Savoy TX | 56256 | $99.00 | `6b753f3f` | `19dd4aebe125b5f7` | No Rentvine bill; Company Owed role blank (API limit) |
| Airview AC | 609 E 1st St, Bonham TX | 56262 | $99.00 | `a10f5171` | `19dd46fa3eef6b9c` | No Rentvine bill; Company Owed role blank; watcher flagged "no property attached" at creation |
| Airview AC | 421 Ridgeview Rd, Sherman TX | 56257 | $198.00 | `31e0e038` | `19dbc42ea865c0e1` | No Rentvine bill; Company Owed role blank (API limit) |
| Airview AC | 192 Diamond Ranch Rd, Whitesboro TX | 56252 | $99.00 | `7dc22769` | `19dda22a9c8c08a7` | No Rentvine bill; Company Owed role blank; watcher flagged "no property attached" at creation |

**Total unposted Airview AC invoices: $594.00**

All these processes were created 2026-04-30T04:46–04:48Z (16–18 min after go-live), meaning they were catch-up processing of pre-existing labeled Gmail messages. All carry the watcher comment: *"NOTE: vendor 'Airview AC' must be set on Company Owed role manually via LS UI — REST API does not support this role."*

Root cause of Rentvine failure: **Airview AC has zero bills in Rentvine** (confirmed via `list_bills` search). The vendor is likely not registered in Rentvine, so the bill POST either failed at vendor lookup or was skipped. No failure notification emails were sent to office@dwcproperty.com for any of these 5 cases — this is a **notification gap** (spec requires notification on any Rentvine step failure).

---

## Property Verification Needed (manual review)

**None found in this stage.** Two watcher processes noted "no property attached" at creation time (192 Diamond Ranch and 609 E 1st St), but both have properties linked now — either the watcher used a fuzzy match and placed them in New Invoice anyway (inconsistent with spec), or a human attached properties after the fact. Either way, they bypassed the Property Verification Needed routing.

---

## Manually-created processes (not watcher output — for context)

| Vendor | Address | LS Process ID | Created | Stage | Evidence it's manual |
|---|---|---|---|---|---|
| Lone Star Seamless Gutters | 602 Carver St, Whitesboro TX | `63e1c501` | 04-30 05:32Z | Paid | Empty comments; Company Owed role IS set (watcher can't do this via API) |
| City of Princeton | 1208 Mesquite Lane, Princeton TX | `358f8dad` | 05-01 06:56Z | Awaiting DWC Payment | Empty comments |
| Mustang Water Supply | 3132 Burmese St, Providence Village TX | `bd6affbf` | 05-01 13:34Z | Awaiting DWC Payment | Null comments |

Note: Lone Star Seamless Gutters has a corresponding Rentvine transaction — bill #733 "Gutter Installation" $1,343.00 on 2026-05-01, ledger Thurstan 100 LLC — which appears to have been manually posted by Darrell.

---

## Notification emails

| Subject | Message ID | Date | Mapped to |
|---|---|---|---|
| [Invoice Watcher] Review needed (no LS deal): Portal | `19ddcde714882170` | 2026-04-30T05:30Z | **No LS deal (orphan)** — pre-shipment catch-up notification; source email from `calydriver@gmail.com` (Darrell's personal Gmail), subject "Portal"; parser couldn't identify vendor; this is pre-watcher-launch backlog, not a live failure |

**No notification emails were found for the 5 Airview AC Rentvine posting failures.** This is a spec violation — the watcher should send office@dwcproperty.com a notification naming the error when any Rentvine step fails.

**Additional observation:** A new Airview AC invoice arrived at 2026-05-01T23:30Z (Invoice #56385, $215.00, subject "Invoice 56385 due from Airview AC - $215.00"). No LS process was visible for it at audit time. This may still be in the watcher's next polling cycle or may require manual handling.

---

## Anomalies

### 1. ⚠️ Rentvine auto-charge silently failed for all 5 watcher-created processes — no notifications sent
Per spec, any Rentvine step failure should trigger a notification email to office@dwcproperty.com. None received. The likely cause is that Airview AC is not in Rentvine as a vendor (zero bills in Rentvine for Airview AC ever). Either the watcher silently swallowed the error, or it has a "skip Rentvine if vendor not found" path that doesn't send notifications. This needs to be debugged — the notification path is the safety net.

### 2. ⚠️ LeadSimple Company Owed role cannot be set via REST API
All 5 watcher-created processes have empty Company Owed role. The watcher correctly flags this with a note in comments but cannot resolve it programmatically. Every Airview AC invoice requires a manual step to link the vendor in LS.

### 3. ⚠️ Two processes flagged "no property attached" but landed in New Invoice, not Property Verification Needed
Per spec, `no match or ambiguous → Property Verification Needed stage`. But 192 Diamond Ranch and 609 E 1st St were created with a "no property attached" warning and appear to have properties linked now. Possible causes: (a) watcher fuzzy-matched and placed in New Invoice anyway (incorrect routing), or (b) properties attached by human after creation. Watcher routing logic for partial matches should be reviewed.

### 4. ℹ️ MCP audit gap: LS custom fields not exposed
`rentvine_charge_id_not_late_fee_id`, `rentvine_property_id`, `rentvine_ledger_id` are not returned by the `get_process` or `list_processes` MCP endpoints. Future audits would benefit from a direct LS API call or a watcher-side logging mechanism that embeds Rentvine IDs in the process `comments` field.

### 5. ℹ️ All watcher activity was a single vendor (Airview AC)
The entire first 48h of watcher processing was one vendor batch. This limits confidence that the happy path (vendor-in-Rentvine, clean property match) has been tested end-to-end. No `New Invoice` → Rentvine bill path was exercised.

### 6. ℹ️ New Airview AC invoice (#56385, $215) arrived near end of audit window
Received 2026-05-01T23:30Z. No LS process visible at audit time. Watcher may process it in next cycle, but same Rentvine failure will occur unless Airview AC is added to Rentvine first.

---

## Verdict

The first 48 hours show a **partially functioning watcher** — the Gmail polling, invoice parsing, and LeadSimple process creation all worked correctly. The watcher correctly parsed 5 Airview AC invoices with high confidence and created well-structured processes with source attribution within minutes of launch. However, **the Rentvine auto-charge component did not fire for a single invoice**, and critically, **no failure notifications were sent**, meaning the spec's failure-alerting safety net is broken. The root cause is that Airview AC is not registered as a vendor in Rentvine, so the bill POST has no vendor to reference — but the watcher swallowed this error silently rather than notifying office@dwcproperty.com. The entire first-48h batch was essentially manual: Darrell reviewed and closed all 5 watcher processes himself without any automated Rentvine charge being posted. Until Airview AC is added to Rentvine as a vendor, and until the silent-failure notification bug is fixed, the automation's core value proposition (removing the manual Rentvine posting step) is not being realized. Next test should be a non-Airview-AC invoice from a vendor that IS in Rentvine to validate the full happy path.
