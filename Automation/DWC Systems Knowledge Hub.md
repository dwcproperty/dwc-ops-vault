---
type: hub
status: active
authority: DWC Systems Knowledge library and current DWC decisions
last_verified: 2026-08-09
superseded_by: null
---

# DWC Systems Knowledge Hub

This page is the Obsidian entry point for DWC platform and process-architecture knowledge.

## Vault navigation

- [[Start Here]] — DWC-Ops home and operating boundary.
- [[Inbox/README]] — capture and review workflow.
- [[Automation/Chat Saves/README]] — historical session archive.

## Canonical local library

The maintained deep-dive files live in:

`C:\Users\Darrell\Documents\DWC Property Group\Systems Knowledge`

Start with:

- [Systems Knowledge README](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/README.md)
- [DWC Process Architecture Packet](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/DWC%20Process%20Architecture%20Packet.md)
- [Knowledge Conflict Register](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/Knowledge%20Conflict%20Register.md)
- [System Inventory](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/System%20Inventory.md)

## Platform deep dives

- LeadSimple: `Systems Knowledge\LeadSimple`
  - [[Systems Knowledge/LeadSimple/LeadSimple Research Index]]
- RentVine: `Systems Knowledge\Rentvine`
- RentEngine:
  - [RentEngine Deep Dive](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/RentEngine/RentEngine%20Deep%20Dive.md)
  - [RentEngine Workflow Build Checklist](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/RentEngine/Workflow%20Build%20Checklist.md)
  - Live account and official-documentation audit last completed 2026-08-09; review the documented open gaps before every build.
- Gmail:
  - [Gmail Deep Dive](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/Gmail/Gmail%20Deep%20Dive.md)
  - [Gmail Workflow Build Checklist](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/Gmail/Workflow%20Build%20Checklist.md)
- Google Drive:
  - [Google Drive Deep Dive](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/Google%20Drive/Google%20Drive%20Deep%20Dive.md)
  - [Google Drive Workflow Build Checklist](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/Google%20Drive/Workflow%20Build%20Checklist.md)
- Linode/custom automation:
  - [Linode and Custom Automation Deep Dive](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/Linode/Linode%20and%20Custom%20Automation%20Deep%20Dive.md)
  - [Linode Workflow Build Checklist](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/Linode/Workflow%20Build%20Checklist.md)

Gmail, Google Drive, and Linode were live-audited 2026-08-09. Check the conflict register for the active bill-monitor failure, Gmail authentication/closure risks, Drive anonymous credential sharing, and server credential/network gaps before building on this stack.
- [Notion governance and live audit](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/Notion/README.md)
- [Obsidian governance and vault audit](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/Obsidian/README.md)
- [monday.com governance and safe testing](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/monday.com/README.md)
- [QuickBooks and RentVine accounting boundary](file:///C:/Users/Darrell/Documents/DWC%20Property%20Group/Systems%20Knowledge/QuickBooks/README.md)

## Mandatory rule for AI helpers

Do not begin a new process by building stages, tasks, forms, messages, or Zaps. First complete the architecture packet, inspect every participating live system, verify uncertain platform behavior against current official documentation, and reconcile older notes against the conflict register.

LeadSimple is a field-driven state machine with related records and contact roles. External systems should write validated facts; LeadSimple conditions should normally control task visibility, communications, reminders, and stage routing.

## Authority order

1. Darrell's current explicit decision.
2. Current live DWC configuration.
3. Current official vendor documentation.
4. Current canonical DWC standards/process architecture.
5. Older Notion, Obsidian, chat-save, and implementation notes.

Legacy notes remain useful history, but they must not silently override current verified behavior.

Last reviewed: 2026-08-09
