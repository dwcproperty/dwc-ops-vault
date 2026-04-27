# LeadSimple Variant Builder — Phase 2 shipped

Session date: 2026-04-25 (continued from earlier sessions today)

## TL;DR

The full **describe → built** loop now works. You describe a process in conversation → Claude drafts a YAML spec → you review → Claude runs `build_process.py` → process exists in LeadSimple as a draft → you review/publish in the UI.

**First production process built today: Acct Illegal Pet.**

Live at: https://app.leadsimple.com/v2/process-types/WMd2_kUaNDSZp6_GpJOqPz22K_vHyBy4B50nQ3TFwl2O2ayJay9D/settings/general

## What got built today

### 1. Auth bootstrap — `lsclient.py`
Playwright logs in once, sniffs the SPA's `Authorization: Bearer <token>` + `browser_id` cookie (both required — auth fails with either alone), caches to `/root/.leadsimple-token.json`. After that everything goes through `requests` directly to `api.leadsimple.com/v2/graphql`. Auto-relogs on 401.

### 2. Read path — `read_master.py`
Five GraphQL queries cover the high-value tabs: General, Stages, Email Templates, Text Templates, Custom Fields, plus a per-stage `WorkflowContainerAccountQuery` drill-in for the workflow steps. Outputs YAML matching `specs/SCHEMA.md`. Round-trips Acct Utility Bills end-to-end.

