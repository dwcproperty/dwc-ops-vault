# Deactivation Playbook

> What happens when you deactivate a Rentvine property, and how the scheduled poll handles the deal afterward.

Companion to [[README]] in this folder. Read it first for the create-side flow.

---

## What "deactivate" actually does in Rentvine

When you deactivate a property in the Rentvine UI:

- `isActive` flips from `1` to `0`.
- `dateTimeDeactivated` is stamped with the moment you clicked the button.
- **The record is preserved**, not deleted. All historical fields (address, portfolio link, owner, year built, contract dates, attached docs, accounting history) stay intact and are recoverable.
- The property **disappears from the active dashboard** and from any tool that filters by active-only.
- The **portfolio and owner records are NOT automatically deactivated**. They linger unless you also clean them up manually. (Currently we leave them.)

## How the poll behaves after a property is deactivated

| Dedup layer | Tool | What it returns for a deactivated property |
|---|---|---|
| Layer 1 — Active address | `mcp__rentvine__find_property_by_address` | **Empty.** This tool is active-only — it has no `is_active` parameter, so deactivated records are invisible to it. |
| Layer 2 — Inactive address | `mcp__rentvine__list_properties` with `is_active: false` + `search="<distinctive street token>"` | **Hits the deactivated record** and returns its `propertyID`, `streetNumber`, `streetName`, `dateTimeDeactivated`. |
| Layer 3 — Owner name | `mcp__rentvine__list_owners` | Currently returns HTTP 404 — degrades to no-op. Layer 2 covers the gap. |
| Layer 4 — Company/LLC portfolio | Chrome search of `/portfolios` | Hits if the deactivated property was under an LLC/agent portfolio with multiple properties. |

The scheduled poll runs all four layers on every Won deal. If Layer 2 fires, the deal is classified `skipped-inactive` and the poll posts a `RENTVINE_DUPLICATE_FOUND:` marker on the LeadSimple deal.

## The cardinal rule

**Reactivate, never recreate.** If a property exists in Rentvine but is deactivated, the correct workflow is to reactivate the existing record manually — never let the automation create a fresh duplicate. The inactive record carries the property's history; a duplicate orphans it.

Why this matters: a duplicate would split rent ledgers, work orders, inspections, and lease history across two propertyIDs, which is painful to merge after the fact.

## When the deal address is missing or abbreviated (the Galaxy / Chen pattern)

If the LeadSimple deal has a blank or abbreviated street on the property record, the poll can't run Layer 2 — there's no distinctive street token to search for. The deal lands as `review-needed` in the summary email and stays that way every night until someone fixes it.

**The corrective workflow** (used 2026-05-10 for the Michael Galaxy & Ge Chen deal, propertyID 47 at 3983 Peregrine Point):

1. Identify the actual street address — usually findable in the deal's activity (email subjects, texts), in RentSign, or via the deactivated Rentvine record itself if you already know the propertyID.
2. Open the LeadSimple deal in the browser. Click the pencil edit icon on the Properties card.
3. Fill **Address 1** with the **full, unabbreviated** street (e.g., `3983 Peregrine Point`, not `3983 Peregrine Pt.`). Press **Tab** or click another field — do **NOT** press Escape, that closes the modal without saving.
4. Update **Property Name** to mirror the same full address. Triple-click to select-all before retyping.
5. Click **Save**.
6. Open the deal's **Note** action and post the marker:
   ```
   RENTVINE_DUPLICATE_FOUND: inactive propertyID <n> at <full address> (portfolioID <n>, deactivated <UTC timestamp>).
   Future scheduled polls: Layer 1 returns empty because the record is inactive; Layer 2 (list_properties search="<distinctive token>", is_active=false) catches propertyID <n>.
   Do not auto-create over the deactivated record — reactivate the existing record manually if/when this owner returns.
   ```

After step 6, the next nightly run will Layer 2-hit and classify the deal as `skipped-inactive` in the email summary, and stop flagging it as review-needed.

## API quirks that matter

Both LeadSimple write paths through the MCP currently return **HTTP 400**:

- `mcp__leadsimple__create_note` — confirmed broken 2026-05-05, 2026-05-08, and 2026-05-10.
- `mcp__leadsimple__update_deal` with `notes` parameter — confirmed broken 2026-05-10.

Until those are fixed, markers must be posted via Chrome browser automation through the LeadSimple **Note** action button on the deal page. The poll falls back to this automatically when the API call errors.

A related quirk: notes posted via the UI **do not appear** in `mcp__leadsimple__get_deal`'s response. The `comments` field returned by `get_deal` only contains the original lead-form submission, not subsequent notes. This means the scheduled poll **cannot reliably detect markers via API** and has to rely on the Rentvine dedup check (Layer 1/2) as the operative skip mechanism. The marker is still useful for human-readable trail and prevents future creates if the API ever surfaces notes.

## Don't deactivate from inside this automation

The scheduled task only reads from Rentvine — it never deactivates anything. Deactivation is a human decision (lease ended, contract terminated, owner sold, etc.) and stays manual. The automation's job is to handle the post-deactivation state correctly on the next poll.

## Run record — 2026-05-10

Today's poll surfaced the Galaxy / Chen review-needed case and we cleaned it up live:

- **Deal:** Michael Galaxy and Ge Chen (`e87c8ae1-d886-4f27-adab-8c26dae7c268`)
- **Rentvine:** propertyID 47 (3983 Peregrine Point, Celina TX 75009), portfolioID 27, deactivated 2026-05-07 21:09:39 UTC
- **Fix:** updated LeadSimple deal property to `3983 Peregrine Point` (was blank line 1, abbreviated name); posted the `RENTVINE_DUPLICATE_FOUND` marker via browser-fallback Note
- **Email summary:** sent to office@dwcproperty.com — Gmail message id `19e13aabfaf320b4`
- **Other Won deals scanned:** 26 more across all 6 pages of the pipeline; 25 confirmed in Rentvine via Layer 1, 1 (Nalin Ratnaike) already-managed via Layer 4 portfolios 28/29. No creates this run.

## Tags
#automation #rentvine #leadsimple #deactivation #playbook
