# DWC Integration Automation Standard

> Canonical cross-system research and build templates are indexed in [[DWC Systems Knowledge Hub]]. Review that hub and its conflict register before using historical implementation notes.

**Systems:** Jotform → Zapier → LeadSimple → RentVine  
**Owner:** DWC Property Group  
**Status:** Draft implementation standard — all related automations remain off until end-to-end testing and owner approval  
**Last reviewed:** 2026-08-06

## Purpose

This is the required architecture for DWC workflows that collect information in Jotform, transfer or validate it in Zapier, manage the operational process in LeadSimple, and read from or write back to RentVine.

The goal is minimal human touch without hiding the workflow logic inside an undocumented Zap. LeadSimple is the operational source of truth for process state and should control task visibility, communications, deadlines, and stage changes whenever it can.

## Non-negotiable design principle

> Zapier and integrations write facts into custom fields. LeadSimple evaluates those facts and controls the workflow.

Do not build a workflow that relies only on Zapier advancing stages. A complete automation normally includes:

1. Structured custom fields.
2. Conditional LeadSimple tasks.
3. LeadSimple email and text templates with merge fields.
4. Conditional automatic stage-change steps.
5. A defined waiting state when no condition is satisfied.
6. An automation-exception route.
7. End-to-end testing before Live Mode is enabled.

## System responsibilities

### Jotform

- Collect structured information that should not arrive through free-form email or text.
- Show only the questions needed for the current request.
- Carry non-editable routing values needed to match a submission to the correct LeadSimple process and contact.
- Return a submission ID and timestamp for auditing.
- Never collect Social Security numbers, bank credentials, credit-report documents, or other restricted information unless a separately approved secure workflow specifically requires it.

### Zapier

- Receive the Jotform submission or RentVine event.
- Identify the automation and form purpose.
- Validate routing identifiers and response tokens.
- Filter or branch by form, response type, applicant, and expected process state.
- Normalize dates, currency, numbers, phone numbers, and choice values.
- Update the correct LeadSimple process and contact custom fields.
- Recalculate household-level status when an applicant-level response changes.
- Update RentVine when LeadSimple reaches a status that must match RentVine.
- Log submission IDs and errors so a Zap can be replayed safely.

Zapier should change a LeadSimple stage directly only when LeadSimple cannot reasonably perform the stage change from its own custom-field logic.

### LeadSimple

- Hold the current operational state.
- Store household-level and applicant-level custom fields.
- Display or hide tasks based on custom-field conditions.
- Send email and text messages from approved templates.
- Schedule reminders within the approved business schedule.
- Automatically change stages when an exclusive routing condition becomes true.
- Remain in the current stage when no valid routing condition exists.
- Route errors to `Automation Exception` instead of guessing.

### RentVine

- Remain the source for the application, property, unit, rent, credit report, lease, signatures, tenant record, and ledger/payment events when those records originate in RentVine.
- Collect stable, routinely required applicant information in the RentVine application before the missing-information workflow runs. This includes household adults, employment and income, rental history, animals, standard contact details, and supported required documents.
- Customize the RentVine application so LeadSimple receives the information needed for screening and lease preparation whenever RentVine supports the field reliably.
- Use the missing-information Jotform only for information that is absent, invalid, requires correction or confirmation, or cannot be collected reliably in the RentVine application.
- Provide stable record IDs to LeadSimple and Zapier.
- Receive matching application/tenant statuses when the LeadSimple workflow requires synchronization.

## Naming rules

Follow the existing DWC LeadSimple naming convention:

- Start with the audience tag, such as `PM`.
- Use letters, digits, and spaces only.
- Use a short lifecycle description.
- Put version notes in the process description, not the title.
- Archive retired processes; do not delete historical processes.

Example: `PM Applicant to Lease`.

## Required identifier and audit fields

Every integration must define the identifiers it uses before building templates or Zaps.

### Process-level examples

- LeadSimple process ID or public routing ID
- RentVine application ID
- RentVine applicant lead ID
- RentVine property ID
- RentVine unit ID
- RentVine lease ID
- RentSign package ID
- RentVine tenant lead ID
- Automation schema/version
- Automation status
- Automation last updated
- Automation last error

### Contact-level examples

- RentVine applicant ID
- Applicant role
- Adult applicant status
- Jotform response token
- Jotform form URL
- Jotform submission ID
- Missing items for that applicant
- Missing-information status
- Missing-information submitted timestamp

## Response-token standard

A Jotform hidden field is not a true secret; the person receiving the form can inspect or alter browser-visible values. Do not place credentials in a form link.

Use an opaque, random response token that:

