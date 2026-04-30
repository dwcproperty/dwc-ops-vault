# DWC Build Inventory

Last updated: 2026-04-29

Master index of every automation, process, form, service, and integration we've built. Update this whenever something new ships.

---

## 🖥 Linode Server

**Host:** 96.126.118.135
**Domain alias:** mcp.collincountyrent.com
**SSH:** `ssh root@mcp.collincountyrent.com` or `ssh root@96.126.118.135`

### PM2 Processes (run `pm2 list` to verify)

| Process | Port | Public URL | Status |
|---|---|---|---|
| webhook | 3000 | — | LIVE |
| leadsimple-mcp | 3001 | mcp.collincountyrent.com/sse | LIVE |
| rentvine-mcp | 3002 | rentvine.collincountyrent.com/sse | LIVE |
| rentengine-mcp | 3003 | rentengine.collincountyrent.com/sse | LIVE |
| obsidian-mcp | 3004 | obsidian.collincountyrent.com/sse | LIVE |

### Files on Server

- `/root/CLAUDE.md` — persistent context for Claude Code (Opus 4.7, 1M context, all MCPs connected)
- `/root/reference-automation.js` — vendor reference automation router with 3 endpoints (written, not yet wired to Jotform)
- `/root/bill-monitor/` — Node service spec drafted (polls tenant + vendor payments, advances LeadSimple stages). NOT YET BUILT.
- `/root/obsidian-vaults/` — synced clones of Darrell-Personal + DWC-Ops, refreshed via cron every 2 minutes

---

## ⚡ Zapier Pro

### Phase 2a — Utility Bill Pipeline ✅ PRODUCTION

**Flow:** Gmail trigger → Claude Sonnet PDF parser → Google Sheets lookup (portfolio owner) → Linode webhook (resolve owner name) → LeadSimple process creation → Paths split by responsibility (Concierge vs. Owner Pays/Vacant).

Tested working on Atmos bill. Fully production.

### Phase 2 — Tenant Notification System ❌ NOT BUILT

5 planned trigger points:
1. Charge posted
2. 2 days before due
3. Due date
4. Late fee applied
5. Payment received

Delivery channel TBD (email vs. text).

---

## 🔁 LeadSimple Processes

| Process | Stages | Status | Notes |
|---|---|---|---|
| DWC Move-In Process | 8 | LIVE | Hidden `deal_id` field on Jotform via merge tag |
| Acct Outstanding Invoices | 5 | LIVE | Parent process |
| Acct Utility Bills | 6 | LIVE | Child of Outstanding Invoices. Includes "Late Fee Applied" stage. Late fee triggers at Tenant Due Date + 1 day ($35) |
| Vendor Onboarding | 12 | ❌ SPEC ONLY | API write returns 403 (read-only key). Must be built manually in UI. |

**API limitation:** All write actions route through Zapier or UI. API key is read-only.

---

## 📝 Jotform Forms

| Form | ID | Status |
|---|---|---|
| Move-In Form | 251564975684069 | LIVE — webhook → Linode → advances LeadSimple deal |
| Vendor Intake | 261094049807057 | BUILT — webhook NOT YET WIRED |
| Vendor Reference Check | 261094412562049 | BUILT |

---

## 🏢 Rentvine Configuration

### Unit-Level Custom Fields

For each utility (Electric, Gas, Water/Trash):
- Account Number
- Vendor
- Responsibility (dropdown): DWC Concierge | Tenant Direct | Owner Pays | Vacant — Owner | N/A

### Accounting

- **Account 4560 "Tenant Utility Reimbursement"** (accountID 38) — charge account for tenant utility reimbursements
- Trust-to-operating sweep flow configured
- Tenant due date = bill due date − 3 days (always)

---

## 💰 QuickBooks Online

- **DWC Tenant Utility Advances** — clearing account on QBO side, paired with Rentvine 4560

---

## 📚 Notion — DWC HQ Workspace

| Asset | ID |
|---|---|
| DWC HQ root page | 34823e9a-a5a3-8139-880d-c59c810edfe7 |
| SOPs Database | 81dbbf37-bde9-47b7-97a4-7cca321d2ca3 |
| Vendors Database | 10ee4a6a-bf8c-4f9e-ae94-0fc3e191c1a3 |
| Templates Library | (in DWC HQ tree) |

8 department pages, 5 seeded SOPs.

---

## 📓 Obsidian

| Vault | Path | Sync |
|---|---|---|
| Darrell-Personal | C:\Users\Darrell\Documents\Obsidian\Darrell-Personal | Obsidian Git → private GitHub → Linode (every 2 min via cron) |
| DWC-Ops | C:\Users\Darrell\Documents\Obsidian\DWC-Ops | Same |

Push interval from laptop: every 5 minutes (Obsidian Git plugin).

---

## 🔌 MCP Connectors (Claude Desktop)

| Connector | URL | Auth |
|---|---|---|
| LeadSimple | mcp.collincountyrent.com/sse | API key baked into Linode env |
| Rentvine | rentvine.collincountyrent.com/sse | API key baked into Linode env |
| RentEngine | rentengine.collincountyrent.com/sse | API key baked into Linode env |
| Obsidian | obsidian.collincountyrent.com/sse | API key baked into Linode env |

No OAuth flows — all keys live on the Linode server.

---

## 🚧 Outstanding / In-Flight

- [ ] Vendor Onboarding LeadSimple process — build manually in UI (API returns 403)
- [ ] Vendor intake webhook — wire Jotform 261094049807057 to Linode
- [ ] Stage 5 UUID for vendor process — retrieve once process is built
- [ ] Gmail App Password — confirm and store
- [ ] Phase 2 tenant notification system — build all 5 triggers
- [ ] `bill-monitor` Node service — build from spec, deploy to PM2

---

## 🎨 DWC Brand Reference

- DWC Blue: `#004B91`
- DWC Orange: `#F47C27`

---

## 🔗 Related Notes

- `Reference/Quick Commands.md` — shell + PM2 cheatsheet
- `Automation/Utility Bill Pipeline/` — Phase 2a build notes
- `Automation/Gmail Resolve/` — Gmail parsing logic
- `Automation/Invoice Watcher/` — invoice monitoring notes
