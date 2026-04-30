# 2026-04-30 — Utility Bills Zap: PDF/HTML Parser Fix

**Status:** Investigation complete. No code changes shipped yet. User stepped away mid-session — pick up here.

## What the user asked for

The Gmail Zap that triggers on label `Utility Bills` failed on the last two **Princeton** utility bills because its parser cannot read PDFs. They want the parser to read PDFs and/or HTML so it pulls the **account number** (and other fields).

## What I found

### 1. The parser capability already exists — the Zap just isn't using it

`/root/dwc-mcp-servers/webhook/index.js` already has a `/webhook/parse-invoice-pdf` route (port 3000) that:
- Mode 1: takes `{message_id, account_email}` → fetches Gmail message, walks parts to gather body text + collect PDF attachments, picks the largest PDF, sends both PDF + email body to `/root/dwc-mcp-servers/invoice-watcher/src/parser.js` (Sonnet, temp 0).
- Mode 2: takes `{pdf_base64?, html_body?, subject?, from?}` for inline content.
- On-disk cache at `/root/dwc-mcp-servers/webhook/parse-cache.json` keyed by `(account_email, message_id)` — guarantees one billable parse per email.
- Returns the schema in `parser.js` `SYSTEM_PROMPT`: `vendor_company, invoice_number, service_date, service_period_start/end, invoice_date, due_date, property_address {line_1, city, state, postal_code}, amount_due, account_number, confidence, notes`.

### 2. Tested live against the two recent Princeton emails

```bash
curl -sS -X POST http://localhost:3000/webhook/parse-invoice-pdf \
  -H "Content-Type: application/json" \
  -H "x-webhook-secret: $WEBHOOK_SHARED_SECRET" \
  -d '{"message_id":"<id>","account_email":"office@dwcproperty.com"}'
```

Webhook secret is in `/root/dwc-mcp-servers/webhook/.env` as `WEBHOOK_SHARED_SECRET`.

| Thread / Message ID | Sender | Format | Parser result |
|---|---|---|---|
| `19db561f0ff325d3` | info@princetontx.us | PDF (`UtilityBill.pdf`) | ✅ high — City of Princeton, account `41-6643-02`, **1208 Mesquite Lane**, $143.82, due 2026-05-10 |
| `19db5620675dc349` | info@princetontx.us | PDF (`UtilityBill.pdf`) | ✅ high — City of Princeton, account `47-3305-01`, **834 Ponderosa Lane**, $116.89, due 2026-05-10 |
| `19db7c01e80e5e0b` | noreply@municipalonlinepayments.com | HTML reminder, **two accounts in one body** | ⚠️ low — parser flagged: "Email shows two separate utility accounts: 47-3305-01 / 834 Ponderosa ($116.89) and 41-6643-02 / 1208 Mesquite ($143.82). Total $260.71. Unable to determine which property address to use." Refused to pick one — correct behavior. |

**Conclusion:** the existing `/webhook/parse-invoice-pdf` solves the user's problem for individual-bill emails. The multi-account reminder email (`municipalonlinepayments.com`) is a duplicate of bills that already arrive individually as PDFs from `info@princetontx.us` — recommend filtering it out of the Zap trigger rather than parsing it.

### 3. The Zap target sheet — `DWC Utility Accounts Lookup`

Spreadsheet ID: `1aniA0p8eGo5pE7dWVT66XS-bXV-my6wlKtahyBXdREQ`
URL: https://docs.google.com/spreadsheets/d/1aniA0p8eGo5pE7dWVT66XS-bXV-my6wlKtahyBXdREQ/edit

This is a **lookup table**, not a per-bill register. Columns A–Q:
A=Vendor Name, B=Account Number, C=Utility Type, D=Property Address (canonical bare format, e.g. `2711 Kimsey`), E=Portfolio ID, F=Property ID, G=Unit ID, H=Lease ID, I=Rentvine Ledger ID, J=Rentvine Vendor ContactID, K=LeadSimple Property ID, L=Rentvine Owner Bill Ledger ID, M=Responsibility (Owner / DWC Concierge), N=QBO Vendor ID, O=Bill Format (e.g. "HTML/visible", "HTML/email", "Multi-account consolidated"), P=Notes, Q=Last Verified.

