---
type: reference
status: needs-review
authority: Darrell's current machine and server configuration
last_verified: null
superseded_by: null
---

# Quick Commands

## Launch Claude Code on the Linode box

From Windows PowerShell:

```powershell
ssh dwc-mcp-server
cd /root/dwc-coding
claude
```

### Current SSH config

Configured on Darrell's Windows machine at `C:\Users\Darrell\.ssh\config`:

```sshconfig
Host dwc-mcp-server
    HostName 96.126.118.135
    User root
    IdentityFile ~/.ssh/dwc_mcp_server
    IdentitiesOnly yes
```

SSH password login is disabled. Root login is allowed by SSH key only.

### Claude Code one-liner

From Windows PowerShell:

```powershell
ssh -t dwc-mcp-server "cd /root/dwc-coding && claude"
```

---

## Linode box info

- Server label / hostname: `dwc-mcp-server`
- IP: `96.126.118.135`
- Working directory for Claude sessions: `/root/dwc-coding`
- CLAUDE.md context file: `/root/dwc-coding/CLAUDE.md` (auto-loads on session start)
- Backups: enabled in Linode, scheduled Sunday `06:00 - 08:00 GMT`
- SSH security as of 2026-07-09:
  - `PasswordAuthentication no`
  - `KbdInteractiveAuthentication no`
  - `PubkeyAuthentication yes`
  - `PermitRootLogin without-password`
- Public firewall: raw public port `3000` is closed. Public webhook traffic should use HTTPS/Nginx.
- Nginx proxy: `https://webhook.collincountyrent.com` -> `127.0.0.1:3000`
- PM2 status after reboot: apps online except `seo-pmw-flipper`, which was already stopped.

## Health check the webhook

```bash
curl https://webhook.collincountyrent.com/webhook/health
```

## Restart webhook after edits

```bash
pm2 restart webhook && pm2 logs webhook --lines 5
```


---

## 📋 Master Build Inventory

For a full list of every automation, process, form, service, and integration we've built, see:

**[[Build Inventory]]**

Update that note whenever something new ships.
