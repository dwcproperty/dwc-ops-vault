---
type: architecture
status: draft-off
authority: DWC process architecture and current live audit
last_verified: null
superseded_by: null
---

# PM Applicant to Lease - Move In Date Confirmation

## Status

- Build estimate: 87% of the full application-to-signed-lease-and-initial-payment process
- LeadSimple process: Draft/off
- Jotform: disabled
- Zapier: Draft/off; Link Builder Zap `375707765` and Return Zap `375708159`
- Webhook: attached to the disabled Jotform; no live submissions accepted
- Applicant communication sent: none

## LeadSimple authority

LeadSimple step conditions remain the business-routing authority. Zapier supplies verified facts and writes exact controlled custom-field values; it does not bypass the native stage routes.

### Move In Date Approval Pending

- Initial staff input displays when confirmation is `Pending`, a requested date exists, approved date is empty, and automation is complete.
- Returned-date staff input displays when confirmation is `Different Date Requested`, a requested date exists, and automation is complete.
- Only the applicable staff step displays.
- Automatic routing to confirmation requires confirmation `Pending`, an approved date, and automation complete.

### Move In Date Confirmation Pending

- Household email and SMS require confirmation `Pending`, approved date, secure form URL, response deadline, and automation `Waiting`.
- Overdue task: `Call applicant - move-in date confirmation overdue`; due immediately after the stored two-business-day deadline. It does not automatically withdraw the application.
- Accepted route requires confirmation `Accepted`, approved date, submission ID, and automation complete; destination is `Lease Preparation`.
- Different-date route requires confirmation `Different Date Requested`, requested date, submission ID, and automation complete; destination is `Move In Date Approval Pending`.

## Jotform contract

- Form: `DWC Move In Date Confirmation`
- Form ID: `262200946591053`
- Exact choices: `Accept Approved Date`; `Request Different Date`
- Alternate date is shown and required only for `Request Different Date`.
- Required: authorized-adult full name, email, certification, signature, signing date.
- Read-only prefills: property address, approved move-in date, response deadline.
- Hidden: LeadSimple process ID, response token, request ID, form purpose, schema version, form version, source, household ID.
- Form is disabled until release.

## Validation contract

The return path must re-read LeadSimple before any write and verify exact form ID, process type/ID, current stage, one-time token, request ID, approved date, response deadline, pending/waiting state, submission ID, duplicate state, signer identity against an authorized adult, certification, signature, signing date, and conditional alternate date.

LeadSimple field `Authorized Adult Applicants JSON` stores the household signer roster. RentVine Intake Mapping Zap `375483019` now builds and maps every adult's normalized name, email, and applicant ID. Its controlled code-only sample returned all three adults correctly; the final LeadSimple update action remains untested to avoid a production write. The validator falls back to this field and fails closed when it is missing or empty.

Only two canonical outcomes may be written:

- `ACCEPT` → `Move In Date Confirmation = Accepted`, submission ID saved, automation complete.
- `DIFFERENT_DATE` → requested date saved, `Move In Date Confirmation = Different Date Requested`, submission ID saved, automation complete.

LeadSimple then performs the stage change using native conditional logic.

## Canonical artifacts

- `docs/automation/zapier/move-in-date-confirmation-link-builder.js`
- `docs/automation/zapier/move-in-date-confirmation-webhook-precheck.js`
- `docs/automation/zapier/move-in-date-confirmation-return-validator.js`
- `docs/automation/zapier/move-in-date-confirmation-self-test.js`
- `docs/automation/zapier/move-in-date-confirmation-test-matrix.md`
- `docs/automation/zapier/rentvine-authorized-adult-signer-roster.js`

The self-test passes valid accept and different-date branches and rejects invalid decisions, missing signatures, and token mismatch.
