# Linode Firewall — Removing Public Port 3000 (webhook MCP)

**Date:** 2026-07-08
**Server:** dwc-mcp-server (96.126.118.135)
**Status:** Approved to remove

## Decision

Remove the Linode Cloud Firewall inbound rule:

```
MCP_Server
TCP 3000
All IPv4, All IPv6
```

This is safe because Nginx proxies HTTPS traffic to `127.0.0.1:3000` over the loopback interface, which is not subject to the Linode Cloud Firewall.

## Why This Is Safe

The Linode Cloud Firewall only filters traffic arriving at the server's external network interface (eth0). When Nginx does `proxy_pass http://127.0.0.1:3000`, that traffic goes over loopback and never touches the firewall.

Result:
- `https://webhook.collincountyrent.com` — still works (Nginx → loopback → app)
- `http://96.126.118.135:3000` — blocked from the internet

## Do NOT Do

- Do not stop the PM2 `webhook` app.
- Do not change the Nginx `proxy_pass` config.
- Only remove the Linode firewall inbound rule exposing public 3000.

## Pre-Flight Check

Before relying purely on the firewall, confirm what interface the webhook app is bound to:

```bash
ss -tlnp | grep :3000
```

- `127.0.0.1:3000` → already loopback-only. Firewall rule is defense-in-depth.
- `0.0.0.0:3000` or `*:3000` → app accepts external connections. Only the firewall is stopping them. Fine for now, but ideally update the app's listen config to `127.0.0.1` so it can't accept external connections regardless of firewall state (belt-and-suspenders).

## Verification (After Removing Rule)

Test order matters — verify production path first, then confirm block:

```bash
# 1. Confirm production HTTPS path still works
curl -I https://webhook.collincountyrent.com

# 2. Confirm direct IP:port is blocked from outside the server
curl -v --max-time 5 http://96.126.118.135:3000
```

Expected:
- First returns HTTP/2 200 (or whatever webhook returns for GET /).
- Second hangs and times out — Linode firewalls DROP rather than REJECT, so timeout is the correct blocked behavior (not "connection refused").

## Broader Audit — Other MCP Subdomains

Same posture likely applies to:
- `mcp.collincountyrent.com` (LeadSimple)
- `rentvine.collincountyrent.com`
- `rentengine.collincountyrent.com`
- `obsidian.collincountyrent.com` (port 3004)

The only inbound Linode firewall rules that should be public on this box:
- **22** — SSH
- **80** — Nginx HTTP (redirect to HTTPS)
- **443** — Nginx HTTPS

Everything else should be internal-only. Nginx handles all public HTTPS termination and forwards to the appropriate localhost port per subdomain.

## Related Context

- Server is Ubuntu 24.04.4 LTS at 96.126.118.135
- Now accessible via SSH alias `dwc-mcp-server`
- PM2 manages: LeadSimple, Rentvine, RentEngine, Obsidian MCP, webhook receiver
- `seo-pmw-flipper` is the only stopped PM2 process (noted separately)

## Follow-Up Actions

- [ ] Remove Linode firewall inbound rule for TCP 3000
- [ ] Run pre-flight `ss -tlnp | grep :3000` check
- [ ] Verify HTTPS webhook endpoint still responds
- [ ] Confirm direct port 3000 times out from external
- [ ] Audit remaining MCP subdomain firewall rules (mcp, rentvine, rentengine, obsidian ports)
- [ ] If any app is bound to `0.0.0.0`, update to `127.0.0.1`
