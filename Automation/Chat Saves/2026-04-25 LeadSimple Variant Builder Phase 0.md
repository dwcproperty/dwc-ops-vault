# LeadSimple Variant Builder — Phase 0 recap

Session date: 2026-04-25 (continued from earlier session)

## Goal

Build automation to create new LeadSimple process types (templates) on demand. 25-100+ "Acct" family variants planned, all spawned from a master template (`Acct Utility Bills`). LeadSimple has no public API for creating process types — every variant has to go through the UI, so we're building a Playwright-driven tool.

## Where things stand

### Done this session

1. **Credentials saved** to `/root/.leadsimple.env` on the Linode (mode 600). No 2FA on the account, plain username + password login works.
2. **Playwright + Chromium installed** at `/root/leadsimple-builder/.venv/`.
3. **Login flow proven** — `id.leadsimple.com/users/sign_in` → OAuth → `app.leadsimple.com/v2`. Session cookie is `_deal_simple_session`. Quirks: SPA never fires `domcontentloaded` for sub-route nav (use `wait_until="commit"`); screenshots hang on font-load if `full_page=True` (use `full_page=False`).
4. **Walkthrough complete** — all 9 tabs of Acct Utility Bills captured via 21 screenshots: General, Users/Roles, Autopilot, Stages & Workflows (overview + each stage's workflow detail), Email Templates (list + open editor), Text Message Templates (list + open editor), Custom Fields (Process / Contact / Property scopes), Version History.
5. **Spec format drafted** at `/root/leadsimple-builder/specs/` — `SCHEMA.md`, `acct_utility_bills_master.yaml` (round-trip of existing master), `example_variant.yaml` (hypothetical Acct Pet Fees variant). Covers all 9 tabs.
6. **Major insight** — LeadSimple uses **GraphQL**. All data calls go to POST `https://api.leadsimple.com/v2/graphql`. Auth is OAuth — login captures an access_token. **Likely we can skip most Playwright complexity for the read path** by using the OAuth token to call GraphQL directly via `requests`.
7. **Phase 0 scraper** built at `/root/leadsimple-builder/scrape.py`. Logs in, walks every settings tab on a fresh page (forces SPA refetch), captures both GraphQL request bodies (the queries themselves) and responses to `/root/leadsimple-builder/scrape_artifacts/`. Last run was in progress when session ended.

### Architecture decisions locked in this session

- **Variant flexibility = full override (option d)**. Every field can vary per variant. Spec is declarative; cloner makes the process type match.
- **Phased build:** 2a General-tab edits → 2b Stage editor → 2c Email/Text template editor → 2d Custom Fields/Users/Autopilot. 2a unblocks metadata-only variants in 1-2 days.
- **Inheritance rule:** scalars inherit from master if omitted in spec; collections (stages, contact_roles, templates) REPLACE entirely if present, INHERIT entirely if omitted. No partial deltas in v1.
- **Custom fields are shared across process types** (LeadSimple's own model). Builder will ADD the variant to existing fields' Process Types lists rather than recreate fields.

## Pick up next session here

1. **Read the captured GraphQL artifacts** at `/root/leadsimple-builder/scrape_artifacts/<latest>/` — index.tsv has the per-tab breakdown, with `.request.txt` (GraphQL query) and `.response.json` (data) per call.
2. **Decide direct-GraphQL vs Playwright for the scraper**. If the GraphQL queries return rich enough data for stages/templates/custom-fields, we can skip Playwright entirely for reads → way faster + more robust. Edits still need Playwright (or GraphQL mutations if exposed).
3. **Then resume on:** finish Phase 0 scraper → validate it round-trips Acct Utility Bills → start Phase 2a (General tab editor) for actually building variants.

## Files / paths

| Thing | Where |
|---|---|
| Credentials | `/root/.leadsimple.env` (mode 600) |
| Playwright venv | `/root/leadsimple-builder/.venv/` |
| Login smoke test | `/root/leadsimple-builder/login_test.py` |
| Spec format | `/root/leadsimple-builder/specs/SCHEMA.md` |
| Master round-trip | `/root/leadsimple-builder/specs/acct_utility_bills_master.yaml` |
| Example variant | `/root/leadsimple-builder/specs/example_variant.yaml` |
| Scraper (Phase 0) | `/root/leadsimple-builder/scrape.py` |
| Scrape outputs | `/root/leadsimple-builder/scrape_artifacts/` |
| Walkthrough screenshots | `/root/leadsimple-builder/walkthrough/01_*.png` – `21_*.png` |

## Acct Utility Bills key facts (for Phase 2a building)

- Process type ID: `d2bab31b-c105-45c9-ae03-e349deb6335c`
- URL slug: `WMd2_kUaNDSZp6_GpZOvMT-2Iv2Rkh_lwW0lJ8Y0DcNy28Tkfq1h`
- Stages (in order): Rentvine Charge Created → Awaiting Tenant Payment → Late Fee Applied → Tenant Paid → Swept to Operating → Contact Tenant → Processing → Billing Complete (completed)
- Default outbound email: `utilities@dwcproperty.com`
- Singular name: `Utility Bills for {{property.street}}`
- Contact roles: Company Owed (Rentvine Vendors), Tenant (Rentvine Tenants)

## Open questions to resolve next session

- Can we issue GraphQL mutations to create process types directly (bypassing Playwright entirely)? Worth a 30-min investigation before committing to Phase 2a Playwright work.
- Lightning-bolt icon on workflow steps = conditions. Spec doesn't model these in v1 yet — flagged.
