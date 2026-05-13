# Resources module — research, gap analysis, plan

## Industry benchmarks
- **MS Project Resource Pool**: shared pool, capacity-vs-demand, manual leveling button. Limited skill matching.
- **Primavera P6 resource leveling**: priority-driven (priority + total float), supports skill-based with calendar conflict resolution. Allows "smoothing" vs "leveling".
- **ALICE Technologies / Procore Workforce Planning**: AI permutation engine, but at the application level the user-facing primitives are skill-matrix, calendar, dispatch board, assignment confirmation.
- **PMBOK / PRINCE2**: explicit RACI per work-package, role-vs-named-individual two-step assignment, certification expiry as risk register entry.
- **HSE / industry norm**: certifications (Working-at-Height, Confined-Space, FLT/CPCS, NPORS, EUSR, PASMA) MUST be valid on the date of work — otherwise auto-block.

## Top-5 user needs (construction PMs / dispatchers)
1. See in 5 seconds who is free this week to do trade X.
2. Get warned BEFORE you assign someone whose ticket expires mid-job.
3. Drag from a foreman request to a dispatcher slot and have it just work.
4. Import last week's time-cards from Excel without 200 clicks.
5. Auto-resolve "weekly work plan" commitments into actual assignments.

## Top-5 industry-standard features
1. Skill-matrix scoring (Required vs Held with proficiency levels).
2. Certification expiry watchers with 60/30/14/7-day alerts.
3. Calendar/availability windows with RRULE recurrence.
4. Resource request → fulfilment workflow (foreman → dispatcher).
5. Time-card / timesheet import (CSV / Excel) with line-level mapping to assignment.

## Top-5 gaps in our code (before this pass)
1. `find_candidates` in service returns repo result with no skill-match scoring — no weighting, no ranking.
2. Certification expiry: synchronous endpoint exists but **no background job** publishing 60/30/14/7-day events.
3. **No time-card import endpoint** (CSV/Excel) — assignments must be hand-built.
4. **No bridge from `schedule_advanced` weekly-work-plan commitments to `resources.Assignment`** — they live in parallel.
5. Conflict detection covers overallocation but **does not consult `oe_resources_availability_window` blocking windows on /propose**.

## Gap table

| DoD item | Status | Action |
|---|---|---|
| Skill matching blocks invalid assignment | partial → real ranked candidate algorithm | IMPLEMENT |
| Cert-expiry alerts (60/30/14/7) | missing background loop | IMPLEMENT scheduler-style helper + events |
| Overallocation conflict | done | — |
| Schedule integration | paper-only | IMPLEMENT real subscriber wave3 → assignment auto-create |
| Time-card import | missing | IMPLEMENT CSV endpoint |
| Notifications | partial (events emit, but no subscribers for `cert_expiring`) | wire cert-expiring subscriber |
