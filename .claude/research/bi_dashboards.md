# Research: bi_dashboards (Module 20)

## Competitive landscape

| Platform | Construction-specific BI strengths |
|----------|------------------------------------|
| Power BI for Construction (Microsoft + RIB) | CPI/SPI cards, drill-through to BOQ, role-based RLS, scheduled subscriptions (daily 8 am, weekly Mon) |
| Tableau | Construction-vertical templates, alert rules with conditions, embedded PDF export |
| Looker | LookML for portable KPI definitions, saved-filter sharing |
| Sisense | ElastiCube for portfolio rollup, write-back |
| Autodesk Construction Cloud BI | Built-in EVM (PMBOK), safety TRIR/LTIFR, daily diary KPIs |

## PMBOK Earned-Value Management formulas

Implemented per PMBOK 7:

| Metric | Formula |
|--------|---------|
| PV (BCWS) | Σ planned_value across tasks/positions at status_date |
| EV (BCWP) | Σ (% complete × budget_at_completion) |
| AC (ACWP) | Σ actual_cost from finance.Expense + procurement.PurchaseOrder |
| CV  | EV − AC |
| SV  | EV − PV |
| CPI | EV / AC |
| SPI | EV / PV |
| EAC (typical) | AC + (BAC − EV) / (CPI × SPI) — construction-typical; assumes both perf indices persist |
| EAC (CPI-only) | BAC / CPI |
| ETC | EAC − AC |
| VAC | BAC − EAC |
| TCPI | (BAC − EV) / (BAC − AC) |

## Top 5 user needs

1. **Earned-value tile pack** — CPI/SPI/EAC/VAC computed from real BOQ+Finance data with one click; not hand-calculated in Excel.
2. **Click any KPI → see underlying rows** (project list / BOQ positions / invoices); right now KPIs are opaque.
3. **Conditional alerts** — "CPI<0.95 AND project.phase==execution" stored as JSON, evaluated every N min.
4. **Scheduled PDF report** — emailed daily 8am / weekly Mon / monthly 1st-business-day to recipients; right now ``file_url`` is None.
5. **Role-based default dashboards** — CFO / PM / Site Manager / Safety Officer / CEO with sensible defaults out of the box (currently seeded but tiles are flat KPIs only).

## Top 5 industry features missing in current code

1. **Real EVM rollup KPIs** — only CPI/SPI exist; EAC, ETC, VAC, CV, SV, TCPI are missing.
2. **Conditional DSL** — alerts today use single condition+threshold; need composite (AND/OR) over multiple KPI/field criteria.
3. **PDF rendering** — ``run_report`` writes rows to JSON but ``file_url`` is always None.
4. **Saved-filter sharing** — has ``scope`` enum but no "share to user X" mechanic.
5. **Benchmark comparison** — no "compare project's CPI to portfolio median" sidebar data.

## Top 5 gaps in code (after gap-analysis)

1. EVM KPIs ``cv / sv / eac / etc / vac / tcpi`` not registered.
2. ``DrillDownResponse`` returns only breakdown + history — needs real underlying records.
3. ``AlertRule.condition`` is a single string ("below" / "above") — no composite DSL.
4. PDF builder absent — ``run_report`` does not produce a binary artifact.
5. ``shared_with_user_ids`` field absent on SavedFilter — sharing impossible.

## Plan (implemented)

- 6 new KPI formulas: ``cv``, ``sv``, ``eac``, ``etc``, ``vac``, ``tcpi``.
- KPI ``benchmark_compare`` helper + ``compute_kpi`` returns ``benchmark`` field (portfolio median).
- Real per-KPI drill-down: each KPI exposes a ``records()`` callable returning the underlying objects.
- Alert DSL: ``AlertRule.expression_json`` (composite). Evaluator walks the tree.
- PDF report builder using ``reportlab`` (already in deps via finance). ``run_report`` writes a PDF, returns S3-style URL.
- ``SavedFilter.shared_with_user_ids_json`` + ``share`` endpoint.
- Chart-export endpoint ``GET /widgets/{id}/export?format=csv|png|svg`` (CSV + SVG, PNG omitted as deps-heavy and deferred).