- Does not contain a name, email address, Social Security number, or meaningful record information.
- Is stored with the expected LeadSimple process and contact.
- Is checked by Zapier before any record is updated.
- Is limited to the intended form purpose.
- Can be marked used or superseded when appropriate.
- Produces an exception instead of updating a record when it is missing, duplicated, expired, or mismatched.

Legacy DWC workflows may use a hidden `deal_id` and a Linode webhook. Document those as legacy implementations. New Zapier workflows should add token validation and idempotency instead of trusting a raw record ID by itself.

## Jotform link pattern

For the current `PM Applicant to Lease` household workflow, Zapier creates one household-specific form URL and writes it to the LeadSimple **process** field `Household Missing Information Form Link`. Applicant-specific workflows may use a contact URL, but the storage scope must match the workflow design.

Recommended routing values:

- `automation_schema_version`
- `form_purpose`
- `response_token`
- `lead_simple_process_id` or public routing ID
- `lead_simple_contact_id` or applicant public ID, only when the form is applicant-specific
- `requested_items_code`

Routing values should be hidden or read-only in Jotform, but security must come from server-side validation in Zapier—not from the field being visually hidden.

### Required prefill standard

Every applicant-facing Jotform must be prefilled with every reliable value already available from RentVine or LeadSimple. The applicant should only have to provide information that is genuinely missing or confirm information that requires confirmation.

Prefill when available:

- Applicant first and last name
- Email address and telephone number
- Current address
- Property applied for
- Application reference
- Household applicant count
- Relevant property, application, process, and contact routing IDs
- Any other non-sensitive value needed by that specific form

Known reference values should be read-only where the applicant should not change them. Routing values should be hidden. Never place Social Security numbers, consumer reports, credit-report contents, credentials, or other sensitive data in a Jotform URL. Zapier must URL-encode each value before shortening the personalized link.

### Required acknowledgment and signature standard

Whenever an applicant or tenant uses a Jotform to grant permission, make a yes/no election, accept or decline an option, acknowledge a disclosure, certify information, or state that they understand a term, the form must also require:

- A signature field completed by the person making the election or acknowledgment
- The signer's printed name, prefilled when reliable data is available
- A signing date and time recorded by the form
- The exact election, acknowledgment text, and form version stored with the submission

The form may not be submitted until the required acknowledgment/election and signature are complete. This rule does not authorize changing Beagle-controlled forms; follow Beagle's required form and signature instructions without alteration.

## LeadSimple custom-field standard

### Process fields

Use process fields for household-wide facts and routing decisions, including:

- Consolidated missing-items summary
- Missing-items count
- Household income
- Property rent
- Income multiple
- Adult applicant count
- Valid score count
- Household average credit score
- Screening result
- Staff decision
- Risk-mitigation fee and decision
- Household deadlines
- Tenant-options status
- Lease and payment status

### Contact fields

Use contact fields for facts belonging to one adult, including:

- Applicant role
- Applicant-specific missing items
- Applicant-specific Jotform URL/token/submission ID
- Credit score and credit status
- Missing-information submission and validation status
- Applicant-level adverse-action status

Do not force applicant-level data into a single household text field when LeadSimple needs to address adults separately.

## LeadSimple stage build standard

Every stage must answer five questions:

1. What facts must be true when the process enters this stage?
2. Which tasks should display for each possible field value?
3. Which communications should send, to whom, and when?
4. What field changes make those tasks disappear?
5. What mutually exclusive conditions move the process to the next stage?

### Conditional tasks

- Use `Display task when` conditions on emails, texts, todos, calls, and stage changes.
- A task that is irrelevant after completion must explicitly test the completion/status field.
- Example: missing-information reminders display only when `Intake Status is not Complete`.
- Example: the staff-review todo displays only when `Staff Decision is Pending`.
- Avoid unconditional tasks unless the action truly applies to every process entering the stage.

### Automatic stage changes

- Use separate automatic stage-change steps for exclusive outcomes.
- Example: `All Adult Scores Valid = Yes` → `Qualification Review`.
- Example: `All Adult Scores Valid = No` → `Missing Information Pending`.
- Example: `Automation Status = Error` → `Automation Exception`.
- Verify the saved destination after selecting it.
- If no route condition is true, leave the process in the stage for staff or external data rather than selecting a default outcome.

### Timing

- Use the stage's workflow schedule for business hours.
- Current applicant-process schedule: Monday–Friday, 8:00 a.m.–8:00 p.m. Central.
- Deadline timestamps must be written to custom fields and used in templates.
- A partial response does not restart a deadline unless the business rule explicitly says to issue a new request.

## LeadSimple communication-template standard

Every email and text template must document:

