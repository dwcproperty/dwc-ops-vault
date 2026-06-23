# Late Fee Relief — Prompt 9: Fix the JotForm → LeadSimple process-matching, then finish the smoke test

> Saved to vault 2026-06-23 (Claude Code) so it survives outside Cowork's outputs folder.
> Authoritative fix-and-finish doc. Hand to a browser-capable agent (Cowork / Claude-in-Chrome)
> for the Zapier + LeadSimple-UI steps. Claude Code owns the JotForm-MCP fixes + MCP verification.
> See the live-state addendum at the bottom (verified via connectors 2026-06-23).

Hand this to an agent that has browser access to the user's LeadSimple, Zapier, and JotForm accounts (e.g. Cowork / Claude-in-Chrome), plus the connected LeadSimple API connector. This is a continuation of an 8-prompt build. Prompt 8 (end-to-end smoke test) failed at Step 4 and we are resuming from that failure. Take it slow and STOP on the first failure rather than push through — a half-tested production flip is worse than a delayed one.

## 0. Background — what was already built (prompts 1–7)
A "Late Fee Relief" automation spanning LeadSimple + JotForm + Zapier:

- LeadSimple process type "Acct Late Fee Relief" — 5 stages, 8 process custom fields, conditional stage branching (Stage 3 → Stage 4 on Approved, Stage 3/Denied → Stage 5), and a Stage-4 task whose due-date binds to a custom field.
  - Currently set to ACTIVE / Live (flipped from Draft during the smoke test so a real process could be started — see section 3). If the test ultimately fails and can't be fixed, revert it to Draft.
- JotForm form 261667477849074 ("One-Time Late Fee Courtesy Request — DWC Property Group") with a configured autoresponder (token {residentFull}), e-signature, and URL-prefill support.
- Prompt-6 Zap "Late Fee Relief — JotForm submission → LeadSimple" (Zap id 369741303, ON):
  - Jotform — New Submission (form 261667477849074)
  - LeadSimple — Search Process (email fallback: free-text Search = tenant email; "Successful if no results = False/halted"). This is the step that fails — see section 1.
  - LeadSimple — Update Process (Process ID from step 2; Stage → "Review & Decision"; Committed Payment Date ← "Date Payment Will Be Made"; JotForm Submission ID / Link ← {id} — https://www.jotform.com/submission/{id})
  - LeadSimple — Create Process Note (Process ID from step 2; note body summarizing the submission)
- Prompt-7 Zap "Late Fee Relief — stamp contact once-per-year date" (Zap id 369748081, OFF, partially wired):
  - Trigger: LeadSimple — Process Changes Stage (no stage filter at trigger level).
  - Filter (only condition 1 wired): Stage Name Exactly matches Record & Close. Still to add: (2) Relief Decision Exactly matches Approved, (3) Relief Granted Date Does not exactly match (blank) i.e. "is not empty".
  - Action: LeadSimple — Update Contact (Contact ← trigger Tenant > ID). Still to add: value mapping Late Fee Relief — Last Granted Date ← trigger Relief Granted Date.

## 1. The failure we are fixing (prompt 8, Step 4)
A real Acct Late Fee Relief process was started for a test tenant and the prefilled JotForm was submitted. The prompt-6 Zap ran (Jun 23, 2026 10:32:01 am) and safely halted:

- Step 1 (JotForm trigger): green — captured Submission Id 6580379207711477973, email office+lfrtest@dwcproperty.com, lease ref L-TEST-001, reason "Test smoke run".
- Step 2 (Search Process, email fallback): HALTED — "safely halted because nothing was found in LeadSimple." The email-based search did not find the process.
- Steps 3 & 4: skipped.

**Root-cause diagnosis.** LeadSimple's Zapier "Search Process" action only exposes a single free-text Search box (no custom-field filter — that's why prompt 6 fell back to email). The test process was started as a sub-process of a Rentvine Tenants lead with no property and no directly-attached tenant contact (its name even rendered as the literal `Acct Late Fee Relief for {{property.street}}`). So the process carries no tenant email as a searchable attribute, and the email lookup returns nothing.
**Conclusion:** the email-fallback matching is not reliable. We need a deterministic way to map a JotForm submission to its LeadSimple process. (The process-type structure itself works fine — stages, sequential Stage-1 tasks, and the currency custom fields all behaved correctly.)

## 2. Part A — Fix the process-matching (do this first)
Goal: a JotForm submission must map to exactly one Acct Late Fee Relief process without fuzzy search.

