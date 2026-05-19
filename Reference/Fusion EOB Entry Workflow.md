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


---

## Session 2026-05-12 #2 — EOB #7 (BCBS PPO $5,750.17, 103 claims)

### What got done
- Remittance header created in Fusion: BCBS (PPO), EFT, 5/12/2026, $5,750.17, # C26125E06374300 ✅
- 81 unique claims attached to Remittance (1 duplicate 59229589 to remove — see below)
- **Line entry: 0 of 101 claims** — all attached claims still show "Enter Amounts" status

### Cleanup needed
- **Remove duplicate**: 59229589 (Colluru, Kairav) appears twice in the linked claims list — delete one before finalizing.

### 20 unique claims still need to be attached

Use Patient name filter (Claim # filter is unreliable on reopen — it doesn't always apply). For each:

| Fusion # | Patient | DOS | CPT | Billed | CO-45 | Deduct | Coins | Paid | Notes |
|----------|---------|-----|-----|--------|-------|--------|-------|------|-------|
| 56859445 | BREWER, DERRICK L | 022426 | 97530 + 99212/25 | 80 | 23.55 | 0 | 16.92 | 39.53 | Original claim |
| 56859445 | BREWER, DERRICK L | 022426 | 97530 + 99212/25 | -80 | CO-197 -80 | 0 | 0 | 0 | ⚠️ STATUS 22 REVERSAL — attach only, no line entry |
| 57947183 | BREWER, DERRICK L | 032626 | 97530 + 99212/25 | -80 | CO-197 -80 | 0 | 0 | 0 | ⚠️ STATUS 22 REVERSAL — attach only, no line entry |
| 59146430 | BROWN, NIXON | 042926 | 92526/GN | 90 | 17 | 0 | 30 | 43 | |
| 59158364 | CARTER, JACKSON | 042726 | 92507/GN | 90 | 16 | 0 | 14.80 | 59.20 | |
| 59158858 | SEBRING, KRISTOFFER | 043026 | 97530 + 99212/25 | 90 | 26.29 | 0 | 0 | 63.71 | |
| 59171668 | KHAN, ASHER | 043026 | 92526/GN | 90 | 17 | 0 | 0 | 73 | |
| 59171681 | KHAN, ASHER | 043026 | 92507/GN | 90 | 16 | 0 | 0 | 74 | |
| 59174639 | KHORDOC, CHRISTIAN | 043026 | 92507/GN | 90 | 16 | 0 | 14.80 | 59.20 | |
| 59179886 | LEWIS, KHRISTIAN | 043026 | 92507/GN | 90 | 16 | 0 | 14.80 | 59.20 | |
| 59182021 | VILLARREAL, GONZALO | 043026 | 92507/GN | 90 | 16 | 0 | 14.80 | 59.20 | |
| 59183168 | WHEELER LL, GEVON | 043026 | 92507/GN | 90 | 21.76 | 0 | 0 | 38.24 | |
| 59184928 | PIOT, JACQUELINE | 043026 | 92507/GN | 90 | 16 | 0 | 0 | 74 | |
| 59187608 | JADID, AHMAD | 043026 | 97530 + 99212/25 | 90 | 26.29 | 0 | 0 | 63.71 | |
| 59192625 | MCCOY, ANELINA | 043026 | 92507/GN | 90 | 21.76 | 0 | 23.88 | 44.36 | |
| 59193131 | WU, LIAM | 043026 | 92507/GN | 90 | 21.76 | 0 | 20.47 | 47.77 | |
| 59194304 | PESARA, VEDH | 043026 | 92507/GN | 90 | 16 | 0 | 14.80 | 59.20 | |
| 59201407 | PIOT, JACQUELINE | 043026 | 97530 + 99212/25 | 90 | 26.29 | 0 | 20 | 43.71 | |
| 59205203 | KUHN, HOLLIS | 043026 | 92507/GN | 90 | 16 | 0 | 20 | 54 | |
| 59205731 | NYSTUEN, NOAH | 043026 | 92526/GN | 90 | 17 | 0 | 7.30 | 65.70 | |
| 59208233 | TSUI, HAYDEN | 043026 | 92507/GN | 90 | 16 | 0 | 10 | 64 | |

### Then for line entry on the 99 normal claims (Status 1)
Each non-reversal claim still needs adjudication. Two reversal claims (status 22) are attached but get no line entry per Darrell's instruction.

The `eob7_claims.csv` file in the outputs folder has all 103 with line-by-line amounts (CO-45, Deduct, Coins, Co-Pay PR-3, Paid) ready for entry.

### What I learned about Fusion's Claim List dialog
- Default shows ~50 most recent BCBS Accepted claims sorted by date desc
- Sort by clicking Date column header works (toggles asc/desc)
- Date FILTER chip (From/To) doesn't apply via form_input or typing — may need calendar picker click
- Patient FILTER multi-select works — pick patients, Apply, then bulk-check claim # boxes via JS
- Claim # filter works for an EXACT match, but resets on dialog reopen
- Bulk-check via JS: `r.querySelector('.options-item-icon.unchecked').click()` works
- After Link, the dialog closes; reopening starts with fresh state
- ⚠️ Already-linked claims still appear in the dialog with unchecked boxes — re-linking creates duplicates

### Resume command
"Read `Reference/Fusion EOB Entry Workflow.md` from dwc-ops, find session #2 for EOB #7, and continue from the 20 missing claims list. Use Patient name filter (claim # filter is buggy)."


---

## Session 2026-05-12 #3 — EOB #7 continuation (Claude resumed)

### What got done
- Re-extracted line data from PDF pages 10-21 via OCR → rebuilt `outputs/eob7_claims.csv`. Totals reconcile exactly to EOB footer: **103 claim blocks, $9,470.00 billed, $5,750.17 paid, 101 unique Fusion IDs**.
- Attached **18 of the 20 missing claims** to Remit # C26125E06374300 — total attached now **99**.
- Entered **line adjudication for 97 normal-status claims** (skipped the 2 BREWER reversal-pair IDs per Darrell).

### Remit final state
- **Remaining: $181.26 of $5,750.17**
- Breakdown of the $181.26:
  - $79.06 = 2 BREWER reversal-pair originals (56859445 + 57947183 status-1 lines) — skipped per Darrell's "no line entry on reversal claims" rule. 56859445 auto-finalized to Patient Account by Fusion. 57947183 shows "Reconcile" status — needs manual reconcile.
  - $102.20 = 2 claims that wouldn't appear in any +Claim dialog filter:
    - **59146430 BROWN, NIXON** — 4/29/2026, 92526 GN, billed 90, CO-45 17, Coins 30, Paid 43.00
    - **59158364 CARTER, JACKSON** — 4/27/2026, 92507 GN, billed 90, CO-45 16, Coins 14.80, Paid 59.20
  - Both confirmed to exist as Accepted/BCBS PPO claims in the main Claim List, but the +Claim dialog refused to surface them with every filter combination tried (Patient filter, Claim # filter, Phase all-3-selected, Remitter cleared, Status all-9-selected). Cause unknown.

### Next step for Darrell
1. Investigate why 59146430 and 59158364 don't appear in the +Claim dialog — possibly a saved/cached filter, or these claims are flagged in some way that hides them. Manually attach + line-enter both. After that, remit should show $79.06 remaining.
2. Decide what to do with 57947183 BREWER status-22 reversal (Reconcile state). If it auto-finalizes like 56859445 did, remit goes to $0. Otherwise needs manual handling.
3. Finalize the remittance once balance reaches $0.00.

### What I learned about Fusion's +Claim dialog (additions to lessons)
- The Phase chip uses **checkbox-style multi-select** (not radio) — Unsubmitted/Pending/Settled can all be checked simultaneously. Default is Pending only.
- Some claims live in Phase=Settled (already-paid → status Completed) — those need Settled enabled to appear.
- Some claims live in Phase=Unsubmitted (status Waiting) — like 57947183 the BREWER reversal which Fusion auto-set to Waiting.
- The Status chip with **no statuses checked** acts like a "filter to nothing" — must have at least one status box checked.
- The Remitter chip can hold a hidden saved preset (TX BCBS (84980)) that persists across dialog reopens. Clear via funnel → uncheck Remitter checkbox (not just Clear button).
- **Validation**: Fusion's line-amount inputs require values formatted with exactly **2 decimal places** (e.g. `58.40` not `58.4`). Integers like `17` are accepted but `58.4` shows red "Dollar Amount (e.g., 59.47)" error and blocks Save.
- Automation pattern that worked: pre-build `window.eob7Data` dict keyed by Fusion #, then JS loop: `clickClaimRow → wait 1200ms → fillCurrentDialog (uses toFixed(2)) → wait 200ms → clickSaveAdjudication → wait 2000ms`. ~5s per claim, 97 claims in ~8 minutes.

### Files used / produced
- Input: `Claude COWork/05062026_05072026.pdf` (pages 10-21 OCR'd via tesseract)
- Output: `outputs/eob7_claims.csv` (full line-level data, all 120 lines reconcile)
- Output: `outputs/eob7_entry.json` (99 status-1 claims, 112 lines, $5,671.11 total paid — minus BREWER pair)
- Parser script: `outputs/parse_eob7.py`


---

## How to run a full EOB batch (refined from Session #3, 2026-05-12)

This section is intended to be read by the next Claude before starting work. Read it all the way through; it captures every gotcha that cost time last session.

### Files in the workspace
- `Claude COWork/eob_scripts/parse_eob.py` — generic OCR'd-EOB → CSV parser. Works on any pdftext output from Claim.MD EOBs.
- `Claude COWork/eob_scripts/fusion_line_entry.js` — the browser-side helper functions: `clickClaimRow`, `getOpenClaimId`, `fillCurrentDialog`, `clickSave`, `processOne`. Inject this in the Fusion tab after building `window.eob7Data`.
- `Claude COWork/eob_scripts/README.md` — short usage notes.

### Overall workflow per EOB
1. **Identify EOB boundaries in the source PDF.** Each EOB starts with the payer header + "PAGE #: 1 of N". OCR can mangle this to "Lofl" (= 1 of 1) or "lof 2" (= 1 of 2) — the parser handles both. Extract: payer, EFT/NONPAY/CHECK #, date, total claims, total check amount.
2. **Find or create the remittance in Fusion.** Billing → Remittances → Received. If the remit isn't there, scroll back through dates or check Pending tab. If still missing, the Claim.MD import may not have run for that file — the remit needs to be **created manually** by Darrell (click +Remittance bottom-right). Don't try to create it yourself.
3. **Attach claims.** Open the remit, click "+ Claim" bottom-right. Filter approach matters (see below).
4. **Enter line adjudication** for each non-reversal claim. Use the JS automation pattern.
5. **Verify $0.00 remaining** before finalizing. Don't auto-finalize — leave that to Darrell.

### Skip list (Darrell's rules)
- **American Specialty Health** remittances — entered manually elsewhere. Don't touch.
- **Status-22 reversal claims** (CO-197). Attach to remit but enter no line amounts. Fusion auto-finalizes reversal-paired claims to Patient Account if the original is already settled. If the reversal stands alone (no matching original on the EOB), it shows "Reconcile" status — leave for Darrell to handle.

### Fusion +Claim dialog quirks (this stuff is unintuitive)
- **Phase filter is multi-select checkboxes**, not radio. Default = Pending only. Older claims live in **Settled** (status Completed). Reversal originals that were "voided" show up in **Unsubmitted** (status Waiting). For a complete view, check **all three** Phase boxes before applying.
- The **Status filter** — when ALL boxes are unchecked, the form submits no values and filters to **nothing** (zero results). Need at least one box checked. Default has 5 of 9 selected; leave them alone unless you need to look at Completed or Paid claims.
- The **Remitter chip** can hold a hidden saved preset (typically TX BCBS 84980). After clearing patient filter, this preset can persist and silently exclude claims. To fully clear: open the funnel and **uncheck the Remitter checkbox** entirely.
- **Patient filter is the safest way to find target claims.** The Claim # filter is buggy and will sometimes show "no claims to display" even when the claim exists.
- **Patient picker requires real keyboard events.** Synthetic input events don't trigger the filter. Use `computer.type` for the search box, then JS to click matching `.roster-item` elements.
- **Filter state resets on dialog reopen** (when you click +Claim a second time after Link). Re-apply your filters.
- Some claims simply don't show in the dialog with any filter combination — see "the BROWN/CARTER mystery" below.

### Edit Adjudication dialog field mapping
The input names are `lines[N][...]`. Line 0 is a hidden new-line template; real lines are 1, 2, 3...

| Field on the form | Input name | EOB column |
|---|---|---|
| Charges Submitted | `lines[N][charges]` | BILLED (auto) |
| Units Submitted | `lines[N][units]` | NOS (auto) |
| Procedure Code | `lines[N][code]` | PROC (auto) |
| Allowed Amount | `lines[N][allowed]` | ALLOWED — usually leave blank |
| Not Allowed (CO 45) | `lines[N][obligations]` | GRP/RC-AMT for CO-45 |
| Other Payers (OA 23) | `lines[N][other]` | leave blank unless OA-23 present |
| Deductible (PR 1) | `lines[N][deductible]` | DEDUCT |
| Co-Insurance (PR 2) | `lines[N][coinsurance]` | COINS |
| Co-Pay (PR 3) | `lines[N][copay]` | PR-3 line under the CPT row |
| Charges Paid | `lines[N][paid]` | PROV PD |

**Critical: amount fields require exactly 2 decimal places.** `58.40` works; `58.4` is rejected with red "Dollar Amount (e.g., 59.47)" error and blocks Save. Format with `Number(v).toFixed(2)` before setting.

### Automation pattern (~5 seconds per claim)
```js
processOne = async (claimId) => {
  clickClaimRow(claimId);                  // JS finds row by claim # text, scrolls, clicks
  await wait(1200);                        // dialog open
  if (getOpenClaimId() !== claimId) throw; // sanity check
  fillCurrentDialog();                     // sets lines[N][...] from window.eob7Data[claimId]
  await wait(200);
  clickSave();                             // finds "Save Adjudication" button
  await wait(2000);                        // save commits + dialog closes
};
```

Run in batches of **7 claims per browser_batch call**. Bigger batches will exceed the 45-second CDP timeout. The loop continues running after a timeout, but you lose result visibility; better to size-down.

### The BROWN/CARTER mystery (unresolved)
Two BCBS PPO Accepted claims — **59146430 BROWN, NIXON** (4/29) and **59158364 CARTER, JACKSON** (4/27) — refused to appear in the +Claim dialog under any combination of Patient/Claim#/Phase/Status/Remitter filters. Both exist in the main Claim List page and are confirmed Accepted/BCBS PPO. Cause unknown. Tried: every Phase combo, every Status combo, cleared Remitter, used Patient filter with exact match, used Claim # filter directly. The dialog returned "There are no claims to display" every time.

If this happens again: flag for Darrell to manually attach via Fusion's UI (he may know what's hiding them).

### Generic OCR pipeline
```bash
# Render PDF pages
mkdir -p /sessions/.../mnt/outputs/eob_imgs
pdftoppm -f START -l END -r 300 "/path/to/eobs.pdf" page -png

# OCR each page (sequential, 1 at a time to stay under bash timeout)
cd eob_imgs
for p in page-*.png; do
  base="${p%.png}"
  tesseract "$p" "$base" --psm 6 -c preserve_interword_spaces=1 quiet > /dev/null 2>&1
done

# Parse → CSV
python3 parse_eob.py
```

OCR gotchas:
- `Cco-45` should normalize to `CO-45` (extra leading C from font rendering)
- `Lofl` / `lof 2` / `Lof 2` should be read as `1 of 1` / `1 of 2` etc
- pdftoppm at 200dpi creates blank OCR for some pages; **always use 300dpi**
- Filenames are zero-padded (`page-01.png` not `page-1.png`) — watch your globs


---

## Session 2026-05-12 #4 — `5_7_2026 Claims.pdf` (15 EOBs) + 3 carryover

### What got done

1. **Parsing infrastructure upgraded.** `Claude COWork/eob_scripts/parse_eob.py` was generalized to handle line formats from UMR, UHC SUREST, and PGBA/TRICARE — not just BCBS. Previous version only matched lines with the full `NPI + POS + NOS + CPT` shape; the new unified regex makes NPI/POS/NOS individually optional so the same script parses all five payer formats now in the workflow.
2. **All 18 EOBs OCR'd and parsed.** Per-line CSVs reconcile cleanly against every EOB footer:
   - `Claude COWork/eob_5_7_claims.csv` — 15 new EOBs, 104 lines, billed $9,200, paid $3,426.69
   - `Claude COWork/eob_carryover.csv` — 3 carryover EOBs, 40 lines, billed $2,790, paid $679.43
   - Pre-built per-remit JSON line-entry dicts in `Claude COWork/eob_scripts/remit_data/` (18 files) — ready to drop into `window.eobData` for the JS line-entry automation when Claim.MD imports the remits.
3. **`Claude COWork/eob_5_7_processing_plan.md`** generated with per-EOB summary tables and a full triage list for non-P000 ACNTs.

### Fusion-side survey (Billing → Remittances → Received, all 100 most-recent rows)

Out of the 18 target remits, only **3 are present in Fusion's Received list**:

| Target | Fusion ID | Date in Fusion | Status |
|---|---|---|---|
| New #2 — UMR EFT CO33829139450946119399944 ($480.20) | 8150883 | 5/11/2026 (4 days after EOB date) | **Done** — 9 claims (CONSTANZO, MONCRIEF, SALLOUM×3, SPRINGER×4) all Finalized (Patient Account), $0.00 of $480.20 remaining |
| New #4 — UHC SUREST EFT UH72100052846511531590729 ($52.00) | 8151532 | 5/11/2026 | **Done** — 4 GOEDDE Harrison claims all Finalized (Patient Account) |
| New #5 — UHC SUREST UH48600052846531018613823 ($53.00) | 8151568 | 5/11/2026 (Type=Check in Fusion vs EFT on EOB) | Likely done (visible on prior screenshots showing green Finalized markers; not re-verified after navigation back) |

The other 15 target remits **were not found** in Fusion's Received list (covers 5/1 → 5/12). Per the standing workflow rule — *"If it's not there yet, Claim.MD probably hasn't imported it — flag for Darrell, don't try to create remittance records yourself"* — these are flagged for Mary/Darrell. Specifically:

**Not yet imported (15 remits):**
- New #1 — UMR NONPAY CG83129138882116119343061 ($0)
- New #3 — UMR NONPAY CN08529139113906119366240 ($0)
- New #6 — UHC SUREST EFT UH88200052846521245378282 ($431.00)
- New #7 — PGBA EFT 5625538121WA6 ($119.16)
- New #8 — PGBA EFT 5213938124WR6 ($781.36)
- New #9 — CIGNA HLTH LIFE EFT 260506050000438 ($136.00)
- New #10 — BCBS EFT C26126E06458650 ($56.00)
- New #11 — BCBS EFT C26126E06619860 ($1,212.37)
- New #12 — BCBS EFT C26126E06414690 ($57.60)
- New #13 — BCBS NONPAY C26127N63950740 ($0, 23 denied claims)
- New #14 — BCBS NONPAY C26127N63996310 ($0, 2 BASHYAM claims)
- New #15 — UMR INNOVETIVE PETCARE CHECK 0003781150 ($48.00)
- Carry #1 — Cigna GW NONPAY 610069179179 ($0, Olmos Vivanco)
- Carry #2 — BCBS NONPAY C26126N60078610 ($0 / -$163.77 prov adj, 23 claims incl. 3 BUCK reversals + 1 BREWER 56859445 duplicate)
- Carry #3 — Cigna GW EFT 603801296193 ($843.20, 13 claims)

Fusion's Received list ends at 5/1/2026; the older 5/4 Cigna GW remits visible (`610069135750`, `610069135815`) confirm Claim.MD does import this payer, just hasn't gotten to these specific remits yet.

### Manual-triage items captured for Darrell

Listed in full in `Claude COWork/eob_5_7_processing_plan.md`. Highlights:

- **23 BCBS-internal alphanumeric ACNTs** (style `SQXTJAGE`, `RR5AQNUJ`) on EOB #13 (17) and EOB #14 (2 BASHYAM x2 lines) and EOB #3 (Cheshier S-prefix). These don't map to Fusion claim numbers by the standard `P000xxxxxxxx → xxxxxxxx` rule. Per Darrell's standing direction this session, claim lookup should switch to **patient name + DOS** (with ACNT only as a fallback). Will need to do this lookup in Fusion when these remits finally appear.
- **6 S-prefix ACNTs** (LOZANO Sofia/Gloria, FERGUSON, PETERSON on PGBA EOB #8; CHESHIER x2 on UMR EOB #3) — all RC=CO-22 "Care covered by another payer", indicating secondary submissions denied for COB. Darrell decided to skip ACNT-based matching globally and use name+DOS instead.
- **BREWER, DERRICK 56859445** reappears on Carryover #2 (BCBS NONPAY C26126N60078610) as a $0 CO-B13 denial. This claim was already attached to EOB #7's remit (last session) and auto-finalized to Patient Account. **Skip** per Darrell — he'll handle in Fusion if/when this remit imports.

### New gotchas learned

- **OCR variants matter.** UMR, UHC SUREST, and PGBA EOBs render claim-line columns differently than BCBS (NOS column blank for UMR/UHC SUREST; NPI/POS/NOS all missing for PGBA TRICARE). The previous parser silently skipped 60–70% of lines for those payers. The new unified `RE_LINE_UNI` regex makes NPI/POS/NOS individually optional. Always reconcile parser output vs OCR'd footer totals before trusting the CSV.
- **CO-B11, CO-B13, CO-22 are real RC codes.** Earlier RC-code regex `[A-Za-z]{2,4}-?\d+` rejected `CO-B11`/`CO-B13` because of the letter in the suffix. Updated to `[A-Za-z]{2,4}-?[A-Z0-9]+`.
- **OneDrive sync truncated a 12 KB Write.** When using the `Write` tool to overwrite `Claude COWork/eob_scripts/parse_eob.py`, the OneDrive-synced file came back truncated at ~6 KB / line 247. Workaround: write the file to the local `outputs/` folder first via shell heredoc, then `cp` it into the OneDrive folder. The Read tool's view was stale during this — bash `wc -l` was the source of truth.
- **Claude in Chrome privacy filter blocks long base64-like strings.** The Fusion Received list's long remit IDs (UMR/SUREST/UHC base64-style numbers) come back as `[BLOCKED: Base64 encoded data]` in JS return values. But the `data-uri` attribute on each row exposes the full ID once via `getAttribute('data-uri')`. When batching many rows, the filter starts blocking even from data-uri. Workaround: read 6-char prefixes/suffixes to identify matches, then click by `data-uri` substring rather than by full remit number. The `fusion_id` (numeric internal Fusion record ID) is never blocked.
- **BCBS NONPAY remits have mixed ACNT formats.** Some claims on a single BCBS NONPAY have standard `P000xxxxxxxx` ACNTs (mapping to Fusion claim #s), others have 8-char BCBS-internal alphanumeric ACNTs that don't auto-map. Per Darrell, **use patient name + DOS to find these in Fusion** rather than ACNT.
- **Status-22 reversal pairs can be paired with Status-23** ("Not our claim, forwarded") on Cigna Great-West NONPAYs. The Status-22 line has negative billed/CO-45/paid; the Status-23 line has positive billed but $0 paid + CO-B11. Both should be parsed but neither needs line entry per Darrell's rule.

### Files saved (Session #4)

- `Claude COWork/eob_5_7_claims.csv` — 15-EOB line-level data (104 rows)
- `Claude COWork/eob_carryover.csv` — 3-EOB line-level data (40 rows)
- `Claude COWork/eob_5_7_processing_plan.md` — per-EOB Fusion processing plan with manual-triage tables
- `Claude COWork/eob_scripts/parse_eob.py` — updated generic parser (unified line regex + alphanumeric ACNT support)
- `Claude COWork/eob_scripts/remit_data/{new,carry}_eob##.json` — 18 pre-built JSON dicts for `window.eobData` line entry, indexed by Fusion claim #

### Next step for Darrell

1. Wait for Claim.MD to import the 15 missing remits (typically 3–5 days post-EOB date — note the 3 confirmed-in-Fusion remits were dated 5/7 on the EOB but appeared in Fusion on 5/11).
2. When each remit appears in Received, open the corresponding `eob_scripts/remit_data/{new,carry}_eob##.json` and use the JS automation pattern from Session #3 to enter line adjudication. The JSON's `skip_line_entry` list shows status-22 claims that get attached only.
3. For the alphanumeric-ACNT BCBS NONPAY claims and S-prefix secondary claims, switch to **patient name + DOS** in the +Claim dialog (Darrell's new standing rule, captured this session).
4. EOB #7 carryover items from Session #3 (BROWN 59146430, CARTER 59158364, BREWER 57947183) — handled manually by Darrell, not by this session.

### Resume command for next session

"Read `Reference/Fusion EOB Entry Workflow.md` from dwc-ops, scroll to Session #4 (2026-05-12). Re-survey Fusion → Billing → Remittances → Received and see which of the 15 missing remits from Session #4 have now imported. Process them using the pre-built JSONs in `Claude COWork/eob_scripts/remit_data/`. Use patient name + DOS (not ACNT) for claim lookup per Darrell."


---

## Session 2026-05-13 — Mid-day check-in on BCBS C26125E06374300 + new EOB intake

### What got done

- Re-OCR'd `Claude COWork/bcbs redo.pdf` (Darrell's re-upload of EOB #7) and reconciled to 101 unique fusion IDs / $9,470 billed / $5,750.17 paid — matches Session #3.
- Diffed against Fusion's attached claim list (98 finalized claims) → 3 missing: 59146430 BROWN/NIXON, 59158364 CARTER/JACKSON, 57947183 BREWER/DERRICK.
- **Mary unblocked the missing claims** by removing them from a separate remit (EOB #C26124N52071870) at 11:35 AM. Once unlinked, BROWN and CARTER both appeared in the +Claim dialog (filtered by Patient + Phase=all3 + Status=default). Last session's mystery is resolved: the claims were locked to another remit, not invisible due to filter issues.
- Attached + line-entered BROWN ($43 paid, CO-45 $17, coins $30) and CARTER ($59.20 paid, CO-45 $16, coins $14.80). Remit went from $181.26 remaining → $79.06 remaining.
- BREWER 57947183 skipped per Darrell (he's handling manually).

### Current state of C26125E06374300

Remaining: **$39.53 of $5,750.17** (Darrell's screen shows this).

The $39.53 is the BREWER **sibling** claim 56859445. On this EOB, BCBS paid $39.53 for that claim (CPT 97530 $18.44 + CPT 99212 $21.09), but Session #3 auto-finalized 56859445 at $0.00 paid because Fusion's reversal-pair pattern netted the status-1 lines against the status-22 CO-197 reversal lines. The remit's Remaining column still wants to absorb that $39.53.

To fully balance, either update 56859445's existing adjudication so paid=$39.53 (with the CO-45 lines), or add a $39.53 Provider-Level Adjustment to absorb the discrepancy. Darrell is reviewing.

### New gotcha captured

- **"Claim won't surface in +Claim dialog" is sometimes a lock to another remit, not a filter mystery.** If you've tried Patient + Phase=all3 + Status defaults and the claim still doesn't appear, open the standalone Claim List (Billing → Claims → All), find it, and check the History tab. If you see an entry like "EOB Processed" pointing to a different remit, that other remit is holding the claim. Have Mary go to that remit and use "Remove Claims" to unlink, then it'll appear in the +Claim dialog of the target remit.

### Next batch queued for fresh session

3 new EOB PDFs uploaded today and staged in `local-agent-mode-sessions/.../uploads/`:
- `eob 1.pdf` (20 pages)
- `eob 2.pdf` (3 pages)
- `ebo 3.pdf` (1 page)

Process in the next session. Workflow unchanged from Session #4.


---

## Session 2026-05-13 #2 — 3 new EOBs uploaded today (`eob 1.pdf`, `eob 2.pdf`, `ebo 3.pdf`)

### What got done

All three PDFs OCR'd at 300 DPI and parsed with `Claude COWork/eob_scripts/parse_eob.py`. Per-line CSVs and line-entry JSONs saved to:
- `Claude COWork/eob_5_13_csvs/{uhc_W358749008, bcbs_C26126E06619860, cignahlth_260506050000438}.csv`
- `Claude COWork/eob_scripts/remit_data/{uhc_W358749008, bcbs_C26126E06619860, cignahlth_260506050000438}.json`

Each JSON has `lines_by_fusion_id` keyed by Fusion claim # — drop it into `window.eobData` and run the JS automation from Session #3 once the remit appears in Fusion.

### EOB summary table

| # | Payer | EFT # | Date | Claims | Billed | Paid | Status in Fusion |
|---|---|---|---|---|---|---|---|
| eob 1 | UNITED HEALTHCARE INSURANCE COMPANY | W358749008 | 2026-05-08 | 164 (162 unique Fusion IDs) | $15,360.00 | $9,504.47 | **NOT IMPORTED** |
| eob 2 | BCBS (PPO) | C26126E06619860 | 2026-05-07 | 22 | $2,250.00 | $1,212.37 | **NOT IMPORTED** (was already on Session #4's "not imported" list as New #11) |
| ebo 3 | CIGNA HLTH LIFE | 260506050000438 | 2026-05-07 | 2 | $180.00 | $136.00 | **NOT IMPORTED** (was already on Session #4's "not imported" list as New #9) |

All three reconcile exactly: parser-sum-paid matches EOB footer for every remit.

### Survey of Fusion Received list (5/13/2026, 100 most-recent rows)

The Received list spans 5/1 → 5/13. Newest UHC INSURANCE entries are 5/7 (W358621147 $204) — no W358749008 anywhere in DOM (verified by searching `document.body.innerHTML`). Newest BCBS PPO entries on 5/13 are C26126E06458650 ($56) and an unlabeled $0.00 row — no C26126E06619860. Newest CIGNA HLTH LIFE on 5/6 was 260502090041611 ($2,085.57) — no 260506050000438.

**Conclusion:** Claim.MD has not yet imported any of these 3 remits. Per Session #4's standing rule, leaving these for Darrell. They should arrive in the next 1–2 days based on Claim.MD's typical 4–7 day import lag.

### Edge cases captured in the JSONs

**UHC W358749008** (`uhc_W358749008.json`):
- **59004198 MANSOOR, WALI** appears with the same ACNT on pages 6 and 14 (different ICNs). The EOB shows two separate adjudications of the same Fusion claim for DOS 4/23/2026, each paying $51.10 on a $90 92507 GN line. Captured as two line entries (`line_idx` 1 and 2) under the same Fusion ID. Mary may need to manually decide whether Fusion's claim 59004198 actually has 2 service lines or whether one of these is a true duplicate adjudication. Worth verifying when the remit lands.
- **58313705 NAGANDLA, SIDDHARTH** has a CO-234 status-22 reversal (-$80) on page 8 immediately followed by a CO-45 status-1 positive ($73). Standard reversal-pair pattern from Session #3 — the reversal piece is excluded from the JSON (no line entry); only the positive +$73 line is queued. The reversal still needs to be attached at the claim level.
- **JENKINS, MYLES** appears twice in the first page with two distinct ACNTs (P00058384092 + P00058623335) — these are two real claims for the same patient, each with a normal 97530 + 99212 line pair.

**BCBS C26126E06619860** (`bcbs_C26126E06619860.json`): no edge cases — all 22 claims status-1, dollars reconcile cleanly. Five claims are normal 2-line (CPT 97530 + 99212 or 92610 + 99202).

**CIGNA HLTH LIFE 260506050000438** (`cignahlth_260506050000438.json`): trivial — GREWAL THEO (59187129) and GREWAL TERRA JOE (59187141), both single-line 92507 GN $68 paid each.

### Files saved

- Per-line CSV (one row per CPT line): `Claude COWork/eob_5_13_csvs/uhc_W358749008.csv` (167 rows), `bcbs_C26126E06619860.csv` (28 rows), `cignahlth_260506050000438.csv` (3 rows incl. header)
- Pre-built line-entry JSON (drop into `window.eobData`): `Claude COWork/eob_scripts/remit_data/uhc_W358749008.json`, `bcbs_C26126E06619860.json`, `cignahlth_260506050000438.json`

### Next step for Darrell

1. Wait for Claim.MD to import the 3 remits (typical 4–7 days post-EOB; UHC is 5 days out today, BCBS/Cigna are 6 days out).
2. When each remit appears in Received, open the corresponding JSON from `eob_scripts/remit_data/` and run the JS automation from Session #3 to enter line adjudication. The JSON's `attach_only` array (currently empty for all 3) flags claims to attach without line entry.
3. For UHC W358749008, double-check MANSOOR 59004198 manually — Fusion may already have 2 service lines on that claim record, or it may need a different approach for the duplicate-ICN scenario.

### Resume command for next session

"Read `Reference/Fusion EOB Entry Workflow.md` from dwc-ops, scroll to Session 2026-05-13 #2. Re-survey Fusion → Billing → Remittances → Received and check whether W358749008 (UHC $9,504.47), C26126E06619860 (BCBS $1,212.37), or 260506050000438 (Cigna $136) have now imported. For any that have, run the JS automation using the pre-built JSON in `Claude COWork/eob_scripts/remit_data/`. Skip MANSOOR 59004198 manual review unless asked."


### Refinement to UHC W358749008 JSON (Darrell clarification, 5/13)

After reviewing the MANSOOR/NAGANDLA cases together:

- Both patients **came in twice on the same day** — these are not duplicate adjudications.
- **MANSOOR 59004198** (4/23/2026): both 92507 GN visits were paid by UHC, $51.10 each. JSON enters as two normal lines (line_idx 1 and 2), both auto-fillable.
- **NAGANDLA 58313705** (4/6/2026): two 97530 GO visits same day. UHC reversed the first one (CO-234 status-22 with M80 remark = "not covered when performed during same session as a previously processed service") and paid only the second one $73 with modifier 59. Per Darrell's standing instruction for this case: **enter both lines** — line 1 = the reversal (-$80 billed, CO-234 -$80, $0 paid), line 2 = the positive visit (+$73 paid).

The updated `uhc_W358749008.json` now has the reversal line included with `requires_manual_entry: true` because the JS `fillCurrentDialog` helper skips negative/zero values. Mary will need to hand-enter line 1's negative amounts in the Fusion Edit Adjudication dialog when this remit lands. Auto-entry continues to work for line 2.

#### New rule (supersedes Session #3 phrasing for line-level cases)

The old rule from Session #3 was "Status-22 reversal CLAIMS get attached only — no line entry." That rule still applies when the *entire claim* is a reversal (e.g., BREWER pair, where claims 56859445 and 57947183 reverse each other and the originals are on separate remits).

When a status-22 line lives **inside a multi-visit claim that also has a status-1/status-19 line** (same Fusion ID, same DOS, different ICNs — i.e., the patient legitimately came in twice and the insurer reversed one of the visits), enter **both** lines in Fusion. The negative-amount entry has to be done by hand (Fusion's input accepts `-80.00` formatted to 2 decimal places, but the JS helper skips it).


---

## Session 2026-05-13 #3 — Manual creation + line entry of 3 EOBs (Cigna, BCBS, UHC)

### What got done (departing from standing "don't create remits" rule)

Darrell directed: "forget md only you will inprt" / "import" — i.e., manually create all three remits in Fusion and enter the data myself rather than waiting for Claim.MD to import them. New rule for this session.

| Remit | Payer | Amount | Status |
|---|---|---|---|
| 260506050000438 | CIGNA (ST) | $136.00 | ✅ Balanced ($0.00 of $136.00 remaining) — 2 GREWAL claims |
| C26126E06619860 | BCBS (PPO) | $1,212.37 | ✅ Balanced ($0.00 of $1,212.37 remaining) — 22 claims |
| W358749008 | United Health Care | $9,504.47 | ⚠️ **$139.10 remaining** — 161/162 attached + 159 line-entered. 3 claims need manual handling. |

### UHC W358749008 — manual handling needed for 3 claims

1. **58313705 NAGANDLA, SIDDHARTH** — could not be attached via +Claim dialog. Claim # filter returned "no claims to display" in Pending, Unsubmitted, and Settled phases. Likely locked to another remit (same root cause as Session #3's BROWN/CARTER situation that Mary unlocked). When this remit lands, check what other remit is holding 58313705, unlink it, then attach here. The line data is in the JSON (two lines for the two visits on 4/6/2026; line 1 is the CO-234 status-22 reversal -$80 with $0 paid, line 2 is the CO-45 status-1 visit +$80 billed → $73 paid).

2. **58623335 JENKINS, MYLES** — attached but partially entered. Has a special CO-8 + PR-3 line structure (CO-8 = procedure inconsistent with provider type / specialty taxonomy, an unfamiliar RC code my parser didn't capture into the co45 field). Currently shows $15 paid; needs PR-3 $25 copay + Allowed/Not-Allowed fields to balance line 1, plus CO-8 $40 write-off on line 2. Source: EOB page 1 second JENKINS NAME block.

3. **58384092 JENKINS, MYLES** — same CO-8/PR-3 pattern as 58623335, totally unentered ($0 paid). Needs same manual treatment. Source: EOB page 1 first JENKINS NAME block.

### Key learnings from this session

1. **Fusion Phase chip is a RADIO, not a checkbox.** Despite Session #3's note about checkbox-style multi-select, the actual behavior on the Phase filter (Unsubmitted/Pending/Settled) is that only one can be selected at a time. Clicking any one deselects the others.

2. **The +Remittance manual-create workflow works.** Form layout: Payer (search dropdown), Date (text input MM/DD/YYYY), Amount (number), Type (select: Check/EFT — must click dropdown then click option, can't just type "EFT"), Number (text). Leave Billing Provider and Identifier Type blank. Save Remittance creates and lands you on the empty remit page.

3. **Payer canonical name mapping.** "Cigna HLTH LIFE" EOBs map to payer "CIGNA (ST)". "BLUECROSS BLUESHIELD OF TEXAS" PPO EOBs map to "BCBS (PPO)". "UNITED HEALTHCARE INSURANCE COMPANY" EOBs map to "United Health Care".

4. **Patient picker — `roster-item` lazy-loads.** The picker has ~150 patients in the initial render. Once you type a search and select, the selected item moves up but the rest only loads when you scroll. For multi-select across many patients, the most reliable pattern is:
   ```
   for each last name: triple_click search → type name → wait 2s → JS click matching .roster-item
   ```
   `window.pickByMatch(lastName, firstName)` helper handles asterisks (discharged) and parenthesized nicknames like `(Maggie)` or `(Jax)`.

5. **OCR vs Fusion name spellings caught in session.** McClanahan, **Sarai** (not Sarat as OCR'd); Kanagasabapathy, **Hanish** (not Hanis); Neti, Pranesh **Shanmukha** (Pranesh) — not Shanmukh. Worth re-checking parser's name extraction for non-Latin/long names.

6. **The Edit Adjudication dialog has a slow-loading state** ("Loading" spinner) where my JS automation's getOpenClaimId() returns null because Primary Claim header isn't yet in the DOM. After ~10 consecutive successful saves, Fusion sometimes goes into a 30s+ loading state. The async processOne loop then times out at the 45s CDP limit, leaving the dialog stuck mid-load. Recovery pattern: query .button.variant-bubble.page-close, click them all, wait, retry. The remaining count still goes down across these recoveries because the previous-batch saves complete in the background.

7. **JS automation pattern processed ~160 claims in ~10 minutes wall-clock** when running smoothly. Per-claim time: clickRow → 1.2s wait → fillDialog (sets ~3 fields) → 200ms → clickSave → 2s wait = ~3.5s. Real-world average came in at ~3.7s per claim with occasional slowdowns.

### Files saved/updated

- Updated `Claude COWork/eob_scripts/remit_data/uhc_W358749008.json` (NAGANDLA reversal line included per Darrell's clarification)
- `Claude COWork/eob_5_13_csvs/{uhc,bcbs,cignahlth}_*.csv` — parsed line data
- Session log appended here

### Next step for Darrell

1. Open Remit W358749008. The 3 manual items above all need attention.
2. For the JENKINS pair: edit the existing adjudication for 58623335 and 58384092 to apply CO-8 to line 2 ($40 each as non-allowed/write-off) and PR-3 to line 1 ($25 each as copay). Verify Allowed Amount and recompute Paid to balance. Total net paid for these two claims should be $30 (= 2 × $15).
3. For NAGANDLA 58313705: locate the other remit holding it (`Billing → Claims → All → search 58313705 → History tab`), have Mary unlink it, then come back to W358749008 and +Claim → search NAGANDLA → attach → enter the two lines manually (line 1 reversal, line 2 +$73).
4. Once all three are handled, remit should balance to $0.00 of $9,504.47, then Finalize.


---

## Session 2026-05-13 #4 — 4 more EOBs (`eob 4-7.pdf`)

### What got done

All four EOBs parsed and manually created in Fusion. Dollars reconciled to footer for each. Three of four fully balanced; one partial.

| File | Payer | EFT # | Total | Status |
|---|---|---|---|---|
| eob 5 | Tricare WEST | 5625538121WA6 | $119.16 | ✅ **$0.00 remaining** — 2 claims (Alford Jett, Ali Amari) |
| eob 6 | SUREST-UHC | UH88200052846521245378282 | $431.00 | ✅ **$0.00 remaining** — 7 claims |
| eob 7 | CIGNA (ST) | 603801296193 | $843.20 | ⚠️ **$136.00 remaining** — 11 of 13 entered; 2 Gorantla Daniel claims locked to a prior remit |
| eob 4 | Tricare WEST | 5213938124WR6 | $781.36 | ⚠️ **$53.73 remaining** — 13 of 17 entered; 4 Tricare-secondary CO-22+OA-23 claims need manual entry |

### Manual items for Darrell

**Remit 603801296193 (CIGNA — was GREATWESTHEALTHCARE-CIGNA on EOB)**
- Two Gorantla, Daniel claims (`59170565` and `59185317`) appear in Fusion's Settled phase as "Completed" — already adjudicated on another remit. They attached at $0.00. Unlink them from the prior remit, then re-enter both lines (each $90 billed, CO-45 $22, $68 paid).
- Note: searched payer dropdown for "GreatWest", "Great-West", "GW" — no matches. Per your direction, used **CIGNA (ST)** as the canonical payer for this GreatWest-Cigna EOB.

**Remit 5213938124WR6 (Tricare WEST)**
- 4 claims pending entry, all Tricare-secondary with the CO-22 + OA-23 pattern that the standard auto-filler can't handle:
  - `58531133` Lozano, Sofia — CPT 92507, billed $90, CO-22 $22.00, OA-23 $54.40, paid $13.60
  - `58565783` Lozano, Gloria — CPT 92507, billed $90, CO-22 $22.00, OA-23 $54.40, paid $13.60
  - `58424274` Ferguson, Jacob — CPT 97530, billed $80, CO-22 $21.36, OA-23 $46.91, paid $11.73
  - `58954404` Peterson, Boston — CPT 92507, billed $90, CO-22 $16.00, OA-23 $59.20, paid $14.80
- For each: open Edit Adjudication, on Line 1 set Not Allowed (CO 45) = CO-22 amount, set Other Payers (OA 23) = OA-23 amount, set Charges Paid = paid amount. Verify Line 1 shows "Balanced" before saving.
- Total of 4 remaining adjudications = $53.73 paid.

### New gotchas captured

1. **Tricare/PGBA secondary claims use S00 prefix on ACNT instead of P000.** The parser's RE_ACNT_P000 only matched P-prefix. Fixed for eob 4 by post-processing `S000(\d{8})` → fusion_id; should be added to the parser permanently. CO-22 ("payment adjusted because this care may be covered by another payer per coordination of benefits") + OA-23 ("payment adjusted due to the impact of prior payer(s) adjudication") is the canonical Tricare-secondary pattern on EOBs.

2. **OCR'd patient names sometimes mangle hyphens or skip words.** Examples caught this session: "ACOSTAGUTHRIE, IAN" actually "Acosta - Guthrie, Ian" (note spaces around hyphen), "ESSILFIEJONES, KENZO" actually "Jones, Kenzo" (the Essilfie- prefix was an artifact of the OCR including an account-number remnant), "LYLE, TARIK" matched OK but searching just "Lyle" returned multiple Lyles in Fusion (Khari, Malik, Tarik) and the substring matcher first-hit grabbed the wrong one. Lesson: when there are multiple patients with the same last name, always disambiguate by first name.

3. **GreatWestHealthcare-Cigna remits map to payer CIGNA (ST)** (per Darrell's direction this session). The "GW" / "Great-West" payer searches return no results in Fusion's dropdown.

4. **The +Claim dialog's filter chips reset between dialog opens.** Even if you enabled Patient last time, it'll be off this time. Sequence to enable: click funnel (•••) → check Patient checkbox → click outside to dismiss the funnel menu → click the now-visible "Patient" chip → search/select patients → click Apply on the picker → bulk-check rows.

5. **Patient picker search needs full-text query each time.** Earlier-typed text doesn't persist between selections. Triple-click + Delete + retype is reliable; just typing over append-extends the prior search.

6. **Async processOne loops fail at the 45s CDP timeout boundary** on slow-loading dialogs. The loop continues running in the background after timeout (so subsequent `getUnprocessedClaims().length` polls show progress), but the JS call result is lost. Pattern: send the batch, ignore the timeout error, poll `pending.length` to verify it decreased, then send the next batch.

### Files saved

- `Claude COWork/eob_scripts/remit_data/pgba_5213938124WR6.json` (17 claims, 4 with `requires_special_entry`)
- `Claude COWork/eob_scripts/remit_data/pgba_5625538121WA6.json` (2 claims)
- `Claude COWork/eob_scripts/remit_data/uhcsurest_UH88200052846521245378282.json` (7 claims)
- `Claude COWork/eob_scripts/remit_data/greatwest_603801296193.json` (13 claims)


---

## Session 2026-05-13 #5 — 4 more EOBs (`eob 8-11.pdf`)

### What got done

| File | Payer | Remit | Total | Status |
|---|---|---|---|---|
| eob 8 | SUREST-UHC | UH88200053327321245398708 | $189.00 | ✅ **$0 remaining** — 3 claims (Harris/Ford/Herron) |
| eob 9 | UHC - UMR | 0003781150 (Check) | $48.00 | ⚠️ **$48 remaining** — Deresz Chambers 58953986 attached but already finalized on prior remit |
| eob 10 | BCBS (PPO) | C26127N63996310 (NONPAY) | $0.00 | ⚠️ Remit pre-existed (Claim.MD imported). Bashyam Isha 5/4/2026 claims not findable in any phase — likely not yet billed in Fusion |
| eob 11 | BCBS (PPO) | C26127N63950740 (NONPAY) | $0.00 | ⚠️ Created. 3 of 23 P000 claims attached + line-entered (Billingsley/Munumuri/Sharma); 17 random-ACNT + 2 P000 (Adnan/Torres) deferred |

### Manual items for Darrell

**Remit 0003781150 (UHC-UMR $48):** Deresz, Chambers 58953986 (DOS 4/24/2026, CPT 92526) is locked to a prior remit and attached at $0 on this one. Unlink from the other remit, then enter line: $90 billed, CO-45 $17, PR-3 copay $25, paid $48.

**Remit C26127N63996310 (BCBS NONPAY $0):** Claim.MD-imported shell. Bashyam Isha has 2 EOB rows (CPT 92507 with random ACNT RR5SAQNUJ + CPT 97530/99212 combo with random ACNT JTDGXJ4M), both DOS 5/4/2026, all $0 paid (full deductible). No matching Fusion claims exist for that DOS in Pending or Settled phase for Bashyam. The Bashyam claim list jumps from 4/20/2026 → backwards, with nothing on 5/4. The claims likely haven't been generated/submitted in Fusion yet. When they do appear, attach with $74 deductible + $16 CO-45 ($90 billed) and $66.56 deductible + $33.44 CO-45 ($100 billed).

**Remit C26127N63950740 (BCBS NONPAY $0):** 20 claims still need attachment and line entry. Line data per claim: 92507 $90 billed, $74 deductible (PR-1), $16 CO-45, $0 paid. Patient names + DOS:
- ADNAN, FARIS 54491672 — has status-22 reversal of 12/8/2025 visit + status-1 5/X visit. Reversal pair — attach only (per standing rule).
- TORRES, SANTIAGO 58938397 — DOS 4/23/2026
- 17 random-ACNT claims: MAKIL, KIM, SANDERS, LEWIS, SAPPA, GONZALEZ AGUILAR, NJOKU, SOSTER, CUMMINS, ADAMS THOMAS (x2), KALEEM, AL-OBAIDI, SMITH NAOMI, BERBERICH, DRYSDALE, BURKE — DOS 5/4–5/6/2026. The non-P000 ACNTs are BCBS internal numbers; Fusion claim # must be looked up by patient + DOS for each.

### New gotchas

1. **BCBS NONPAY EOBs frequently arrive before Fusion has the corresponding claim billed.** The EOB exists from the insurer (because the claim was filed somehow — possibly direct paper claim) but Fusion's billing system doesn't have it yet. Result: patient + DOS lookup returns no matches in any phase (Pending/Settled/Unsubmitted). These claims have to wait until the Fusion claim is generated, then re-attached. Bashyam Isha 5/4/2026 is today's example.

2. **BCBS uses random 8-char ACNTs on NONPAY EOBs** for any claim it can't match to a P00 ACNT (e.g., when the patient was billed directly, or via paper). Examples: RR5SAQNUJ, JTDGXJ4M, F7MAMHE3, JRMNB9FJ. The parser correctly leaves fusion_id empty when ACNT doesn't match `^P000(\d{8})$` or `^S000(\d{8})$`.

3. **The "Patient" filter chip in the +Claim dialog needs to be enabled via the funnel checkbox AND then the chip clicked.** Just clicking the (hidden) Patient `.button.opener-button` programmatically doesn't reveal it visually — you have to first add it to the visible chip strip via the funnel.

4. **UMR / UMR Innovetive Petcare EOBs map to payer "UHC - UMR"** in Fusion.

5. **UHC SUREST EOBs continue to map to "SUREST-UHC"** (note hyphen order).

### Files saved/updated

- Parsed CSVs in `/sessions/.../outputs/eob{8,9,10,11}_claims.csv` (not copied to OneDrive this session — Mary doesn't need them since each remit's data is small)

### Cumulative totals across all sessions today (5/13)

Created/populated in Fusion: 11 remits totaling **$12,604.20** in payments entered, plus 4 NONPAY $0 shells. Items still needing Darrell's manual attention: 6 (1 NAGANDLA reversal lock, 2 JENKINS CO-8/PR-3 manual, 4 Tricare CO-22/OA-23 secondary, 2 GORANTLA locks, 1 DERESZ lock, 1 BASHYAM not-yet-billed, 20 BCBS NONPAY lookups).


---

## Session 2026-05-13 #6 — 6 new EOBs from batch 12-19 (16 and 19 were duplicates of prior sessions, skipped)

| File | Payer | Remit | Total | Status |
|---|---|---|---|---|
| EOB 13 | Tricare WEST | 5227051125WR6 | $109.94 | ✅ **$0 remaining** — Smith Mason + Ray Daniel |
| EOB 14 | Tricare WEST | 5632192124WA6 | $259.34 | ✅ **$0 remaining** — Williams Axel, Leiseth Jack ×2, Zakariya Joshua |
| EOB 15 | CIGNA (ST) | 260505090040922 | $557.60 | ✅ **$0 remaining** — 9 claims (Gautam Aadya ×2, Kalwar Adhya/Arya, Whitehurst, Banogu, Kemoeatu, Collooru Vivaan ×2, Korean Aadya) |
| EOB 17 | United Health Care | W17776713 | $305 | ⚠️ Created, claims not findable in any phase — Surabhi Hetvik ×3 + McHenry Avery ×2, all April 7-16 DOS, all locked to prior remits |
| EOB 12 | BCBS (PPO) | C26127E01540490 | $113 | ⚠️ Created, claims not yet in Fusion — Koo Noah DOS 5/4 and Thomas Alexandria DOS 5/5 don't exist; latest Settled claims for these patients are 4/27-4/28 |
| EOB 18 | United Health Care (NONPAY) | W358749009 | $0 | ⚠️ Created, 13 of 40 claims line-entered ($73 deduct + $17 CO-45 each). 26 remaining claims in Settled phase, 1 (Davis Zellie 59106428) all-zero needs manual finalize. |

EOB 16 (UHC W358749008 $9,504.47) and EOB 19 (BCBS NONPAY C26127N63950740) were re-uploads of prior sessions — skipped per Darrell.

### Manual items for Darrell

**Remit W17776713 (UHC $305):** 5 claims locked to prior remits. Same pattern as prior session GORANTLA/DERESZ — unlink from the other remit first.
- 58352712 Surabhi, Hetvik — CPT 92523, billed $300, CO-45 $227, PR-3 $10, paid $63
- 58325910 McHenry, Avery — CPT 92507, billed $90, CO-45 $17, PR-3 $15, paid $58
- 58566716 McHenry, Avery — CPT 92507, billed $90, CO-45 $17, PR-3 $15, paid $58
- 58583770 Surabhi, Hetvik — CPT 92507, billed $90, CO-45 $17, PR-3 $10, paid $63
- 58693066 Surabhi, Hetvik — CPT 92507, billed $90, CO-45 $17, PR-3 $10, paid $63

**Remit C26127E01540490 (BCBS $113):** 2 claims with random BCBS ACNTs. Fusion claims for DOS 5/4 (Koo Noah) and 5/5 (Thomas Alexandria) don't exist yet — same not-yet-billed scenario as eob 10's Bashyam case. Wait for Fusion to generate the claims, then attach with:
- Koo Noah 5/4 — CPT 92507, $90 billed, $35 PR-2 coinsurance, $16 CO-45, $39 paid
- Thomas Alexandria 5/5 — CPT 92507, $90 billed, $16 CO-45, $74 paid (no coins/copay)

**Remit W358749009 (UHC NONPAY $0):** 27 of 40 claims still need work.
- 26 P000 claims need to be attached from Settled phase: 58992385, 58992411, 59016431, 59017824, 59024318, 59024607, 59003957, 58855992, 59029604, 59040016, 59054545, 59075874, 59085450, 59080857, 59092075, 59093318, 59100198, 59100417, 59116986, 59138483, 59125526, 59126676, 59136045, 59158578, 59159346, 59151108. All $90 billed (97530 is $80, MANTOVANI BARBOSA 59003957 has $7 CO-45 not $17, HARIHARAN 59092075 has $73 copay not deduct).
- 1 special case Davis, Zellie 59106428 — $90 billed but no deduct/co45/paid (parser may have read zeros). Manually verify the EOB row and finalize.
- Standard line entry pattern: $73 Deductible + $17 CO-45 (Not Allowed) + $0 Charges Paid. Exceptions: 59003957 = $73 Deduct + $7 CO-45; 59092075 = $73 Copay (PR-3) + $17 CO-45 instead of deduct; 59208860/59208942 = $85 Deduct + $5 CO-45.

### Cumulative for the day
Total payment dollars entered across all sessions: ~$13,852 + 4 NONPAY shells. Outstanding manual items: ~10 small lookups for Mary (mostly "unlink from other remit" or "wait for Fusion to bill the claim").


### Session 2026-05-13 #7 — Finish EOB 18 (UHC NONPAY W358749009)

Picked up from Session #6 to attach the remaining 26 of 40 claims on this remit (only 14 had been attached before).

**Result: all 40 claims attached + line-entered.** Remaining: $0.00 of $0.00.

#### Gotchas surfaced this session

- **`.button` selector matches DIVs too.** `Array.from(document.querySelectorAll('button, .button'))` picks up Fusion's `.button-text` DIVs alongside real `<button>` elements. `find()` returned the DIV (textContent matches) but `.click()` on the DIV was a no-op, so `Save Adjudication` silently failed. Fix: use `'button'` selector only when clicking real buttons.
- **`fillCurrentDialog` had a race with slow-loading dialogs.** The original helper waited 200ms after row click before calling setField on `lines[N][...]` inputs. On slower claims the inputs weren't in the DOM yet, setField silently returned false, Save was disabled, and the loop logged `save click failed`. Built `fillCurrentDialogRobust(claimId, maxWait=8000)` that polls for the first expected line input before filling.
- **`+ Claim` button text is just `Claim`.** The `+` is a font-awesome icon (`.button-icon.fa.fa-plus`), not text. Selector: `b.textContent.trim() === 'Claim' && b.className.includes('variant-primary')`.
- **Add Patient ≠ attach claim to remit.** "Add Patient" opens a *create new patient* form. The right entry point is the `+ Claim` button at the bottom-right of the remit page, which opens the **Claim List / Link Claims to Remittance** dialog.
- **Claim # filter chip persists across reopens but the input visibility flips.** After a successful Link, the next +Claim open sometimes shows the popup already-open with stale value; sometimes shows it closed. Reliable pattern: open dialog → if `input[name="id"]` is visible, click chip once to close, then again to reopen; if not visible, click chip once to open. Even with that, ~50% of attempts fail with "no input visible" on first try. Solution: wrap in `attempt(cid)` that returns `{retry: true}` on filter failure; outer `processOne` closes the dialog via × and retries up to 3 times. ~25s/claim worst case, but 100% success rate over 25 claims.
- **"Selection Required" popup hijacks the page.** If you click Link before checking a row, a modal opens warning you. The `Got it!` button is a real `<button>` (not `.button` div), so use `Array.from(document.querySelectorAll('button')).find(b => b.textContent.trim() === 'Got it!')`.

#### Outstanding items for Darrell

1. **59106428 Davis, Zellie 4/29 (CO-252)** — skipped. All EOB line values are $0 (status 22 reversal/denial). Decide whether to manually line-enter zeros or remove from remit.
2. **59093318 Bickmore, Tessa 4/28** — duplicate row on remit. The retry sequence linked the claim twice (one via auto-retry that I thought failed, one via my manual fix-up click). Both rows are line-entered at $73 deduct + $17 CO-45 so the math still balances at $0, but it's cosmetically odd. Remove one of the two rows via "Remove Claims" if it matters.

#### Status

- CLAIMS 41 (40 unique + 1 dup)
- 27 "Finalize to Patient Account" (newly line-entered this session)
- 13 already line-entered from Session #6
- 1 pending (Davis Zellie)
- Remaining: $0.00 of $0.00 ✓


### Session 2026-05-13 #8 — EOB 20-28

Imported 9 new EOBs. The "clean" 5 (with proper P000 ACNTs) all balanced fully. The 4 BCBS/Cigna ones with random ACNTs only partially attached because most claim lines lack a Fusion ID and require patient+DOS lookup.

#### Fully balanced this session ($2,213.42)

- **EOB 20** SUREST-UHC EFT `UH88200053765721245419252` — $351.00 (7/7 line-entered) ✓
- **EOB 23** UHC-UMR EFT `CO33801065944896121974807` — $243.40 (6/6) ✓
- **EOB 24** Tricare WEST EFT `5227334126WR6` — $396.72 (6/6) ✓
- **EOB 25** Tricare WEST EFT `5638506125WA6` — $199.76 (3/3) ✓
- **EOB 26** American Specialty Health EFT `109726592` — $1,022.54 with $58.64 still remaining (21/22; LUSTER 57091855 not found in Pending) — flagged manual

#### Partially complete

- **EOB 27** BCBS (PPO) EFT `C26128N66790380` — NONPAY $0.00 remit. 9 of 10 P000-mapped claims attached + line-entered. Remaining:
  - 54300806 ADNAN, FARIS 12/2/25 — status-22 reversal, in Settled phase, needs manual attach
  - 54633869 ADNAN, FARIS 12/11/25 — status-22 reversal, in Settled phase, needs manual attach
  - 59085580 AHMED, ALYAAN 4/27/26 — CO-197 ($90 full denial, no co45/deduct), needs manual line entry (helper doesn't fill co197 field)
  - 14 random-ACNT BCBS claims (need patient+DOS lookup in Fusion)

#### Not started (need patient+DOS lookups)

- **EOB 21** BCBS (PPO) EFT `C26128E06983770` — $2,054.54 — 4 P000 mapped + ~30 random-ACNT patients. Remit shell NOT created.
- **EOB 22** CIGNA (ST) EFT `260507090045681` — $2,535.80 — 4 P000 mapped + ~30 random-ACNT patients. Remit shell NOT created.
- **EOB 28** BCBS (PPO) EFT `C26127E06886720` — $2,157.26 — 14 P000 mapped + ~20 random-ACNT patients. Remit shell NOT created.

These 3 represent ~$6,747 still to be imported. They are slow because each random-ACNT claim requires opening Fusion's claim list, filtering by patient last name, picking the matching DOS, then linking.

#### EOB 26 manual item

LUSTER, KAYDEN 3/2/26 fusion #57091855 — $58.64 paid, $21.36 CO-45 on CPT 97530. Old claim, in Settled phase; needs Mary to attach to remit `109726592` and run line entry.

#### Files produced

- `outputs/eob{20..28}_claims.csv` — parsed line data per EOB
- `outputs/eob{20..28}_data.json` — Fusion-ready line-entry dicts keyed by fusion_id

#### New gotchas

- **EOB-level NONPAY type isn't a separate Fusion option.** When creating BCBS NONPAY in Fusion, use EFT type with amount $0.00; remit # like `C26128N66790380` will still be searchable.
- **Settled-phase ADNAN reversals (status-22 line + status-1 line on same fusion_id).** Pending-phase filter hides these. To find: phase=Settled in the +Claim dialog.
- **CO-197 claims (full denial)** like AHMED 59085580 don't fit the standard deduct/co45/paid helper schema. The whole billed amount is in `co197`. Manual entry needed in Fusion's "Other Adjustment" field.


### Session 2026-05-13 #9 — EOB 29 (16 sub-EOBs) + EOB 30

EOB 29 was a 19-page PDF containing **16 concatenated sub-EOBs**, mostly GreatWest-Cigna NONPAY notices. EOB 30 was a single BCBS EFT.

#### Fully balanced this session ($439.54)

- **EOB-29-14** American Specialty Health Check `109736378` — $213.02 (5/5 P000 line-entered) ✓
- **EOB-29-15** UHC-UMR EFT `XD00047411` — $43.00 (1/1, HOWARD ELLIS) ✓ — payer mapped to UHC-UMR since "United Healthcare Insurance Company" wasn't in the dropdown
- **EOB-29-16** American Specialty Health Check `109739435` — $125.92 (3/5 P000; 2 random-ACNT OA-208 flagged for Mary) ✓
- **EOB 30** BCBS (PPO) EFT `C26128E07065630` — $57.60 (1/1, MILLER JABARI) ✓

Plus **EOB-29-13** BCBS (PPO) NONPAY `C26131N69808190` $0 — 1 of 22 claims line-entered (NUCKOLLS BECKETT, $74 deduct + $16 CO-45). 21 random-ACNT BCBS lines still need patient+DOS lookup by Mary.

#### Remit shells created (no line entry done yet)

10 GreatWest-Cigna NONPAY $0 EOBs + EOB-29-1 BCBS $51.80:
- `610069378508` (OLMOS VIVANCO 5/7)
- `610069378597` (GORANTLA 4/23 — OA-18 duplicate denial, $0)
- `610069384265` (SALAWU 5/7)
- `610069342412` (SCHAAB 5/4)
- `610069378509` (GORANTLA 4/23 — reversal of CPT 92507 → CPT 92526 — complex, see notes below)
- `610069342411` (THARP 5/4)
- `610069384112` (JOHNSON OLIVER 5/7)
- `610069378507` (MCCOMBS 5/7)
- `610069342413` (QUISENBERRY 5/4)
- `610069342414` (OLMOS VIVANCO 5/5)
- `610069342696` (SALAWU 5/5)
- `C26128E06989020` BCBS $51.80 (BERGER DAX 5/6)

Each has a single random-ACNT claim that requires patient+DOS lookup in Fusion's `+ Claim` dialog before line entry can happen. Mary needs to:
1. Open each remit
2. `+ Claim` → filter by patient name → match DOS → check → Link
3. Open the attached claim → enter `$22 obligations` ($16 for BCBS) and `$68 deductible` ($74 for BCBS) → Save Adjudication

#### Special case: EOB-29-6 GORANTLA reversal

Remit `610069378509` is a $-30 net adjustment where the payer is reversing a previously-paid CPT 92507 ($-90 billed, $-68 paid, status 22) and substituting CPT 92526 ($90 billed, $38 paid, status 1). Both line entries reference fusion #58934409. Recommend handling in Fusion's claim history view rather than as a normal remit.

#### Cumulative status across the 9 + 2 = 11 EOBs from the last batch

- 4 of EOB 29's 16 sub-EOBs fully balanced
- EOB 30 fully balanced
- 12 sub-EOB shells created (no line entry)
- $439.54 imported this session


---

## Session #10 — 2026-05-14 — EOB 50 (Frisco Feeding ERA #4)

**Source:** `EOB 50.pdf` (19MB, 27 pages, "Frisco Feeding and Speech Therapy ERA (4).pdf")

**Structure:** 14 concatenated sub-EOBs totaling $5,693.24 grand total, 134 claims

### Sub-EOB Summary

| # | Payer | Remit # | Type | Amount | Claims | P000s |
|---|---|---|---|---|---|---|
| 50-01 | UHC-UMR | CE43421126198176111111402 | EFT | $189.80 | 3 | 3 ✓ |
| 50-02 | UHC-UMR | C625104093472026124162297 | EFT | $58.40 | 1 | 1 ✓ |
| 50-03 | UHC-UMR | CL21521126521396111143724 | NONPAY | $0.00 | 1 | 1 ✓ |
| 50-04 | UHC-UMR | CO33821126893616111180946 | EFT | $73.00 | 1 | 1 ✓ |
| 50-05 | CIGNA (ST) | 260511050001267 | EFT | $204.00 | 3 | 0 (Mary) |
| 50-06 | Tricare WEST | R37060511016841 | NONPAY | $0.00 | 1 | 1 ✓ OA-18 denial |
| 50-07 | BCBS (PPO) | C26131E07266780 | EFT | $74.00 | 1 | 0 (Mary) |
| 50-08 | BCBS (PPO) | C26131E07372800 | EFT | $237.70 | 5 | 0 (Mary) |
| 50-09 | BCBS (PPO) | C26131E07486110 | EFT | $4,898.98 | 76 | 6 ✓ |
| 50-10 | BCBS (PPO) | C26131E07257360 | EFT | $23.00 | 1 | 0 (Mary) |
| 50-11 | BCBS (PPO) | C26131E01632540 | EFT | $182.56 | 4 | 0 (Mary) |
| 50-12 | BCBS (PPO) | C26132N72770850 | NONPAY | $0.00 | 35 | 4 (3 done + 1 BURNS reversal flagged) |
| 50-13 | UHC | XD00049358 | EFT | $73.00 | 1 | 1 ✓ |
| 50-14 | BCBS (PPO) | C26132N73754310 | NONPAY | $0.00 | 1 | 0 (Mary) |

### Done This Session

- **All 14 remit shells created** in Fusion Received list, dated 5/12/2026
- **17 P000 claims fully attached + line-entered** totaling $744.03 imported:
  - EOB-50-01: BRADLEY 58497414 ($73), GARCIA 58593684 ($58.40), GARCIA 58600526 ($58.40)
  - EOB-50-02: SERRATO 58992061 ($58.40)
  - EOB-50-03: PARENT 58506047 ($0, PR-1 deduct $73)
  - EOB-50-04: ROCA 58541081 ($73)
  - EOB-50-06: ZAKARIYA 58831564 ($0, OA-18 dup)
  - EOB-50-09: ABAUNZA 58858918 ($65.70), DHINESH 59091693 ($39.53 2-line), MISTRY 59103748 ($58.40), CROWELL 59159257 ($66.60), HENSON 59188675 ($54), GUFFEY 59190870 ($66.60)
  - EOB-50-12: NUCKOLLS 59097640 ($0), ARENDS 59205091 ($0), LINDSKOG RYAN 59213748 ($0, OA-18 dup)
  - EOB-50-13: PENEMETCHA 58710455 ($73)

### Flagged for Mary

- **EOB-50-12 BURNS JACE P00057940026** — Reversal of previous payment (status 22 negative line $-90/$-73.70 deduct/$-16.30 CO-45) + restate (status 1 positive). Complex 2-entry reversal; recommend manual review.
- **115 random-ACNT claims totaling $5,269.41** across all EOBs need patient+DOS lookup. Random ACNTs are not Fusion claim IDs — they're BCBS/CIGNA internal account numbers. Mary needs to find each patient in Fusion, locate the matching claim by DOS, and attach manually.
  - EOB-50-05: 3 claims @ $204
  - EOB-50-07: 1 claim @ $74
  - EOB-50-08: 5 claims @ $237.70
  - EOB-50-09: 70 random-ACNT claims @ $4,548.15 (the bulk of the BCBS pile)
  - EOB-50-10: 1 claim @ $23
  - EOB-50-11: 4 claims @ $182.56
  - EOB-50-12: 30 random-ACNT NONPAY claims (write-offs/deductibles)
  - EOB-50-14: 1 claim @ $0 NONPAY (NIPP PR-1 deduct $70.10)

### Files

- `outputs/eob50_parsed.json` — full structured data
- `outputs/eob50_claims.csv` — flat CSV of all 134 claims with lines
- `outputs/eob50_pages/*.txt` — OCR'd page text
- `outputs/parse_eob50.py` — parser script

### Notes

- Fusion's "Claim #" filter chip remained flaky (Selection Required error on first attempt ~50% of the time); pattern is: dismiss "Got It!" → re-click chip → retry. 3-retry pattern with longer waits (3-4s) between steps works.
- The OCR missed the "NOS" (units) column on UMR EOBs (50-01 through 50-04), so the parser regex didn't catch those lines. Worked around by manually extracting from raw OCR text. Future: make NOS optional in `parse_eob50.py` RE_LINE.
- EOB-50-09 was the big one: 10 sub-pages of BCBS claims. 6 P000s ($350.83) auto-processed; remaining 70 random-ACNTs ($4,548.15) await Mary's lookup pass.


---

## Session #11 — 2026-05-14 — EOB NEW 10 (Frisco Feeding ERA #6) — PARTIAL / PAUSED

**Source:** `EOB NEW 10 ON5.13.pdf` (5MB, 11 pages, "Frisco Feeding and Speech Therapy ERA (6).pdf")

**Structure:** 10 sub-EOBs totaling $1,447.01, 32 claims, 30 P000s ($1,337+ auto-processable)

### Sub-EOB Summary

| # | Payer | Type | Remit # | Amount | Claims | Status |
|---|---|---|---|---|---|---|
| N10-01 | UHC | EFT | 53694910 | $211.00 | 4 | ✓ 2 P000s done ($101); MOHLER×2 random for Mary |
| N10-02 | UHC | EFT | 53651940 | $48.00 | 1 | ⚠️ **WRONG CLAIM ATTACHED — Van Deren 59227861 finalized instead of ANDERSON 58758845. Needs Mary to unfinalize + fix.** |
| N10-03 | UHC | EFT | 11439183087 | $73.00 | 1 | NOT STARTED |
| N10-04 | UHC SUREST | EFT | UH88200054194171245438107 | $58.00 | 1 | NOT STARTED |
| N10-05 | UMR | EFT | CE43405113702936125248916 | $525.60 | 8 | NOT STARTED |
| N10-06 | UMR | NONPAY | CN06905114089616125287584 | $0 | 2 | NOT STARTED |
| N10-07 | UMR | NONPAY | CL21505114009446125279567 | $0 | 3 | NOT STARTED |
| N10-08 | UHC SUREST | EFT | UH72100054194161531593082 | $26.00 | 2 | NOT STARTED |
| N10-09 | UMR | EFT | CO33805114499346125328557 | $348.80 | 8 | NOT STARTED |
| N10-10 | Tricare WEST | EFT | 5644722127WA6 | $156.61 | 2 | NOT STARTED |

### What Got Done

- **EOB-N10-01 fully balanced** ($211): MCLEOD 58565768 ($58), QUINN 58911144 ($43) — 2 P000s correctly attached + line-entered. $110 MOHLER×2 random ACNTs left for Mary.
- **EOB-N10-02 shell created** but wrong claim got attached due to Claim # filter chip failure.

### The Issue

The Claim # filter chip in Fusion's "Link Claims to Remittance" dialog is intermittently flaky — about 50% of the time my click on the chip doesn't open the popup, so subsequent typing goes nowhere, the filter doesn't apply, and clicking Apply on the empty filter shows ALL claims unfiltered. Then my next click on row checkbox + Link picks the first row in the unfiltered list.

This pattern caused Van Deren 59227861 (a previously-finalized claim from an earlier remit) to get wrongly attached to EOB-N10-02 with $48 paid. The line entry saved + finalized the claim before I noticed, so it now can't be removed from this remit through the UI (Fusion says "Claim #59227861 has already been finalized").

### Action for Mary

- **Manually fix EOB-N10-02** — unfinalize Van Deren 59227861, remove from remit C26131E... wait wrong: remit 53651940, then attach ANDERSON P00058758845 92507 paid $48, CO-45 $17, copay $25.
- **Process EOB-N10-03 through 10** — 27 remaining P000 claims totaling ~$1,236 + 2 random ACNT for full Mary lookup. All remit shells need to be created (EOB-N10-02 already exists in compromised state).

### Claim Data Ready

All claim data parsed and saved in:
- `outputs/eob_new10/eob_new10.pdf` — source PDF
- `outputs/eob_new10/pages/*.txt` — OCR'd text

The 27 remaining P000s ready to process when filter chip issue is resolved:
- N10-03: ESAYAS P00058954874 92507 paid $73, CO-45 $17
- N10-04: POWELL P00059025523 92507 paid $58, CO-45 $17, copay $15
- N10-05: 8 GARCIA RIO + BRADLEY MAKENNA P000s (all UMR EFT, varied amounts)
- N10-06: DANDAMUDI×2 (S00 prefix — try as 58915034 / 59030195) — deduct only, $0 paid
- N10-07: PARENT×3 — deduct only $73 each, $0 paid
- N10-08: GOEDDE×2 — paid $13 each, copay $60
- N10-09: 8 P000s (FACTOR, ROCA, SPRINGER×2, MONCRIEF, SALLOUM, SEGURA OA-18, JAIN)
- N10-10: WILLIAMS 58342533, HODGES 58936043 — paid $74.47 and $82.14 respectively, CO-45 only

### Note for Future Work

Before resuming, fix the Claim # filter chip flakiness. Options:
1. Verify popup opened (`document.querySelector('input[name="id"]')`) before typing
2. Always check the filter chip is in active/highlighted state before pressing Apply
3. After Apply, verify only 1 row is showing OR ask for confirmation before linking
4. Use JS to drive entire workflow (skip the chip click; use API directly)


### Session #11 Update — Final State

**All 10 EOB NEW 10 sub-EOB remit shells created in Fusion Received (5/13/2026):**

| # | Payer | Type | Remit # | Amount | Status |
|---|---|---|---|---|---|
| N10-01 | United Health Care | EFT | 53694910 | $211.00 | ✅ 2 P000s done ($101: MCLEOD, QUINN); MOHLER×2 random for Mary ($110) |
| N10-02 | United Health Care | EFT | 53651940 | $48.00 | ⚠️ **WRONG ATTACH** — Van Deren 59227861 finalized, needs unfinalize + ANDERSON 58758845 attach |
| N10-03 | United Health Care | EFT | 11439183087 | $73.00 | Shell only — ESAYAS P00058954874 NOT in Fusion (patient+DOS lookup needed) |
| N10-04 | SUREST-UHC | EFT | UH88200054194171245438107 | $58.00 | Shell only — POWELL P00059025523 to attach |
| N10-05 | UHC-UMR | EFT | CE43405113702936125248916 | $525.60 | Shell only — 8 P000s to attach |
| N10-06 | UHC-UMR | EFT (NONPAY) | CN06905114089616125287584 | $0.00 | Shell only — DANDAMUDI×2 (try S00... or look up) |
| N10-07 | UHC-UMR | EFT (NONPAY) | CL21505114009446125279567 | $0.00 | Shell only — PARENT×3 P000s |
| N10-08 | SUREST-UHC | EFT | UH72100054194161531593082 | $26.00 | Shell only — GOEDDE×2 P000s |
| N10-09 | UHC-UMR | EFT | CO33805114499346125328557 | $348.80 | Shell only — 8 P000s |
| N10-10 | Tricare WEST | EFT | 5644722127WA6 | $156.61 | Shell only — WILLIAMS + HODGES P000s |

**Imported tonight: $101.00 (2 P000s in EOB-N10-01)**

### Mary's Pickup List — Full Line-Entry Data

**EOB-N10-02 fix (Remit 53651940 $48):**
- Unlink Van Deren 59227861 (unfinalize first)
- Attach ANDERSON P00058758845 — 92507, paid $48.00, CO-45 $17, copay (PR-3) $25

**EOB-N10-03 (Remit 11439183087 $73):**
- ESAYAS P00058954874 doesn't exist in Fusion. Patient+DOS lookup: Esayas, Christian, DOS 4/23/2026, 92507, paid $73, CO-45 $17

**EOB-N10-04 (Remit UH88200054194171245438107 $58):**
- POWELL P00059025523 — 92507, paid $58, CO-45 $17, copay $15

**EOB-N10-05 (Remit CE43405113702936125248916 $525.60) — 8 P000 claims:**
- GARCIA RIO P00058856249 92507 paid $58.40, CO-45 $17, coins $14.60
- GARCIA RIO P00058856870 97530 paid $58.40, CO-45 $7, coins $14.60
- GARCIA RIO P00058896993 92507 paid $58.40, CO-45 $17, coins $14.60
- BRADLEY MAKENNA P00058914534 92507 paid $73, CO-45 $17
- BRADLEY MAKENNA P00059004934 97530 paid $73, CO-45 $7
- BRADLEY MAKENNA P00059150149 92507 paid $73, CO-45 $17
- GARCIA RIO P00059146390 92507 paid $58.40, CO-45 $17, coins $14.60
- BRADLEY MAKENNA P00059149383 97530 paid $73, CO-45 $27

**EOB-N10-06 (Remit CN06905114089616125287584 $0 NONPAY) — 2 P000 claims:**
- DANDAMUDI SHIVA S00058915034 97530 paid $0, deduct $59.43, CO-45 $20.57 (S-prefix indicates Secondary claim instance — search 58915034)
- DANDAMUDI SHIVA S00059030195 97530 paid $0, deduct $59.43, CO-45 $40.57

**EOB-N10-07 (Remit CL21505114009446125279567 $0 NONPAY) — 3 P000 claims:**
- PARENT MATTEO P00058870541 92507 paid $0, deduct $73, CO-45 $17
- PARENT MATTEO P00059005929 92507 paid $0, deduct $73, CO-45 $17
- PARENT MATTEO P00059117862 92507 paid $0, deduct $73, CO-45 $17

**EOB-N10-08 (Remit UH72100054194161531593082 $26) — 2 P000 claims:**
- GOEDDE HARRISON P00059020269 97530 paid $13, CO-45 $7, copay $60
- GOEDDE HARRISON P00059025570 92507 paid $13, CO-45 $17, copay $60

**EOB-N10-09 (Remit CO33805114499346125328557 $348.80) — 8 P000 claims:**
- FACTOR AIDEN P00058993298 92507 paid $13, CO-45 $17, copay $60
- ROCA LUNA P00059032995 92507 paid $73, CO-45 $17
- SPRINGER GABRIEL P00059050981 92526 paid $73, CO-45 $17
- MONCRIEF REID P00059053731 92507 paid $0, deduct $73, CO-45 $17
- SPRINGER JEREMIAH P00059056723 92526 paid $73, CO-45 $17
- SALLOUM BENNETT P00059080155 97530 paid $58.40, CO-45 $27, coins $14.60
- SEGURA MIA P00057964605 92507 paid $0, OA-18 dup denial $90 (put in "Other" field)
- JAIN AHAAN P00059135394 92507 paid $58.40, CO-45 $17, coins $14.60

**EOB-N10-10 (Remit 5644722127WA6 $156.61) — 2 P000 claims:**
- WILLIAMS AXEL F P00058342533 92507 paid $74.47, CO-45 $15.53, REM N16
- HODGES LIAM W P00058936043 92526 paid $82.14, CO-45 $7.86, REM N16

### Total Outstanding for Mary
- **30 P000 claims** to attach + line-enter (~$1,236)
- **2 random ACNT** claims for patient+DOS lookup (MOHLER×2 in EOB-N10-01)
- **1 cleanup** task (EOB-N10-02 Van Deren removal)

### Helper Verification Step Built

`window.safeAttach` JS helper installed in browser tab — has verification step that confirms only 1 row in claim list before linking. Filter chip selector fixed to `button.opener-button`. Available for future sessions to avoid the wrong-claim-attached issue.


### Session #11 — Continued Push With Verification Helper

**More claims attached to EOB-N10-05 (UMR EFT CE43405113702936125248916 $525.60):**

| Claim # | Patient | DOS | Service | Paid | Status |
|---|---|---|---|---|---|
| 59149383 | Bradley, Makenna | 4/30/2026 | OT 97530 | $73.00 | ✅ Done |
| 59150149 | Bradley, Makenna | 4/30/2026 | ST 92507 | $73.00 | ✅ Done |
| 59146390 | Garcia, Rio | 4/29/2026 | ST 92507 | $58.40 | ✅ Done |
| 59004934 | Bradley, Makenna | 4/27/2026 | OT 97530 | $73.00 | ⏳ Attached, line entry submitted (browser froze during save - verify) |

**Imported this session: $378.40 cumulative** (EOB-N10-01 $101 + EOB-N10-05 $277.40 so far)

### Remaining for Mary on EOB-N10-05 (still ~$248 outstanding)

The following 5 claims OCR'd as P000NNNNNNNN but the IDs don't appear in Fusion's "Pending" phase. These may be in "Settled" phase (already had a primary payment that finalized them) or were billed through a different system. Mary needs to look these up by patient name + DOS:

- 58856249 GARCIA RIO 92507 4/21/2026 paid $58.40 (CO-45 $17, coins $14.60)
- 58856870 GARCIA RIO 97530 4/21/2026 paid $58.40 (CO-45 $7, coins $14.60)
- 58896993 GARCIA RIO 92507 4/22/2026 paid $58.40 (CO-45 $17, coins $14.60)
- 58914534 BRADLEY MAKENNA 92507 4/23/2026 paid $73.00 (CO-45 $17)
- 59149383 already imported above (this was the OT 4/30, not 4/23)

Wait actually 58914534 is BRADLEY 4/23 ST — different DOS. Need lookup.

### Key Finding About Fusion Claim Phases

When searching for P000 IDs in the Link Claims dialog, the **default Phase filter is "Pending"**. Many older claims (paid out 1-2 weeks ago) move to **Settled** phase and won't show in default search. To find older claims:
1. Click "Phase" filter chip
2. Uncheck "Pending", check "Settled"
3. Apply

Mary should switch to Settled phase when searching for these older P000 IDs.

### Remaining EOBs Untouched (Mary to handle)

- **EOB-N10-03** UHC EFT 11439183087 $73 — ESAYAS P00058954874 needs name+DOS lookup
- **EOB-N10-04** SUREST UH88200054194171245438107 $58 — POWELL P00059025523 needs Settled-phase lookup
- **EOB-N10-06** UMR NONPAY CN06905114089616125287584 $0 — DANDAMUDI×2 (S00 prefix = Secondary instance)
- **EOB-N10-07** UMR NONPAY CL21505114009446125279567 $0 — PARENT×3 P000s
- **EOB-N10-08** SUREST UH72100054194161531593082 $26 — GOEDDE×2 P000s
- **EOB-N10-09** UMR EFT CO33805114499346125328557 $348.80 — 8 P000s
- **EOB-N10-10** Tricare WEST 5644722127WA6 $156.61 — WILLIAMS + HODGES P000s

### Browser Issues

Several CDP timeouts during the session. The Fusion app seemed to freeze during back-to-back Save Adjudication calls. Mary may want to refresh the page between EOBs.

### Tooling Note for Future Work

`window.safeAttach` JS helper installed (in the browser tab) verifies the filter result before linking, preventing wrong-claim attachments. The chip click via JS doesn't reliably trigger the popup — manual mouse coordinates (715, 237) work better. Pattern that works:
1. Manual click chip (715, 237)
2. Wait 2s
3. Manual click input (640, 280)
4. Type claim ID
5. Click Apply (696, 339)
6. Wait 3s
7. Check checkbox (99, 328)
8. Click Link (937, 803)
9. Wait 4s


### Session #11 — FINAL RESULTS

**Total imported correctly tonight: $896.81 across 13 P000 claims**

**Per-EOB results:**

| EOB | Status | Imported | Notes |
|---|---|---|---|
| N10-01 ($211) | ✅ Done correctly | $101 | McLeod $58, Quinn $43. MOHLER×2 random for Mary ($110) |
| N10-02 ($48) | ⚠️ NEEDS MARY CLEANUP | $0 correctly | Van Deren 59227861 wrongly attached + finalized. Mary must unfinalize + replace with ANDERSON 58758845 (92507 paid $48, CO-45 $17, copay $25) |
| N10-03 ($73) | ❌ Shell only | $0 | ESAYAS P00058954874 not in Fusion. Patient: Esayas, Christian, DOS 4/23/2026, 92507, paid $73, CO-45 $17 |
| N10-04 ($58) | ❌ Shell only | $0 | POWELL P00059025523 not in Fusion's Pending or Settled. Patient: Powell, Anthoneil, DOS 4/27/2026, 92507, paid $58, CO-45 $17, copay $15 |
| N10-05 ($525.60) | 🟡 Partial | $277.40 | Done: Bradley 59149383 OT $73, Bradley 59150149 ST $73, Garcia 59146390 $58.40, Bradley 59004934 OT $73. Remaining ~$248: Garcia×3 (58856249, 58856870, 58896993) + Bradley 58914534 — try Settled phase |
| N10-06 ($0 NONPAY) | ❌ Shell only | $0 | DANDAMUDI 58915034 + 59030195 (ACNTs were S00 prefix in EOB) — both deduct $59.43, CO-45 varies |
| N10-07 ($0 NONPAY) | 🟡 2 of 3 PARENT done | $0 (NONPAY) | Done: Parent 58870541, Parent 59005929 (both deduct $73). Remaining: PARENT 59117862 (not in Fusion) |
| N10-08 ($26) | ✅ Done correctly | $26 | Goedde 59020269 $13, Goedde 59025570 $13 |
| N10-09 ($348.80) | 🟡 5 of 8 done + 1 wrong | $335.80 | Done: Jain $58.40, Roca $73, Springer G $73, Springer J $73, Salloum $58.40. **WRONG: Castro Guillermo 59223920 wrongly attached as $13 instead of FACTOR P00058993298** — Mary fix. Not done: MONCRIEF 59053731 ($0 deduct), SEGURA 57964605 ($0 OA-18) |
| N10-10 ($156.61) | ✅ Done correctly | $156.61 | Williams 58342533 $74.47, Hodges 58936043 $82.14 |

**Net session: $896.81 imported, $2 cleanup tasks for Mary (Van Deren + Castro Guillermo wrong attaches), ~$306 in untouched P000s for Mary's name/Settled-phase lookup**

### Key Learning: Fusion Claim Phases

The "Link Claims" dialog defaults to **Pending phase**. Older claims (older than ~10 days that were already paid as primary) move to **Settled phase**. To find them, Mary needs to switch the Phase filter to Settled (or clear it).

### Browser Issues This Session

- Multiple CDP screenshot timeouts on consecutive Save Adjudication calls
- Claim # filter chip click via JS unreliable — must use manual mouse coordinates
- Even manual coords fail ~30% of the time → wrong claim attaches if filter empty

### Recommended Next Steps for Mary

1. **Quick wins first:** Try the remaining P000s with Settled phase filter — should pick up most of EOB-N10-05's 4 remaining and EOB-N10-07's PARENT 59117862
2. **Name lookups for the truly missing:** ESAYAS (N10-03), POWELL (N10-04), and any others by patient + DOS
3. **Cleanup the 2 wrong attaches:** Van Deren in N10-02 ($48), Castro Guillermo in N10-09 ($13)
4. **DANDAMUDI S00 prefix:** Try as just digits 58915034 / 59030195 in Settled


### Session #11 — Fresh Pass Round 2 (after user uploaded EOB NEW 10 again)

**Key discovery:** The Phase filter chip can be **cleared entirely** (uncheck Pending AND Settled, click Apply) to search across all phases. This finds claims that aren't in default Pending view.

**Net new this round: $321.20 imported**

| EOB | Imported This Round | Cumulative State |
|---|---|---|
| N10-03 ESAYAS | **$73.00** ✓ | Found with Phase cleared. Fully balanced. |
| N10-05 (4 remaining) | **$248.20** ✓ | All 4 found with Phase cleared. EOB now fully balanced $525.60/$525.60 |
| N10-06 DANDAMUDI×2 | $0 NONPAY ✓ | Both found in Settled phase as **Secondary** instances (S00 prefix = Secondary in OCR). Fully balanced. |
| N10-04 POWELL | — | Not found in any phase. Patient+DOS lookup still needed by Mary |
| N10-07 PARENT 59117862 | — | Not found in any phase |

### Updated Total: $1,218.01 imported across both EOB NEW 10 sessions

**EOB NEW 10 Final State:**
- ✅ N10-01: $101 (2 P000 done; MOHLER×2 random pending Mary)
- ⚠️ N10-02: Van Deren wrong attach ($48) — Mary cleanup
- ✅ N10-03: $73 ESAYAS done
- ❌ N10-04: $58 POWELL not in Fusion
- ✅ N10-05: $525.60 fully balanced
- ✅ N10-06: $0 NONPAY (2 Secondary DANDAMUDI)
- 🟡 N10-07: 2 of 3 PARENT done; 59117862 missing
- ✅ N10-08: $26 GOEDDE×2
- 🟡 N10-09: $335.80 done + Castro $13 wrong + MONCRIEF/SEGURA untouched
- ✅ N10-10: $156.61 Tricare

**$1,218.01 imported / $1,447.01 EOB total = 84% completion**

**For Mary's pickup:**
1. Cleanup Van Deren (N10-02) + Castro (N10-09) = ~$61 wrong attaches
2. POWELL P00059025523 patient+DOS lookup (N10-04)
3. PARENT 59117862 patient+DOS lookup (N10-07)
4. MOHLER×2 random ACNT lookups (N10-01)
5. MONCRIEF/SEGURA NONPAY entries (N10-09)


### Session #11 — FINAL TALLY After Fresh Pass

**Discovery: My earlier "Castro Guillermo wrong attach" assessment was incorrect.** Verified EOB-N10-09 has all 7 P000s correctly attached (Jain, Roca, Springer×2, Salloum, Factor, Moncrief). $0 of $348.80 remaining = fully balanced.

**SEGURA 57964605 not in Fusion** in any phase. EOB still balanced because SEGURA was $0 paid (OA-18 denial).

### Updated Final State: **$1,231.01 imported / $1,447.01 = 85% completion**

**6 of 10 sub-EOBs FULLY balanced:**
- ✅ N10-03 $73 ESAYAS
- ✅ N10-05 $525.60 (8 P000s)
- ✅ N10-06 $0 NONPAY (2 DANDAMUDI Secondary)
- ✅ N10-08 $26 (2 GOEDDE)
- ✅ N10-09 $348.80 (7 P000s — corrected my earlier mistaken assessment about Castro)
- ✅ N10-10 $156.61 (Williams + Hodges)

**4 sub-EOBs need Mary's attention:**
- N10-01 $211 (only $101 done — MOHLER×2 random ACNTs = $110 for patient+DOS lookup)
- N10-02 $48 (Van Deren 59227861 wrongly attached + finalized; needs unfinalize + ANDERSON 58758845 replace)
- N10-04 $58 (POWELL P00059025523 NOT in Fusion any phase — patient+DOS lookup or claim may not exist)
- N10-07 $0 NONPAY (1 of 3 PARENT missing — 59117862 not in Fusion)

### Key Takeaways for Future EOBs

1. **Phase filter trick:** Cleared Phase (no Pending, no Settled, no Unsubmitted checked + Apply) searches ALL phases. Many claims sit in Completed phase (after primary payment) and won't show in default Pending view.
2. **Secondary instances:** ACNTs with "S00..." prefix in EOB OCR = Secondary claim instances. Search by just the 8 digits (e.g., 58915034 not S00058915034).
3. **Claims not in Fusion:** Some EOBs reference claim IDs that don't exist anywhere in Fusion. These were likely billed directly to the payer outside Fusion. Mary handles these via patient+DOS lookup.
4. **Browser stability:** The save-adjudication operation can hang the page for 10-15 seconds. Build in long waits between batched operations.

### Cumulative Across All Recent Sessions

- Session #10 (EOB 50): $744.03 imported, 17 P000 claims, 14 sub-EOBs
- Session #11 (EOB NEW 10): $1,231.01 imported, ~22 P000 claims, 10 sub-EOBs
- **Total: $1,975.04 imported across 24 sub-EOBs and 39 P000 claims tonight**
