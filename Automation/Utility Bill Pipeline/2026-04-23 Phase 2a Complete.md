# 2026-04-23 — Phase 2a(i) + 2a(ii) COMPLETE

**Session outcome:** Full end-to-end automation for DWC Concierge utility bills is now working — vendor email → parsed → routed → QBO draft bill + Rentvine tenant charge. This was a multi-session build; today's session cracked the Rentvine tenant-charge endpoint that blocked us yesterday.

## What's live (technically)

1. **Zapier happy path** — Gmail trigger → Anthropic parse → Sheets lookup → Filter → LeadSimple search + create process → QBO draft bill creation. Verified working on Atmos Belgian bill (Sample B).
2. **Linode webhook `/webhook/rentvine-tenant-charge`** — accepts POST with shared-secret auth, creates a tenant-facing lease charge on the Rentvine lease ledger.
3. **Tenant charges now hit the correct endpoint** (`POST /accounting/leases/{leaseID}/charges`) and show up on the lease's unpaid charges list, tenant portal, and tenant balance. Verified on Belgian Drive lease 33 — balance went $350.99 → $405.72 after test charge.

## The critical endpoint discovery

Spent most of today hitting the wrong Rentvine endpoint. Both our custom MCP's `create_bill` and the `/accounting/bills` endpoint it wraps are for **vendor AP bills** (money DWC owes vendors). They require `payeeContactID` and create transactions with `transactionTypeID: 7` tied to a bill record. **These do not appear on the tenant's ledger.**

The correct endpoint for tenant-facing charges (late fees, NSF, utility reimbursement, etc.) is:

```
POST /api/manager/accounting/leases/{leaseID}/charges
```

Payload (just 4 fields, no ledgerID or payee needed — Rentvine derives everything from the leaseID in the URL):

```json
{
  "datePosted": "2026-04-23",
  "amount": "0.01",
  "chargeAccountID": "38",
  "description": "APT TEST - Will delete"
}
```

Response creates a `transactionTypeID: 1` charge with `billID: null` that shows up on the lease's unpaid charges list.

**Reverse-engineered via Chrome DevTools** — manually added a lease charge in Rentvine UI, captured the XHR call in the Network tab.

## Rentvine data model notes (important for future work)

- **`leaseID`** and **`ledgerID`** are different things. The lease has its own ID (33 for Belgian); the ledger the charge hits is 53 (Moor Lawson LLC portfolio). Rentvine auto-resolves the ledger from the leaseID when using the charges endpoint.
- **ChargeAccountID 38 = account 4560 "Tenant Utility Reimbursement"** — this is the account code that tells Rentvine the charge is tenant-facing and subject to the trust-to-operating sweep.
- **`payeeContactID` is only for bills** (vendor AP), not for charges. Our early attempts needed 218 for Atmos because we were using the wrong endpoint.

## Bug found in existing Rentvine MCP

The custom Rentvine MCP's `create_bill` tool description says *"Create a charge against a lease (late fees, NSF fees, etc)"* — but it actually hits `/accounting/bills`, which creates vendor AP bills, not lease charges. This tool has effectively been broken for its stated purpose since it was written. **Fix: either repurpose `create_bill` to call the correct lease-charges endpoint, or add a new `create_lease_charge` tool alongside it.** Leaving the broken tool in place is dangerous — it will create accounting errors for anyone who trusts the description.

## Webhook infrastructure

Location: `/root/dwc-mcp-servers/webhook/index.js`  
Backups: `index.js.bak`, `index.js.bak2`, `index.js.bak3`  
Port: 3000  
Running as PM2 process `webhook` (id 3)

Env sources loaded in order:
1. `/root/dwc-mcp-servers/leadsimple/.env` (LEADSIMPLE_API_KEY)
2. `/root/dwc-mcp-servers/Rentvine/.env` (RENTVINE_ACCOUNT, ACCESS_KEY, SECRET)
3. `/root/dwc-mcp-servers/webhook/.env` (WEBHOOK_SHARED_SECRET)

Routes:
- `GET /webhook/health` — uptime check, no auth
- `POST /webhook/rentvine-tenant-charge` — requires `x-webhook-secret` header
- `POST /` (default, catch-all) — existing Jotform → LeadSimple deal advance handler (unchanged, preserved)

