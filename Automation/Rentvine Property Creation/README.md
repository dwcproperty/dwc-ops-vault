# Rentvine Property Creation Automation

> Auto-creates Rentvine portfolio + owner + property records when a deal in the LeadSimple "DWC Owner Leads" pipeline moves to **Signed Contract** (Won).

**Status:** Live as of 2026-05-05 — first end-to-end run completed successfully.
**Runs in:** Cowork (Claude desktop app), scheduled task `rentvine-property-from-won-lead`, every 15 minutes.
**NOT in:** Claude Code CLI. The prompt file is portable if we ever want to migrate.

---

## How it works

1. **Trigger:** A LeadSimple deal in pipeline `DWC Owner Leads` moves to stage `Signed Contract` (status: won).
2. **Cowork polls** LeadSimple every 15 min for new Won deals (deduped by the `RENTVINE_PROPERTY_CREATED:` marker in deal comments).
3. For each new Won deal, Claude:
   - Pulls owner contact + property data from the LeadSimple deal.
   - Falls back to the county Central Appraisal District (CAD) for missing beds / baths / sqft / year built / legal description.
   - Looks for a signed contract envelope on RentSign (matched by owner email).
   - Creates a Rentvine **Portfolio** with one or more owner contacts attached.
   - Creates the Rentvine **Property** with the full field map below.
   - Posts a back-comment to the LeadSimple deal so future polls skip it.

## Key IDs (do not change)

| Thing | ID |
|---|---|
| Pipeline: DWC Owner Leads | `96033e00-eaf8-4173-a453-3224d8e20e9b` |
| Stage: Signed Contract (Won) | `c69cbe47-1175-4d64-87a2-aa6ae9ba5361` |
| Dedup marker (in deal comment) | `RENTVINE_PROPERTY_CREATED:` |

## Files

- **Prompt file (canonical):** `C:\Users\Darrell\AppData\Roaming\Claude\local-agent-mode-sessions\541a22d9-6826-46c9-bfcf-57aa0dc0f0ab\38c1c629-92ad-4661-8632-42668ed632fa\local_0188e1b0-dbde-47b8-9b01-efad28cdaf96\outputs\rentvine_create_property_prompt.md`
- **Scheduled task:** `C:\Users\Darrell\Documents\Claude\Scheduled\rentvine-property-from-won-lead\SKILL.md`

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
- Address: autocomplete fills city / state / zip / county.
- Name: blank. Legal Description: from CAD.
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

## Test run record

### 2026-05-05 — Naga Sudheer Ravela
- **Deal:** `b94c3fdb-1e0e-44bf-925f-f7b7b451b84a`
- **Property:** [16712 Hidden Cove Dr, Celina, TX 75009](https://dwcpropertygroup.rentvine.com/properties/67) (Property #67)
- **Portfolio:** [Naga Sudheer Ravela & Priyanka Surapaneni](https://dwcpropertygroup.rentvine.com/portfolios/33) (Portfolio #33)
- **CAD source:** Denton CAD PropID `960942`
- **Result:** ✅ All fields populated correctly. 4 BR / 3 Full Baths / 2,984 sqft / Year 2023.
- **Manual cleanup needed:**
  - Attach signed contract (RentSign envelope not found)
  - Verify bath count (CAD plumbing=3, entered as 3 full / 0 half — could be 2.5)
  - Add email/phone for Priyanka if available

## Known issues / open items

- [ ] **LeadSimple `create_note` API returns HTTP 400.** The scheduled task can't post the dedup marker via API. Fallback: post via Chrome browser automation on the deal page. Risk: if the marker doesn't get posted, the next poll will try to create a duplicate property — `find_property_by_address` should catch it but worth monitoring.
- [ ] **Rental Info section was missed in initial form documentation** because it only renders after a Portfolio is selected. Now baked into the prompt.
- [ ] **Cowork-bound runtime.** Task only runs when the laptop is on, signed in, and has Chrome open with Rentvine logged in. If we want unattended runs, we'd need a server-side Claude Code setup with a LeadSimple webhook.
- [ ] **No CAD coverage for non-DFW counties.** If we ever take on properties outside Collin/Denton/Dallas/Tarrant/Grayson/Fannin/Rockwall, add the new CAD URL to the prompt.
- [ ] **Pre-approve tools** by clicking "Run now" once on the scheduled task in Cowork's Scheduled section, so future runs don't pause for permission prompts.

## Workflow steps the automation does NOT do (still manual)

- Attach the signed management contract PDF to the property (RentSign doesn't have envelopes for older deals — manual upload required).
- Set up the unit's listing for marketing.
- Configure tenant screening criteria.
- Send the welcome email to the new owner.

## Tags
#automation #rentvine #leadsimple #onboarding