**Option A (recommended) — carry the LeadSimple Process ID through the form**
- In JotForm form 261667477849074, add a short-text field to carry the LeadSimple Process ID (or repurpose the existing Account / Lease Reference Number field — URL-prefill name textbox5 — to carry the process ID). Make it prefillable and, if possible, hidden/read-only.
- Update the LeadSimple Stage-1 "Generate prefilled JotForm link" task so the generated URL injects the current process's ID into that field — via a LeadSimple merge token for the process ID if one exists, otherwise instruct the PM to paste it from the process URL (.../processes/<PROCESS_ID>).
- Rework the prompt-6 Zap (369741303): remove the email-based Search Process (step 2) and feed the process ID from the JotForm submission straight into Update Process and Create Process Note. If a lookup is still desired, use LeadSimple "Find Process by ID" (exact), not the free-text search.

**Option B** — make the process searchable, keep email search: attach the tenant contact and a property when starting the process so the email is searchable, then verify empirically whether Search Process matches a linked-contact email (uncertain).

**Option C** — search by lease reference (textbox5, e.g. L-TEST-001) if the process can be made searchable by it.

**Acceptance for Part A:** a fresh test submission drives the prompt-6 Zap all three actions GREEN, advancing the test process to Stage 3 (Review & Decision) with JotForm Submission ID/Link + Committed Payment Date populated and a new process note.

**Also fix this JotForm bug:** the "Total Balance Acknowledged" field has a maximum value of 100, so it rejects realistic dollar amounts (1500.00 was rejected with "Input should not be greater than the maximum value: 100"). Remove/raise the max, or confirm it should stay optional/blank.

## 3. Part B — Re-run the submission and verify (Step 4 redo)
Use the existing test artifacts (section 6). The test process is currently in Stage 1; reset it to Stage 1 if a prior partial run moved it.

- Generate the (now process-ID-bearing) prefilled link and submit the form: Reason = "Test smoke run"; Date Payment Will Be Made = 3–5 business days out; check all 5 acknowledgments; sign; Date Signed = today.
- Verify (STOP and report if any fail): (a) autoresponder email arrives at office+lfrtest@dwcproperty.com addressed to "Test Tenant LFR Smoke"; (b) Prompt-6 Zap History shows all 3 steps green; (c) the test process is now Stage 3 with JotForm Submission ID/Link + Committed Payment Date populated and a new note.

## 4. Part C — CHECKPOINT (finish prompt-7), then Steps 5–12
Once the process reaches Stage 3, a real Process Changes Stage event exists.

**CHECKPOINT — finish the prompt-7 Zap (369748081, OFF):**
- Re-test the trigger; pick the test process's real stage-change sample.
- In the Filter, keep condition 1 and add: (2) Relief Decision Exactly Matches Approved; (3) Relief Granted Date Does Not Exactly Match (blank).
- In Update Contact, map Late Fee Relief — Last Granted Date ← trigger Relief Granted Date.
- Test the action (filter should halt for a Stage-3 sample — expected). TURN THE ZAP ON.

**Then complete Steps 5–12:**
- 5 — Stage 3 (Approved): set Relief Decision = Approved; confirm auto-advance to Stage 4.
- 6 — Stage 4: skip real Rentvine void; advance to Stage 5.
- 7 — Stage 5: set Relief Granted Date = today; prompt-7 Zap auto-stamps the contact; add closing note; close.
- 8 — Verify: prompt-7 Zap green; contact's Late Fee Relief — Last Granted Date = today.
- 9 — Once-per-year re-trigger: second process, Relief Decision = Denied, advance to Stage 5/close; verify prompt-7 Zap HALTS.
- 10 — Cleanup: delete both test processes; clear the contact's Last Granted Date; delete the test lead; keep the contact (tag "smoke test").
- 11 — Flip Active: only if Steps 1–10 pass → leave Acct Late Fee Relief Active.
- 12 — Publish 06 Delinquencies: verify the draft still has the two prompt-4 additions, then Publish.

**Stop-on-failure / revert:** if any verification fails and can't be fixed, STOP, report, and revert Acct Late Fee Relief to Draft.

## 5. Report back
Test email used; which fix (A/B/C) and why; stage-by-stage Zap timing; any failures and handling; whether Steps 11 & 12 were reached; final state of Acct Late Fee Relief (Active/Draft), 06 Delinquencies (Live w/ steps / Live w/o / Draft), both Zaps (ON/OFF).