## Shared secret

```
WEBHOOK_SHARED_SECRET=09058436753e360bc2c1bddc621afac932cdc3aae2d28510c01571a8844352b8
```

Used in: Linode `.env` + Zapier Webhooks step headers.

## Working curl for reference

```bash
curl -X POST http://localhost:3000/webhook/rentvine-tenant-charge \
  -H "Content-Type: application/json" \
  -H "x-webhook-secret: 09058436753e360bc2c1bddc621afac932cdc3aae2d28510c01571a8844352b8" \
  -d '{
    "leaseID": 33,
    "amount": 54.73,
    "description": "Utility: Gas (Atmos) for March 2026",
    "datePosted": "2026-03-04"
  }'
```

Response includes `transactionID` and `rentvine_response.transaction` for downstream use.

## Sheet schema — simpler than expected

Originally added columns for Rentvine Ledger ID and Vendor ContactID thinking we'd need both. **Neither is actually needed for the tenant-charge endpoint.** The Zap only needs:

- `Lease ID` (Rentvine leaseID) — primary key
- `Utility Type`, `Vendor Name`, `Property Address` — for description building
- `LeadSimple Property ID`, `Qbo Vendor Id` — for LeadSimple + QBO steps
- `Responsibility` — for Paths routing (DWC Concierge / Owner Pays / Tenant Direct / Vacant)

Keep the Ledger ID and Vendor ContactID columns as reference data (useful for Phase 2a(iii) Owner Pays / Vacant-Owner flows which likely DO use the bills endpoint), but they're not wired into the DWC Concierge Path A flow.

## Cleanup done this session

- [x] Bill 721 voided (wrong property — Renate Racher, 539 West Lookout)
- [x] Bill 722 voided (correct portfolio but wrong transaction type)
- [x] Transaction 3651 ($0.01 APT TEST) delete from lease 33 in Rentvine UI
- [x] Transaction 3652 ($54.73 TEST 4) delete from lease 33 in Rentvine UI

## Still TODO before flip-live

### Immediate (next session or tomorrow)
- [ ] **Wire Zapier Path A Step 13A** — Webhooks by Zapier POST to `https://<webhook-domain>/webhook/rentvine-tenant-charge` with leaseID, amount, description, datePosted + shared secret header
- [ ] **End-to-end Zap test** — trigger off a real Atmos email, confirm QBO draft bill + Rentvine tenant charge both fire
- [ ] Flip Zap to ON
- [ ] Final cleanup: delete test processes on Atmos Energy LeadSimple lead, delete test bills in QBO

### Phase 2a(iii) — Owner Pays / Vacant-Owner
- [ ] Figure out correct endpoint for posting charges to the owner's portfolio ledger (likely IS `/accounting/bills` with payeeContactID, but needs verification)
- [ ] Wire Path B in Zapier

### Phase 2b — Payment Detection
- [ ] Rentvine webhook for tenant payment events → trigger LeadSimple stage advance "Tenant Paid"

### Phase 2c — Late Fee Automation
- [ ] $35 late fee charge at Tenant Due Date + 1 day
- [ ] Trigger: LeadSimple stage "Late Fee Applied" reaches trigger date

### Stage automations
- [ ] 7 notification templates (email/SMS) not yet attached to LeadSimple stages

### Vendor contactID discovery
- [ ] Atmos = 218 (confirmed)
- [ ] Map the rest: TXU Energy, City of Aubrey, City of Celina, City of Princeton, City of Murphy, CoServ, Culeoka Water Corporation, Mustang Water, GCEC, Farmers Electric, Inframark
- [ ] Use search_transactions with vendor name, check contactID on bill-linked payment transactions

### Other
- [ ] Backfill Responsibility column for all sheet rows (only Belgian populated)
- [ ] Fix Rentvine MCP's broken `create_bill` tool (either repurpose or add correct `create_lease_charge` tool)
- [ ] Fix data flags: Belgian TXU number verification, Kinglet Atmos digit count (9 vs 10), Ponderosa vendor mislabel, Burmese Inframark acct#

## Key entity IDs (for reference)