- Template name
- Process it belongs to
- Triggering stage
- Recipient role(s)
- Display condition(s)
- Automatic/manual setting
- Delay and business schedule
- Merge fields used
- Link field used
- Completion field that hides future reminders
- Failure behavior when email, phone, or link is missing

### Email rules

- DWC email subjects begin with `DWC | `.
- Household-wide emails use process merge fields such as a consolidated missing-items summary.
- Send the same household email to all adult applicants when the message concerns the entire household.
- Do not place one adult's private response link in a household email unless the email intentionally contains a named list of every adult's links.

### Text rules

- Texts should be brief and action-oriented.
- Applicant-specific texts use contact merge fields.
- Household texts in `PM Applicant to Lease` use the process fields `Missing Items Summary` and `Household Missing Information Form Link`; the same household message and link are sent to all adult applicant contact roles.
- Each adult receives only that adult's secure Jotform link when a future workflow is intentionally applicant-specific.
- Do not instruct applicants to text sensitive information.
- A reminder template must still contain the deadline and the exact action required.

### Merge-field verification

Before Live Mode:

1. Confirm the LeadSimple merge-tag slug for every custom field.
2. Preview with a test process containing realistic values.
3. Confirm blank/default behavior.
4. Confirm links are clickable and resolve to the correct applicant/form.
5. Confirm household fields do not accidentally show another applicant's private data.

## Standard Jotform-return Zap

### Trigger

- Jotform `New Submission`, or a dedicated Catch Hook when required.

### Required steps

1. Capture the raw submission and submission ID.
2. Filter by the expected Jotform form ID.
3. Read `form_purpose`, `automation_schema_version`, and `response_token`.
4. Reject a blank or malformed token.
5. Locate the expected LeadSimple process/contact.
6. Confirm the process is in an allowed stage.
7. Confirm the token/form purpose belongs to that process/contact.
8. Check whether the submission ID was already processed.
9. Validate required answers and attachments.
10. Update the applicant contact fields.
11. Set the applicant submission status and timestamp.
12. Recalculate household missing items and counts.
13. Set household `Intake Status = Complete` only when every required adult/item is complete.
14. Allow LeadSimple's conditional tasks and automatic stage change to react.
15. Record the submission ID, Zap run, and update timestamp.
16. On failure, write an actionable error and route/notify through the automation-exception design.

### Zapier filters and paths

Use Filters for hard stop conditions:

- Wrong form ID
- Missing token
- Token mismatch
- Duplicate submission
- Process not in an allowed stage

Use Paths for legitimate business variants:

- Missing-information response
- Credit-correction response
- Risk-mitigation decision
- Tenant-options response
- Move-in-date response
- Other future form purposes

Do not create many nearly identical Zaps if one well-documented router Zap with controlled Paths is easier to maintain. Split Zaps when permissions, risk, ownership, or error handling materially differ.

## RentVine synchronization standard

- Store RentVine record IDs in LeadSimple before attempting updates.
- Filter the Zap by the exact RentVine application, applicant, lease, tenant, or ledger event.
- For payments, confirm the charge type and amount—not merely that any payment occurred.
- For signed leases, confirm every required signer completed the package.
- When LeadSimple moves to `Withdrawn Incomplete`, update the corresponding RentVine application status and record the result.
- Never activate a lease if the required payment gate is not cleared.
- Route ambiguous RentVine events to an exception instead of inferring success.

## Error and recovery standard

Each automation must define:

- Retry-safe/idempotent behavior
- Duplicate-submission behavior
- Missing-record behavior
- Missing-contact-information behavior
- Invalid-token behavior
- API outage behavior
- Human override field or stage
- Error message visible in LeadSimple
- Staff notification rule
- Replay instructions

Use `Automation Status`, `Automation Last Error`, and `Automation Last Updated` consistently. Never silently fail.

## Testing and release checklist

Keep LeadSimple in Draft until all tests pass.

### Test records

- One adult, complete application
- Multiple adults, complete household
- One adult missing information
- Multiple adults with different missing items
- Invalid/no-hit credit for one adult
- Partial Jotform response
- Duplicate Jotform submission
- Invalid or altered token
- Missing email
- Missing mobile number
- Zapier or API failure
- Staff override
- Deadline expiry

### Assertions

- Correct tasks display and irrelevant tasks remain hidden.
- Correct email/text template is attached.
- Correct recipients receive the communication.
- Merge fields and Jotform URLs resolve correctly.
- Completed fields hide future reminders.
- Exactly one automatic stage route becomes true.
- RentVine receives the correct matching status.
- Duplicate events do not create duplicate messages or updates.
- Errors produce a visible exception.

### Release procedure

