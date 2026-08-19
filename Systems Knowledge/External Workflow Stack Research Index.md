---
type: index
status: active
authority: DWC Systems Knowledge library
last_verified: 2026-08-09
superseded_by: null
---

# External Workflow Stack Research Index

Last verified: 2026-08-09

This page indexes the canonical DWC research for Zapier, Jotform, and RentSign. The source files live in the DWC Property Group workspace so process builders use one maintained standard rather than copying stale Notion notes.

## Canonical references

- [Zapier Deep Dive](../../../DWC%20Property%20Group/Systems%20Knowledge/Zapier/Zapier%20Deep%20Dive.md)
- [Zapier Workflow Build Checklist](../../../DWC%20Property%20Group/Systems%20Knowledge/Zapier/Workflow%20Build%20Checklist.md)
- [Jotform Deep Dive](../../../DWC%20Property%20Group/Systems%20Knowledge/Jotform/Jotform%20Deep%20Dive.md)
- [Jotform Form Build Checklist](../../../DWC%20Property%20Group/Systems%20Knowledge/Jotform/Form%20Build%20Checklist.md)
- [RentSign Deep Dive](../../../DWC%20Property%20Group/Systems%20Knowledge/RentSign/RentSign%20Deep%20Dive.md)
- [RentSign Document Build Checklist](../../../DWC%20Property%20Group/Systems%20Knowledge/RentSign/Document%20Build%20Checklist.md)
- [DWC Process Architecture Packet](../../../DWC%20Property%20Group/Systems%20Knowledge/DWC%20Process%20Architecture%20Packet.md)
- [Knowledge Conflict Register](../../../DWC%20Property%20Group/Systems%20Knowledge/Knowledge%20Conflict%20Register.md)

## Rules that must survive every build

1. External systems write verified facts; LeadSimple owns workflow conditions and routing.
2. Hidden Jotform values are not trusted. Re-read authority, validate token/form/version/process/stage/deadline, and enforce idempotency.
3. A Zap is not release-ready because its validator passes; its business Paths, writeback, read-back, error route, and replay safety must also be complete.
4. RentSign completion means every required party signed the exact non-expired package and the executed document exists. “Sent” is not “signed.”
5. Keep forms and Zaps disabled/draft during controlled end-to-end testing. Never test by contacting a real applicant or signer.

## Urgent live discrepancies

- The enabled tenant-options Jotform still posts to the legacy Linode endpoint while the modern Zapier return is Draft/off and incomplete.
- The move-in confirmation Jotform's live webhook does not match the hook fingerprint recorded in the implementation notes.
- Live Zap titles/IDs conflict with the local registry, including two Zaps titled Move In Date Confirmation Link Builder.
- RentSign `Lease Agreement` and `Lease Documents` are different templates with different forms and signer capacities; processes must name the exact template ID.

Notion contains useful historical notes, but the legacy `deal_id`/Linode pattern must not be treated as the current secure standard.