| Entity | ID | Notes |
|---|---|---|
| Belgian Drive leaseID | 33 | Use for tenant-charge endpoint |
| Belgian Drive ledgerID | 53 | Moor Lawson LLC portfolio — for bills endpoint only |
| Belgian Drive propertyID | 38 | |
| Belgian Drive unitID | 35 | |
| Moor Lawson LLC portfolioID | 19 | |
| Moor Lawson LLC owner contactID | 216 | |
| Tenant: Triston Freeman contactID | 227 | |
| Tenant: Madison Bowers contactID | 228 | |
| Atmos Energy vendor contactID | 218 | From bill 626 / transaction 3248 |
| ChargeAccountID 38 | Account 4560 "Tenant Utility Reimbursement" | For utility reimbursement charges |
| QBO Atmos vendor ID | 85 | |
| QBO account | "DWC Tenant Utility Advances" | Expense / Other Misc Service Cost |

## Session notes

- User pushed back twice today when I suggested stopping at blockers. Fair call — both times we pushed through, the fix was straightforward. DevTools reconnaissance was the unlock for the tenant-charge endpoint. Lesson for future sessions: when hitting an API dead-end, don't guess endpoints forever — DevTools inspection of the working UI action is the fastest path.
- Three iterations of the webhook code tonight (bak, bak2, bak3). Each time it was a paste-the-whole-file replace rather than nano surgery — user prefers this and it's less error-prone.


---

## 2026-04-23 (Continued) — Phase 2a LIVE + Phase 2b primitive built

After writing the journal entry above, pushed through several more hours and got:

### Phase 2a — FULLY LIVE in production

- **Nginx configured** for public webhook access: `webhook.collincountyrent.com` with Let's Encrypt TLS, proxying to `127.0.0.1:3000`. DNS A record pointed at `96.126.118.135`.
- **Zapier Step 11 wired** — Webhooks by Zapier POST to `https://webhook.collincountyrent.com/webhook/rentvine-tenant-charge` with shared secret header. Test fired successfully end-to-end, creating transaction 3654 ($220.73 Atmos charge) on Belgian Drive lease 33.
- **Full end-to-end verified:** Gmail → Anthropic parse → Sheets → Filter (DWC Concierge only) → LeadSimple process → QBO draft bill → Rentvine tenant-facing charge. All automated.

### Phase 2b core primitive — BUILT

New webhook route: `POST /webhook/check-charge-paid`

Input:
```json
{ "leaseID": 33, "rentvine_transaction_id": 3642 }
```

Output (unpaid case):
```json
{
  "success": true,
  "is_paid": false,
  "transaction_id": "3642",
  "total_amount": 201.5,
  "amount_paid": 0,
  "amount_unpaid": 201.5,
  "description": "TXU Energy April",
  "datePosted": "2026-04-22",
  "isOverdue": true,
  "lease_balance": { "unpaidTotalAmount": 350.99, ... }
}
```

Output (paid/settled case):
```json
{
  "success": true,
  "is_paid": true,
  "transaction_id": "3000",
  "amount_unpaid": 0,
  "note": "Transaction is not in lease unpaidCharges; treated as paid/settled"
}
```

**Logic:** Calls Rentvine's `/api/manager/leases/export?leaseIDs[]=X` endpoint, scans the `unpaidCharges` array for the target transactionID. If found → still unpaid, return amount. If not found → paid/voided/settled. This endpoint scans unpaidCharges rather than looking up the transaction directly because Rentvine's lease-level view is what the tenant sees, which matches what we care about for late-fee decisions.

Both test cases passed April 23 2026.

### Important discovery — why existing "Amount Receivable" integration won't work for utilities

Darrell's `06 Delinquencies Late Rent` process uses Step Conditions on a custom field called `Amount Receivable`, populated by an existing Rentvine→LeadSimple integration. Initially we considered reusing this pattern for utilities.

**Blocker:** `Amount Receivable` is an AGGREGATE of all receivables on the lease (rent + utilities + late fees + everything). We cannot tell from that field whether a specific utility charge has been paid vs. whether the tenant still owes rent but paid the utility.

**Solution:** We need charge-specific tracking. Store each utility charge's Rentvine `transactionID` on its LeadSimple process, and use the `/webhook/check-charge-paid` endpoint to ask "is THIS specific charge paid?" at decision time.

### Rentvine webhook options explored but not pursued

