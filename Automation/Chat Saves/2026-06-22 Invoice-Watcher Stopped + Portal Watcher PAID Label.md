# 2026-06-22 — Invoice-Watcher Stopped + Utility Portal Watcher PAID Label

Session recap. Three pieces of work, all on the box at `/root/dwc-mcp-servers/`.

## 1. invoice-watcher — STOPPED & decommissioned
- User asked to discontinue the every-120-second invoice watcher entirely.
- Action: `pm2 stop invoice-watcher` → `pm2 delete invoice-watcher` → `pm2 save`. Won't restart on reboot.
- **Effect:** Gmail-labelled (`DWC/Invoice-Inbound`) vendor invoices **no longer auto-flow** into the LeadSimple *Acct Outstanding Invoices* deal. No more AI parsing / Rentvine charge posting from email.
- The internal **Phase 2 Rentvine Bill Bridge** poller lived inside this same process, so it stopped too — but it was paused in dry-run anyway, so no live behavior change there.
- Code/config/state untouched at `/root/dwc-mcp-servers/invoice-watcher/`. Revive with `pm2 start ecosystem.config.js` from that dir. It also held the first/only Anthropic API key on the box.

## 2. Farmers Electric login failure — diagnosed as transient
- Nightly utility-portal-watcher email flagged: *"Farmers Electric: login did not reach an authenticated page."*
- Root cause: the SmartHub portal returned a server-side toast *"An unexpected error has occurred. Please contact customer service"* with the login form blanked — a **portal-side hiccup**, not bad creds or broken code.
- Evidence: CoServ (same SmartHub adapter/platform) logged in fine the same night; Farmers itself worked 6/19–6/21; error counter was at 1 (alert threshold is 3); the older 6/19 error captures were just from initial adapter build.
- Re-ran manually → logged in clean, pulled the bill (`BLUFF CREEK EST L-031 B-B, MURPHY, TX 75094 — $134.20 due 2026-07-01`). Then ran the **full cycle** (`node index.js --once`): all 11 portals OK, summary email sent. No fix needed.

## 3. New feature: PAID / balance-cleared label in the portal watcher
User wanted a paid bill to show distinctly, not as a generic "changed."

**Detection semantics** (`state.js diffBills`, keyed on account + statementDate):
- **NEW** = statementDate never seen for that account (fresh billing cycle).
- **CHANGED** = same statement, `amountDue` moved (e.g. corrected bill) → orange `[CHANGED from $X]`.
- **PAID / CLEARED** (new) = a CHANGED where prev amount was `> 0` and new amount is `<= 0`; flagged `cleared:true`, rendered green `[PAID — cleared from $X]` with an "(incl. N paid)" count in the email + Obsidian summary.
- **UNCHANGED** = silent.

**Files changed:** `state.js` (detect clear), `runner.js` (thread flag + `clearedCount`), `notify.js` (green badge in HTML + plain-text + summary line). Verified by simulation both ways ($134.20→$0 = PAID; $134.20→$150 correction = still CHANGED). Service restarted (`pm2 restart utility-portal-watcher`) + `pm2 save`.

**Caveats user understands:**
1. "Paid" fires only once the *portal's displayed balance* hits $0, so it can lag the real payment by a night or two depending on how fast the utility posts. It reads balance, not a payment confirmation.
2. If a portal rolls a paid balance into a brand-new statement instead of zeroing the old one, it shows as NEW, not PAID — inherent to the portal's data presentation.

## State after session
- 11 portals live in utility-portal-watcher (txu, atmos, princeton, murphy, aubrey, celina, mustang, thecolony, coserv, farmerselectric, providence). Celina = 0 accounts by design (no active account).
- invoice-watcher: down, off pm2.
- Nightly utility bill check still running (2 AM Central). bill-monitor (30-min) still running.
