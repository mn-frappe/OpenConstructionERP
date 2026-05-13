# Carbon module — research, gap analysis, plan

## Industry benchmarks
- **GHG Protocol Corporate Standard**: Scope 1 (direct fuel), Scope 2 (purchased energy: location vs market method), Scope 3 (upstream/downstream — 15 categories).
- **EN 15978**: Whole-life assessment with LCA stages **A1-A3** (product), **A4** (transport to site), **A5** (construction install), **B1-B7** (use phase), **C1-C4** (end-of-life), **D** (benefits beyond system boundary).
- **ISO 14064-1**: org-level GHG inventory. **ISO 14064-2**: project-level reductions.
- **RICS Whole Life Carbon Assessment**: UK building / infra standard.
- **EPD International / Ökobaudat / ICE / EC3**: public EPD databases. GWP-100 (100-year Global Warming Potential, kgCO2e).
- **OneClick LCA / Tally / eTool**: commercial tools — assign EPD to BOQ position, get embodied carbon.
- **TCFD / ISSB S2**: climate-related disclosure framework. Requires Scope 1/2/3 + intensity + targets + transition plan.
- **Grid factors**: IEA, DEFRA (UK), EPA eGRID (US), Umweltbundesamt (DE) — country-year emission factors for grid electricity.

## Top-5 user needs
1. See embodied carbon on every BOQ line auto-computed from material × quantity.
2. Paste an EPD ID or URL and get the GWP into the system.
3. Compare two materials side-by-side: 23% less carbon if we switch concrete grade.
4. Drop a project area (m² GFA) and get intensity kgCO2e/m².
5. Export an ISSB / TCFD report at year-end without rebuilding numbers.

## Top-5 industry-standard features
1. **EN 15978 lifecycle stages** (A1-A3 / A4 / A5 / B / C / D) per embodied entry.
2. **EPD ingestion** by identifier from Ökobaudat / ICE / EC3.
3. **BOQ position → carbon auto-calc** via material-factor lookup.
4. **Country / year grid emission factor table** for Scope 2.
5. **TCFD / ISSB-aligned report export** with required disclosure sections.

## Top-5 gaps in our code (before this pass)
1. `EmbodiedCarbonEntry.stage` exists with `a1a3` default but **no validator** that it's one of the 11 allowed stages — typos accepted.
2. `epd_database_sync_hook` is a stub returning `[]` — no real Ökobaudat / ICE ingestion.
3. **No BOQ-line subscriber** — when a BOQ position is created/updated, no carbon entry is auto-built.
4. **No country-year grid factor table** — every Scope 2 entry needs the user to look up the factor by hand.
5. Reports save totals dict but **don't render a structured TCFD / ISSB output** (sections, narrative, intensity metrics).

## Gap table

| DoD item | Status | Action |
|---|---|---|
| Embodied auto from QTO | partial (compute fn exists, no subscriber) | IMPLEMENT BOQ subscriber + endpoint |
| Alternative materials picker | done | — |
| Scope 1/2/3 split | done | — |
| ESG report generation | partial (totals only) | IMPLEMENT TCFD / ISSB structured report builder |
| EN 15978 stage enum | missing validator | ADD allowed set + validator |
| EPD ingestion | stub | IMPLEMENT identifier-based ingestion endpoint |
| Country grid factors | missing | IMPLEMENT static table + lookup endpoint |
| Intensity metrics | partial | EXTEND with per-m² GFA / NIA / per-€1M revenue |