1. Export or screenshot the final stage/condition map.
2. Document all custom fields and allowed values.
3. Document every Jotform ID and Zap ID.
4. Run end-to-end tests with non-production records.
5. Review communication copy.
6. Confirm RentVine writes against test records.
7. Enable LeadSimple Live Mode only after approval.
8. Monitor the first real records and document corrections.

## Current PM Applicant to Lease decisions

- One LeadSimple process represents the entire household.
- Every adult applicant is attached as a contact.
- The consolidated missing-items email and text are sent to all adult applicants.
- The household receives one secure missing-information Jotform link. The same link is sent to all adults, and the submitted response updates the household process.
- All adults require a valid credit score.
- Household credit score is the rounded average of adult scores.
- Income requirement is 3× rent.
- Missing-information period is three business days.
- Workflow hours are Monday–Friday, 8:00 a.m.–8:00 p.m. Central.
- Missing-information reminders stop when `Intake Status = Complete`.
- Incomplete applications move to `Withdrawn Incomplete` and synchronize to RentVine.
- Staff makes the final approve, approve-with-risk-mitigation, more-information, hold, deny, or withdraw decision.
- The process remains in Draft during construction.

## Current missing-information implementation registry

This section records the intended corrected architecture. It is not a statement that release testing has passed.

### LeadSimple

- Process: `PM Applicant to Lease`
- Scope: one process per applicant household, with every adult attached as a contact
- Required stage order:
  1. `Intake Validation`
  2. `Missing Information Link Preparation`
  3. `Missing Information Pending`
  4. `Missing Information Response Review`
  5. Return to `Intake Validation` after a validated response
- `Missing Information Link Preparation` is a waiting gate. It must not send applicant communications. The link-builder Zap writes the response token and household form URL, then advances the process to `Missing Information Pending` only after both values exist.
- `Missing Information Pending` owns the initial email/text, reminders, deadline, and incomplete-withdrawal route. This prevents a race in which LeadSimple renders a message before Zapier has saved the link.
- `Response Validated - Continue Screening` must display only when `Missing Information Validation Result is VALIDATED`.
- The staff-review task must display only when `Missing Information Validation Result is NEEDS_REVIEW`.
- LeadSimple Live Mode remains off until the test checklist below is complete and the owner approves release.

### Process-field dictionary for the household response

| LeadSimple process field | Type/purpose | Required use |
|---|---|---|
| `Household Missing Information Form Link` | URL | Personalized shortened Jotform link used by both email and text templates |
| `Missing Information Requested Items Code` | Text | Authoritative server-side list of requested categories; never trust the submitted hidden-field copy as the source of truth |
| `Missing Information Response Token` | Text | Opaque random token stored before notice delivery and matched on return |
| `Missing Information Form Submission ID` | Text | Jotform submission ID used for traceability and duplicate protection |
| `Missing Information Response Summary` | Text | Normalized readable summary of submitted information |
| `Missing Information Validation Result` | Multiple choice | Exact values `VALIDATED` and `NEEDS_REVIEW` |
| `Missing Information Validation Result (Legacy Text)` | Text | Legacy only; must not control routing or receive new Zap mappings |
| `Missing Items Summary` | Text | Consolidated household list shown in email and text |
| `Missing Items Count` | Number | Remaining household item count |

### Jotform

- Form: `DWC | Rental Application Missing Information`
- Form ID: `262175267632056`
- Production URL: `https://form.jotform.com/262175267632056`
- Purpose: collect only the household information requested by `Missing Information Requested Items Code`.
- Hidden routing values include form purpose, schema version, response token, LeadSimple process ID, RentVine application ID, and requested-items code.
- Hidden or read-only fields are a convenience, not a security control. Zapier must compare the submitted token and routing values with the LeadSimple process and use the requested-items code retrieved from LeadSimple for validation.
- The form must not collect a Social Security number or consumer report.
- Permission, certification, acknowledgment, and yes/no decisions require printed name, signature, and Jotform submission timestamp.
- Field limits, conditional visibility, required status, prefills, and read-only behavior must be verified in the builder and in a submitted test before release.

### Zapier

#### Missing-information link builder

- Zap ID: `375483019`
- Trigger: a `PM Applicant to Lease` process enters `Missing Information Link Preparation`.
- Required actions: verify process and stage; generate an opaque response token; build an encoded URL for Jotform `262175267632056`; shorten it; save the token and URL to the LeadSimple process; confirm both saved values; then move the process to `Missing Information Pending`.
- Failure behavior: leave the process out of the applicant-notice stage, record an actionable automation error, and notify staff through the exception design.
- Status: draft/off. A controlled test on `TEST - PM Applicant to Lease - DO NOT PROCESS` verified the stage-isolation filter, generated a populated six-category Jotform URL, prefilled the available applicant values, and displayed all six conditional sections. The safe fixture had no linked RentVine property, so property/reference population still requires a RentVine-linked test record. Notification, timing, and error-path testing remains required.

