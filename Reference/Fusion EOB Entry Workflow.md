# Fusion EOB Entry Workflow

How to enter insurance Explanation of Benefits (EOBs) / Remittance Advice into Fusion Web Clinic (`app.fusionwebclinic.com`) for Frisco Feeding and Speech Therapy.

Fusion has no API — this is a manual UI workflow. Claude can drive it via browser automation, but operator confirmation matters because it touches real billing data.

## Quick mental model

Each EOB from an insurance company is a **Remittance** in Fusion. A Remittance contains one or more **Claims** (one per patient/service). Each Claim has one or more **Lines** (one per CPT procedure code).

**Process is always 3 steps:**
1. Create the Remittance header (payer + date + total $ paid + Remit #)
2. Attach the relevant Claims (search by patient name, link)
3. For each Claim, enter Line-level amounts (CO-45, Deductible, Co-Pay, etc.) so they balance to the total billed

## Field mapping: EOB → Fusion

### Remittance header (Add Remittance dialog)

| Fusion field | EOB source | Notes |
|---|---|---|
| Payer | EOB header (top-left) | Search by partial name. "Cigna" → "CIGNA (ST)"; "BCBS" → "BCBS (PPO)" or "BCBS (HMO)"; "American" → "American Specialty Health (CIGNA OT)" |
| Date | Date received in Fusion | Use today's date (not the EOB date) |
| Amount | EOB total "PROV PD" / Check Amt | $0.00 for NonPay EOBs; actual paid amount otherwise |
| Type | Always **EFT** | Per Darrell — even if EOB says Check or NonPay |
| Number | NonPay # or EFT # from EOB header | The remit ID |
| Billing Provider | Leave blank | Not required |
| Identifier Type | Leave blank | Not required |

### Claim line (Edit Adjudication dialog)

For each claim's line items, fill these per CPT code:

| EOB line value | Fusion field | When |
|---|---|---|
| Billed | Charges Submitted | Always (auto-filled) |
| 1 (NOS) | Units Submitted | Always (auto-filled) |
| **CO-45** adjustment | **Not Allowed CO 45** | Always when CO-45 is on the line |
| Deduct | **Deductible PR 1** | When deductible > 0 |
| Coins | **Co-Insurance PR 2** | When coinsurance > 0 |
| PR-3 amount | **Co-Pay PR 3** | When copay > 0 |
| OA-23 amount | **Other Payers OA 23** | When other payer (rare) |
| Prov Pd | **Charges Paid** | When insurance paid > 0 (paying EOBs) |

**Allowed Amount field** — leave blank. The "Allowed OR Not Allowed" pair: we always use Not Allowed (the CO-45 adjustment side).

Line must end up **"Balanced"** (Charges Submitted = sum of all amounts on the line). When balanced, the line shows green "Balanced" badge.

Once all lines on a claim are balanced, the header shows **"$0.00 of $X.XX Remaining"** — then Save Adjudication.

## Common adjustment / remark codes

| Code | Meaning | Where it goes in Fusion |
|---|---|---|
| CO-45 | Charge exceeds fee schedule (most common write-off) | Not Allowed CO 45 |
| CO-B13 | Previously paid | (Adjustments section — uncommon) |
| OA-B11 | Claim transferred to proper payer | (Status changes, no $ entered) |
| OA-23 | Other payer's adjustment | Other Payers OA 23 |
| PR-1 | Deductible Amount | Deductible PR 1 |
| PR-2 | Coinsurance Amount | Co-Insurance PR 2 |
| PR-3 | Co-payment Amount | Co-Pay PR 3 |
| PR-27 | Expenses after coverage terminated | (Special — sends total to patient) |
| PR-187 | Health Savings Account payment | (Cigna-specific, may need special handling) |
| PI-144 | Incentive adjustment | (American Specialty Health-specific, may add via Adjustments) |
| N216 | Coverage not offered for this service | Remark code only |

## Critical gotchas

### 1. Claim status determines if line entry is possible
- **Accepted** status → Edit Adjudication is editable, you must enter line amounts
- **Completed** status → form is read-only / locked. Just create the remittance and attach the claim; line amounts cannot be entered. This usually means the claim was already settled by another payer/EOB.
- **Approach for Completed claims:** create remit, attach, leave line entry blank — Fusion auto-finalizes them as "Finalized (Patient Account)".

### 2. Filter by PATIENT NAME, not Claim # (Claim # filter is buggy)
The Claim # filter sometimes fails silently and lets you accidentally check a wrong claim. **Always use Patient name filter** instead:
1. In Claim List, click the **funnel icon** (top-right of dialog)
2. Check "Patient" to enable the Patient filter
3. Close funnel, click "Patient" filter chip
4. Type patient last name → results auto-populate
5. Check the patient → Apply
6. Then check the specific claim row → Link

### 3. ACNT number on EOB = Fusion Claim #
The EOB's "ACNT P00059145190" maps to Fusion claim number **59145190** (drop the "P000" prefix). Use this to verify the right claim got linked.

### 4. Remitter filter defaults to existing claims for the payer
When adding claims to a new remittance, Fusion pre-filters to claims matching the payer. If your EOB's claims aren't appearing, also clear/check the Remitter and Phase filters (Phase often defaults to "Pending" only — Settled claims won't show).

## Step-by-step (concrete)

### A. Create new remittance
1. Billing → Remittances → Received
2. Click "+ Remittance" (bottom-right)
3. Fill: Payer (search), Date (today), Amount ($ from EOB), Type (EFT), Number (NonPay/EFT #)
4. Leave Billing Provider and Identifier Type blank
5. Click "Save Remittance"
6. New remittance appears at top of list

### B. Attach claims to remittance
1. Click on the new remittance row to open it
2. Click "+ Claim" (bottom-right)
3. Open funnel → enable "Patient" filter (only needed once per dialog open)
4. Click "Patient" chip → search by last name → check patient → Apply
5. Check the box on the correct claim row (match Date of Service to EOB)
6. Click "Link claims to remittance"
7. **Repeat for each claim on the EOB.** Each "+ Claim" cycle resets filters, so you need to re-enable Patient filter each time.

### C. Enter line amounts (Edit Adjudication)
1. Click on the attached claim row
2. Edit Adjudication dialog opens (Summary tab first)
3. For each Line tab on the left:
   - Click into "Not Allowed CO 45" field → type CO-45 amount
   - Click into "Deductible PR 1" → type Deduct amount (if any)
   - Click into "Co-Insurance PR 2" → type Coins amount (if any)
   - Click into "Co-Pay PR 3" → type PR-3 amount (if any)
   - Click into "Charges Paid" → type Prov Pd amount (if any)
4. Verify Line shows "Balanced" badge (green)
5. After all Lines balanced, header shows "$0.00 of $X.XX Remaining"
6. Click "Save Adjudication" (bottom-right)
7. Back at remittance, claim now shows "Finalize to Patient Account" with green dot

### D. Finalize the remittance (when all claims done)
- When every claim is balanced/finalized and Remittance shows $0.00 of $X.XX Remaining, click "Finalize" (bottom-right). Status: Finalized.

## EOB Inventory — file `05062026_05072026.pdf` (12 EOBs)

This batch was started 2026-05-12. Status as of that session:

| # | Report ID | Payer | Remit # | Date | Claims | $ Paid | Status |
|---|---|---|---|---|---|---|---|
| 1 | 74643049 | Cigna Great-West (use CIGNA ST) | 610069179138 | 5/05 | 1 (Salawu) | $0 | ✅ Remit created + claim attached. Adjudication was locked (Completed status) — no line entry needed. |
| 2 | 74643107 | Cigna Great-West | 610069179179 | 5/05 | 1 (Olmos Vivanco) | $0 | ⏳ Pending — likely Completed status like #1 |
| 3 | 74658469 | BCBS | C26125N55456530 | 5/05 | 2 (Boorman x2) | $0 | ✅ Already in Fusion (green dot) — no action |
| 4 | 74658478 | BCBS | C26125N55460290 | 5/05 | 27 | $0 | ✅ Done by Claude session 2026-05-12 |
| 5 | 74695046 | BCBS | C26125E06146020 | 5/06 | 1 (Cook) | $66.60 | ✅ Done with line amounts: CO-45 $16, Coins $7.40, Paid $66.60 |
| 6 | 74697990 | BCBS | C26125E06215560 | 5/06 | 3 (Boorman x2 + Montemayor) | $171.00 | ✅ Done — each claim: CO-45 $18, PR-3 $15, Paid $57 |
| 7 | 74703193 | BCBS | C26125E06374300 | 5/06 | **103** | $5,750.17 | ⏳ Pending — massive, save for dedicated session |
| 8 | 74746318 | BCBS | C26125N56078610 | 5/06 | 23 (reversals) | $0, -$163.77 prov adj | ⛔ SKIP — reversal EOB. Darrell will do manually. |
| 9 | 74740729 | American Specialty Health | 109701611 | 5/06 | 1 (Saunders Grayson) | $0 | ⏳ Pending — remit created in session, claim not yet linked |
| 10 | 74921362 | American Specialty Health | 109701283 | 5/06 | 2 (Jackson Ruey, has PI-144 Incentive) | $39.03 | ⏳ Pending — has unfamiliar PI-144 code |
| 11 | 74685944 | Cigna Great-West | 6038012943143 | 5/06 | 13 (multiple patients incl PR-187) | $843.20 | ⏳ Pending — has unfamiliar PR-187 code |
| 12 | 74921450 | American Specialty Health | 109708176 | 5/06 | 5 (Aretha, Harris, Vorcannon + Incentives) | $66.96 | ⏳ Pending — has PI-144 Incentive |

### Per-claim details for pending EOBs (for re-entry in fresh session)

**EOB #9 — ASH NonPay 109701611 — 1 claim**
- Saunders, Grayson | ACNT P00058915629 → Fusion 58915629 | DOS 4/22/2026 | 97530 | Billed 80, Allowed 0, Deduct 58.64, CO-45 21.36, Paid 0

**EOB #10 — ASH Check 109701283 — 2 claims (same patient)**
- Jackson, Ruey | ACNT P00058870807 → Fusion 58870807 | Insured CLOUD, DYMOND | DOS 4/22/2026 | 97530 | Billed 80, CO-45 21.36, PR-3 20, Paid 38.64
- Jackson, Ruey | same ACNT, 2nd line | DOS 4/22/2026 | "1 INCENTIVE" | Billed 0, PI-144 -0.39, Paid 0.39 (negative adj = bonus)

**EOB #11 — Cigna Great-West EFT 6038012943143 — 13 claims, $843.20 paid**
| Patient | ACNT | DOS | Proc | Billed | Allowed | Deduct | Coins | CO-45 | Other | Paid |
|---|---|---|---|---|---|---|---|---|---|---|
| BALA, DASHWIN | P00058828896 | 4/21 | 92507 | 90 | 68 | 0 | 13.60 | 22 | - | 54.40 |
| LADOUCEUR, ROBERT M | P00059101580 | 4/29 | 92507 | 90 | 68 | 0 | 13.60 | 22 | - | 54.40 |
| GORAPLLA, DANIEL | P00059170565 | 4/22 | 92507 | 90 | 68 | 0 | 0 | 22 | - | 68 |
| STEWART, JORDAN N | P00059168715 | 4/30 | 92507 | 90 | 68 | 0 | 6.80 | 22 | - | 61.20 |
| SALEEM, ABILEAH | P00059164720 | 4/22 | 92507 | 90 | 68 | 0 | 6.80 | 22 | - | 61.20 |
| MANASAN, MADELINE | P00059130180 | 4/22 | 92507 | 90 | 68 | 0 | 0 | 22 | - | 68 |
| MARDUYEV (or similar) | P00059172670 | 4/30 | 92507 | 90 | 68 | 68 | 0 | 22 | PR-187 -68 | 68 |
| AGARKURE, JIBREEL | P00059288212 | 5/02 | 92507 | 90 | 68 | 0 | 0 | 22 | - | 68 |
| AGARKURE, JIBREEL (2nd) | P00059298212 (verify) | 5/02 | 92507 | 90 | 68 | 0 | 0 | 22 | - | 68 |
| SALAMU, JAMIL | P00059208145 (verify) | (verify) | 92507 | 90 | 68 | 68 | 0 | 22 | - | 0 |
| CAIRO | P00059211177 | (verify) | 92507 | 90 | 68 | 0 | 6.80 | 22 | - | 61.20 |
| BALA, DASHWIN (2nd) | P00058918426 | 4/22 | 92507 | 90 | 68 | 0 | 13.60 | 22 | - | 54.40 |

Note: Names may be slightly misread from OCR — verify against the actual PDF before entry.

**EOB #12 — ASH Check 109708176 — 5 claims, $66.96 paid**
- ARETHA, AURORA | ACNT P00058853016 | DOS 4/22 | 97530 | Billed 80, CO-45 21.36, PR-3 45, Paid 13.64
- ARETHA, AURORA | same ACNT | 4/22 | "1 INCENTIVE" | Billed 0, PI-144 -0.14, Paid 0.14
- HARRIS, CALVIN | ACNT P00058832082 | Insured HARRIS, CRYSTAL | DOS 4/26 | 97530 | Billed 80, Coins 5.86, CO-45 21.36, Paid 52.78
- HARRIS, CALVIN | same ACNT | 4/26 | "1 INCENTIVE" | Billed 0, PI-144 -0.40, Paid 0.40
- VORCANNON, WESTON | ACNT P00059014061 | Insured VORCANNON, RUSSETT | DOS 4/28 | 97530 | Billed 80, Coins 5.86, PR-3 (verify), CO-45 21.36, Paid 58.64

**EOB #2 — Cigna Great-West NonPay 610069179179 — 1 claim**
- Olmos Vivanco, Martin | ACNT P00059180521 → Fusion 59180521 | DOS 4/30/2026 | 92507 GN | Billed 90, Allowed 68, Deduct 68, CO-45 22, Paid 0
- Likely Completed status (similar to Salawu) → just create remit + attach, line entry locked

**EOB #7 — BCBS EFT C26125E06374300 — 103 claims, $5,750.17 paid**
Too large to enumerate here. Source PDF pages 10-21 of `05062026_05072026.pdf`. Multi-page EOB with many Brewer Derrick claims plus other patients. Recommend dedicated session.

## How to resume in a fresh session

1. Open Fusion in Chrome with Claude for Chrome extension active
2. Log into Fusion as Mary
3. Navigate to Billing → Remittances → Received
4. Tell Claude: "Read `Reference/Fusion EOB Entry Workflow.md` from dwc-ops, then resume the pending EOBs from file `05062026_05072026.pdf`."
5. Claude will pick up where this session left off

## Source files

- EOB PDF: `C:\Users\Owner\OneDrive\Desktop\Claude COWork\05062026_05072026.pdf`
- Earlier EOB spreadsheet (for reference on format): `C:\Users\Owner\OneDrive\Desktop\Claude COWork\test eob v2.xlsx`
