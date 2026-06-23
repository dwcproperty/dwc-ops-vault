# 2026-06-23 — Gmail Connector: Permanent Reauth Fix + Monitor

Follow-up to [[2026-06-22 Domain-Wide Gmail MCP Build]]. The five-mailbox
`gmail-domain` connector had been dying **~weekly** with
`RefreshError: Reauthentication is needed`. Root-caused it, fixed it permanently,
and added a watchdog. On the box at `/root/dwc-mcp-servers/`.

## Root cause (confirmed in the Google Admin console)
The box's **Application Default Credentials ran as the human user `office@`**, and
the Workspace policy **Security → Access and data control → Google Cloud session
control** was set to **"Require reauthentication" every 16 hours** for Cloud
Platform scope. Google's own note says that policy *also applies to non-Google apps
using Cloud Platform scope* — i.e. our gcloud ADC. So the ADC token was being
force-expired on a timer. (It was **NOT** a "Testing" OAuth-consent-app expiry — ADC
uses Google's published gcloud client. The box is not a GCE VM, so it must auth as
some identity.)

## The fix — dedicated non-expiring automation identity
- New Workspace user **`automation@dwcproperty.com`** in a new OU **`Automation`**.
- That OU's **Google Cloud session control = "Never require reauthentication"**
  (overridden from the root OU). This is the actual cure.
- License: **Cloud Identity Free** (no Gmail seat → $0). Turned OFF Business Standard
  auto-assign for the Automation OU and removed the paid license from the user.
- IAM (project `dwc-gmail-mcp`):
  - `automation@` → `roles/iam.serviceAccountTokenCreator` **on the SA**
    `gmail-domain-readonly@dwc-gmail-mcp.iam.gserviceaccount.com`.
  - `automation@` → `roles/serviceusage.serviceUsageConsumer` on the project
    (so it can set the ADC quota project).
  - **Removed** `office@`'s old Token Creator role on the SA (least privilege).
- Box ADC re-authed as `automation@` (`gcloud auth application-default login`,
  quota project `dwc-gmail-mcp`). Verified impersonation + all 5 mailboxes.

> Supersedes the [[2026-06-22 Domain-Wide Gmail MCP Build]] detail that said ADC =
> `office@`. ADC is now `automation@`. `office@` no longer has any role on the SA.

## The watchdog — `gmail-domain-monitor`
- New PM2 service `gmail-domain-monitor` (`/root/dwc-mcp-servers/gmail-domain-monitor/`),
  **node-cron hourly** (`0 * * * *`, America/Chicago), `pm2 save`d.
- Runs `gmail-domain-mcp/healthcheck.py`, which impersonates **each** of the 5
  mailboxes and calls `getProfile` — exercising the full
  ADC → signJwt → DWD → Gmail chain (catches reauth death AND deeper breakage).
- On failure emails **office@** via the **separate office@ Gmail OAuth token**
  (`gmail-resolve-monitor/auth`, `gmail.modify`) — deliberately a *different*
  credential than the one it watches, so the alarm still fires when the
  gmail-domain ADC itself is broken.
- State machine (`state.json`): alert on `ok→down`, recovery on `down→ok`, reminder
  at most once/24h while down. Subjects: `DWC | Gmail Connector DOWN - action
  needed` / `... still DOWN (since …)` / `... recovered`.
- **Tested for real**: induced a live failure (moved ADC aside) → monitor emailed a
  real DOWN alert with the live error → restored → emailed recovery. Verified
  healthy after.

## Quick-ref
- Manual health check: `gmail-domain-mcp/.venv/bin/python gmail-domain-mcp/healthcheck.py`
  (exit 0 = healthy).
- Runbook (recovery steps): `gmail-domain-mcp/REAUTH-RUNBOOK.md`.
- Monitor logs: `gmail-domain-monitor/logs/`; state: `…/state.json`.
- If a real alert ever lands, the email body names the live error + links the runbook.
- Source now committed to GitHub (`dwcproperty/dwc-mcp-servers`, `main`).