## 6. Key identifiers (2026-06-23)
**LeadSimple**
- "Acct Late Fee Relief": UUID d00f54a7-46e5-4b91-bd7e-53acb7d3314c; LS id WMd2_kUaNDSZp6_GpJKuMT22KKjHyUtjMKkVbVXzUt5h0AukSawm.
- Stages → IDs: Gather Info 79f0f87c-a7c7-42fe-be60-fef0a94b92be; Awaiting Submission 559f0b03-8f78-4704-a649-cafc9779ea0a; Review & Decision 21c3ce6e-20a5-4407-bb79-abb6b257a556; Verify Payment 1f7159c8-efa5-4888-9ce6-a44f1aaea532; Record & Close 77cecb81-2cd3-45cf-9e2b-77947a1afb85.
- Contact field key: late_fee_relief_last_granted_date (date, global).
- "06 Delinquencies Late Rent" id d5390b5d-f766-41f8-9f41-0cb50579aa02.
- Rentvine Tenants pipeline 3142904f-de86-401c-8623-bd2b105ad2d8 (Active stage e56030bf-..., Pending 08bf0e8f-...).
- ⚠️ LeadSimple connector write actions (create_process/create_deal) are broken via MCP — start processes/leads through the UI (a process starts from a host Lead via Sub Processes → "+ Add a related process").

**Test artifacts (still in place)**
- Contact "Test Tenant LFR Smoke": id 45d41db3-159b-4872-a185-13e862a080f6; email office+lfrtest@dwcproperty.com; URL https://app.leadsimple.com/v2/contacts/S9p36UEKM0_S4_3cpJGvPib4LaeVw4i2ycm4G2L1qgwQ9TSGvyY=
- Rentvine Tenants lead (Pending): https://app.leadsimple.com/v2/pipelines/WNxp-EwAKQXP5PzQq5W1MTivKaxPA7Osc1YsL54xbCakdeSA/deals/TNB48Q9YfljT7_vbro6qNTr9KEOYhFsjtIEhTkM1bQunBQA=
- Test process (Stage 1; Late Fees $50, NSF $0, Outstanding $1,450): .../processes/WMd2_kUaNE_V7v3frpajKTiqKKiTgE86jclvbqp3VgL26g8aXQ== (UUID 234b4c02-f016-404a-94fc-4ba7442f19af)
- Failed submission Id 6580379207711477973 (Date Payment 06-26-2026).

**JotForm** — form 261667477849074; prefill params residentFull, textbox2 (address), email3, phoneNumber, textbox5 (lease ref — candidate to repurpose for the process ID). "Total Balance Acknowledged" has a max-100 bug.

**Zapier** — prompt-6 Zap 369741303 (ON); prompt-7 Zap 369748081 (OFF, partial). Connections: "LeadSimple API Key for Darrell Calhoun", "Jotform Darrell_Calhoun".

---

## Live-state addendum — verified via MCP connectors 2026-06-23 (Claude Code)
- Process type `Acct Late Fee Relief` (d00f54a7-…) confirmed **LIVE**, owner Darrell Calhoun. All 5 stage IDs above confirmed.
- Test process `234b4c02-f016-404a-94fc-4ba7442f19af` confirmed in Stage 1 (Gather Info), **properties [] and contact_roles [] (orphaned)** — name still renders literally as `Acct Late Fee Relief for {{property.street}}`. Created fresh today 15:21 UTC, not closed.
- Test contact `45d41db3-…` confirmed; `late_fee_relief_last_granted_date` = null (not yet stamped).
- JotForm 261667477849074 confirmed live ("One-Time Late Fee Courtesy Request"). Fields seen: Resident Full Name, Property Address, Email Address, Phone Number, Account / Lease Reference Number, Payment Arrangement, Reason Payment Is Late, Date Payment Will Be Made, Total Balance Acknowledged, 5 acknowledgment checkboxes, Resident Signature (e-sign), Date Signed.

### Execution split (who does what)
- **Claude Code (MCP) can do:** JotForm form edits (add hidden process-ID field / fix the max-100 bug) via Jotform connector; fire the re-test submission via create_submission; verify LeadSimple process/contact state read-only; update_contact for cleanup.
- **Cowork / browser must do:** rewire Zap 369741303, finish + turn on Zap 369748081, edit the LeadSimple Stage-1 "Generate prefilled JotForm link" task, start/advance/close processes (LS create-writes are broken via MCP), flip Active/Draft, Publish 06 Delinquencies. Zap run-history verification is also browser-only.

---

