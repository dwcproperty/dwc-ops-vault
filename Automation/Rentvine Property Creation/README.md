# Rentvine Property Creation Automation

> Auto-creates Rentvine portfolio + owner + property records when a deal in the LeadSimple "DWC Owner Leads" pipeline moves to **Signed Contract** (Won).

**Status:** Live. Hardened on 2026-05-05 with three-layer dedup and a mandatory end-of-run email summary.
**Schedule:** Daily at **2 AM local** (cron `0 2 * * *`, ~2:07 AM after dispatch jitter). Can also be run manually from Cowork → Scheduled at any time.
**Runs in:** Cowork (Claude desktop app), scheduled task `rentvine-property-from-won-lead`.
**NOT in:** Claude Code CLI. The prompt file is portable if we ever migrate.
**Notify on completion:** in-app notification + email summary to office@dwcproperty.com.

---

## How it works

1. **Trigger:** A LeadSimple deal in pipeline `DWC Owner Leads` moves to stage `Signed Contract` (status: won).
2. **Cowork polls** LeadSimple once daily for new Won deals (deduped by the `RENTVINE_PROPERTY_CREATED:` marker in deal comments AND the three-layer Rentvine dedup check below).
3. For each new Won deal, Claude:
   - Pulls owner contact + property data from the LeadSimple deal.
   - **Runs the three-layer dedup check** (active address → inactive address → owner-name fallback). On any hit, stops and posts a typed marker comment instead of creating.
   - Falls back to the county Central Appraisal District (CAD) for missing beds / baths / sqft / year built / legal description.
   - Looks for a signed contract envelope on RentSign (matched by owner email).
   - Creates a Rentvine **Portfolio** with one or more owner contacts attached.
   - Creates the Rentvine **Property** with the full field map below, **using the naming standard verbatim**.
   - Posts a back-comment to the LeadSimple deal so future polls skip it.
4. **Emails a summary** to office@dwcproperty.com listing every deal processed and its disposition.

## Key IDs (do not change)

| Thing | ID |
|---|---|
| Pipeline: DWC Owner Leads | `96033e00-eaf8-4173-a453-3224d8e20e9b` |
| Stage: Signed Contract (Won) | `c69cbe47-1175-4d64-87a2-aa6ae9ba5361` |
| Dedup marker (in deal comment) | `RENTVINE_PROPERTY_CREATED:` |
| Inactive-match marker | `RENTVINE_DUPLICATE_FOUND:` |
| Owner-only-match marker | `RENTVINE_REVIEW_NEEDED:` |
| Blocked marker | `RENTVINE_BLOCKED:` |

## Files

- **Prompt file (canonical):** `C:\Users\Darrell\Documents\Claude\Scheduled\rentvine-property-from-won-lead\rentvine_create_property_prompt.md`
- **Scheduled task (SKILL.md):** `C:\Users\Darrell\Documents\Claude\Scheduled\rentvine-property-from-won-lead\SKILL.md`
- Both files now live next to each other in the persistent scheduled-task folder, so every run can read the prompt regardless of session.

## Naming standard (MANDATORY)

Every property record uses the **full, unabbreviated** address. Applies both to new creation and dedup search keys:

- **Directionals in full**: `North`, `South`, `East`, `West` — never `N`, `S`, `E`, `W`.
- **Suffixes in full**: `Drive`, `Street`, `Lane`, `Avenue`, `Boulevard`, `Road`, `Court`, `Place`, `Way`, `Trail`, `Circle`, `Cove` — never `Dr`, `St`, `Ln`, etc.
- **Unit / apt / # tokens go in `Address 2`** — never in line 1 of the street address.
- Property `Name` field: leave blank, OR mirror the full address.
- Example: `529 West Lookout Drive` (line 1) + `#226` (line 2) — NOT `529 W Lookout Dr #226` and NOT `529 West`.

Why: existing Rentvine data mixes formats (e.g., propertyID 25 has `address: "529 West Lookout Drive"` but `name: "529 West Lookout Dr"`). The naming standard prevents that going forward.

