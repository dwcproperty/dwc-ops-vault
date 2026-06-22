# LeadSimple Process Naming Convention

All LeadSimple process titles should be prefixed with one of the four category codes below. The prefix tells you at a glance which department owns the workflow.

## Prefix definitions

| Prefix | Domain | Scope |
|--------|--------|-------|
| **Acct** | Accounting | Money in / Money out |
| **Ops** | Operations | Execution / coordination |
| **PM** | Property Management | Resident / Owner lifecycle |
| **MNT** | Maintenance | Repair-specific workflows |

## Quick rules

- **Acct** — anything where dollars move: bills, invoices, owner draws, deposits, refunds, late fees, collections.
- **Ops** — cross-functional execution and coordination work that doesn't fit the other three (e.g. onboarding, internal handoffs, vendor setup, recurring administrative tasks).
- **PM** — anything tied to the resident or owner relationship lifecycle: applications, move-ins, renewals, move-outs, owner statements, owner communication.
- **MNT** — repair-specific workflows: work orders, inspections, vendor dispatch, repair approvals, warranty issues.

## Examples

- `Acct — Outstanding Invoices`
- `Acct — Owner Distribution Hold`
- `Ops — New Property Onboarding`
- `Ops — Vendor W-9 Collection`
- `PM — Lease Renewal`
- `PM — Move-Out Inspection Coordination`
- `MNT — Standard Work Order`
- `MNT — Emergency After-Hours Repair`

## When in doubt

Ask: *"What is the primary purpose of this process?"*

- If the answer is fundamentally about **money changing hands** → **Acct**
- If it's about a **resident or owner relationship event** → **PM**
- If it's about a **physical repair to the property** → **MNT**
- Everything else (coordination, execution, internal admin) → **Ops**


---

## Updates

- **2026-06-22** — Convention reaffirmed. New process **`Acct — Late Fee Relief`** (tenant late-fee waiver intake via JotForm) classified under **Acct** (late fees = money). When creating any new LeadSimple process, apply one of the four prefixes above (Acct / Ops / PM / MNT) with the `Prefix — Name` em-dash format. Older processes use mixed prefixes (numbers, BAC, HOA) and can be renamed to this standard over time.
