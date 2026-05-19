# Inspections Portal — Handoff (paused 2026-05-19)

**Status:** Design complete, awaiting implementation. Plan approved by user but execution paused per user request before any code was written.

**Resume prompt:**
> Resume the inspections-portal build from `/root/.claude/plans/i-see-other-property-vivid-kahan.md`

## Quick recap of what's planned

New service at `/root/dwc-mcp-servers/inspections-portal/` that mirrors OnSightPros property-inspection reports as DWC-branded pages at `inspections.collincountyrent.com/<address-slug>`.

- **Stack:** Express + better-sqlite3 + EJS + Passport Google OAuth, PM2 cluster on port 3017 — matches `finances-tracker`/`processes-tracker` pattern.
- **Scraper:** Playwright logs into OnSightPros, intercepts XHRs to `api.onsightpros.com/api/v1/*`, harvests bearer token, downloads photos/panos/videos from `walkthru-data.s3.us-east-2.amazonaws.com`. Sha256 dedupe.
- **Viewer:** Server-rendered EJS with self-hosted **PhotoSwipe v5** (lightbox + pinch-zoom = the "magnifying glass") and **Pannellum v2** (360° panoramas). Native `<video>` for clips.
- **Admin:** `/admin` (Google OAuth allowlist) — paste OnSight URL, see live SSE progress, list/re-scrape/email-owner/archive.
- **Audience:** Property owners — periodic inspection delivery. DWC branding throughout, "Download PDF" + "Powered by DWC Property Management" footer.

## What user needs to provide on resume

1. **OnSightPros login credentials** — `ONSIGHT_EMAIL` + `ONSIGHT_PASSWORD` for an account that can view the test report at `https://app.onsightpros.com/report/1D1E3C6F-FA98-6843-9B25-99B7A55B8A72`. Will go in `.env`.
2. **(Optional)** Property address for the test report — if known. Otherwise we'll read it from the scraped data.
3. **(Implicit)** Permission to edit `/etc/nginx/sites-available/inspections.collincountyrent.com` (currently a static-file vhost — will be replaced with a reverse proxy to port 3017) and run `certbot` to refresh the cert if needed.

## Existing state on the box

- The nginx vhost `inspections.collincountyrent.com` already exists pointing at `/var/www/inspections` (static). This will be **replaced**, not added — back it up before swapping.
- Playwright is already installed at `/root/dwc-mcp-servers/seo-content-engine/node_modules/playwright` (version confirmed working). The new service can either depend on its own copy via `package.json` or symlink — to be decided at implementation.
- Google OAuth client is reused from finances-tracker: `462267609532-njj6kap226ar6sb0mo4l9obbhdg223m8.apps.googleusercontent.com`. Will need to add `inspections.collincountyrent.com/auth/google/callback` to authorized redirect URIs in Google Cloud console.

## Open decisions deferred to implementation

- **PDF rendering** — use Playwright headless (preferred, reuses existing dep) or `puppeteer` (extra dep). Defaulting to Playwright.
- **Whether to email owner automatically** on import-success vs require an explicit click. Plan currently says explicit click — user can override.
- **Session secret rotation** — generate fresh `SESSION_SECRET` per service (don't share with finances-tracker).

## Related work landed today (already shipped — independent)

- `/kpi` page on finances-tracker is live with RPD, monthly P&L (gross/expenses/net/owner-pay/retained), vacancy/delinquency, renewal rate, WO times. See `project_kpi_dashboard_live.md` in memory.

## File map

- **Plan:** `/root/.claude/plans/i-see-other-property-vivid-kahan.md`
- **This handoff:** `dwc-ops/Automation/Inspections Portal/Handoff — paused 2026-05-19.md`