## Three-layer dedup check (MANDATORY before any create)

If **any** layer returns a hit (active OR inactive), the run STOPS for that deal — never creates a duplicate.

| Layer | Tool | What it catches | On hit |
|---|---|---|---|
| 1 — Active address | `mcp__rentvine__find_property_by_address` | Active properties at the exact address | Skip; post `RENTVINE_PROPERTY_CREATED:` with the existing property URL |
| 2 — Inactive address | `mcp__rentvine__list_properties` with `is_active: false`, `search="<distinctive street token>"` | Properties that exist but were deactivated | Stop; post `RENTVINE_DUPLICATE_FOUND: inactive propertyID <n> at <address> — reactivate manually` |
| 3 — Owner name | `mcp__rentvine__list_owners` with the owner's last name | Existing owner under a different/typo'd address | Stop; post `RENTVINE_REVIEW_NEEDED: existing owner may already have this property — verify manually` |

**Inactive matches mean reactivate, never recreate.**

Known tool quirks the procedure works around:
- `find_property_by_address` is **active-only** (no `is_active` param). Layer 2 is the safety net.
- `find_property_by_address` trim has a **directional bug**: `"529 West Lookout Drive"` trims to `"529 West"` and returns nothing. Layer 2 catches this by searching the distinctive part of the street name (e.g. `Lookout`).
- `list_owners` currently returns HTTP 404 in some cases — Layer 3 may degrade to no-op until that's fixed; Layers 1+2 still run.

## End-of-run email summary

Every run sends an HTML email to `office@dwcproperty.com` via the Zapier Gmail "Send Email" action. The email contains:

- One-line headline: counts of `created` / `skipped-active` / `skipped-inactive` / `review-needed` / `blocked`
- Per-deal table with deal name, property address, disposition, propertyID (matched or created), and clickable links to LeadSimple and Rentvine
- A "Data quality / follow-ups" section for anything needing human attention (RentSign envelope not found, naming-convention violations in existing records, deals with empty addresses, etc.)
- Any `RENTVINE_BLOCKED:` / `RENTVINE_DUPLICATE_FOUND:` items with their exact reason

If the email send fails the run continues; the in-app notify-on-completion fires as a backup.

## User defaults

| Setting | Value |
|---|---|
| Portfolio Name | Owner Full Name (joint owners: "First & Second" e.g. "Naga Sudheer Ravela & Priyanka Surapaneni") |
| Reserve Amount | $300.00 |
| Assignee | Pulled from LeadSimple deal `assignee` |
| Contract source | RentSign (skip + flag if not found) |
| Management Fee Setting | Default ($150 mgmt / 50% lease / $400 renewal) |
| Co-owners from CAD | Add to portfolio with even % split (50/50, 33/33/33, etc.); secondary owners get name + property mailing address only |
| County | Trust the CAD's reported county (e.g. Celina TX 75009 = Denton, NOT Collin) |

## Property type mapping (LeadSimple → Rentvine)

| LeadSimple value | Rentvine value |
|---|---|
| Single Family / Single Home / Single Family Home | `Single Family Home` |
| Townhouse / Townhome | `Townhouse` |
| Condo / Condominium | `Condo` |
| Duplex | `Duplex` |
| Apartment | `Apartment` |
| Multiplex / Tri-plex / Four-plex | `Multiplex` |
| Mobile / Manufactured | `Mobile Home` |
| (anything else) | default to `Single Family Home`, flag for review |

Rentvine accepts: Single Family Home, Apartment, Condo, Townhouse, Duplex, Multiplex, Loft, Mobile Home, Commercial, Garage, Lot.

## County CAD lookup endpoints (DWC service area)

| County | Search URL |
|---|---|
| Collin (Plano, Frisco, McKinney, Allen, Princeton — sometimes Celina) | https://www.collincad.org/propertysearch |
| Denton (Denton, Lewisville, Providence Village, Aubrey, **Celina 75009**) | https://www.dentoncad.com/property-search |
| Dallas | https://www.dallascad.org |
| Tarrant | https://www.tad.org |
| Grayson (Sherman, Denison) | https://www.graysoncad.org |
| Fannin (Bonham) | https://esearch.fannincad.org |
| Rockwall | https://www.rockwallcad.com |