Rentvine outbound webhooks exist, but only two event types are available: `Lease Charge Created` and `Lease Charge Updated`. No dedicated "Payment Received" event. Initial plan was to subscribe to `Lease Charge Updated` and infer payments from amountPaid changes, but this is significantly more complex than the pull-at-decision-time approach we ultimately built.

**Decision: pull model wins.** Event-driven (push) would cost ~40-60x more in Zapier tasks at scale compared to one-shot check at the critical moment. Push doesn't help because real-time "paid" visibility is already available in Rentvine UI.

### Architecture decision: Linode-side logic, not Zapier polling

For the late-fee decision (Phase 2c next), we'll have LeadSimple fire a webhook to Linode at Tenant Due Date + 1 day. Linode checks the charge status via `/webhook/check-charge-paid`, then either:
- If paid → advances LeadSimple deal to "Paid Complete" stage (or similar)
- If unpaid → advances deal to "Apply Late Fee" + creates $35 late fee via existing `/webhook/rentvine-tenant-charge` endpoint

Still TODO: figure out LeadSimple API for advancing a PROCESS (not deal) through specific stages.

### Existing Rentvine MCP `create_bill` — STILL BROKEN

Despite today's discoveries, the existing MCP's `create_bill` tool description remains misleading. It hits `/accounting/bills` (vendor AP) but claims to be for lease charges. Tool description needs fixing OR a new `create_lease_charge` tool added that hits `/accounting/leases/{id}/charges`. Deferred.

### Infrastructure state after tonight

**Linode webhook routes (all live):**
- `GET /webhook/health` — uptime check, no auth
- `POST /webhook/rentvine-tenant-charge` — creates lease charge (Phase 2a)
- `POST /webhook/check-charge-paid` — checks charge payment status (Phase 2b primitive)
- `POST /` (default) — Jotform → LeadSimple deal advance (unchanged)

**File:** `/root/dwc-mcp-servers/webhook/index.js` (backups: `.bak` through `.bak5`)

**Env sources:**
1. `/root/dwc-mcp-servers/leadsimple/.env` — LEADSIMPLE_API_KEY
2. `/root/dwc-mcp-servers/Rentvine/.env` — RENTVINE_ACCOUNT, ACCESS_KEY, SECRET
3. `/root/dwc-mcp-servers/webhook/.env` — WEBHOOK_SHARED_SECRET

**Nginx config:** `/etc/nginx/sites-enabled/webhook` — proxies `webhook.collincountyrent.com:443` to `localhost:3000` with Let's Encrypt cert (auto-renewing).

### Remaining TODO (not done tonight)

1. **Phase 2c — Late fee automation**
   - LeadSimple "Acct Utility Bills" child process: add stage at Tenant Due Date + 1 day that fires webhook
   - New Linode route `/webhook/late-fee-check-and-apply` that calls `check-charge-paid`, then conditionally creates $35 late fee + advances LeadSimple
   - Need to figure out LeadSimple process-advancement API (stage IDs, authentication, endpoints)

2. **Storing Rentvine transactionID on LeadSimple process**
   - Add custom field `Rentvine Transaction ID` to "Acct Utility Bills" process
   - Zapier Step 12 (after Step 11): LeadSimple "Update Process" action storing the transactionID from Step 11's webhook response

3. **Cleanup (before Zap flipped to full production):**
   - [ ] Delete test charge 3654 ($220.73) from Belgian lease 33 ledger
   - [ ] Delete test Atmos Energy processes in LeadSimple
   - [ ] Delete test Atmos bills in QBO
   - [ ] Fill `Responsibility` column for non-Belgian rows in sheet

4. **Sheet maintenance:** Fill remaining vendor contactIDs (TXU, CoServ, City of Aubrey, City of Celina, City of Princeton, City of Murphy, Culeoka Water, Mustang Water, GCEC, Farmers Electric, Inframark) — though note this field is now optional for the lease-charge endpoint; only needed if using the bills endpoint for Owner Pays or Vacant flows.

5. **Fix existing Rentvine MCP `create_bill` tool description** — still misleading.

6. **Phase 2a(iii) — Owner Pays / Vacant Owner charges**
   - Different endpoint from tenant charges (probably `/accounting/bills` with payeeContactID, per the earlier dead-end we hit)
   - Path B (Owner Pays) and Path D (Vacant-Owner) in Zapier


