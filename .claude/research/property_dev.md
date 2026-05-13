# Property Development & Buyer Portal — Research

## Industry references
- **Procore Construction Financials for developers** — unit pricing, sales-stage tracking, buyer options gating against construction phase, change-order propagation to ProjectContract.
- **Yardi Voyager Affordable / Condo** — reservation→deposit→contract→handover state machine with jurisdiction-specific deposit forfeiture (UK 10% non-refundable beyond cooling-off, Spain LOE Art. 1454 retraction at 2x deposit, GCC RERA escrow lock).
- **Procorem (UK)** — snag list per unit, BSA-2022 (Building Safety Act) defect severity registry, golden-thread document trail.
- **BuilderTREND / CoConstruct** — buyer selection portal: option picker → freeze deadline → production handoff.
- **Showpad/Spinify** — sales pipeline kanban + reservation calendar.
- **PRA UK CP21/22** — buyer-protection deposit holding; SRA Property Standards on handover documentation.
- **CDM 2015 Reg 32–35** — handover dossier (H&S file) must accompany unit handover.

## Top 5 user-needs
1. **Reservation→contract pipeline that automatically forfeits deposit per jurisdiction** when buyer pulls out after cooling-off period.
2. **Per-unit snag list** with severity (cosmetic/minor/major/safety per BSA), photo evidence, target-fix dates, and warranty-period auto-extension when snag is safety class.
3. **Buyer portal** showing handover docs (warranty cert, manuals, key handover record, H&S file) + ability to submit defect claims.
4. **Development-level P&L rollup** showing sold value vs build cost vs marketing vs commissions, real-time.
5. **Sales pipeline kanban + reservation calendar** so sales agents can see leads moving across lead/reserved/contracted/completed and pending viewings.

## Top 5 industry features
1. State-machine driven sales lifecycle with deposit-forfeiture rules per jurisdiction (PRA/RERA/LOE).
2. Snag list with BSA severity categorisation + photo attachment + warranty clock.
3. Buyer portal handover document bundle.
4. 3D configurator with compatibility rule engine + freeze deadlines.
5. Cost-revenue rollup at development level with finance integration.

## Top 5 gaps in current code (delta from PRD §22)
1. **No deposit forfeiture engine** — `buyer.status` flips but no money calculation. Need `compute_deposit_forfeiture(buyer, jurisdiction, cancelled_at)` returning percentage/amount with rule citation.
2. **No P&L rollup** — `development_sales_dashboard` aggregates plot/buyer counts but no revenue-vs-cost breakdown. Need `development_pnl_rollup(dev_id)` reading from finance.
3. **No reservation calendar / kanban data endpoint** — frontend would have to compute itself.
4. **No handover-doc bundle** — `Handover` model has only `customer_signature_ref`. Need handover-document list per plot for buyer portal.
5. **No defect-submission cross-link** — `WarrantyClaim.linked_service_ticket_id` is unbound; need event-publish to spawn Service ticket on raise.
