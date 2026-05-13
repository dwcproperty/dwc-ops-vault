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

| EOB line value | Fusion field | When |
|---|---|---|
| Billed | Charges Submitted | Always (auto-filled) |
| NOS | Units Submitted | Always (auto-filled — usually 1 but sometimes 2) |
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
| **PR-187** | Health Savings Account payment (Cigna Great-West) | **Unknown — Darrell handles manually** |
| **PI-144** | Incentive adjustment (American Specialty Health) | **Unknown — Darrell handles manually** |
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

### 4. Each "+ Claim" cycle may reset filters
After linking one claim, opening "+ Claim" again to add the next one may show a fresh dialog with default filters. Re-enable Patient filter via the funnel each time if needed.

## Step-by-step (concrete)

### A. Create new remittance
1. Billing → Remittances → Received
2. Click "+ Remittance" (bottom-right)
3. Fill: Payer (search), Date (today), Amount ($ from EOB), Type (EFT), Number (NonPay/EFT #)
4. Leave Billing Provider and Identifier Type blank
5. Click "Save Remittance"

### B. Attach claims to remittance
1. Click on the new remittance row to open it
2. Click "+ Claim" (bottom-right)
3. Open funnel → enable "Patient" filter
4. Click "Patient" chip → search by last name → check patient → Apply
5. Check the box on the correct claim row (match Date of Service to EOB)
6. Click "Link claims to remittance"

### C. Enter line amounts (Edit Adjudication)
1. Click on the attached claim row
2. For each Line tab: fill in Not Allowed CO 45, Deductible PR 1, Co-Insurance PR 2, Co-Pay PR 3, Charges Paid as applicable
3. Verify line shows "Balanced" badge
4. After all lines balanced, header shows "$0.00 of $X.XX Remaining"
5. Click "Save Adjudication"

### D. Finalize the remittance (when all claims done)
- When every claim is balanced/finalized and Remittance shows $0.00 of $X.XX Remaining, click "Finalize" (bottom-right)

## Session log — file `05062026_05072026.pdf` (12 EOBs)

### Session 2026-05-12 with Claude

**Completed (5 EOBs in Fusion):**

| # | Report ID | Payer | Remit # | Amount | Status |
|---|---|---|---|---|---|
| 4 | 74658478 | BCBS | C26125N55460290 | $0 | ✅ Done (27 claims entered earlier in session) |
| 5 | 74695046 | BCBS PPO | C26125E06146020 | $66.60 | ✅ Done — Cook Isaiah, CO-45 $16, Coins $7.40, Paid $66.60 |
| 6 | 74697990 | BCBS PPO | C26125E06215560 | $171.00 | ✅ Done — 3 claims (Boorman x2 + Montemayor), each $57 paid |
| 9 | 74740729 | American Specialty Health | 109701611 | $0 | ✅ Done — Saunders Grayson, CO-45 $21.36, Deduct $58.64 |
| 1 | 74643049 | Cigna (CIGNA ST) | 610069179138 | $0 | ⚠️ Remittance created + claim attached (Salawu Jamal claim was already Completed status, line amounts couldn't be entered) |

**Already in Fusion before session (no action needed):**

| # | Report ID | Payer | Remit # | Status |
|---|---|---|---|---|
| 3 | 74658469 | BCBS | C26125N55456530 | ✅ Already entered (green dot in Received list) |

### SKIPPED EOBs — needs manual entry (8 EOBs)

These were skipped during the 5/12 session due to unfamiliar codes, scope, or other issues. **Each requires manual entry in Fusion by Darrell:**

| # | Report ID | Payer | Remit # / Check # | Amount | Reason skipped |
|---|---|---|---|---|---|
| 2 | 74643107 | Cigna Great-West | 610069179179 | $0.00 | Likely Completed-status claim (Olmos Vivanco). Create remit + attach claim, line entry will be locked. 1 claim only. |
| 7 | 74703193 | BCBS PPO | C26125E06374300 | $5,750.17 | **Massive — 103 claims across 12 PDF pages.** Many Brewer Derrick claims plus others. Recommend dedicated session. |
| 8 | 74746318 | BCBS | C26125N56078610 | $0.00 (-$163.77 prov adj) | Reversal EOB (STATUS CODE 22). Darrell handling manually per direction. |
| 10 | 74921362 | American Specialty Health | 109701283 | $39.03 | 2 claims for Jackson Ruey, includes **PI-144 Incentive** code (-$0.39, unfamiliar). Manual. |
| 11 | 74685944 | Cigna Great-West | 6038012943143 | $843.20 | 13 claims, includes **PR-187 HSA payment** code (-$68, unfamiliar). Manual. |
| 12 | 74921450 | American Specialty Health | 109708176 | $66.96 | 5 claims (Aretha, Harris, Vorcannon), includes **PI-144 Incentive** code (unfamiliar). Manual. |

**Total skipped dollar value:** ~$6,699.33 (sum of #7, #10, #11, #12 paid amounts; #2 and #8 are $0/negative).

### Notes for the manual-entry skips

**EOB #2 (Olmos Vivanco, Cigna $0):**
- Patient: Olmos Vivanco, Martin | ACNT P00059180521 → Fusion claim 59180521 | DOS 4/30/2026 | 92507 GN | Billed 90, Allowed 68, Deduct 68, CO-45 22, Paid 0
- Plan Type: LOCAL PLUS
- Expected: claim will be in Completed status (like Salawu), so just create remit + attach. No line entry needed.

**EOB #10 (Jackson Ruey, ASH $39.03):**
- Both claims same patient, same DOS, same ACNT
- Line 1 — 97530: Billed 80, CO-45 21.36, PR-3 (copay) 20, Paid 38.64
- Line 2 — "1 INCENTIVE": Billed 0, PI-144 -0.39 (negative = bonus), Paid 0.39
- The Incentive line might need to be entered via the Adjustments section at the bottom of the Edit Adjudication form

**EOB #11 (Cigna Great-West $843.20, 13 claims):**

| Patient | ACNT | Fusion claim # | DOS | Proc | Billed | Allowed | Deduct | Coins | CO-45 | PR-187 | Paid |
|---|---|---|---|---|---|---|---|---|---|---|---|
| BALA, DASHWIN | P00058828896 | 58828896 | 4/21 | 92507 | 90 | 68 | 0 | 13.60 | 22 | - | 54.40 |
| LADOUCEUR, ROBERT M | P00059101580 | 59101580 | 4/29 | 92507 | 90 | 68 | 0 | 13.60 | 22 | - | 54.40 |
| GORAPLLA, DANIEL | P00059170565 | 59170565 | 4/22 | 92507 | 90 | 68 | 0 | 0 | 22 | - | 68 |
| STEWART, JORDAN N | P00059168715 | 59168715 | 4/30 | 92507 | 90 | 68 | 0 | 6.80 | 22 | - | 61.20 |
| SALEEM, ABILEAH | P00059164720 | 59164720 | 4/22 | 92507 | 90 | 68 | 0 | 6.80 | 22 | - | 61.20 |
| MANASAN, MADELINE | P00059130180 | 59130180 | 4/22 | 92507 | 90 | 68 | 0 | 0 | 22 | - | 68 |
| MARDUYEV (verify) | P00059172670 | 59172670 | 4/30 | 92507 | 90 | 68 | 68 | 0 | 22 | -68 | 68 |
| AGARKURE, JIBREEL | P00059288212 | 59288212 | 5/02 | 92507 | 90 | 68 | 0 | 0 | 22 | - | 68 |
| AGARKURE, JIBREEL (#2) | (verify) | (verify) | 5/02 | 92507 | 90 | 68 | 0 | 0 | 22 | - | 68 |
| SALAMU, JAMIL (verify) | P00059208145 | 59208145 | (verify) | 92507 | 90 | 68 | 68 | 0 | 22 | - | 0 |
| CAIRO | P00059211177 | 59211177 | (verify) | 92507 | 90 | 68 | 0 | 6.80 | 22 | - | 61.20 |
| BALA, DASHWIN (#2) | P00058918426 | 58918426 | 4/22 | 92507 | 90 | 68 | 0 | 13.60 | 22 | - | 54.40 |

Names may have OCR errors — verify against the PDF (pages 27-28 of `05062026_05072026.pdf`).

**EOB #12 (ASH $66.96, 5 claims):**
- ARETHA, AURORA | ACNT P00058853016 → 58853016 | DOS 4/22 | 97530 | Billed 80, CO-45 21.36, PR-3 45, Paid 13.64
- ARETHA, AURORA | same | DOS 4/22 | "1 INCENTIVE" | Billed 0, PI-144 -0.14, Paid 0.14
- HARRIS, CALVIN | ACNT P00058832082 → 58832082 | Insured HARRIS, CRYSTAL | DOS 4/26 | 97530 | Billed 80, Coins 5.86, CO-45 21.36, Paid 52.78
- HARRIS, CALVIN | same | DOS 4/26 | "1 INCENTIVE" | Billed 0, PI-144 -0.40, Paid 0.40
- VORCANNON, WESTON | ACNT P00059014061 → 59014061 | Insured VORCANNON, RUSSETT | DOS 4/28 | 97530 | Billed 80, Coins 5.86 (verify), CO-45 21.36, Paid 58.64

**EOB #7 (BCBS PPO $5,750.17, 103 claims):**
- Too large to enumerate here. Source PDF pages 10-21 of `05062026_05072026.pdf`.
- Multiple multi-line claims (97530+99212 combos for some patients) plus many single 92507 claims.
- Most claims show normal CO-45 + Deductible/Coinsurance/Paid pattern.
- Recommend dedicated session of 1-2 hours to grind through systematically.

## How to resume in a fresh session

1. Open Fusion in Chrome with Claude for Chrome extension active
2. Log into Fusion as Mary
3. Navigate to Billing → Remittances → Received
4. Tell Claude: "Read `Reference/Fusion EOB Entry Workflow.md` from dwc-ops and resume the SKIPPED EOBs from file `05062026_05072026.pdf`."
5. Claude will pick up where this session left off — or you can install the `fusion-eob-entry` skill (saved to `Claude COWork\fusion-eob-entry-SKILL.md`) for automatic workflow recall.

## Source files

- EOB PDF: `C:\Users\Owner\OneDrive\Desktop\Claude COWork\05062026_05072026.pdf`
- Earlier $0 EOB spreadsheet (reference): `C:\Users\Owner\OneDrive\Desktop\Claude COWork\test eob v2.xlsx`
- Skill file: `C:\Users\Owner\OneDrive\Desktop\Claude COWork\fusion-eob-entry-SKILL.md`
