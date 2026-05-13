# Module 5 — Equipment & Fleet — research + gap analysis

## Industry context

Reference platforms: B2W Track (fleet operations), HCSS Equipment360,
EquipmentShare T3, Tenna (telematics + ERP), Samsara (telematics), Fleetio.
Heavy-civil benchmark is HCSS; rental-house benchmark is Wynne/RentalMan.

## Top 5 user-need findings

1. **Maintenance triggers must be enforced, not advisory** — buyers expect
   that crossing a service hour-meter threshold automatically generates a
   work order before someone has a chance to schedule the unit on a job. The
   single biggest complaint about cheaper fleet tools is "the alert pops up
   but no work order is created — we miss servicing".
2. **Inspection compliance must block assignment** — heavy-civil markets
   (OSHA US, BetrSichV DE, LOLER UK lifting equipment) all require a current
   inspection certificate. A unit with an expired annual inspection must
   refuse to accept a new project assignment.
3. **Fuel-cost rollup to project finance** — when a unit is on rental to a
   project, every fuel-log entry should automatically credit the equipment
   cost-center and debit the project's equipment budget line.
4. **Telematics ingest should auto-detect anomalies** — sustained high
   coolant temp, fuel-rate spikes, sustained idle (idling > threshold), and
   geofence violations.
5. **Depreciation by method** — straight-line is too coarse for most fleets;
   declining-balance and units-of-production (per engine hour) are the
   accepted alternatives.

## Top 5 industry-standard features

1. Maintenance schedule auto-WO trigger.
2. Inspection compliance + assignment gate.
3. Fuel/parts cost rollup to project P&L.
4. Telematics ingest (engine hours, GPS, fuel level).
5. Multi-method depreciation.

## Top 5 gaps in current code

1. Telematics ingest exists but never re-evaluates maintenance schedules — a
   reading that crosses the threshold doesn't cause a WO; the user has to
   call `/generate-due-work-orders` manually.
2. Fuel log writes exist but no event is emitted, so finance never sees a
   cost. `equipment.fuel_logged` event is missing.
3. Depreciation only supports `linear`; declining-balance raises
   NotImplementedError.
4. Project rental rollup to BOQ Equipment line not wired.
5. No telematics anomaly detection (idle-time, geofence) — `raw_payload`
   stored but never inspected.

## PRD Definition-of-Done — gap analysis

| DoD item | State |
|---|---|
| Models + migrations | ✅ |
| Live map of fleet | ❌ frontend only |
| PPM trigger 50 hrs ahead | ✅ `generate_due_work_orders(lookahead_hours=50)` |
| Expired inspection blocks assignment | ✅ `is_blocked_from_assignment` |
| Auto project billing | 🟡 rental cost computed but not posted to finance |
| Fuel vs norm comparison | ❌ |
| Semantic search docs | ❌ deferred |

## Integration points actually wired vs paper-only

| Integration | Status |
|---|---|
| Asset Register | ⚪ |
| Service (cross-asset shared model) | ❌ separate models |
| Finance (depreciation / billing) | 🟡 depreciation pure-fn exists, no event |
| Resource Planning | ❌ |
| Procurement (fuel, parts) | ❌ |
| HSE (incidents with equipment) | ❌ |
| Mobile App | ❌ |
| Project Controlling | ❌ |

## Implementation plan (this pass)

1. Make `record_telemetry` re-evaluate maintenance schedules on every reading
   when `equipment.hour_meter` crosses a known `next_due_meter` threshold and
   auto-create a WO via `generate_due_work_orders(equipment_id=...)`.
2. Emit `equipment.fuel_logged` and `equipment.parts_logged` events with
   project_id resolved from the active rental, currency, and cost.
3. Add declining-balance method to `depreciation_value_at()` (a real
   implementation, not a stub).
4. Add `equipment.depreciation_snapshot` periodic job + event used by
   finance (added later or skip-if-no-finance-subscriber; we publish the
   event but do not bind to a finance subscriber here).
5. New endpoint `POST /api/v1/equipment/equipment/{id}/telematics-ingest`
   accepting raw provider payloads + a `provider` discriminator, normalising
   to `TelemetryReadingCreate` then calling `record_telemetry`.