#### Missing-information submission

- Zap ID: `375481302`
- Trigger: a new submission from Jotform `262175267632056`.
- Required hard stops: wrong form/purpose; wrong schema; blank or malformed token; missing process ID; missing LeadSimple process; process in a disallowed stage; token mismatch; duplicate submission ID.
- The Zap must retrieve the LeadSimple process before validation. `Missing Information Requested Items Code` from LeadSimple is authoritative; the submitted hidden copy may be compared for tampering but may not decide which answers are required.
- After validation, update the process fields, record the submission ID and timestamp, and move the process to `Missing Information Response Review`. LeadSimple then controls the validated or staff-review route.
- Status: draft/off. A controlled end-to-end submission test on `TEST - PM Applicant to Lease - DO NOT PROCESS` passed the purpose/schema/routing filter, exact stored-token filter, all six category validations, required document upload, certification/signature capture, LeadSimple update, and one-use token conversion. The final result was `VALIDATED`, `Missing Items Count = 0`, `Missing Items Summary = NONE`, submission ID `6618859685515394453`, and stage `Missing Information Response Review`. Negative and timing paths remain required.

### Return-Zap mapping checklist

Every item below must be mapped from the Jotform trigger or the retrieved LeadSimple process into the validator before release:

- LeadSimple process ID
- Stored LeadSimple response token and submitted response token
- Stored LeadSimple requested-items code and submitted requested-items code
- Jotform submission ID and submission timestamp
- Employer, job title, gross monthly income, employment start date, and employer phone
- Landlord name, landlord phone, rental address, rental dates, monthly rent, and reason for leaving
- Animal type, name, breed, age, and weight
- Requested-document answer or uploaded document references
- Credit-correction confirmation
- Other requested information

The update action must map validation result, response summary, remaining-items summary/count, submission ID, and timestamp to the exact process fields above. It must not overwrite canonical household income with a blank or partial answer.

### Communication merge-field checklist

- Initial and reminder email subjects begin with `DWC | `.
- Emails and texts use `{{process.missing_items_summary | default}}` for the consolidated household list.
- Emails and texts use `{{process.household_missing_information_form_link | default}}` for the secure household link.
- Old contact fields such as `contact.missing_items_by_applicant` and `contact.missing_information_form_link` are not used in this household workflow.
- Applicant notices do not send until the household link field is populated.
- Initial and reminder messages are addressed to all adult applicant contact roles.

### Release status and required proof

No production test is recorded as passed in this document. The following must be demonstrated with non-production records before either Zap is published or LeadSimple Live Mode is enabled:

Controlled action-level verification completed on 2026-08-06:

- The final LeadSimple action was changed from its expired saved connection to the existing working LeadSimple connection; the change was scoped to that step.
- Link-builder action saved the shortened form URL and token, then moved the safe test process to `Missing Information Pending`.
- Return action saved the submission ID and response output, replaced the token with `USED-6618493117713165791`, and moved the safe test process to `Missing Information Response Review`.
- These action tests did not publish either Zap, enable LeadSimple Live Mode, or contact an applicant.

Controlled end-to-end response verification completed on 2026-08-06:

- A fresh populated link carried all six requested category codes plus the available applicant name, email, phone, current address, process ID, and response token.
- Jotform displayed and accepted income, rental-history, animal, document, RentVine-credit-correction, and other-information responses with a required certification and drawn signature.
- The return Zap captured submission `6618859685515394453`, passed the purpose/schema/process filter and exact stored-token filter, and produced `VALIDATED` with zero remaining items.
- The validator now emits `NONE` instead of a blank remaining-items value so LeadSimple does not retain a stale missing-items summary after successful completion.
- LeadSimple readback confirmed stage `Missing Information Response Review`, validation result `VALIDATED`, missing count `0`, missing summary `NONE`, the complete response summary, and token `USED-6618859685515394453`.
- The all-six-category email/form was intentionally a stress-test. Normal production requests must include only the gaps remaining after the customized RentVine application is evaluated.

### RentVine Residential application audit — 2026-08-07

The active global-default RentVine template is `Residential`. After the audit, two required yes/no exception questions were added and saved:

- `Are there any occupants who will live at the property that you have not listed in the Occupants section?`
- `Are there any pets, service animals, or emotional support animals that will live at the property that you have not listed in the Animals section?`

RentVine displays a Reason field with each question. Final preview verification showed exactly these two questions and two yes/no groups; temporary placeholder questions created during configuration were removed before the final save.

