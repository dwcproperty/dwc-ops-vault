# Phase 2 — Rentvine Bill Bridge

Extension to [[Invoice Watcher]] that takes the LeadSimple deal created in Phase 1 and writes a matching **vendor AP bill in Rentvine**. Vendor = payee. Property = the asset the expense posts against. The portfolio/owner is resolved automatically by Rentvine via the property's portfolio assignment.

> **Out of scope:**
> - **Utility bills.** Utility invoices flow through their own Gmail label (**`Utility Bills`** — exact spelling, no prefix, no slash) and a separate processor that ALSO drops deals into the **Acct Outstanding Invoices** pipeline. Phase 2 must NEVER touch those deals. Identification logic below.
> - **QuickBooks Online.** Vendor bills do NOT mirror to QBO. **Rentvine is the system of record for AP.**

## Current state (clarification)

Phase 1 (the Invoice Watcher today) creates a **LeadSimple deal only**. It does NOT write a Rentvine bill. It does NOT write a QBO bill. Bookkeeping currently re-keys every invoice into Rentvine by hand. Phase 2 closes that gap for general vendor invoices.

## Why this exists

When a non-utility vendor invoice is forwarded to the inbound mailbox and labelled `DWC/Invoice-Inbound`, the Invoice Watcher parses it and creates a LS deal in "New Invoice." Phase 2 picks up from there and writes the corresponding Rentvine vendor bill — fully automated, no manual review, properly coded to the right expense GL, with the property attached so the owner's books are charged.

## How Phase 2 distinguishes its deals from Utility Bills deals

Both processors write to the same LS pipeline ("Acct Outstanding Invoices"). Phase 2 must only act on deals created by the Invoice Watcher.

**Approved approach (Darrell, 2026-05-01):** New LS custom field `invoice_source` (text). Phase 1 (Invoice Watcher) sets it to `"invoice-inbound"`. The Utility Bills processor must be updated to set its own value (recommended: `"utility-bills"`). Phase 2 only processes deals where `invoice_source = "invoice-inbound"`.

## High-level flow

```
LS deal in "New Invoice"
   AND invoice_source = "invoice-inbound"
   AND rentvine_bill_id is empty
      │
      ▼
bridge-rentvine.js (poll every 60s)
      │
      ├─▶ vendor lookup in Rentvine
      │       └─▶ NOT FOUND or AMBIGUOUS → advance LS to "Vendor Onboarding Needed". Stop.
      │
      ├─▶ property resolution (LS property → Rentvine propertyID)
      │       └─▶ NOT FOUND / AMBIGUOUS / MISSING → advance LS to "Property Verification Needed". Stop.
      │
      ├─▶ GL account lookup for vendor
      │       └─▶ NOT MAPPED → advance LS to "Property Verification Needed" with reason. Stop.
      │
      ├─▶ duplicate check (vendorID + invoiceNumber)
      ├─▶ POST vendor bill to Rentvine
      ├─▶ attach original PDF
      ▼
update LS: write rentvine_bill_id, advance stage to "Bill Created in Rentvine",
add timeline note. Done — fully automated, no manual review at any step.
```

**No manual gate at any point.** Per Darrell, Phase 2 runs end-to-end without human approval. If anything goes wrong, the deal lands in one of the two Review stages and a human picks it up there.

## Architecture

- **Where it lives:** new module inside `invoice-watcher`, not a separate PM2 service. Path: `/root/dwc-mcp-servers/invoice-watcher/bridge-rentvine.js`. Reuses the same Gmail/LS clients, `.env`, and logs.
- **Trigger:** poll LeadSimple every 60s for deals in stage `New Invoice` AND `invoice_source = "invoice-inbound"` AND `rentvine_bill_id` is empty. Idempotent — already-processed deals self-exclude via the `rentvine_bill_id` check.
- **Why poll, not stage-change webhook:** webhooks from LS are best-effort and we don't want a single missed event to silently strand a bill. Poll is recoverable.
- **Concurrency:** single worker; deals processed serially. Volume is well under one bill per minute.

## Vendor resolution

