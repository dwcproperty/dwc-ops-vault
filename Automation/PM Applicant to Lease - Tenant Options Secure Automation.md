---
type: architecture
status: draft-off
authority: DWC process architecture and current live audit
last_verified: 2026-08-08
superseded_by: null
---

# PM Applicant to Lease - Tenant Options Secure Automation

**Status:** Draft/off — controlled testing required  
**Updated:** 2026-08-08  
**Owner:** DWC Property Group

## Architecture

LeadSimple is the workflow and state authority. Jotform collects the signed household election. Zapier validates the external event and writes verified facts. LeadSimple step conditions decide when email, SMS, staff work, and stage movement appear.

## LeadSimple stage

`Tenant Options Pending` contains:

- Guarded automatic email and SMS to all contact roles.
- Automatic route `Continue to Move In Date Approval Pending` only after a validated submission.
- Manual task `Call applicant - tenant options deadline expired`, hidden unless all conditions are true:
  - `Tenant Options Status = Expired`
  - `Automation Status = Error`
  - `Automation Last Error = TENANT_OPTIONS_DEADLINE_EXPIRED`

The overdue task is skipped while hidden, so it does not block the normal submitted route.

## Jotform

- Existing form: `New Tenant Option Form`
- Form ID: `251564975684069`
- Preserved existing option questions.
- Added hidden LeadSimple process ID, opaque response token, form version `tenant-options-v2`, and link source `leadsimple-tenant-options`.
- Added required signer email, household certification, and E-Signature.

## Zaps

### PM Applicant to Lease | Tenant Options Link Builder

- Zap ID: `375704730`
- Draft/off.
- Exact release gate: process type/stage, Automation Complete, valid approval/risk combination, Tenant Options Not Sent, and no existing token/link/submission.
- Creates a three-business-day deadline inside the 8:00 a.m.–8:00 p.m. business window.
- Maps short form link, token, deadline, Tenant Options Pending, and Automation Waiting.
- At the deadline, re-fetches LeadSimple and fails closed unless the exact request remains pending.
- Only a valid timeout maps Expired/Error/`TENANT_OPTIONS_DEADLINE_EXPIRED`.

### PM Applicant to Lease | Tenant Options Return

- Zap ID: `375705864`
- Draft/off.
- Webhook endpoint: `https://hooks.zapier.com/hooks/catch/20991120/46qg8y2/`
- Validates form ID, process ID, token, form version, source, submission ID, signed certification, current LeadSimple stage/status, deadline, duplicate state, and approval state.
- Proceed path maps Submitted, submission ID, Automation Complete, requested move-in date, Move In Date Confirmation Pending, Tenant Protection choice, CurbCommand, and Utility Convenience.
- Withdraw path maps Submitted, submission ID, Automation Complete, and Staff Decision Withdraw.

## Canonical code

- `docs/automation/zapier/tenant-options-link-builder.js`
- `docs/automation/zapier/tenant-options-webhook-precheck.js`
- `docs/automation/zapier/tenant-options-return-validator.js`
- `docs/automation/zapier/tenant-options-deadline-validator.js`

All four pass `node --check`.

## Remaining release work

1. Sign in to Jotform integrations and attach the webhook endpoint to form `251564975684069`.
2. Use a synthetic signed payload with no real applicant communication.
3. Prove fail-closed rejection for wrong form, token, version, source, stage, duplicate ID, expired deadline, missing signature, and invalid controlled options.
4. Prove Proceed and Withdraw paths update only the intended safe test process.
5. Confirm the native LeadSimple email/SMS and stage route remain condition-gated.
6. Publish only after the controlled release checklist passes.

## Safety

No Zap is published. No LeadSimple action write was tested on a production record. No applicant communication was sent.

Overall application-to-signed-lease-and-initial-payment completion estimate: **77%**.