- RentVine already requires the core personal identifiers needed for screening: name, applicant role, mobile phone, email, date of birth, SSN/ITIN, driver's-license number, and license state.
- Emergency Contacts is active with minimum required `1`; contact name, relationship, and phone are required.
- Address History is active with minimum required `1`; Employment History is active with minimum required `1`.
- Income Verification and Identity Verification are active. Their document sections each require at least `1` item.
- General Documents is active with minimum required `2`, and its instructions request government-issued ID and the applicable proof-of-income records.
- Occupants and Animals are active with minimum required `0`. This appropriately allows a household with no minor occupants or animals to continue, but it does not create an explicit affirmative `none` declaration.
- Questions is active with the two disclosure questions above. Terms remains inactive with no custom items. Addenda is active and contains `Applicant Consent Form`.
- The application instructions already state that every occupant age 18 or older must submit a separate application and that a valid SSN is required for each applicant.

Design decision: RentVine remains the primary collection point for employment/income, rental history, animals, documents, emergency contact, identity, and household composition. The missing-information Jotform must not routinely request those categories again. It is used only when the RentVine submission is absent, incomplete, invalid, contradictory, or requires a correction/confirmation. The two required exception questions distinguish a true zero-entry household from an omission: a `No` answer confirms nothing was omitted; a `Yes` answer and its Reason require reconciliation before intake is complete. Do not change the credit/risk-mitigation disclosure wording until that wording receives its separate compliance review.

### RentVine intake transport verification — 2026-08-07

- A read-only test of the connected RentVine application reader returned structured applicant, property, requested move-in, current-address, prior-address, employment/salary, occupant, animal, vehicle, emergency-contact, and screening-report-status data from an existing application.
- The tested response did not expose uploaded files, the new custom-question answers, a numeric credit score, or a general other-income collection. Absence from one response is not proof that every application omits those keys; each must be tested with a controlled application containing the relevant data.
- Zapier's public app directory has no RentVine integration; the direct RentVine app URL returns `404`.
- The RentVine Webhooks settings were inspected without adding a webhook. Available events are Work Order Created/Updated, Lease Created/Updated, Property Created/Updated, Unit Created/Updated, and Lease Charge Created/Updated. There is no application-created or application-submitted event.
- Therefore, do not design application intake around a nonexistent RentVine application webhook. Use the existing RentVine-to-LeadSimple new-lead sync as the event source. A LeadSimple stage/process trigger starts the intake automation, which then retrieves the matching RentVine application.
- Before production mapping, submit one controlled application using the revised `Residential` template and prove: household/applicant identifiers, both custom-question answers and Reasons, uploaded-document metadata, all income variants, screening status, numeric score availability, and multi-adult grouping.
- If the supported RentVine API cannot return a required field, use a narrowly scoped browser-assisted exception step only for that field. Do not make browser reading the default when a supported structured source is available.

### Three-adult historical application dry run — 2026-08-07

A previously approved, converted three-adult application was used only for read-only extraction and matching. No RentVine or LeadSimple records were changed, no stages were advanced, no reports were rerun, and no email, text, Jotform link, or other communication was sent.

- RentVine application `66` contains three applicant records with distinct applicant IDs `82`, `83`, and `84`. This proves `applicationID` and `applicantID` are different identifiers; passing an application ID to an applicant-detail lookup can return an unrelated person.
- The connected structured reader returned each adult under the correct household application and exposed the standard personal, property, current/prior address, employment, occupants, animals, vehicles, emergency-contact, and screening-report-status sections.
- Summing only the three current-employment monthly amounts reproduced RentVine's household income total and its rent-to-income ratio. A prior employment record also contained a salary amount, proving that household income must use records marked current rather than summing every employment-history row.
- The adults submitted different requested move-in dates. Do not silently select one applicant's date or an earliest/latest value; preserve the applicant-level requests and require the approved household move-in date before lease creation, as already designed.
- The manager summary displayed three individual numeric scores and a rounded household average. The connector response exposed report status but not the numeric score, so numeric-score retrieval still needs a supported structured route or a narrowly scoped manager-screen read.
- All required native form controls visible for the three historical submissions were populated. The manager view displayed the two newly added disclosure questions with blank answers because those questions were added after this household applied. Those blanks are `not testable on this historical application`, not missing-information findings.
- LeadSimple contains both an `applicant` contact and a later `tenant` contact for each of the three adults. This confirms that the future move-in process must deliberately use tenant contacts, while application intake must use applicant contacts.
- The applicant contacts' `RV Applicant ID` custom fields are blank, and no `PM Applicant to Lease` process instance exists for this historical household. Do not match or update by name alone. At intake, save the RentVine application ID and every adult applicant ID before any household routing.
- Because the household is approved and converted, it must not be pushed through the new workflow. Use it only to prove extraction logic; perform workflow writes on the dedicated safe test process or on a future controlled pilot with outbound communication disabled.