## Part A progress — JotForm fixes DONE via MCP (2026-06-23, Claude Code)
- **max-100 bug FIXED:** "Total Balance Acknowledged" (qid 12, name `q12_number10`) — maxValue cleared, now accepts any positive amount.
- **Hidden carrier field ADDED:** label **"LeadSimple Process ID"**, **qid 27**, qname/name **`leadsimpleProcess`**, type textbox, `hidden=Yes`, not required, placed just before Submit. ssoPrefillKey `LeadSimpleProcessID`.
  - **URL prefill param to use in the Stage-1 link:** `?leadsimpleProcess=<PROCESS_ID>` (e.g. `https://form.jotform.com/261667477849074?leadsimpleProcess=234b4c02-f016-404a-94fc-4ba7442f19af`). To be confirmed with one live test submission.
  - In the Zap "New Submission" trigger this surfaces as **"LeadSimple Process ID"** — map it into Update Process / Create Note as the Process ID.
- **OPEN for Cowork to confirm empirically:** does LeadSimple's Zapier "Update Process" Process ID field want the **UUID** (`234b4c02-…`) or the **encoded LS id** (`WMd2_kUaNE_V7v3frpajKTiqKKiTgE86jclvbqp3VgL26g8aXQ==`)? Prefill default = UUID; switch if the action rejects it.

## Part A (Zap) + Part B (re-test) COMPLETE & VERIFIED (2026-06-23)
- **Zap 369741303 rewired** (Cowork): email Search Process step deleted; Update Process + Create Note now map Process ID ← trigger "LeadSimple Process ID". Published v2, ON. **Process ID format = UUID** (e.g. `234b4c02-…`); encoded `WMd2…==` id NOT used.
- **Stage-1 link task updated** (Cowork) to append `&leadsimpleProcess=<PROCESS_UUID>`. ⚠️ No LeadSimple merge token exists for the process UUID, and the process-page URL shows the encoded id (not the UUID the Zap needs) — so the per-tenant link-builder must fetch the UUID via the LS API (get_process), not copy it from the address bar.
- **Hidden-field URL prefill WORKS:** Zap run 013ffa18… received "LeadSimple Process ID" = `234b4c02-…` non-empty. No unhide needed.
- **Re-test submission** (Cowork, browser — MCP create_submission can't submit this form: e-signature `control_signature` unsupported): submission id 6580451557714329112. All 3 Zap steps green. Process advanced **Stage 1 → Stage 3** (reset to Stage 1 at 12:26, submission drove it to Review & Decision at 12:32). JotForm Submission ID/Link + Committed Payment Date populated; new process note written.
- **Verified by Claude Code (MCP):** (a) get_process → Stage 3 "Review & Decision"; (b) Gmail → autoresponder at office+lfrtest@ 17:33:17Z, greeting "Hi Test Tenant LFR Smoke," (residentFull token OK).
- **max-100 fix confirmed live:** form accepted Total Balance Acknowledged = 1500.00.
- **Data nit (not a defect):** Committed Payment Date came through as 06-26 not 06-30 — the JotForm "Date Payment Will Be Made" lite-date widget kept its Month/Day/Year sub-fields and ignored the combined-field edit. Only affects the manually-entered test value; the date propagated through the pipeline correctly. No production impact (tenants fill the widget normally). No resubmit needed.
- **Infra note:** gmail-domain MCP returned "Reauthentication is needed (gcloud auth application-default login)" — ADC expired; used the native claude.ai Gmail connector as fallback. gmail-domain needs ADC refresh before it's usable again.

### Still TODO (Part C — Cowork browser): finish Zap 369748081, then Steps 5–12.
Claude Code can verify Step 8 (contact `late_fee_relief_last_granted_date` = today) and do the Step-10 contact-field clear via MCP update_contact.

## Part C progress (2026-06-23)

### CHECKPOINT — Zap 369748081 finished + turned ON (v1) ✅
Cowork wired and published it ON:
- **Trigger:** LeadSimple "Process Changes Stage". Re-tested; selected the test process's real sample (UUID 234b4c02…, type "Acct Late Fee Relief", stage "Review & Decision"). Sample exposes `custom_relief_decision` + `custom_relief_granted_date` (both empty at Stage 3, as expected).
- **Filter (3 conditions, AND):** (1) Stage Name Exactly = "Record & Close"; (2) Custom Relief Decision Exactly = "Approved"; (3) Custom Relief Granted Date Does Not Exactly Match (blank) → i.e. not empty.
- **Action:** Update Contact → "Late Fee Relief — Last Granted Date" ← trigger "Custom Relief Granted Date". Contact identifier left as the original-build mapping ("Tenant > ID").
- **Per-step test intentionally skipped:** Zapier per-step tests bypass the filter, so testing against the Stage-3 sample would have fired a real Update Contact with an empty date. Real stamping path is exercised at Step 7 instead.
- Cowork flags: (a) filter condition-3's value had to be set via a scripted DOM click (normal click wasn't registering) — verified on screen; (b) the contact-identifier token was not re-verified to resolve (see blocker below).

