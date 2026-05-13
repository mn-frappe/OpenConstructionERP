# CRM module — research, gap analysis, plan

## Industry benchmarks
- **Salesforce Construction Cloud**: account hierarchy (Parent / Subsidiary), opportunity scoring fields (BANT-derived). Pipeline kanban + forecast.
- **HubSpot for AEC**: lead scoring on engagement + firmographics, pipeline stages mapped to RFQ → proposal → shortlist → award.
- **Pipedrive for contractors**: lightweight, sales pipeline by deal value, activity-driven.
- **BANT framework**: Budget / Authority / Need / Timeline — sales qualification standard.
- **Construction BD pipeline**: Lead → Qualified → RFQ received → Proposal submitted → Shortlisted → Award → Lost (with reason).

## Top-5 user needs
1. See pipeline kanban with deal values and stage probabilities.
2. Score opportunities so the BD team focuses on the right ones.
3. Walk the account-hierarchy tree (Owner → GC → Sub) to spot upsell.
4. When opportunity is won, automatically open a bid package or project.
5. See revenue forecast for next quarter weighted by stage probability.

## Top-5 industry-standard features
1. **BANT scoring** (Budget / Authority / Need / Timeline weighted average).
2. **Account hierarchy** with parent / child + visualisation.
3. **Bid-management integration**: opportunity won → bid package + invitee list.
4. **Activity timeline** (calls / meetings / emails / notes / tasks).
5. **Weighted pipeline forecast** by stage probability per quarter.

## Top-5 gaps in our code (before this pass)
1. No `parent_account_id` on Account → no hierarchy.
2. No BANT-style scoring fields or method.
3. `crm.opportunity.won` event fires but **no subscriber wires it to `bid_management`**.
4. Activity timeline retrieves activities but not in unified chronological feed (per-entity only).
5. Forecast computed on call but no period-weighted breakdown by stage.

## Gap table

| DoD item | Status | Action |
|---|---|---|
| Pipeline kanban + weighted | done (compute_pipeline_metrics) | — |
| Forecast | done (compute_forecast) | — |
| Lead → opportunity conversion | done | — |
| Activity log | done (per-entity) | EXTEND with unified timeline endpoint |
| BANT scoring | missing | IMPLEMENT pure scoring + endpoint |
| Account hierarchy | missing | ADD parent_account_id + tree endpoint |
| Won → bid package | event emitted, no subscriber | IMPLEMENT subscriber |
| Weighted by stage forecast | partial | extend with stage-grouped output |
| Won → project | covered via project subscriber elsewhere | leave |
