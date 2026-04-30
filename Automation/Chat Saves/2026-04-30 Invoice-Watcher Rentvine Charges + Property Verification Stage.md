# 2026-04-30 — Invoice-Watcher: Rentvine Charges + Property Verification Stage

**Status:** Live in production. Stopped mid-session to take a call. Pick up here.

## What shipped this session

### 1. Property Verification Needed stage (LS)
- New stage added in LeadSimple UI on Outstanding Invoices process type.
- UUID: `7fe085ca-8d28-4d98-807c-a32c499901de`
- Wired via `PROPERTY_VERIFICATION_STAGE_ID` env var in `/root/dwc-mcp-servers/invoice-watcher/.env`.
- When a deal is created with no property match OR ambiguous match → lands in this stage instead of New Invoice. Stage's first step (assigned to Darrell) drives manual property entry.
- Two pre-existing review-status deals still in New Invoice — need to be moved to the new stage **manually in LS UI** (PUT can't transition stages):
  - Airview AC - 609 E 1st St (`a10f5171-d0fb-4d2f-9c3a-df3fd0dc6542`)
  - Airview AC - 192 Diamond Rnch Rd (`7dc22769-0cb8-4a57-9a84-21654998ee48`)

### 2. Notification email when review without LS deal (path A)
- Path A = parser flagged `no_vendor` or `low_confidence` → email tagged Review but no LS deal exists.
- Now sends a Gmail notification to `office@dwcproperty.com` (configurable via `NOTIFY_EMAIL` env var).
- `gmail.modify` scope already covers `messages.send` — no re-auth needed.
- Catch-up notification was sent for one pre-existing case: message `19ddc9b7a41a9b26` from Darrell Calhoun, subject "Portal" — almost certainly NOT an invoice (mislabelled), confirms the conservative gate is doing its job.

### 3. Rentvine owner-ledger charge auto-post
- After successful LS deal create with single property match: invoice-watcher now posts a charge to Rentvine.
- **Endpoint:** `POST /api/manager/accounting/bills` (the MCP `create_bill` tool's underlying endpoint — accepts any ledgerID despite the misleading "late fees, NSF fees" description).
- **Body shape:**
  ```json
  {
    "billDate": "YYYY-MM-DD",
    "dateDue": "YYYY-MM-DD",
    "description": "<vendor> — <property address>",
    "charges": [{ "ledgerID": <int>, "chargeAccountID": 71, "amount": "0.00", "description": "..." }]
  }
  ```
- **Charge account:** 71 = "Contractor" (Rentvine account number 6450). Set via `RENTVINE_CONTRACTOR_ACCOUNT_ID` env (default 71).
- **Ledger lookup:** Rentvine has no `/ledgers` endpoint accessible to this token. Workaround: portfolio→primary-owner-ledger discovered by walking `/accounting/transactions/search` and reading any historical txn's `primaryLedgerID`. Cached on disk in `/root/dwc-mcp-servers/invoice-watcher/ledger-cache.json`. Already cached: portfolio 28 (Thurstan 100 LLC) → ledger 77.
- **LS write-back:** on successful charge, three custom fields are PUT back onto the LS deal:
  - `rentvine_charge_id_not_late_fee_id` ← bill ID from Rentvine response
  - `rentvine_property_id`
  - `rentvine_ledger_id`
- **Failure mode (any step fails):** LS deal stays put, Rentvine fields blank, email notification sent to office@dwcproperty.com naming the error and showing the action ("post the charge manually, paste the bill ID into the Rentvine Charge ID field").
- **Skip cases:** no property match (review path), or amount ≤ 0.

## Verification status

- All three changes deployed via `pm2 reload invoice-watcher --update-env` (PID 941751 at session end).
- Read-only end-to-end test passed: Rentvine property lookup, portfolio→ledger discovery, cache write all confirmed working against live Rentvine.
- **Not yet verified live:** the actual Rentvine POST. The first inbound invoice with a property match will trigger the first real charge. First run will log the full Rentvine response (`raw: ...`) so we can verify the `billID` field name extraction is correct. If `rentvine_charge_id_not_late_fee_id` shows up empty on the LS deal after a successful charge, the response field name is different than expected — narrow the extractor.

## Pick-up items when resuming

1. **Move the two pre-existing review deals to Property Verification Needed stage manually in LS UI** (links above).
2. **Watch the first live Rentvine charge.** When the next vendor invoice email comes in:
   - Check `pm2 logs invoice-watcher` for `rentvine charge: about to post` and `rentvine charge: posted {raw: ...}`.
   - Confirm the LS deal got `rentvine_charge_id_not_late_fee_id` populated (visible in LS UI).
   - Verify the charge actually appears on the owner ledger in Rentvine.
3. **Optional follow-up offered but not yet scheduled:** /schedule a 24-48hr audit agent that reads pm2 logs + LS deals + Rentvine transactions and confirms everything tied up cleanly across the first few real charges.

## Key file paths touched

- `/root/dwc-mcp-servers/invoice-watcher/src/rentvine.js` (new)
- `/root/dwc-mcp-servers/invoice-watcher/src/pipeline.js` (Rentvine integration + notify-on-no-deal-review + property-verification stage routing)
- `/root/dwc-mcp-servers/invoice-watcher/src/config.js` (rentvine block, propertyVerificationStageId, notifyEmail)
- `/root/dwc-mcp-servers/invoice-watcher/src/leadsimple.js` (`createOutstandingInvoiceDeal` accepts `stageId` override)
- `/root/dwc-mcp-servers/invoice-watcher/src/gmail.js` (`sendEmail` added)
- `/root/dwc-mcp-servers/invoice-watcher/.env` + `.env.example` (PROPERTY_VERIFICATION_STAGE_ID, NOTIFY_EMAIL, RENTVINE_*)
- `/root/dwc-mcp-servers/invoice-watcher/ledger-cache.json` (new — portfolio→owner ledger cache)
- Deleted: `/root/dwc-mcp-servers/invoice-watcher/scripts/backfill-natural-links.js` (one-shot fix already in code)

## New auto-memory written this session

- Updated `reference_leadsimple_api_quirks.md`: stage_id can NOT be transitioned via PUT, only set at create time. Added to the "create-only" list alongside property_ids and contact_roles_attributes. MEMORY.md index hook updated.
