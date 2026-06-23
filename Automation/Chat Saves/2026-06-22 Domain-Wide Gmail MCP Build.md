> [!update] 2026-06-23 — superseded in part
> The ADC identity changed from `office@` to a dedicated **`automation@dwcproperty.com`**
> (non-expiring, in a session-control-exempt OU) to stop the ~weekly reauth failures,
> and a watchdog was added. `office@` no longer holds any role on the SA. See
> [[2026-06-23 Gmail Connector — Permanent Reauth Fix + Monitor]]. The sections below
> reflect the original 2026-06-22 build.

# 2026-06-22 — Domain-Wide Gmail MCP Server (all 5 mailboxes, keyless DWD)

Built a single MCP server that searches/reads **all five `dwcproperty.com`
Workspace mailboxes at once**, getting past the native Claude Gmail connector's
one-account-at-a-time limit. On the box at `/root/dwc-mcp-servers/gmail-domain-mcp/`.

**Why:** to find whether a Gmail bounce to tenant Madison Bowers
(`m.bowers2019@gmail.com`) existed in `dcalhoun@` or `info@`. (It did — see bottom.
That hunt later led to the LeadSimple SMS fix — see `2026-06-22 LeadSimple SMS Send Fix`.)

Mailboxes: `info@`, `dcalhoun@`, `office@`, `mary@`, `benefits@` — all `@dwcproperty.com`.

## Design
- Minimal purpose-built **Python** server (FastMCP) — chosen over the big
  taylorwilsdon/google_workspace_mcp because the credential is a domain master key,
  so a small auditable surface + least privilege matters.
- **Read-only**: only scope `https://www.googleapis.com/auth/gmail.readonly`.
- **Allow-list** of the 5 mailboxes enforced on every call (`gmail_client.build_service`).
- Tools: `list_mailboxes`, `search_messages`, `search_all_mailboxes` (one query
  across all five), `get_message`, `get_recent`.

## KEYLESS domain-wide delegation (the curveball)
The Google Cloud org enforces `iam.disableServiceAccountKeyCreation`, so **no
downloadable service-account key** is possible — and we deliberately did NOT weaken
that policy. Instead:
- Service account `gmail-domain-readonly@dwc-gmail-mcp.iam.gserviceaccount.com`
  (project `dwc-gmail-mcp`, Client ID `108565876578734460439`), DWD-authorized in
  the Admin console for **exactly** `gmail.readonly`.
- The box impersonates the SA via the **IAM `signJwt` API** (Google signs the DWD
  assertion server-side with its managed key — no key file on disk), then exchanges
  it (jwt-bearer) for a per-mailbox access token.
- That `signJwt` call authenticates with **Application Default Credentials** (a
  non-admin user) holding `roles/iam.serviceAccountTokenCreator` on the SA + Service
  Usage Consumer on the project. **[As of 2026-06-23 the ADC user is `automation@`,
  not `office@` — see the update banner at top.]**
- gcloud installed at `/opt/google-cloud-sdk/`; ADC at
  `/root/.config/gcloud/application_default_credentials.json` (0600).

**5/5 delegation test passed** — recent messages listed for every mailbox.

## Clients (all three covered)
1. **Claude Code on this box** — registered `claude mcp add gmail-domain --scope
   user` (local stdio). Working immediately.
2. **claude.ai web + the Windows "Cowork" desktop app** — the desktop app turned
   out to be an **MSIX/Cowork build = the claude.ai web client**: it ignores local
   `mcpServers` entirely (config at `...\Packages\Claude_pzs8sxrjxfjjc\...`, only a
   `claude.ai-web.log`), and only accepts **remote OAuth** connectors. So both use
   one OAuth endpoint.

### The OAuth endpoint
- URL (add as custom connector): **`https://gmail.collincountyrent.com/sse`**
  (subdomain → 96.126.118.135, Let's Encrypt TLS, auto-renew).
- Real OAuth 2.0: DCR + authorization_code + **PKCE S256**, **password-gated** login.
  All flow + security checks pass (wrong password / bad PKCE / no-token rejected).
- Architecture:
  - pm2 `gmail-oauth-gateway` = Flask `oauth_gateway.py` (gunicorn, `127.0.0.1:3021`)
    — discovery, `/register`, `/authorize` (login form), `/token`, `/validate`.
  - pm2 `gmail-domain-mcp` = `mcp-proxy --host 127.0.0.1 --port 3020 --apiKey <key>
    -- <venv python> server.py`.
  - nginx (`/etc/nginx/sites-available/gmail.collincountyrent.com`, mode 640) routes
    OAuth paths to 3021 and gates `/sse`+`/mcp` via `auth_request` → gateway
    `/validate`, then injects the internal X-API-Key. The bearer gate is never absent.
- **Add it:** Settings → Connectors → Add custom connector → that URL → enter the
  connector password → Authorize.

## Secrets (all 0600, off-repo in `/root/secrets/`)
- `gmail-mcp-apikey.txt` — internal X-API-Key (nginx → mcp-proxy)
- `gmail-oauth-pw.hash` — connector login password hash
- `gmail-oauth-store.json` — OAuth clients/codes/tokens
- SA key: **none exists** (keyless).
- Connector login password held by Darrell (not in the vault). Rotate anytime.

## Security posture
Read-only `gmail.readonly` (code + Admin DWD); keyless (no SA key on disk);
5-mailbox allow-list; OAuth + TLS on the public endpoint; org "secure by default"
key policy left fully intact.

## The motivating finding — Madison Bowers bounce
Lives in **`dcalhoun@`** (NOT `info@`). Two soft "Delay" notices (Jun 20 & 21) from
`mailer-daemon@googlemail.com`: **SMTP `452 4.2.2` — recipient inbox out of storage
space** (her Gmail is full). No permanent failure → likely delivered on retry, but
her full inbox will keep delaying mail. → prompted the SMS heads-up to her.

## Maintenance quick-ref
- Rotate connector password: `generate_password_hash` into `gmail-oauth-pw.hash`.
- Revoke all sessions: delete `gmail-oauth-store.json` + `pm2 restart gmail-oauth-gateway`.
- Revoke the whole SA: remove Client ID from Admin → DWD, or pull the ADC user's
  Token Creator role.
- Add a scope later: one line in `config.py` `SCOPES` **and** re-authorize that exact
  scope in Admin → Domain-wide delegation (both required).
- README at `/root/dwc-mcp-servers/gmail-domain-mcp/README.md` has full detail.