---

## 2026-04-24 Morning Session — Owner Name Auto-Populate

### Problem
AOI processes were being created with Owner Name field blank. Manual entry not acceptable — needed automated lookup.

### Solution: New Linode endpoint `/webhook/get-portfolio-owner`

Since every property in Rentvine has a portfolioID and every portfolio has an owner, portfolio lookup works universally (even for vacant units without leases).

**Endpoint lives on Linode webhook at:**
`POST https://webhook.collincountyrent.com/webhook/get-portfolio-owner`

**Input:** `{"portfolioID": 19}`

**Output:**
```json
{
  "success": true,
  "portfolioID": "19",
  "owner_name": "Moor Lawson LLC",
  "primary_owner_name": "Moor Lawson, LLC",
  "primary_owner_contact_id": "216",
  "all_contacts": [...]
}
```

### Rentvine endpoint discovery
Direct single-record GET works: `GET /api/manager/portfolios/{id}`
Returns `{portfolio: {...}}` shape with `contacts` as a real array (not JSON string).

First attempt used `/portfolios/export?portfolioIDs[]=` pattern (matching our working lease endpoint) — that returned 404. Rentvine's API is inconsistent; portfolios don't have an export path, but leases do.

### Zapier integration
Added new Step 5 (Webhooks by Zapier POST) between Sheets Lookup and AOI Create Process. Maps `portfolioID` from sheet → calls endpoint → `owner_name` response field mapped into Step 8 (Create AOI) Owner Name custom field.

Tested successfully — AOI now auto-creates with Owner Name populated.

### Code change
Function `getPortfolioOwner()` uses `/api/manager/portfolios/${id}` path. Also handles `contacts` field which may be JSON string (from search endpoint) or actual array (from single-record endpoint) — defensive parsing works for both.

Files:
- `/root/dwc-mcp-servers/webhook/index.js.bak7` — pre-morning backup
- `/root/dwc-mcp-servers/webhook/index.js.bak8` — pre-URL-fix backup


---

## 2026-04-24 (Continued) — Owner Pays Endpoint Complete

Built the Phase 2a(iii) Owner Pays endpoint that was deferred last night.

### New endpoint: `POST /webhook/rentvine-owner-bill`

Creates a bill on the owner's unit ledger (Type 4) for utility charges where owner is responsible. Funds come from trust account, never hits QBO.

**Input:**
```json
{
  "propertyID": 38,
  "payeeContactID": 218,
  "ledgerID": 55,
  "chargeAccountID": 46,
  "amount": 149.49,
  "billDate": "2026-04-20",
  "dateDue": "2026-04-28",
  "serviceAccountNumber": "20-0032-03"
}
```

**Defaults:**
- `chargeAccountID` = 46 (Owner Utilities, 6402) if omitted
- `billDate`/`dateDue` = today if omitted

### Rentvine API discovery

The `/api/manager/accounting/bills` endpoint needs a very specific payload shape — captured via Chrome DevTools network tab:

```json
{
  "billDate": "YYYY-MM-DD",
  "dateDue": "YYYY-MM-DD",
  "propertyID": "38",
  "payeeContactID": "218",
  "serviceAccountNumber": null,
  "tenantAmount": 0,
  "utilityPeriodStartDate": null,
  "utilityPeriodEndDate": null,
  "leaseCharges": [],
  "charges": [
    {
      "ledgerID": "55",
      "amount": 0.01,              // CRITICAL - also required alongside fromPayer/toPayee
      "markupAmount": 0,
      "discountAmount": 0,
      "fromPayer": 0.01,
      "toPayee": 0.01,
      "chargeAccountID": "46"
    }
  ]
}
```

First attempt missed the `amount` field inside charges[] (only had fromPayer/toPayee). Rentvine returned: `{"charges[0].amount":["Amount is required"]}`. Good error message.

### Ledger type discovery

Huge discovery: **the sheet's "Rentvine Ledger ID" column is actually the PORTFOLIO ledger (Type 2) — different from what Owner Pays needs.**