The Zap presumably: parse → sheet lookup by `(vendor, account_number)` → pulls property/lease/ledger IDs → drives downstream LS/Rentvine work. Not yet confirmed end-to-end; user knows the Zap shape.

### 4. Two issues that affect Princeton matching

**(a) Account-number formatting mismatch.** Sheet col B stores accounts **dash-stripped** (e.g. `41664302`). PDFs show them **dashed** (`41-6643-02`). Parser currently returns the dashed form. Need to normalize for lookup.

**(b) Wrong account number in row 13.** Sheet row 13 (`City of Princeton` / `834 Ponderosa`) has account `67154012` in col B. The actual 834 Ponderosa account from the live PDF is `47-3305-01` (= `47330501` stripped). `67154012` is the City of The Colony account that appears in row 19 — looks like a leftover from the earlier "Vendor corrected from 'City of The Colony' data error" cleanup (col P note). **Sheet needs row 13 col B updated from `67154012` to `47330501`.**

Row 18 (`City of Princeton` / `1208 Mesquite` / `41664302`) is correct — strips cleanly to match the PDF's `41-6643-02`.

## Recommended 3-part fix (NOT YET SHIPPED — waiting on user)

1. **Tiny additive parser change**: have `/webhook/parse-invoice-pdf` also return `account_number_digits` (non-digits stripped) alongside `account_number`. Additive only — won't break invoice-watcher's consumption of the same endpoint.

2. **Rewire the Zap parser step** to Webhooks-by-Zapier `POST /webhook/parse-invoice-pdf` with body `{message_id: <Gmail message id>, account_email: "office@dwcproperty.com"}`. Header `x-webhook-secret: <WEBHOOK_SHARED_SECRET from .env>`. Map the response fields onto whatever the Zap currently feeds the sheet lookup. Use `account_number_digits` for the col B lookup.

   - Need the public webhook URL (the host the Zap currently reaches). Check existing Zaps that already hit `/webhook/rentvine-owner-bill` etc — they have it.

3. **Fix sheet row 13** — col B `67154012` → `47330501`. Also clear or update col P note since the data error is now corrected.

4. **Filter the trigger**: the Zap should ignore senders matching `noreply@municipalonlinepayments.com` (or restrict to only individual-bill senders). Their bills come through twice — once as the multi-account reminder HTML, once as individual PDFs from `info@princetontx.us`.

## Open questions for the user (option I asked before they stepped away)

> Want me to:
> - **(a)** make the parser change + fix row 13 + write up the Zap handoff steps for you to apply, or
> - **(b)** also rebuild the Zap end-to-end (I'd need the Zap's current trigger/action shape — paste a screenshot or step list and I can match it), or
> - **(c)** something else?

User left without answering.

## Concurrent-session note

Earlier today another session was open on the same machine working on **invoice-watcher Rentvine charges + Property Verification stage** (file: `Automation/Chat Saves/2026-04-30 Invoice-Watcher Rentvine Charges + Property Verification Stage.md`). That session shipped code; this session has not. They share `parser.js` and `parse-invoice-pdf` so be aware before edits.

## Side win this session

The user added the **Zapier MCP connector** mid-conversation. Saved a reference memory at `reference_zapier_connector.md`. Net add: Google Sheets (used in this session), QuickBooks Online, Outlook, DocuSign, Calendly, Canva, RentCast, Rentometer, Parseur, PDFfiller, plus generic `make_api_get_request`/`make_api_mutating_request`. Native connectors (Gmail, Drive, LeadSimple, Jotform) still preferred for those overlaps.

## Quick re-orient when picking up

1. Read this file.
2. Confirm `/webhook/parse-invoice-pdf` still works:
   ```
   curl -sS -X POST http://localhost:3000/webhook/parse-invoice-pdf \
     -H "Content-Type: application/json" \
     -H "x-webhook-secret: $(grep WEBHOOK_SHARED_SECRET /root/dwc-mcp-servers/webhook/.env | cut -d= -f2)" \
     -d '{"message_id":"19db561f0ff325d3","account_email":"office@dwcproperty.com"}'
   ```
   Expect a cache-hit success with City of Princeton / 1208 Mesquite / $143.82.
3. Ask user: a / b / c?