### BLOCKER hit on tenant resolution (genuine) ⚠️ — being handled via hardcode-for-test
- The test process is **orphaned**: `contact_roles: []`, `properties: []` (confirmed via get_process). So prompt-7's Update Contact "Tenant > ID" identifier has nothing on the process itself to resolve.
- Cowork confirmed the **"Acct Late Fee Relief" process type defines NO contact-role slot**, so there is no UI affordance to attach a contact/Tenant to the process. (LeadSimple normally derives Tenant from a linked property's lease.) The test contact "Test Tenant LFR Smoke" is only a **Tenant on a Lead**, not on any property.
- **House rule honored:** did NOT create a throwaway property to carry the tenant. [[feedback_never_create_property]]
- **Resolution for the smoke test:** temporarily **hardcode** the test contact id `45d41db3-159b-4872-a185-13e862a080f6` in prompt-7's Update Contact identifier (same technique used to first prove prompt-6's process UUID), run Steps 5–9, then **REVERT** to the dynamic "Tenant > ID" mapping so production doesn't stamp the test contact.
- **Side-finding:** the autoresponder also shows on the contact's LeadSimple Conversations ("We've received your late fee relief request…") — confirms Step-4 email path from the contact side too.

### OPEN PRODUCTION QUESTION (must resolve before the Step-11 Active flip)
How does prompt-7 identify the right contact to stamp **in production**?
- If production starts the Late Fee Relief process **on the tenant's property** → the lease supplies the Tenant role → "Tenant > ID" resolves. ✅
- If production starts it as an **orphaned sub-process of a lead** (as the build/test did) → "Tenant > ID" may be empty live → prompt-7 can't stamp. ❌ Would need either: (a) production SOP = start the process on the property, or (b) add a Tenant contact-role to the process type, or (c) a different contact identifier.
- **Awaiting Darrell's call on the production start-flow.** Cowork was also asked to report what "Tenant > ID" resolves to in the test sample (populated from the parent lead, or empty) — that datapoint informs this decision.

### Remaining steps after the stamp test
8 (Claude Code MCP-verifies contact `late_fee_relief_last_granted_date` = 2026-06-23) · 9 (Denied → halt) · 10 (revert identifier; delete both test processes + lead; keep+tag contact; Claude Code clears the contact date via update_contact) · 11 (flip Active — gated on the production question above) · 12 (Publish "06 Delinquencies Late Rent" d5390b5d-… after confirming the two prompt-4 additions).

### Steps 5–8 DONE + stamp VERIFIED (2026-06-23)
- **Stamp test ran via hardcoded contact** (prompt-7 Update Contact temporarily set to literal `45d41db3-…`, published v2 "TEMP smoke test — hardcoded contact (revert before production)").
  - Step 5: Relief Decision = Approved → conditional branch produced the Approved "Advance to Stage 4" task → Verify Payment & Apply Relief.
  - Step 6: Stage-4 tasks completed (Rentvine void marked done WITHOUT a real void) → Stage 5.
  - Step 7: Relief Granted Date = 2026-06-23 set **just before** moving to Record & Close (the Zap triggers on the stage change, so the date must be present at that moment for the filter to pass) → process Completed.
- **Step 8 — prompt-7 Zap GREEN:** run 013ffa18-6e27… (v2, 01:37:23 pm) Process Changes Stage → Filter "passed the rules" (Record & Close + Approved + Granted Date present) → Update Contact "Sent 1 new Contact." The 01:32 run on the Stage-4 change correctly showed Filtered (0). Filter verified both directions.
- **Claude Code MCP verification:** contact `45d41db3-…` `late_fee_relief_last_granted_date` = `1782190800` = **2026-06-23** (00:00 Central); contact updated_at `2026-06-23T18:37:25Z` matches the Zap run. ✅

