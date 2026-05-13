# Schedule Advanced — Research

## Industry references
- **Last Planner System® (Glenn Ballard, LCI 2000)** — four-tier planning hierarchy: Master → Phase (pull) → Look-ahead (6 wks, with constraints) → Weekly Work Plan (commitments) → PPC (Promises Completed / Promises Planned) → RNC (Reasons for Non-Completion) Pareto.
- **Lean Construction Institute (LCI) Reliable Promising Guide** — definition of "committed" (the foreman publicly stated yes on the floor) vs "planned" (in the system but not promised).
- **AACE International RP 29R-03** — Forensic Schedule Analysis (TIA, time-impact-analysis method).
- **AACE 17R-97 / 18R-97** — Cost & schedule control, EVM (Earned Value Management) S-curves.
- **Primavera P6** — CPM forward+backward pass producing total/free float per activity, critical-path highlighting.
- **MS Project & Asta Powerproject** — resource-leveling heuristics + CPM.
- **PMBOK 7th ed.** — EVM core formulae: SPI = EV/PV, CPI = EV/AC, EAC = AC + (BAC-EV)/CPI.
- **DCMA 14-point assessment** — schedule quality checks (logic, leads, lags, relationship types, hard constraints, etc.).

## Top 5 user-needs
1. **Real PPC computed per weekly-work-plan** with promises completed / promises planned (already exists — keep).
2. **RNC Pareto chart** with canonical category enum (manpower/material/equipment/info/weather/predecessor/changes/quality/other) — already exists.
3. **Constraint analyzer** — per look-ahead activity, list all open constraints and auto-flag "ready" vs "not-ready" before WWP planning meeting.
4. **CPM forward+backward pass** giving each activity ES/EF/LS/LF/total-float/free-float + critical-path indicator (total_float ≤ 0).
5. **Earned-Value SPI/CPI** per phase plus cumulative S-curves.
6. (Plus) **Time-Impact Analysis (TIA) endpoint** — given delay event days+activity, recompute project completion date through the CPM.

## Top 5 industry features
1. PPC + RNC pareto (LPS core).
2. CPM forward+backward with float.
3. Earned-value SPI/CPI.
4. Constraint analyzer (ready/not-ready).
5. Time-impact analysis for EoT claims.

## Top 5 gaps in current code (delta from PRD §9)
1. **No CPM engine** — `compute_baseline_delta` shows variance but no critical-path computation. Need pure helper `cpm_pass(activities, deps)` returning ES/EF/LS/LF/total_float per activity.
2. **No constraint ready-flag aggregator** — `Constraint` rows exist but no endpoint that returns per-task `is_ready=true|false` based on open constraints.
3. **No EVM/SPI/CPI helper** — no field for actual_cost or budget on activities; no aggregator. Need pure helper `compute_evm(activities, today)` returning PV/EV/AC + SPI/CPI per phase.
4. **No TIA endpoint** — `compute_baseline_delta` is a one-shot; we need an endpoint that takes a delay event and returns updated completion date.
5. **No RNC chart export** — `rnc_pareto_for_project` returns dict but no sorted-Pareto (desc-by-count) variant + cumulative-percent column the UI needs.
