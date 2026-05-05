# Phase 2 — Handoff (paused 2026-05-01)

Pickup point for Phase 2 of the [[Phase 2 — Rentvine Bill Bridge|Rentvine Bill Bridge]] build. Read this first when you come back.

## Status: built, in dry-run, waiting on three UI tasks before we can flip live.

The code is done and committed. The PM2 service `invoice-watcher` (id 14) is running with the bridge enabled in **dry-run mode**. Phase 1 (email → LS deal) is healthy and Anthropic-keyed. Phase 2 (LS deal → Rentvine bill) is wired but currently filtering 0 deals in scope, because the `invoice_source` field doesn't exist in LS yet — that's the first of the three things below.

## What I need you to do (three small things)

### 1) Add 4 custom fields to the LS process type

LS REST returns 403 on custom-field create — it has to be done in the LS UI. Open process type **Acct Outstanding Invoices** (`982dd2b8-6391-46f5-820d-764aad18ebc9`) and add these four:

| Key | Type | Label |
|---|---|---|
| `invoice_source` | Text | Invoice Source |
| `rentvine_vendor_id` | Text | Rentvine Vendor ID |
| `rentvine_bill_id` | Text | Rentvine Bill ID |
| `bridge_failure_reason` | Text | Bridge Failure Reason |

Then on the box, auto-backfill the label_ids:

```sh
cd /root/dwc-mcp-servers/invoice-watcher
node scripts/ls-probe.js --apply
```

### 2) Add 2 stages to the same pipeline

Two of the four review stages already exist (Vendor Onboarding Needed, Property Verification Needed — confirmed via API). Just these two are missing:

| Stage name | Status |
|---|---|
| `Bill Created in Rentvine` | working (success terminal) |
| `Bill Already in Rentvine` | working (duplicate terminal) |

After creating them, paste the IDs into `/root/dwc-mcp-servers/invoice-watcher/.env`:

```
BILL_CREATED_STAGE_ID=...
BILL_ALREADY_IN_RENTVINE_STAGE_ID=...
```

Then `pm2 restart invoice-watcher`.

### 3) Update the Utility Bills Zap

In Zapier, find the zap that creates a deal in the **Acct Outstanding Invoices** pipeline from the `Utility Bills` Gmail label. Add a custom-field write step setting `invoice_source = utility-bills` on the deal-create.

Without this, the bridge would pick up Utility Bills deals and post them as non-utility vendor AP bills — exactly what the filter is designed to prevent.

## When you've done 1+2+3, tell Claude

Resume prompt to use:

> Resume the Invoice Watcher Phase 2 build. I've done the three handoff items: created the 4 LS custom fields, created the Bill Created / Bill Already in Rentvine stages and pasted their IDs into .env, and updated the Utility Bills Zap to set invoice_source = "utility-bills". Read [[Phase 2 — Handoff (paused 2026-05-01)]] in the dwc-ops vault for context, then pick up at the dry-run verification step.

Claude will then:

1. Run `ls-probe --apply` to backfill label_ids.
2. Verify the bridge picks up real deals in dry-run (logged, no Rentvine writes).
3. Pick a known-good repair-vendor invoice + property and run that single deal live (`BRIDGE_DRY_RUN=false`).
4. Force every failure mode: `vendor_not_found`, `vendor_ambiguous`, `property_not_found`, `property_ambiguous`, `gl_map_missing`, `duplicate_existing`, Utility-Bills-ignored.
5. Flip the spec from "draft, ready for build" to "live in production".

## What's already done (so you don't worry it got lost)

- **Rentvine MCP extended** (`/root/dwc-mcp-servers/Rentvine/index.js`): added `find_property_by_address`, `create_vendor_bill`, `attach_file_to_bill`, `list_bills_by_vendor`. Fixed the broken `list_vendors` / `get_vendor` paths (they were 404'ing on `/contacts/vendors`; real path is `/vendors`). PM2 restarted, smoke tested. Committed as `318622a`.
- **Chart of accounts pulled and approved** — 16 non-utility expense GLs identified. You signed off on the mapping.
- **`vendor-gl-map.json` seeded** — 41 active DWC vendors mapped across 13 GL buckets. Personal-name vendors (Jeremy Hollar, Iman Pasha, etc.) intentionally left out so they bounce to Vendor Onboarding Needed on first invoice — you said all 11 are real contractors but we'd let them bounce for one-time human classification.
- **Architecture deviation you approved** — bridge reads `vendor.defaultBillChargeAccountID` from Rentvine first (9 vendors already have this set), only falls back to `vendor-gl-map.json` when null.
- **Phase 1 changed**: now stamps `invoice_source = "invoice-inbound"` on every new deal it creates. Removed the inline `createOwnerCharge` block from `pipeline.js` — Phase 2 owns Rentvine writes now.
- **Bridge built**: `src/bridge-rentvine.js` (per-deal processor), `src/bridge-poller.js` (60s loop, runs alongside Gmail poller in same PM2 process). Default dry-run.
- **PDF attach mechanism confirmed**: single multipart `POST /api/manager/files` with `objectTypeID=4`, `objectID=<billID>` does it. No two-step needed.
- **Slack alert path wired** for `ls_update_failed` (the only critical failure mode — bill exists in Rentvine but LS write failed). Falls back to email-to-self if `SLACK_WEBHOOK_URL` is empty.
- Committed as the second invoice-watcher Phase 2 commit. `git -C /root/dwc-mcp-servers log --oneline -3` to see them.

## Operational anchors

- Service: PM2 `invoice-watcher` (id 14), port 3005.
- Logs: `/var/log/dwc/invoice-watcher.log` and `pm2 logs invoice-watcher`.
- Health: `curl -s http://localhost:3005/health` — the `bridge` block shows dryRun status, last tick stats, and current scope counts.
- Bridge config in `.env`: `BRIDGE_DRY_RUN=true` (default), `BRIDGE_POLL_INTERVAL_SEC=60`, `BRIDGE_ENABLED=true`.
- To go live for one test deal: change `BRIDGE_DRY_RUN=false` in `.env`, `pm2 restart invoice-watcher`. To go back to dry-run: flip back to `true`, restart.

## Files touched this session

```
/root/dwc-mcp-servers/Rentvine/index.js                            (extended)
/root/dwc-mcp-servers/Rentvine/_smoke-phase2.mjs                   (new)
/root/dwc-mcp-servers/invoice-watcher/src/pipeline.js              (Phase 1 update)
/root/dwc-mcp-servers/invoice-watcher/src/rentvine.js              (extended)
/root/dwc-mcp-servers/invoice-watcher/src/bridge-rentvine.js       (new)
/root/dwc-mcp-servers/invoice-watcher/src/bridge-poller.js         (new)
/root/dwc-mcp-servers/invoice-watcher/src/index.js                 (boot bridge)
/root/dwc-mcp-servers/invoice-watcher/src/config.js                (new env vars)
/root/dwc-mcp-servers/invoice-watcher/scripts/ls-probe.js          (--apply flag)
/root/dwc-mcp-servers/invoice-watcher/leadsimple-field-map.json    (4 new pending fields)
/root/dwc-mcp-servers/invoice-watcher/vendor-gl-map.json           (new — 41 vendors)
/root/dwc-mcp-servers/invoice-watcher/.env                         (new vars added)
/root/dwc-mcp-servers/invoice-watcher/.env.example                 (new vars added)
```

## Related

- [[Phase 2 — Rentvine Bill Bridge]] — the spec
- [[Invoice Watcher]] — Phase 1 service note