- Link preparation saves the correct token and URL before any notice is sent.
- Email and text previews show the consolidated items, working link, deadline, and correct recipients.
- Complete response produces `VALIDATED` and only the validated LeadSimple route. **Passed for the controlled all-six-category response; downstream automatic route execution remains to be observed.**
- Partial or invalid response produces `NEEDS_REVIEW` and only the staff task.
- Animals, documents, credit correction, other information, monthly rent, and reason for leaving are received and validated correctly.
- Altered token, altered requested-items code, missing routing values, wrong purpose/schema, and a process in the wrong stage are rejected without updating the process.
- A repeated Jotform submission ID is idempotent and does not send duplicate messages or repeat updates.
- Three-business-day timing and the Monday–Friday, 8:00 a.m.–8:00 p.m. Central schedule behave as approved.
- Withdrawal updates the matching RentVine application to `Withdrawn Incomplete`.
- Missing email, missing mobile number, integration error, and manual staff override follow documented exception behavior.

## Documentation required for every future process

Create or update one process-specific page containing:

- Purpose and owner
- LeadSimple process name and ID
- Stage map
- Custom-field dictionary
- Conditional-task truth table
- Automatic stage-routing truth table
- Email and text template inventory
- Jotform inventory and hidden/read-only fields
- Zap inventory with trigger, filters, paths, and actions
- RentVine records/events used
- Deadlines and business hours
- Exception and staff-override rules
- Test cases and release status
- Change log

## Related Notion references

