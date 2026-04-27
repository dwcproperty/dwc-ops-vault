# 2026-04-25 — Chat Save: LeadSimple Process Automation Research

**Status:** Paused mid-conversation. Pick up here next session.

## What we were exploring

Whether you can programmatically create new LeadSimple **process types** (templates) instead of clicking through the UI manually.

## Key findings

### 1. No public API for creating process types
- Tested directly: `POST https://api.leadsimple.com/rest/process_types` returns **403 "Unauthorized to access this endpoint"**
- `GET /process_types` works fine (200) — read access only
- LeadSimple's API docs claim the Zapier API key is "only valid for Zapier" but the MCP works for all our reads, so docs may be outdated. Either way, write access to process types is admin-UI-only.

### 2. Process Type creation in UI is a 5-step flow
1. Basic info (name, naming convention, category)
2. Team & contact roles (multi-tab)
3. Default email
4. Communication settings (Relevant vs Broad)
5. **Stages & Workflows** — the complex one: backlog/active/completed/cancelled buckets, conditional logic, wait steps, contact role assignment

### 3. Big unlock — LeadSimple has a template clone feature
- Library (left sidebar) → pre-built templates → "Copy Into My Account" → "Use This Process"
- ~4 clicks to clone, then edit only what's different
- This is the path to take for variant-heavy work without engineering effort

## Three-tier recommendation we landed on

| Tier | Approach | Effort | Best for |
|---|---|---|---|
| 1 | Build master templates once in UI, clone for variants | 2-3 hrs per master, ~5 min per variant | Most use cases |
| 2 | Playwright automates clone + rename + custom-field tweaks only | 4-6 hrs build | Many variants of similar processes |
| 3 | Full Playwright build of stages & workflows from YAML spec | 2-3 days build + ongoing UI-change maintenance | Many genuinely different processes |

**Recommended:** Start with Tier 1. Revisit Playwright only if templating doesn't cut it.

## Open checks deferred (for next session if pursuing Playwright)

- **2FA status on Darrell's LeadSimple account** — affects whether direct login automation works or we need session cookie reuse
- **Hands-on UI inspection** — walk through "New Process Type" once to confirm what selectors Playwright would target

## Tools & access confirmed during this session

- LeadSimple MCP available with these tools (read-heavy + some writes — but no `create_process_type`)
- Rentvine, RentEngine, Obsidian, Gmail, Google Drive, Notion, Jotform, QuickBooks all wired up
- LeadSimple API key at `/root/dwc-mcp-servers/leadsimple/.env`
- Confirmed working contact: Renate Racher (id `47060805-65f8-4ea3-95f3-1bccaeed6a8e`) — owner, real record, do NOT use as test target

## Other things created this session

- `/root/dwc-coding/CLAUDE.md` — auto-loads on future Claude Code sessions for DWC context
- `/root/dwc-coding/hello.js` — sanity-check script (verified Linode → Claude Code working)
- `Reference/Quick Commands.md` in this vault — `dwc` alias for one-step ssh+claude launch

## To pick this back up

Open Claude Code on the Linode (`dwc` if you set up the alias) and say: *"Pick up the LeadSimple process automation conversation from 2026-04-25 — what's the next step?"* I'll read this note and continue from the open checks above.