CAD pages give: Bedrooms, "Plumbing" count (= total baths), Living/Main Area sqft, Year Built, Legal Description, joint deed owners.

## Form fields

### Add Portfolio (`/portfolios/add`)
Owners section → **Add New Owner** → fill Name (required), Email, Phone Number + Type (Mobile), then click "Yes, lets add one" for Address and use the autocomplete. Save the slide-out.
Repeat for each co-owner found on the deed.
Set Percent Owned for each owner (e.g. 50/50). Total must = 100%.
Portfolio Name auto-populates from owner names.
Fiscal Year End: December (default). Other fields blank.
Click **Save** → portfolio ID appears in URL.

### Add Property (`/properties/add`)
- Portfolio: search by owner name, select.
- Address: autocomplete fills city / state / zip / county. **Manually fix the street to the full unabbreviated form** if autocomplete abbreviated it. **Strip any unit/apt/# token from this field** — that goes in Address 2.
- Address 2: unit/apt/# token (required if the deal address contains one).
- Name: blank (or mirror the full address).
- Legal Description: from CAD.
- Assignee: deal assignee.
- Property Type: mapped value.

> **The Rental Information section is hidden until a Portfolio is selected.** Easy to miss. It contains:
> - **Market Rent Amount** (REQUIRED — from deal `value`, e.g. 2350)
> - Security Deposit Amount (blank → $0)
> - Square Footage
> - Bedrooms (radio 1–10+)
> - Full Baths (radio 1–10+)
> - Half Baths (radio 1–10+, only if `.5` fraction)

- Reserve Amount: $300.00.
- Date Contract Begins: date the deal moved to Won.
- Year Built: from CAD.
- Attach Files: signed contract PDF.

## Run history