1. Take parsed `vendor_company` from the LS deal's `raw_parsed_data`.
2. Hit Rentvine MCP: `list_vendors(search=vendor_company)`.
3. Match logic:
   | Result | Action |
   |---|---|
   | Exact case-insensitive name match | use it; cache `rentvine_vendor_id` on deal |
   | Single fuzzy match (normalized: lowercased, alphanumeric only) | use it; cache `rentvine_vendor_id` |
   | Multiple matches | route to **Vendor Onboarding Needed** with reason `vendor_ambiguous` |
   | Zero matches | route to **Vendor Onboarding Needed** with reason `vendor_not_found` |

**No vendor auto-creation.** Unknown vendors must go through the proper vendor onboarding flow (Jotform intake → reference checks → Rentvine vendor record with W-9, insurance, banking). Phase 2 stops cleanly and lets the existing onboarding process do its job.

## Property resolution

Phase 1 already attaches a LeadSimple property to the deal. Phase 2 just needs to translate that to a `rentvine_property_id`.

1. Read the LS property's address from the deal.
2. Hit Rentvine MCP: `find_property_by_address(line1, city, state, zip)`.
3. Match logic:
   | Result | Action |
   |---|---|
   | Single match | write `rentvine_property_id` to deal, proceed |
   | Multiple matches | route to **Property Verification Needed** with reason `property_ambiguous` |
   | Zero matches | route to **Property Verification Needed** with reason `property_not_found` |
   | LS property field empty | route to **Property Verification Needed** with reason `property_missing_on_deal` |

## Charge account mapping (vendor → expense GL)

Lives in: `/root/dwc-mcp-servers/invoice-watcher/vendor-gl-map.json`. Schema:

```json
{
  "<normalized_vendor_name>": {
    "chargeAccountID": <number>,
    "label": "<human readable>",
    "category": "repairs|supplies|professional_services|hoa|insurance|legal|other"
  }
}
```

Initial GL map covers **non-utility** vendor categories only. To be seeded during Phase 2a from Rentvine chart of accounts.

Example shape (specific `chargeAccountID` values pulled at build time):

```json
{
  "<example_plumber>":      { "chargeAccountID": 0, "label": "Repairs - Plumbing",       "category": "repairs" },
  "<example_hvac_company>": { "chargeAccountID": 0, "label": "Repairs - HVAC",           "category": "repairs" },
  "<example_locksmith>":    { "chargeAccountID": 0, "label": "Repairs - Locksmith",      "category": "repairs" },
  "<example_lawn_care>":    { "chargeAccountID": 0, "label": "Landscaping",              "category": "other" },
  "<example_attorney>":     { "chargeAccountID": 0, "label": "Legal Services",           "category": "legal" },
  "<example_hoa>":          { "chargeAccountID": 0, "label": "HOA Dues",                 "category": "hoa" }
}
```

> **Utility vendors are deliberately omitted.** They flow through the separate Utility Bills processor.

Behavior:

