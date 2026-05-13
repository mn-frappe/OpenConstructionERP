# Contracts module — research, gap analysis, plan

## Industry benchmarks
- **FIDIC Red/Yellow/Silver/Green books**: Red = employer's design, Yellow = contractor design, Silver = EPC, Green = short form. Sub-clauses are standardised (Sub-Clause 14.x for payment, 8.x for time).
- **JCT Standard / Design-Build / Minor Works**: Section 4 = payment, Section 6 = injury/damage, Section 9 = settlement. Heavily UK-centric.
- **NEC4 ECC**: Activity Schedule (Options A/C) vs Bill of Quantities (Options B/D) vs Target Contract (C/D) vs Cost Reimbursable (E). Pain-share / Gain-share native.
- **AIA A201 / A102 / ConsensusDocs**: US standard form. SOV (Schedule of Values), retainage (typically 5-10% release at substantial completion), conditional vs unconditional lien waivers.
- **Procore Contracts / Aconex Contracts / Causeway / RIB CostX**: SOV grid (line × billed-to-date × current-period × retainage), automated retainage release stage, lien waiver per period attached to draw.

## Top-5 user needs
1. Track SOV: how much of each line is billed vs earned vs paid?
2. Release retention in tiers: 50% at substantial completion, 50% at punch-list closeout.
3. Attach a signed lien waiver to every payment certificate (US compliance).
4. When a Variation gets approved, contract total must update without manual data entry.
5. Approved progress claim must create a finance invoice automatically.

## Top-5 industry-standard features
1. **Schedule of Values (SOV) tracker** with percent-complete per line.
2. **Tiered retention release workflow** (substantial completion + final completion).
3. **Lien waiver registry** (conditional / unconditional, partial / final).
4. **Contract-type clause templates** (FIDIC clause numbers, JCT clause numbers, AIA article numbers).
5. **Auto-invoice on certified progress claim**.

## Top-5 gaps in our code (before this pass)
1. No SOV per-line `billed_to_date` / `earned_to_date` summary view (data exists in claim lines, no aggregation).
2. Retention release: column `retention_release_event` exists but workflow / split logic absent.
3. No lien-waiver model at all.
4. Contract `terms` is a freeform JSON — no clause-template structure tied to FIDIC/JCT/AIA.
5. Approved/certified claims emit events but **no finance subscriber → invoice creation** wired.

## Gap table

| DoD item | Status | Action |
|---|---|---|
| Lump-sum + GMP + cost-plus + T&M claim generators | done | — |
| Status state-machine | done | — |
| Gainshare math | done | — |
| LD math | done | — |
| SOV per-line progress aggregation | missing | IMPLEMENT `sov_status` service + endpoint |
| Tiered retention release | missing | IMPLEMENT model + service + endpoint |
| Lien waivers | missing | IMPLEMENT model + endpoint |
| Clause templates (FIDIC / JCT / AIA) | missing | IMPLEMENT seed catalogue |
| Variation → contract value auto-update | partial (apply_change_order_to_contract method exists, but no subscriber) | wire `variations.*` subscriber |
| Progress claim certified → finance invoice | event emitted, no subscriber | IMPLEMENT subscriber in `_wave1_contracts_subscribers` |
