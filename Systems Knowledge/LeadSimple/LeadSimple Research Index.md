---
type: index
status: active
authority: DWC Systems Knowledge library
last_verified: 2026-08-09
superseded_by: null
---

# LeadSimple Research Index

**Canonical research:** `C:\Users\Darrell\Documents\DWC Property Group\Systems Knowledge\LeadSimple\LeadSimple Deep Dive.md`  
**Build checklist:** `C:\Users\Darrell\Documents\DWC Property Group\Systems Knowledge\LeadSimple\Process Build Checklist.md`  
**Verified:** 2026-08-09

## Core rule

A LeadSimple process is a field-driven state machine with attached property/unit records, contact roles, structured state, stop/wait steps, conditional routes, and controlled communication. It is not complete when only stages and task labels exist.

## Live audit flags

- `PROPERTY HOA Setup`, `TENANT HOA Registration and Amenity Access`, and `ANNUAL HOA Verification` were observed in **Live Mode**.
- `PM Applicant to Lease` is Draft, but its General settings have no applicant contact roles and no default sender.
- `TENANT HOA Registration and Amenity Access` and `ANNUAL HOA Verification` show no associated custom fields.
- `ANNUAL HOA Verification` also shows no contact roles and no default sender.
- Current DWC UI says email/SMS templates are shared account-wide and edits update every use. Treat template edits as global-impact.
- Retired process names currently use `OLD` and `LEGACY DO NOT START`, while the playbook specifies a `ZZZ` retirement pattern.
- `PROPERTY HOA Setup` has a wrong-type legacy property field: `HOA DWC Registration Status Legacy Error` is Currency and must not be used for logic.

## Existing related notes

- [[Automation/DWC Integration Automation Standard]]
- [[Automation/Chat Saves/2026-04-25 LeadSimple Variant Builder Phase 0]]
- [[Automation/Chat Saves/2026-04-25 LeadSimple Variant Builder Phase 2 Shipped]]
- [[Automation/PM Applicant to Lease - Intake Mapping and QA]]
- [[Automation/PM Applicant to Lease - Move In Date Confirmation]]
- [[Automation/PM Applicant to Lease - Tenant Options Secure Automation]]

The two canonical workspace files contain the source-linked product model, DWC configuration findings, contradictions, failure modes, build sequence, and release checklist.
