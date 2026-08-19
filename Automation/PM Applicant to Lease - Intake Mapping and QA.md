---
type: architecture
status: controlled-test-reference
authority: DWC process architecture and verified test evidence
last_verified: null
superseded_by: null
---

# PM Applicant to Lease — Intake Mapping and QA

Status: design, read-only source verification, isolated LeadSimple test-process creation, and the first controlled intake write-through are complete. Production automation remains disabled.

## Safety boundary

- Read-only RentVine source: historical application `66`, applicant IDs `82`, `83`, and `84`.
- Never update this application, its applicants, screening reports, lease, or tenant records; never rerun screening.
- Never send email, text, or Jotform notices from this test.
- The only permitted intake write target is LeadSimple process `c69fde70-8a25-4ec4-a41c-b5b5007c3cc8` (`TEST - PM Applicant to Lease - INTAKE ONLY - DO NOT PROCESS`).
- Keep LeadSimple Live Mode off and all relevant Zaps unpublished/test-only.

## Intake mapping

| RentVine source | LeadSimple target | Rule |
|---|---|---|
| Application ID | `RV Application ID` process field | Authoritative household key; never pass it to an applicant-detail lookup. |
| All adult applicant IDs | `RentVine Adult Applicant IDs` process audit list | Store a normalized unique set. |
| Individual applicant ID | `RV Applicant ID` applicant-contact field | Required before routing; never match by name alone. |
| Applicant role | `Applicant Role` contact field | Intake uses applicant contacts, not later tenant contacts. |
| Name, email, mobile | Standard contact fields | Prefill only; not authoritative cross-system keys. |
| SSN/ITIN and license data | Presence/validity flags only | Raw values remain in RentVine. |
| Property/unit IDs | RentVine property/unit process fields | Exact source linkage. |
| Current RentVine rent | `Property Rent` | Authoritative 3× denominator. |
| Applicant requested move-in date | Applicant contact requested-date field | Preserve each answer. |
| All requested dates | Process summary plus conflict flag | Differing dates require staff decision; never select earliest/latest automatically. |
| Staff-approved household date | `Approved Move-In Date` | Must remain blank until staff approval; required before lease creation. |
| Current address and rental history | Applicant contact fields/summary | Do not fabricate missing history. |
| Employment rows marked current | Applicant current-employer fields and income subtotal | Exclude prior employment from current income. |
| Adult current-income subtotals | `Household Monthly Income` | Sum normalized current rows only. |
| Household income ÷ current rent | `Income Multiple Calculation` plus `Income Requirement Met` | Store the exact formula as text and use the choice field for logic. LeadSimple Number fields round this ratio to a whole number; `Income Multiple Rounded - Audit Only` must never drive qualification logic. |
| Occupants and animals | Household count/summary | Retain source-applicant provenance for deduplication. |
| New disclosure answers and Reasons | Unlisted-occupant/animal fields and reconciliation flags | `Yes` requires reconciliation. Historical pre-question blanks are `not testable`, not missing. |
| Emergency contact | Applicant contact fields | Applicant-level data. |
| Uploaded-file metadata/presence | Applicant document-status flags | Actual retrieval still needs a supported endpoint or controlled new-template proof. |
| Screening status | Applicant credit/report-status field | Do not infer numeric score from report status. |
| Numeric score | Applicant `Credit Score` | All adults require a valid score; supported structured retrieval remains unresolved. |
| Adult/valid-score counts | Process counts | Counts must match before averaging. |
| Rounded adult-score average | `Household Average Credit Score` | Calculate only when every adult has one valid score. |
| Applicant gaps | Applicant missing-item/status fields | Preserve applicant provenance. |
| Consolidated gaps | Process missing summary/count/intake status | Drives one household notice only after the communication gate is authorized. |
| Unsupported or mismatched source | Automation exception fields | Fail closed; do not advance or communicate. |

## Verified historical-source facts

- Exactly three unique adult applicant IDs belong to application `66`.
- Current-employment-only monthly amounts total `$9,300`; rent is `$2,195`; ratio rounds to `4.24x` and passes 3×.
- Three numeric scores produce a rounded household average of `538`.
- Applicant requested dates conflict; the approved move-in date must remain a staff decision.
- Required native form controls were complete.
- The two new questions are blank because the application predates them.
- LeadSimple contains both applicant and tenant contacts for each adult; applicant `RV Applicant ID` values are blank.

## Retrieval status

- Supported/API-ready: application ID, applicant IDs through application export, status, and application → lease → tenant linkage.
- Current structured reader: standard personal/property/address/employment/occupant/animal/vehicle/emergency-contact data and screening-report metadata.
- Not proven through the supported reader/API: numeric score, applicant-file enumeration, and custom-question answers.
- Manager-screen/PDF reads exist for the unresolved values, but must remain a narrowly scoped exception path until RentVine supplies documented endpoints or connector extensions.

## Release tests