### 3. Variant builder (clone-master path) — `build_variant.py`
For metadata-only variants. `createProcessType` → `cloneProcessType(clone: [STAGES, EMAIL_TEMPLATES, TEXT_MESSAGE_TEMPLATES, CUSTOM_FIELDS])` → poll until clone content lands (it's async, ~3s) → `updateProcessType` for metadata deltas. Tested cleanly.

### 4. From-scratch builder — `build_process.py` ⭐
**This is the big one.** Takes a complete YAML spec (no master required) and constructs the entire process via GraphQL mutations:

```
createProcessType
  → updateProcessType (communication flags)
  → updateProcessTypeDueDateFields
  → createContactRole(s)
  → createEmailTemplate(s)
  → createTextMessageTemplate(s)
  → createCustomField(s)
  → createStage(s)
  → destroyStage(s) for auto-created Backlog/Working/Completed defaults
  → createStep(s) per stage
  → updateStep(s) for emailTemplateId / textMessageTemplateId / instructions / autoSend
    (createStep doesn't accept these — must update afterward)
  → updateProcessTypeDefaultStage
```

Cleanup-on-failure rolls back via `destroyProcessType` + `destroyCustomField` (custom fields are account-scoped — they don't cascade with the process type, so we track and clean separately).

### 5. Helpers
- `timing.py` — parses spec timing strings ("immediately", "5 minutes later", "2 days before custom_tenant_due_date") → `(delayMinutes, delayRelativeTo)` for `createStep`
- `destroy_process_type.py <id>` — quick cleanup
- `pull_inputs.py` — re-introspect input shapes if the schema changes
- `extract_queries.py` — regenerate `queries.py` from `scrape_artifacts/`

## Architecture decisions locked

- **GraphQL-direct, no Playwright at runtime.** Playwright is only used during initial login to capture the bearer token; the full builder runs via `requests`. Confirmed by introspection: 291 mutations available, including every create/update/destroy we need.
- **YAML is the human artifact.** Specs live at `specs/<process>.yaml` and match the format documented in `specs/SCHEMA.md`. You read/edit YAML; I drive the API.
- **Draft mode is fine.** Process types are created in DRAFT — you publish manually in the UI after review. The `changeProcessTypeMode` mutation exists if we want to automate that later.

## Gotchas baked into memory (so they don't bite again)

1. Process names = letters/digits/spaces only (no punctuation)
2. URL slug ≠ UUID. The canonical Relay node ID for Acct Utility Bills is `WMd2_kUaNDSZp6_GpZOvMT-2Iv2Rkh_IwW0lJ8Y0DcNy28Tkfq1h` — note CAPITAL `I` in `_IwW0`, not lowercase `l`. Visually identical; cost a debug session
3. `cloneProcessType` is async — poll until counts match
4. `cloneProcessType`'s `clone` enum doesn't include general-tab settings — those need explicit `updateProcessType` after
5. `createProcessType` auto-creates 3 default stages (Backlog/Working/Completed) — destroy them AFTER your spec stages exist (server refuses "delete the last stage")
6. `createStep` does NOT take template IDs — two-step pattern: createStep, then updateStep with `emailTemplateId` / `textMessageTemplateId` / `instructions` / `autoSend`
7. Custom fields are account-scoped — duplicate label across builds collides; cleanup must destroy them separately
8. Mutation payloads use `newEdge { node { id } }` for stages/steps/templates, but `contactRoleEdge` / `assigneeRoleEdge` / `customFieldEdge` for those three
9. `UpdateProcessTypeDefaultStage` uses `processTypeId` + `defaultStageId`, not `id` + `stageId`
10. `StepKindEnum` has `CHANGE_STAGE` (not `STAGE_CHANGE`); `StageStatusEnum` has `BACKLOG`/`WORKING`/`COMPLETED`/`CANCELED`

## How to use this going forward

In any Claude Code session on the Linode:

> "Let's build Acct Smoking Violation. When a tenant smokes inside, I want to send them a notice and charge a $100 fee, then..."

Claude will:
1. Read memory (auto-loaded) → know about this project
2. Look at `specs/acct_illegal_pet.yaml` as a template
3. Draft `specs/<new_process>.yaml`
4. Show it to you, ask any judgment-call questions (fee amounts, follow-up timing, etc.)
5. Run `build_process.py specs/<new_process>.yaml` once approved
6. Give you the URL to review in LeadSimple

Iteration loop if you want to tweak after build:
```
edit specs/<process>.yaml
./destroy_process_type.py <id>
./build_process.py specs/<process>.yaml
```

(Idempotent rebuild is a future polish.)

To resume this exact conversation: `claude --continue`. For a fresh session: just `claude` — memory loads either way.

## Open question for next session — automation/webhooks on stage change

Darrell asked: can we have stage changes automatically trigger external actions (e.g. post a charge to Rentvine) instead of leaving them as TODOs?

What we know:
- `upsertAutomationRule` mutation exists (Autopilot tab) — input shape not yet introspected
- Step kind `PROCESS` can start another process type, so cross-process automation already works inside LeadSimple
- We have direct Rentvine API access via the MCP tools (`create_bill`, `create_work_order`, etc.)

What we don't know yet:
- Whether LeadSimple has a native outbound webhook step / autopilot action
- Whether there's a built-in Rentvine integration for "post a charge"
- Whether Zapier/Make/n8n is the cleanest middleman

**Next session plan:** ~30 min to introspect autopilot/webhook/integration surface in LeadSimple's GraphQL schema, plus check if a Rentvine action is exposed natively. After that we'll know whether the answer is "native LeadSimple feature" (1 day to wire), "Zapier middleman" (couple hours), or "custom bridge service that forwards LeadSimple webhook → Rentvine MCP" (1-2 days).

## Files / paths reference

| Thing | Where |
|---|---|
| Code root | `/root/leadsimple-builder/` |
| Credentials | `/root/.leadsimple.env` (mode 600) |
| Token cache | `/root/.leadsimple-token.json` (auto-managed) |
| Schema dumps | `/root/leadsimple-builder/schema/` (mutations.json, builder_inputs.json) |
| Specs | `/root/leadsimple-builder/specs/<process>.yaml` |
| Spec format | `/root/leadsimple-builder/specs/SCHEMA.md` |
| First production spec | `/root/leadsimple-builder/specs/acct_illegal_pet.yaml` |
| Latest scrape (reference) | `/root/leadsimple-builder/scrape_artifacts/1777107486_WMd2_kUaNDSZ/` |

## Acct Illegal Pet snapshot

Process ID: `WMd2_kUaNDSZp6_GpJOqPz22K_vHyBy4B50nQ3TFwl2O2ayJay9D`

- 4 stages: Notice Sent → Inspection Scheduled → Re-notice Required → Resolved
- 8 workflow steps total (paired email+text on each notice; 10-day auto-advance from Notice Sent → Inspection Scheduled; manual advance from Inspection Scheduled to either Re-notice or Resolved)
- 2 email templates with full HTML+text bodies
- 2 text-message templates
- 6 process custom fields (Pet Type, Pet Description, Discovery Date, Inspection Date, Fee Amount, Pet Status After Inspection)
- Tenant contact role linked to Rentvine Tenants pipeline
- Default starting stage: Notice Sent
- Status: DRAFT — review and publish in UI when ready