- [LeadSimple Process Naming Convention](https://app.notion.com/p/LeadSimple-Process-Naming-Convention-35c23e9aa5a381218b3ec14e23938ab0)
- [Move-In Workflow (Jotform → LeadSimple)](https://app.notion.com/p/Move-In-Workflow-Jotform-LeadSimple-35223e9aa5a38127bc97f5fc21a70aff)
- [LeadSimple Cheat Sheet](https://app.notion.com/p/LeadSimple-Cheat-Sheet-34823e9aa5a3818eb152db0f98870091)
- [Jotform Cheat Sheet](https://app.notion.com/p/Jotform-Cheat-Sheet-34823e9aa5a3818c84c1fd39e76c0b34)
- [RentVine Cheat Sheet](https://app.notion.com/p/Rentvine-Cheat-Sheet-34823e9aa5a38174b8e4db0a05a3945d)

## Change log

- **2026-08-08:** Completed the Draft/off move-in-date confirmation architecture. LeadSimple now uses mutually exclusive conditional staff-input steps, a guarded confirmation stage, household email/SMS, a two-business-day staff-call exception, an accepted route to `Lease Preparation`, and a different-date loop back to staff approval. Created disabled Jotform `DWC Move In Date Confirmation` (`262200946591053`) with controlled choices, conditional alternate date, required signer identity/certification/signature/signing date, read-only approved values, and hidden token/schema/routing fields. Added fail-closed link/precheck/authoritative-validator code plus a passing local self-test. Overall build estimate is 83%; no communication, Zap publication, LeadSimple Live Mode, or production write was triggered.
- **2026-08-08:** Added `[[PM Applicant to Lease - Tenant Options Secure Automation]]`. Hardened Jotform `251564975684069`, completed Draft/off link-builder Zap `375704730`, drafted signed-return Zap `375705864`, added the native LeadSimple expired/error staff task, and documented the remaining Jotform webhook connection and synthetic release tests. Overall build estimate is 77%; no communication or production write was tested.
- **2026-08-07:** Created and tested Draft Zap `PM Applicant to Lease | Intake Validation` (ID `375555217`). It triggers on LeadSimple process creation, uses an exact test-only process/stage/name guard, validates the eleven authoritative household inputs, and returns deterministic normalized values, an idempotency key, and a source fingerprint. The controlled test returned `READY_FOR_DRAFT_ENRICHMENT`, `errors = NONE`, and `productionReady = false`. No downstream write or communication action was added. Release remains blocked pending supported RentVine numeric-score, custom-question, upload-metadata, and other-income retrieval plus exact lead-to-process and applicant-contact ID linkage.
- **2026-08-07:** Created isolated LeadSimple process `TEST - PM Applicant to Lease - INTAKE ONLY - DO NOT PROCESS` (ID `c69fde70-8a25-4ec4-a41c-b5b5007c3cc8`) in `Intake Validation`. Verified Draft Mode, zero contact roles, zero properties, no uncompleted starting-stage tasks, and a stored no-communication/no-RentVine-write safety comment. The earlier missing-information test process was not reset or altered.
- **2026-08-07:** Completed the first controlled intake write-through on the isolated process. Verified application `66`, adult applicant IDs `82,83,84`, three adult scores, average score `538`, household income `$9,300`, rent `$2,195`, exact ratio `4.24x`, and conflicting requested dates without attaching contacts/property or sending communication. LeadSimple Number fields rounded `4.24` to `4`, so the field was relabeled `Income Multiple Rounded - Audit Only`; `Income Multiple Calculation` preserves the exact formula and `Income Requirement Met` is the authoritative conditional-logic field. The process remains in Draft Mode, `Intake Validation`, with `Automation Status = Waiting`.
- **2026-08-06:** Completed a controlled all-six-category Jotform return test. Verified prefill, conditional sections, document upload, signature, purpose/schema/token filters, complete validation, LeadSimple response writeback, the `Missing Information Response Review` stage, and one-use `USED-<submission ID>` token behavior. Repaired the final action's expired LeadSimple connection and changed successful remaining-items output to `NONE` to prevent stale summaries. Both Zaps remain draft/off.
- **2026-08-06:** Confirmed that the RentVine application should collect stable standard applicant information first. The missing-information Jotform is an exception/correction path and must request only gaps remaining after RentVine data is evaluated.
- **2026-08-06:** Reconciled the standard after a read-only audit. Added the `Missing Information Link Preparation` gate before applicant notices, documented Jotform `262175267632056`, Zap IDs `375483019` and `375481302`, the authoritative LeadSimple-field/token design, the complete validator mapping checklist, and the required release tests. Both Zaps and LeadSimple Live Mode must remain off until the corrected configuration passes those tests.
- **2026-08-06:** Added process-level LeadSimple field `Missing Information Response Summary` (text) and `Missing Information Validation Result` (multiple choice with fixed values `VALIDATED` and `NEEDS REVIEW`) to `PM Applicant to Lease` only. Verification showed that LeadSimple text-field conditions support only empty/any-value comparisons, so the first text version was renamed `Missing Information Validation Result (Legacy Text)` and removed from Zap mappings and workflow decisions.
- **2026-08-06:** Verified the deterministic validator inputs in Zap `PM Applicant to Lease | Missing Information Submitted` (Zap ID `375481302`) for `INCOME`, `RENTAL_HISTORY`, `ANIMALS`, `DOCUMENTS`, `CREDIT_CORRECTION`, and `OTHER`. The authoritative requested-items input now comes from the retrieved LeadSimple process. An action-level test passed on the safe test process; full trigger/filter path testing remains pending.
- **2026-08-06:** Added LeadSimple active stage `Missing Information Response Review`. The return Zap now records the submission ID, response summary, validation result, remaining-item summary, and remaining-item count, then moves the household into this stage. Controlled testing moved only the safe test household to this stage; the Zap remains unpublished.
- **2026-08-06:** Confirmed the two intended workflow rules in `Missing Information Response Review`: the stage change displays only for exact `VALIDATED`, and the staff-review task displays only for exact `NEEDS_REVIEW`.
- **2026-08-06:** Deliberately did not map the validator's income output directly into canonical `Household Monthly Income`. A partial or blank response must not overwrite a previously verified household total; income will be written only through a later conditional, household-aware path.
- **2026-08-06:** Replaced the earlier per-adult missing-information-link design with one household link. Created and verified the LeadSimple process fields `Household Missing Information Form Link`, `Missing Information Requested Items Code`, `Missing Information Response Token`, `Missing Information Form Submission ID`, and the primary-applicant prefill fields.
- **2026-08-06:** Corrected Zap `PM Applicant to Lease | Missing Information Link Builder` (Zap ID `375483019`) to create a response token and encoded link for form `262175267632056`. It filters for `Missing Information Link Preparation` and advances to `Missing Information Pending` only after the URL and token are saved. A controlled action test passed on the safe test process; the Zap remains unpublished pending the full release checklist.
- **2026-08-06:** Drafted Zap `PM Applicant to Lease | Missing Information Submitted` (Zap ID `375481302`) for the household-link design. The intended filters require form purpose `missing_information`, schema `v1`, a token, a process ID, and an exact stored-token match. Full mappings, authoritative stored requested-items validation, duplicate protection, and end-to-end behavior remain subject to the release checklist. The Zap remains unpublished.
- **2026-08-06:** Removed the `AUTO` process-title suffix by owner direction. Automation is documented in descriptions and workflow documentation, not process names.
- **2026-08-06:** Initial standard created from the PM Applicant to Lease design session. Established LeadSimple-first conditional logic, per-adult secure links, Zapier return validation, RentVine synchronization, and end-to-end testing requirements.