### KEY PRODUCTION FINDING — Tenant > ID is EMPTY for orphaned processes
- Cowork searched the live prompt-7 trigger sample for the test process: **"Tenant" → No matches found** (the orphaned process exposes no tenant field; get_process confirms `contact_roles: []`).
- **Conclusion:** as wired, prompt-7 only stamps when the process carries a tenant contact-role. Processes started **without** one (orphaned sub-process of a lead — how the build/test started them) will NOT resolve a tenant live.
- **DECISION NEEDED before Step 11 (flip Active).** Options:
  (a) **Production SOP = start the Late Fee Relief process on the tenant's property** (lease supplies Tenant > ID). Cleanest if processes are always property-linked. No Zap change.
  (b) **Add a Tenant contact-role to the "Acct Late Fee Relief" process type** so the PM links the tenant directly when starting it (works even off a lead). Process-type change + confirm the role surfaces as "Tenant" in the trigger.
  (c) Different identifier (e.g., a tenant-email custom field on the process written by prompt-6, matched in prompt-7) — more rework.
- **Recommendation (pending Darrell):** (a) if the real start-flow is always property-anchored; otherwise (b). Do NOT flip Active until this is settled, or production approvals silently fail to stamp (breaking once-per-year enforcement).

### Remaining: Step 9 (Denied → verify HALT) · Step 10 (REVERT hardcode → "Tenant > ID"; delete both test processes + lead; keep+tag contact; Claude Code clears the contact date via update_contact) · Step 11 (flip Active — GATED on the decision above) · Step 12 (Publish "06 Delinquencies Late Rent").

### Steps 9–10 DONE (2026-06-23)
- **Step 9 — Denied path HALTS ✅:** second process (Denied, db5369f4) advanced to Record & Close → prompt-7 Zap "This filter successfully stopped your run" (0 tasks). Once-per-year guard verified (only Approved stamps).
- **Step 10 — revert + cleanup:**
  - **Revert ✅:** Zap 369748081 Update Contact back to dynamic "Tenant > ID" (hardcoded id gone, verified on screen). Published **v3, ON**. Production will not stamp the test contact.
  - **Deletions:** both test processes (234b4c02 Approved, db5369f4 Denied) + the test lead deleted.
  - ⚠️ **Cascade:** deleting the test lead **cascade-deleted its primary contact 45d41db3** (URL 404s; search "Test Tenant LFR" → none). So "keep + tag smoke test" couldn't be done — nothing left to tag. Net: ALL test data removed (processes + lead + contact). Claude Code's planned API clear of the stamp is now moot (contact gone). Lesson: deleting a LeadSimple lead cascade-deletes its primary contact — tag/detach the contact BEFORE deleting the lead next time.
- JotForm test submissions (6580379207711477973, 6580451557714329112) still exist in JotForm — harmless test data; optional to delete.

### HOLDING at Step 11 — production tenant-resolution decision (see options above)
Test is fully green end-to-end; the ONLY blocker to flipping Active is deciding how prompt-7 resolves the tenant in production (Tenant > ID confirmed empty on orphaned processes). Awaiting Darrell's choice: (a) SOP = start process on the tenant's property; (b) add Tenant contact-role to the process type (+re-test); (c) other identifier. Step 12 (Publish "06 Delinquencies Late Rent") still pending after Step 11.

### DECISION — production tenant-resolution = path (a), NO rework (2026-06-23, Darrell)
Production Late Fee Relief processes are spawned from **"06 Delinquencies Late Rent"** (d5390b5d-f766-41f8-9f41-0cb50579aa02), which already carries the **tenant, property, and owner**. The spawned process therefore inherits the property → the lease supplies **Tenant > ID** → prompt-7 resolves and stamps the correct contact. The orphaned-process failure seen in testing was a test-setup artifact (we started from a bare Rentvine Tenants lead). **No Zap or process-type change required.**

**The one production-correctness check (do at Step 12):** confirm the two prompt-4 additions in 06 Delinquencies start the Late Fee Relief process **carrying the property/tenant** (so Tenant > ID is non-empty live), not as a bare sub-process.

**Safety net:** monitor the FIRST real Late Fee Relief case end-to-end — confirm (1) the spawned process carries the tenant, and (2) prompt-7 stamps that tenant's `late_fee_relief_last_granted_date`. Claude Code can MCP-verify that first real stamp.

### Cleared for Steps 11–12
- **Step 11:** leave "Acct Late Fee Relief" (d00f54a7-…) **Active** (already Active from the test).
- **Step 12:** in "06 Delinquencies Late Rent" draft, verify the two prompt-4 additions are present AND property-link the spawned process, then **Publish**.
After this the automation is fully live.
