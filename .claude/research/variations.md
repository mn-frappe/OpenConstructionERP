# Variations & Site Measurements — Research

## Industry references
- **FIDIC Red Book 2017 Sub-Clause 13** — Variations procedure: Engineer instruction → Contractor's proposal (Sub-Clause 13.2) → Engineer's value (13.3.1) → cost record (13.3.2). Sub-Clause 13.5 covers provisional sums. Sub-Clause 13.6 covers Daywork.
- **JCT SBC/Q 2016 Clause 5** — Variations Valuation Rules (5.6 — Schedule 2 rates; 5.7 — fair valuation; 5.8 — additional/omitted work).
- **NEC4 ECC Clause 60-65 — Compensation Events** — Notification (Cl. 61.3 within 8 weeks), quotation by Contractor (Cl. 62.3 within 3 weeks), Project Manager assessment (Cl. 62.5 within 4 weeks if no agreement).
- **BS 6079-1:2019** — Project Management — Time-cost-quality framework; recommended Daywork sheet structure (Labour, Plant & Equipment, Material with markup).
- **AICPA Construction Audit Guide** — measured-mile method for disruption claims; baseline-productivity vs impacted-productivity comparison.
- **SCL Delay & Disruption Protocol 2nd ed** — time-impact-analysis (TIA) procedure for EoT claims; concurrent delay treatment.
- **Procore Change Management** — instruction → estimate → owner approval → execution → cost-record → final-account workflow.
- **Aconex CM** — variation lifecycle with mobile site-measurement capture + customer signature.

## Top 5 user-needs
1. **FIDIC clause 13 dedicated workflow** with sub-clause field stamping (which clause was triggered) and Engineer's-determination snapshot.
2. **NEC4 compensation-event countdown timer** — 3 weeks bidder quotation + 4 weeks PM assessment SLA, with status flag.
3. **BS 6079-compliant Daywork sheet** with explicit Labour/Plant/Material rows + percentage markup column.
4. **EoT claim with critical-path link** — must reference a `schedule_advanced` activity whose criticality (`is_critical_path=true`) drives whether the days are granted.
5. **Disruption claim with measured-mile baseline** — explicit baseline productivity rate (units/hour) vs impacted productivity rate, period comparison.
6. (Plus) **Auto contract-sum update event** on VO approval to update `oe_contracts.total_value`.

## Top 5 industry features
1. Standards-aware lifecycle (FIDIC 13, JCT 5, NEC4 CE).
2. SLA timer with overdue flag.
3. BS 6079 Daywork sheet with markup.
4. Measured-mile disruption methodology.
5. TIA-aware EoT claim.

## Top 5 gaps in current code (delta from PRD §14)
1. **No clause-citation field** — `VariationRequest`/`VariationOrder` have `classification` but no contract-clause reference, so audit trail to FIDIC/JCT/NEC4 is informal.
2. **No NEC4 quotation/assessment timer** — `submitted_at`/`decision_at` exist but no SLA logic computing overdue or escalation flag.
3. **No Daywork markup column** — `DayworkSheet` has `total_amount` but no markup-percentage; auditors can't tell what was OH&P vs raw cost.
4. **No measured-mile fields** — `DisruptionClaim` has `cost_amount` but no `baseline_productivity` / `impacted_productivity` numeric fields.
5. **No critical-path enforcement on EoT** — `ExtensionOfTimeClaim.critical_path_impact` is a boolean set manually; should be validated by lookup against `schedule_advanced.BaselineDelta` for the affected activity.
6. (Plus) **No contract-sum bump event** — VO approval doesn't emit anything for `oe_contracts` to consume.
