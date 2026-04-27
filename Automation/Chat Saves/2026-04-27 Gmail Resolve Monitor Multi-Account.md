# 2026-04-27 — Gmail Resolve Monitor: Multi-Account Expansion

## Outcome
`gmail-resolve-monitor` (pm2 id 12) now polls **3 Gmail accounts** instead of 1. Live and verified.

- `office@dwcproperty.com` (already running)
- `info@dwcproperty.com` (added)
- `dcalhoun@dwcproperty.com` (added)

First successful 3-account cycle ended **2026-04-27 19:34:10Z**:
- `accounts=3`, `closed=0`, `errors=0`, `alreadyProcessed=525`, `skipNoMatch=1131`
- ~3.5 min wall time on the 30-min cron — comfortably under window.

## What changed
1. Refactored monitor from single-token to per-account token loop.
2. Token storage moved to `auth/<email>.token.json` (mode 600) — one OAuth token per account, all under `gmail.modify` scope.
3. Migrated existing `office@` token to the new path layout and restarted pm2.
4. Ran fresh OAuth consent for `info@` and `dcalhoun@` via the non-interactive `node redeem-code.js <email> "<redirect-url>"` flow (browser the consent URL → paste back the `localhost` redirect → token persisted).

## Re-consent recipe (for future accounts / token expiry)
1. Build consent URL with client id `462267609532-ajgpeskipe25h9hl06g2p2pp8sjo2785.apps.googleusercontent.com`, redirect `http://localhost`, scope `https://www.googleapis.com/auth/gmail.modify`, `access_type=offline`, `prompt=consent`.
2. User opens URL in a browser logged in as the target account, approves, copies the (broken) `http://localhost/?...code=...` redirect URL.
3. `node redeem-code.js <email> "<redirect-url>"` writes `auth/<email>.token.json`.
4. `pm2 restart gmail-resolve-monitor` to pick up the new token; confirm next cycle log shows `accounts=N` matching the token count.

## Health check
Look for this line each cycle:
```
[YYYY-MM-DDTHH:mm:ssZ] --- Cycle start (DRY_RUN=false, accounts=3) ---
```
If `accounts=` drops below 3 after a cycle, a token has gone missing/expired — re-consent the affected account.

## Memory updated
`project_gmail_resolve_monitor.md` now reflects the multi-account layout, per-account token paths, and the consent-URL recipe.

## Side note
The previous session ended mid-task because the `pm2 logs … until grep` command hit its 10-minute timeout while waiting for the cycle to finish. The cycle had actually completed cleanly — not a quit, just an unwatched success. Verified after-the-fact from the pm2 log buffer.