### 2026-05-05 — first end-to-end create (Naga Sudheer Ravela)
- **Deal:** `b94c3fdb-1e0e-44bf-925f-f7b7b451b84a`
- **Property:** [16712 Hidden Cove Dr, Celina, TX 75009](https://dwcpropertygroup.rentvine.com/properties/67) (Property #67)
- **Portfolio:** [Naga Sudheer Ravela & Priyanka Surapaneni](https://dwcpropertygroup.rentvine.com/portfolios/33) (Portfolio #33)
- **CAD source:** Denton CAD PropID `960942`
- **Result:** All fields populated correctly. 4 BR / 3 Full Baths / 2,984 sqft / Year 2023.
- **Manual cleanup needed:** attach signed contract (RentSign envelope not found); verify bath count (CAD plumbing=3, entered as 3 full / 0 half — could be 2.5); add email/phone for Priyanka if available.

### 2026-05-05 — dry run with hardened logic (post-fix verification)
- 19 Won deals scanned (the earlier "27" in the prior automated report was inflated).
- **Layer 1 active matches: 17.** Includes Naga (67), Cottontail (43), Alexandra (65), Belgian (38), Campeiro (42), Burmese (37), Aledo (40), Kinglet (39), Kimsey quadplex (41 + units 68–71), Central Court (36), Welsh Lane 2044 (31), Welsh Lane 2028 (32, *but stored as "Welsh Road" — see data quality*), Ramsey (21), Hamil (14), Robinia (29), Lottie (20), Sterling (16), Salisbury (22).
- **Layer 2 inactive matches: 2** — would have been duplicates under the old logic:
  - Cheryl Torres / [2263 Cashmere Way, Princeton](https://dwcpropertygroup.rentvine.com/properties/34) — propertyID **34** (deactivated 2025-12-03).
  - Renate Racher / [529 West Lookout Drive #226, Richardson](https://dwcpropertygroup.rentvine.com/properties/25) — propertyID **25** (deactivated 2025-12-30). Layer 1 missed this because of the directional-prefix trim bug; Layer 2 caught it via `search="Lookout"`.
- **Truly-new (would create): 0.** No writes performed.
- **Sarat Sathuluri** (1837 Whispering Pines Drive, Celina) was Section B-1 of the prior auto-report but is no longer in Won — stage may have moved or deal was deleted. Worth a glance.
- **Data quality flagged for cleanup:**
  - PropertyID 32 stored as `2028 Welsh Road` — should be `2028 Welsh Lane` per the deal and the neighbor at 2044.
  - PropertyID 37 stored as `3132 Burmese St` — should be `Burmese Street` per the new naming standard.
  - PropertyID 25 has `name: "529 West Lookout Dr"` while `address: "529 West Lookout Drive"` — `name` is abbreviated and should match the address.

## Known issues / open items

- [ ] **LeadSimple `create_note` API returns HTTP 400.** Markers can't be posted via API. Fallback options: post via Chrome browser automation on the deal page, OR fix the API request shape. Until resolved, every nightly run keeps re-evaluating the same already-matched deals (cheap, no harm) but no marker is left behind.
- [ ] **`mcp__rentvine__list_owners` returns 404.** Layer 3 of dedup degrades to no-op. Layers 1+2 still run.
- [ ] **`find_property_by_address` directional-prefix trim bug** (`"529 West Lookout Drive"` → `"529 West"`). Layer 2 mitigates. Worth fixing in the underlying MCP eventually.
- [ ] **Cowork-bound runtime.** Task only runs when the laptop is on, signed in, and has Chrome open with Rentvine logged in. For unattended runs we'd need a server-side Claude Code setup with a LeadSimple webhook.
- [ ] **No CAD coverage outside DFW counties.** Add new CAD URL to the prompt if we expand.
- [ ] **Pre-approve tools** by clicking "Run now" once on the scheduled task in Cowork's Scheduled section so future runs don't pause for permission prompts.
- [ ] **Data tidies in Rentvine** (do these one time, manually, when convenient): rename propertyID 32 from `Welsh Road` to `Welsh Lane`; rename propertyID 37 to `Burmese Street`; align propertyID 25 name with full address.

## Workflow steps the automation does NOT do (still manual)

- Attach the signed management contract PDF to the property (RentSign doesn't have envelopes for older deals — manual upload required).
- Set up the unit's listing for marketing.
- Configure tenant screening criteria.
- Send the welcome email to the new owner.
- **Reactivate** an existing inactive property when a Layer 2 hit fires (the automation flags it; reactivation is a human decision).

## Tags
#automation #rentvine #leadsimple #onboarding


---

## See also

- [[Deactivation playbook]] — what happens when a property is deactivated in Rentvine, how the poll responds, and the corrective pattern for deals with blank/abbreviated addresses (Galaxy/Chen 2026-05-10).

## Run history (cont.)

### 2026-05-10 — full-pipeline scan + live deactivation remediation
- **Scope:** All 6 pages of the DWC Owner Leads pipeline (127 total deals, 27 Won).
- **Created:** 0. All addressable Won deals returned Layer 1 active matches in Rentvine.
- **Skipped-active:** 25 (verified via `find_property_by_address`).
- **Already-managed (Layer 4):** 1 — Nalin Ratnaike → Portfolios 28 & 29 (Thurstan 100 LLC / Red Brick Properties LLC).
- **Live remediation:** Michael Galaxy & Ge Chen (deal `e87c8ae1`) — Rentvine propertyID 47 (3983 Peregrine Point, Celina) was deactivated 2026-05-07. Deal had a blank `address`. Browser fallback used to fix Address 1 + Property Name to the full street, then post a `RENTVINE_DUPLICATE_FOUND` marker. Next poll will Layer 2-hit and classify `skipped-inactive`.
- **Email summary:** sent to office@dwcproperty.com (Gmail message id `19e13aabfaf320b4`).
- **API issues confirmed again:** both `mcp__leadsimple__create_note` and `mcp__leadsimple__update_deal` (with `notes`) return HTTP 400. Notes posted via UI don't appear in `get_deal`'s payload either, so the scheduled poll has no API path to detect markers — Rentvine dedup is the operative skip check.

#deactivation
