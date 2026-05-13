# Module 1 — Service & Maintenance — research + gap analysis

## Industry context

Reference platforms: ServiceMax (asset-centric field service), Salesforce Field
Service Lightning, IFS Field Service Management, BlueFolder, Jobber. In
construction-adjacent verticals: Procore (warranty workflows), iTWO civil
(post-handover service), Sage 300 Construction (service module).

## Top 5 user-need findings

1. **Tickets must come in from many channels** — phone, email, customer portal,
   email-to-ticket parsing. Reviews of competing tools repeatedly complain that
   adding "phone-in" tickets manually breaks SLA tracking because the
   `reported_at` timestamp slips.
2. **SLA escalation only matters if it actually notifies** — most failures are
   silent (the SLA breach happens but nobody is paged). Buyers want both
   in-app and email escalation with role-based routing.
3. **Cross-link to NCR / RFI / Change order** — when an engineer finds an
   additional defect on site, they expect a one-click conversion of the
   debrief item into an NCR or a Change Order so the office side does not have
   to re-key the data.
4. **Warranty work auto-zeroing** — when a ticket is on an asset that is still
   inside its warranty window, every WO line should be created with `total=0`
   for the customer, but kept at full cost internally for cost tracking.
5. **PPM with checklist-bound completion** — a PPM (planned preventive
   maintenance) work order should not be markable as "complete" until every
   required item on the linked inspection checklist has been ticked.

## Top 5 industry-standard features

1. Procurement integration for spare parts (PR auto-creation from WO material
   line that does not exist in stock).
2. Finance integration: WO billing → invoice generation with line items.
3. Customer portal intake with full audit trail.
4. SLA matrix per priority with breach events for escalation.
5. Mobile-first signature capture on completion + offline-capable.

## Top 5 gaps in current code

1. No `source` channel on ServiceTicket — every ticket looks "manual" even
   when it came from the portal.
2. No real procurement integration — material lines on a WO are dead text.
3. No real finance integration — `service.work_order.billed` event exists but
   no subscriber actually creates an Invoice row.
4. No NCR creation path from a debrief — the PRD calls for "find similar
   issue → file NCR" as a one-click action but the code has no endpoint.
5. SLA breach is computed only on demand inside `get_contract_dashboard` and
   never publishes an event — escalation cannot be wired without scanning the
   whole table from the UI.

## PRD Definition-of-Done — gap analysis

| DoD item | State |
|---|---|
| All entities created + migrations | ✅ Done (v3010_service) |
| 5 PRD scenarios end-to-end | 🟡 partial — Scenarios 1 + 4 work; 2 (portal intake), 3 (SLA escalation event), 7 (NCR from WO) missing |
| Dispatcher can create + assign in 2 minutes | ✅ |
| Mobile signature persisted | ✅ (customer_signature column + complete endpoint) |
| Finance invoices from closed WO | ❌ event emitted but no subscriber |
| Real-time dashboard | ✅ (`/contracts/{id}/dashboard`) |
| SLA escalation works | 🟡 — SLA-due computed but no breach event / no notification subscriber |
| PPM auto-generation | 🟡 schedules exist but no cron that creates WOs from them |
| Semantic search debrief | ❌ deferred (Qdrant gate) |
| Portal intake | ❌ no real endpoint that creates a ServiceTicket from a portal session |

## Integration points actually wired vs paper-only

| Integration | Status |
|---|---|
| Contacts (customer_id) | ✅ FK in place |
| Asset Register | ⚪ separate ServiceAsset; not promoted to/from generic Asset |
| Procurement (PR auto-create) | ❌ paper-only |
| Finance (invoice from WO) | 🟡 event emitted only |
| Subcontractor Mgmt (assign sub engineer) | ❌ technician_id is plain string, not linked to Subcontractor |
| CRM Pipeline | ❌ |
| Customer Portal intake | ❌ |
| HSE permit-to-work | ❌ |
| Quality NCR | ❌ |

## Implementation plan (this pass)

1. Add `source` enum to ServiceTicket (manual / portal / email / api / auto_ppm).
2. Add `POST /api/v1/service/portal-tickets/` portal-facing endpoint with
   real RLS gate via PortalAccessRule(`service_contract`).
3. Add `SLABreachScan` service method + `service.sla.breached` event +
   notification subscriber.
4. Add `POST /api/v1/service/work-orders/{id}/file-ncr` that creates a real
   NCR row linked back to the WO via metadata.
5. Add `service.work_order.billed → finance` subscriber that creates a real
   Invoice with line items mirroring the WO items.
6. Add `service.work_order.material_requested → procurement` subscriber that
   creates a real PurchaseOrder draft when a WO item has
   `metadata.procurement_required=true`.