1. **Immutable-source guard:** snapshot source and target; abort unless source IDs and safe target ID are exact and all communication interlocks are off.
2. **ID mismatch protection:** application ID used as applicant ID, unrelated applicant, duplicate, missing ID, or fourth ID fails closed with no target change.
3. **Three-adult grouping:** shuffled/replayed IDs still produce one household and exactly three unique adults.
4. **Applicant/tenant separation:** intake never selects or updates tenant contacts.
5. **Current-income aggregation:** result is `$9,300`, `$2,195`, `4.24x`, 3×=`Yes`; prior salary is excluded.
6. **Score completeness:** three valid scores yield average `538`; missing/duplicate scores stop before averaging and never rerun reports.
7. **Requested-date conflict:** retain all requests; set conflict; leave approved date blank; block lease creation.
8. **Legacy-question exemption:** historical blanks do not create missing items or applicant notices.
9. **Incomplete/mixed household:** missing, unrelated, or cross-application applicants stop before household finalization.
10. **Idempotent replay:** repeated payload does not double income, scores, activities, processes, or messages.
11. **Target-field assertion:** only authorized intake fields and internal checkpoint change; no applicant-facing/lease/payment/decision stage.
12. **Post-run integrity:** RentVine and historical LeadSimple contacts unchanged; zero email, text, Jotform notices, or screening reruns.

## Evidence required

For each test retain redacted inputs, expected/actual fields, safe target ID, timestamps, stage before/after, Zap path, and zero-communication proof. Do not publish Zaps or enable Live Mode based only on field-level success.

## Isolated LeadSimple intake target

- Process ID: `c69fde70-8a25-4ec4-a41c-b5b5007c3cc8`
- Name: `TEST - PM Applicant to Lease - INTAKE ONLY - DO NOT PROCESS`
- Link: `https://app.leadsimple.com/v2/process-types/WMd2_kUaNDSZp6_GpJWvMTy2KqmVxxtQjz6GeFF1snxvHeEmV1hG/processes/WMd2_kUaNE_W5fnYr5mjKTH8LqyTAOB9Az5PxqkNcTrY4mMlJA==`
- Starting stage: `Intake Validation`
- Due: August 31, 2026 at 8:00 p.m. Central
- Verified: zero contact roles, zero properties, no uncompleted tasks in the starting stage, and the process type remains in Draft Mode.
- Safety comment is stored on the process: no applicant contacts, no property, no outbound communication, no RentVine writes, and controlled intake-field mapping/QA only.
- The earlier missing-information test process was not reset or altered.

## Controlled write-through result — 2026-08-07

- Passed on the isolated LeadSimple target only; no historical RentVine data or LeadSimple contacts were changed.
- Stored `RV Application ID = 66` and `RentVine Adult Applicant IDs = 82,83,84`.
- Stored `Adult Applicant Count = 3`, `Adult Scores Received Count = 3`, `Household Average Credit Score = 538`, `All Adult Scores Valid = Yes`, and `Intake Status = Not Reviewed`.
- Stored `Household Monthly Income = $9,300`, `Property Monthly Rent = $2,195`, `Income Multiple Calculation = 9300 / 2195 = 4.24x`, and `Income Requirement Met = Yes`.
- Stored `Requested Move In Dates Summary = 2026-08-01; 2026-08-02`, `Move In Date Conflict = Yes`, and left `Approved Move In Date` blank.
- Stored `Automation Status = Waiting`; the process remains in `Intake Validation` and Draft Mode.
- Verified through the read-only LeadSimple API that the process still has zero contact roles and zero properties.
- LeadSimple rounded the original Number field to `4`. It has been relabeled `Income Multiple Rounded - Audit Only`; the exact text calculation and the explicit yes/no qualification field are authoritative.
- No email, text, Jotform notice, screening rerun, stage advance, or RentVine write occurred.

## Draft intake-validation Zap — 2026-08-07

- Zap name: `PM Applicant to Lease | Intake Validation`
- Zap ID: `375555217`
- Status: Draft and unpublished. Do not publish until every production blocker below is cleared with a new controlled application.
- Trigger: LeadSimple `Process Created`.
- Test-only filter: process type exactly `PM Applicant to Lease`, stage exactly `Intake Validation`, and process name exactly `TEST - PM Applicant to Lease - INTAKE ONLY - DO NOT PROCESS`.
- Validator input mapping: LeadSimple process ID; RentVine application ID; normalized adult applicant IDs; adult count; score count; household average score; household income; property rent; requested move-in dates; all-scores-valid flag; and 3×-income flag.
- Fresh code-step test result: `READY_FOR_DRAFT_ENRICHMENT`, `errors = NONE`, `productionReady = false`, idempotency key `rv-intake:v1:dwc:66`, source fingerprint `68334e7b`.
- Normalized result: applicant IDs `82,83,84`; adult count `3`; score count `3`; average `538`; income `9300`; rent `2195`; income multiple `4.24`; income requirement `Yes`; requested dates `2026-08-01; 2026-08-02`; conflict `Yes`; approved date blank.
- The validator fails closed for missing IDs, count mismatches, invalid scores, invalid income/rent, a mismatched 3× flag, or missing requested dates.
- The Zap has no downstream write or communication action. Its Zap note records the read-only household, no-communication rule, test-only guard, successful test identifiers, and release blockers.
- Production blockers: supported RentVine numeric-score source; custom-question answers and Reasons; upload metadata; other-income retrieval; exact RentVine/LeadSimple lead-to-household-process bridge; and reliable applicant-contact ID linking.