- If vendor is in map → use the mapped `chargeAccountID`.
- If vendor is NOT in map → route to **Property Verification Needed** with reason `gl_map_missing`. (This is a setup gap, not a missing vendor — the vendor exists in Rentvine but we haven't told the bridge what GL account to use.)

## Bill creation payload

`POST /manager/bills` (Rentvine API):

| Rentvine field | Source |
|---|---|
| `vendorID` | resolved Rentvine vendorID |
| `propertyID` | resolved Rentvine propertyID |
| `billDate` | parsed `invoice_date` |
| `dueDate` | parsed `due_date` |
| `invoiceNumber` | parsed `invoice_number` |
| `description` | `"{vendor_name} — invoice {invoice_number}"` |
| `lineItems[0].chargeAccountID` | from GL map |
| `lineItems[0].amount` | parsed `amount_due` |
| `lineItems[0].memo` | `"Account #{account_number}"` if present, else blank |
| `attachment` | original invoice PDF (mechanism TBD in Phase 2a — single POST or two-step) |

Returns: `billID`.

> Single line per bill is the default. Property is the cost center; we don't split line items.

## Round-trip to LeadSimple

On successful bill creation:

1. Update LS deal custom fields:
   - `rentvine_bill_id` ← billID
   - `rentvine_property_id` ← propertyID (if not already set)
   - `rentvine_vendor_id` ← vendorID
2. Advance stage: `New Invoice` → `Bill Created in Rentvine`.
3. Add a deal note: `"Rentvine bill #{billID} created {ISO timestamp}. Vendor #{vendorID}, Property #{propertyID}, ${amount} on chargeAccount #{chargeAccountID}."`

## Rentvine MCP prerequisite

The current Rentvine MCP exposes `create_bill`, but it's wired for **lease-ledger tenant charges**, not vendor AP bills. We need to extend the MCP first.

New tools to add to `/root/dwc-mcp-servers/rentvine/`:

| Tool | Wraps | Purpose |
|---|---|---|
| `create_vendor_bill` | `POST /manager/bills` | Phase 2 core write |
| `list_vendors` | `GET /manager/vendors?search=` | vendor resolution (lookup only — no create) |
| `find_property_by_address` | `GET /manager/properties?search=` + smart match | property resolution |
| `attach_file_to_bill` | `POST /manager/files` then link | PDF attachment |

Effort: ~1 day to extend the existing Rentvine MCP server.

## Error handling and Review routing

Two Review states only, per Darrell:

| Failure | Reason code | LS stage |
|---|---|---|
| Vendor lookup ambiguous | `vendor_ambiguous` | **Vendor Onboarding Needed** |
| Vendor lookup zero results | `vendor_not_found` | **Vendor Onboarding Needed** |
| Property lookup ambiguous | `property_ambiguous` | **Property Verification Needed** |
| Property lookup zero results | `property_not_found` | **Property Verification Needed** |
| Property missing on LS deal | `property_missing_on_deal` | **Property Verification Needed** |
| Vendor not in GL map | `gl_map_missing` | **Property Verification Needed** |
| GL map references non-existent Rentvine account | `gl_account_invalid` | **Property Verification Needed** |
| Rentvine API 5xx | `rentvine_5xx` | Retry 3× with exponential backoff (5s, 30s, 120s); on final failure → **Property Verification Needed** |
| Rentvine API 4xx | `rentvine_4xx` | Log full payload, route to **Property Verification Needed** immediately |
| LS update fails after Rentvine bill created | `ls_update_failed` | Critical — bill exists in Rentvine but LS doesn't know. Retry LS update 3× and alert via Slack/email |
| Duplicate bill found in Rentvine | `duplicate_existing` | Write existing billID to LS, advance to "Bill Already in Rentvine" (terminal) |

Every Review event writes a structured log entry to `/var/log/dwc/invoice-watcher.log` with deal id, reason code, and payload snapshot. The reason code is also written to a `bridge_failure_reason` LS custom field so the assignee knows why it landed in their queue.

## Duplicate detection

Before any POST:

1. Query `list_bills(vendorID=X, invoiceNumber=Y)` on Rentvine.
2. If a match exists → do not create. Write the existing billID to LS and move the deal to `Bill Already in Rentvine`.
3. Protects against re-runs after `processed.json` edits, label flips, or someone forwarding the same email twice.

## Stages required in LS — Acct Outstanding Invoices pipeline

Existing stage preserved: `New Invoice` *(Phase 1 lands here; Phase 2 reads from here)*

New stages to add:

- `Bill Created in Rentvine` *(success terminal)*
- `Bill Already in Rentvine` *(duplicate terminal)*
- `Vendor Onboarding Needed` *(human queue: spawn vendor onboarding pipeline)*
- `Property Verification Needed` *(human queue: fix property mapping or GL map)*

## New LS custom fields required

| Field | Type | Purpose |
|---|---|---|
| `invoice_source` | text | "invoice-inbound" set by Phase 1; lets Phase 2 ignore Utility Bills deals. **MUST also be set by the Utility Bills processor** to its own value (e.g., "utility-bills") |
| `rentvine_vendor_id` | text | resolved Rentvine vendorID |
| `rentvine_bill_id` | text | (already exists, just gets populated) |
| `rentvine_property_id` | number | (already exists, just gets populated) |
| `bridge_failure_reason` | text | reason code when routed to a Review stage |

## Implementation phases

### 2a — Rentvine MCP extension
- [ ] Pull current Rentvine chart of accounts; identify non-utility expense account IDs
- [ ] Add `create_vendor_bill`, `list_vendors`, `find_property_by_address`, `attach_file_to_bill` tools
- [ ] Determine PDF attachment mechanism (single POST vs two-step)
- [ ] PM2 restart `rentvine-mcp`
- [ ] Smoke test in MCP playground

### 2b — LS pipeline + Phase 1 update
- [ ] Add `invoice_source`, `rentvine_vendor_id`, `bridge_failure_reason` custom fields to the process type
- [ ] Add the 4 new stages to the Acct Outstanding Invoices pipeline
- [ ] Update Invoice Watcher (Phase 1) to set `invoice_source = "invoice-inbound"` on every new deal
- [ ] Coordinate with Utility Bills processor maintainer to set its own `invoice_source` value (e.g., `"utility-bills"`) so Phase 2 can ignore those deals

### 2c — Bridge module
- [ ] Build `bridge-rentvine.js` with the resolution and write logic
- [ ] Build initial `vendor-gl-map.json` with non-utility vendors (top 5–10 by volume)
- [ ] Wire LS round-trip and timeline note
- [ ] Wire Slack alert for `ls_update_failed`
- [ ] Wire stage routing for the two Review states with reason codes

### 2d — Testing
- [ ] Pick a known repair vendor and a known property
- [ ] Send a real non-utility invoice through `DWC/Invoice-Inbound`
- [ ] Verify: vendor resolved, property resolved, bill created, GL coded correctly, PDF attached, LS deal updated and advanced — **all without human intervention**
- [ ] Force each failure mode and verify the correct Review stage:
  - Unknown vendor → Vendor Onboarding Needed
  - Bad address → Property Verification Needed
  - Missing GL map entry → Property Verification Needed
- [ ] Send a `Utility Bills` labelled email through and verify Phase 2 ignores it (i.e., `invoice_source` filter works)

### 2e — Rollout
- [ ] Process the backlog of any deals already sitting in `New Invoice` from Phase 1 testing
- [ ] Monitor first 50 production bills for misroutes
- [ ] Expand GL map as new non-utility vendors appear

## Approved decisions (Darrell, 2026-05-01)

1. ✅ **Source-of-deal identification:** new `invoice_source` text custom field on the LS deal. Phase 1 sets to `"invoice-inbound"`. Utility Bills processor must be updated to set its own value.
2. ✅ **PDF attachment:** mechanism (single POST vs two-step) determined empirically in Phase 2a.
3. ✅ **Auto-advance from the start.** No manual review gate at any stage. Phase 2 runs end-to-end automatically. Failures land in one of the two Review stages.

## Explicit non-goals

- **Do not touch utility-bill deals.** Utility invoices use the Gmail label `Utility Bills` (exact spelling) and a separate processor. They land in the same LS pipeline but Phase 2 must ignore them via the `invoice_source` filter.
- **No QBO mirroring.** Vendor bills live in Rentvine only. (Per Darrell, 2026-05-01.)
- **No vendor auto-creation.** Unknown vendors go to Vendor Onboarding Needed. The existing onboarding pipeline handles them.
- **No manual review gate.** Phase 2 is fully automated. (Per Darrell, 2026-05-01.)
- **No payment scheduling.** Phase 2 creates the bill. Payment is a separate workflow.
- **No tenant chargeback.** That's the [[../Utility Bill Pipeline/]]'s job, and only for utilities.

## Future work (post-Phase-2)

- Auto-pay scheduling for trusted vendors above a confidence threshold and below a dollar cap
- Bill approval routing for amounts over $X (owner-specific thresholds)
- Webhook from Rentvine on bill paid → auto-close the LS deal
- Trigger vendor onboarding Jotform automatically when a deal lands in Vendor Onboarding Needed

## Related

- [[Invoice Watcher]] — Phase 1
- [[../Utility Bill Pipeline/]] — separate processor for utility invoices (label `Utility Bills`); out of scope for this spec
- [[../Gmail Resolve/]] — Gmail OAuth source

---

*Spec drafted 2026-05-01. Status: draft, ready for build. Blocked behind: (1) Anthropic API key on the Linode box for Phase 1 to come online, (2) Rentvine MCP extension (Phase 2a), (3) LS pipeline + custom field setup (Phase 2b).*