Rentvine has multiple ledger types:
- **Type 2** = Portfolio/owner ledger (what v1/v2 sheet had)
- **Type 4** = Unit ledger (what Owner Pays bills need)
- Type 5 or similar = Lease ledger (for tenant-facing charges, but we don't use it directly)

Example for Belgian: portfolio ledger 53 (Type 2, Moor Lawson LLC) vs unit ledger 55 (Type 4, for 2025 Belgian unit 35).

### Probe-swept all Type 4 unit ledgers for the portfolio

Ran: `for i in $(seq 1 150); do curl /api/manager/accounting/ledgers/{i}; done` — filtered for `ledgerTypeID:"4"`. Got the full unitID → unit-ledger mapping in 90 seconds.

All 12 DWC unit ledgers captured for v3 sheet.

### v3 sheet created

**Columns added/filled in v3:**
1. **Rentvine Vendor ContactID** (column J) — was mostly blank, now fully populated for 10 of 12 utility vendors. Inframark = not a registered vendor in Rentvine yet (Darrell has never paid them). GCEC found under "Grayson County Electric Coop" (contactID 255).
2. **Rentvine Owner Bill Ledger ID** (column L, NEW) — Type 4 unit ledger for all 23 rows.

Vendor name cleanups to match Rentvine's registered names exactly:
- CoServ → CoServe
- Mustang Water → Mustang Water Supply
- Farmers Electric → Farmers Electric Coop
- GCEC → Grayson County Electric Coop
- City of The Colony → City of The Colony Water Department

**File ID:** `1bJId6fciPZzwgUMfUVMWhktk7TrEcDJDNGsqdMf00tM`
URL: https://docs.google.com/spreadsheets/d/1bJId6fciPZzwgUMfUVMWhktk7TrEcDJDNGsqdMf00tM/edit

Old v2 can be archived once Zapier Step 4 is repointed at v3.

### Reusable pattern for future: sequential ledger ID probe

For finding ledgers by type:
```bash
for i in $(seq 1 200); do
  R=$(curl -s -H "Authorization: Basic ${AUTH}" \
    "https://${account}.rentvine.com/api/manager/accounting/ledgers/${i}")
  if echo "$R" | grep -q '"ledgerTypeID":"4"'; then
    # found a unit ledger
  fi
done
```

### State after this session

**Endpoints live (5):**
- `/webhook/rentvine-tenant-charge` — Concierge tenant charges
- `/webhook/rentvine-owner-bill` — Owner Pays bills (NEW)
- `/webhook/get-portfolio-owner` — Auto-fill Owner Name
- `/webhook/check-charge-paid` — Payment detection primitive
- `/webhook/late-fee-decision` — Late fee atomic decision
- `POST /` default — Jotform → LeadSimple deal advance (unchanged)

**Backups:** `.bak7` through `.bak10` at `/root/dwc-mcp-servers/webhook/`

### Remaining TODO

1. **Zapier Path B wiring** — use new `/webhook/rentvine-owner-bill` endpoint when Responsibility = Owner Pays OR Vacant – Owner
2. **Zapier Step 4 repoint to v3 sheet** — currently pointing at v2
3. **Phase 2c LeadSimple wiring** — stage action to fire `/webhook/late-fee-decision` at Tenant Due Date + 1
4. **Separate Zap: Payment detection** — triggered by LeadSimple stage, fires `/webhook/check-charge-paid`, closes Acct Utility Bills process when paid
5. **Separate Zap: AOI closure** — triggered when owner-pays bill paid in Rentvine, closes AOI
6. **Fix existing Rentvine MCP `create_bill` tool** description (still misleading)


---

## 2026-04-24 — PHASE 2A FULLY COMPLETE

Owner Pays path (Path B) tested successfully end-to-end in Zapier. Real Owner Pays invoice created in Rentvine via webhook.

**Full Zap architecture now live:**

```
Steps 1-4  :  Gmail trigger → Anthropic parse → Code unpack → Sheets lookup (v3)
Step 5     :  Webhook: get-portfolio-owner (auto-fills Owner Name on AOI)
Step 6-7   :  Filter / Formatter (tenant due = bill due - 3 days)
Step 8     :  LS Search Lead (Rentvine Vendors)
Step 9     :  LS Create Process: Acct Outstanding Invoices (always fires)
Step 10    :  Paths by Zapier
             ├─ Path A: Responsibility = DWC Concierge
             │   A1  LS Create Process: Acct Utility Bills
             │   A2  QBO Create Bill (Account Based)
             │   A3  Webhook: /webhook/rentvine-tenant-charge
             │   A4  LS Update Process: Acct Utility Bills
             │        (Rentvine Transaction ID + Utility Charge Amount Unpaid)
             └─ Path B: Responsibility = Owner Pays / Vacant – Owner
                 B1  Webhook: /webhook/rentvine-owner-bill
```

**All Linode endpoints in production:**
- `/webhook/rentvine-tenant-charge` (Path A — Concierge tenant charges) ✅ LIVE
- `/webhook/rentvine-owner-bill` (Path B — Owner Pays bills) ✅ LIVE
- `/webhook/get-portfolio-owner` (Owner Name auto-fill) ✅ LIVE
- `/webhook/check-charge-paid` (payment detection primitive) ✅ BUILT, not wired yet
- `/webhook/late-fee-decision` (atomic late fee) ✅ BUILT, not wired yet

### Next separate Zaps (per Darrell's decision to keep builds isolated)

1. **Late Fee Zap** — triggered by LeadSimple Acct Utility Bills stage at Tenant Due Date + 1, fires `/webhook/late-fee-decision`
2. **Payment Detection Zap (Concierge)** — monitors Rentvine charges for payment, closes Acct Utility Bills on paid
3. **Payment Detection Zap (Owner Pays)** — monitors Rentvine owner bills for payment, closes Acct Outstanding Invoices on paid

All three use existing endpoints — no new Linode code needed for now.

### Remaining non-blocking TODOs

- Archive/delete v2 sheet now that v3 is authoritative
- Fix existing Rentvine MCP `create_bill` tool description (still misleading)
- Backfill Kimsey #101/#102/#104 utility accounts when bills arrive
- Add Inframark as vendor in Rentvine when first bill arrives
- Verify Kinglet Atmos account number length (9 digits vs standard 10)


---

## 2026-04-24 Late Afternoon — Claude Code Installed

Big tooling shift: installed Claude Code on Linode. From now on, future builds can happen entirely in Darrell's terminal — no copy-paste dance with Claude chat.

### Setup
- Installed via `npm install -g @anthropic-ai/claude-code`
- Version: 2.1.119 (Opus 4.7, 1M context, Claude Max)
- Working directory: `/root/dwc-coding/`
- Auto-loaded context file: `/root/dwc-coding/CLAUDE.md` (67 lines covering: architecture overview, Linode infra, webhook endpoints, env files, ledger types, sheet IDs, standing instructions)

### MCPs already connected in Claude Code session
- Rentvine (full read/write)
- LeadSimple (deals, pipelines, contacts, processes, tasks, conversations)
- RentEngine (showings, lockboxes, prospects)
- Obsidian (read/write/search both vaults)
- Gmail
- Google Drive
- Notion
- Intuit QuickBooks
- Jotform

### How to use it next session
1. SSH to Linode: `ssh root@96.126.118.135`
2. `cd /root/dwc-coding`
3. `claude`
4. Tell Claude Code: "Read CLAUDE.md and catch yourself up on the latest journal entry"

CLAUDE.md will load automatically; the journal-read primes it on most recent work.

### Why this matters
Darrell's plan: future processes (late fee Zap, payment detection Zap, etc.) will be built entirely in code on Linode rather than Zapier. Claude Code makes this possible without Darrell having to copy-paste between Claude chat and his terminal. Eventually the existing utility intake flow may also be ported off Zapier (~weekend of work, mostly Node + Gmail push notifications).

### TODO list (carries forward to next session)
1. Late Fee Zap (or Linode-coded equivalent) — fires `/webhook/late-fee-decision` at Tenant Due Date + 1
2. Payment Detection Zap (Concierge) — closes Acct Utility Bills when charge paid
3. Payment Detection Zap (Owner Pays) — closes AOI when owner bill paid
4. Archive/delete v2 sheet now that v3 is authoritative
5. Fix Rentvine MCP `create_bill` tool description (still misleading)
6. Backfill Kimsey #101/#102/#104 utility accounts when bills arrive
7. Add Inframark as Rentvine vendor when first bill arrives
8. Verify Kinglet Atmos account number length (9 vs 10 digits)
