# Module 4 — Subcontractor Management — research + gap analysis

## Industry context

Reference platforms: SmartBid (prequal + bidding), SAP Fieldglass (services
procurement), GCPay & Textura by Oracle (payment applications + lien waivers),
Procore Subcontractor Default Insurance + Procore Pay, Compass / GCBuilt for
prequal scoring. UK/Middle East: Aconex bidding, Conject. US lien-waiver flow
is the dominant pattern; UK retention-bond flow is its closest cousin.

## Top 5 user-need findings

1. **Compliance documents must drive payment release** — buyers want a hard
   block on payment app submission until W-9 / VAT registration / insurance
   COI / safety stats are present and not expired. Reviews of competing tools
   consistently flag "subs got paid even though insurance had lapsed" as the
   #1 risk event.
2. **Lien waivers are not optional in the US market** — a payment app must be
   accompanied by a conditional waiver on submit and an unconditional waiver
   after payment; the system should track both.
3. **Performance ratings must be event-driven** — manual rating entry never
   gets done. NCRs, HSE incidents and schedule slips should automatically
   degrade the score; positive ratings should automatically rise when
   milestones are achieved.
4. **Schedule of Values (SOV) view per agreement** — the user wants to see
   planned-value vs claimed vs certified vs paid per work package, in a
   single grid with running totals.
5. **Retention release events** are usually two-tier (50% at practical
   completion, 50% at end-of-defects). The release schedule should be
   configurable per agreement.

## Top 5 industry-standard features

1. Multi-criteria prequalification scoring with configurable weights
   (financial / safety / quality / experience).
2. Compliance-document expiry registry with 60/30/7-day reminders + payment
   block.
3. Performance scorecard fed by NCR + HSE + schedule events.
4. Lien-waiver registry (conditional/unconditional × progress/final).
5. Retention ledger with configurable release events.

## Top 5 gaps in current code

1. No real ingestion of NCR / HSE events into the rating engine — `compute_rating`
   is a pure function but nothing publishes to it.
2. `next_payment_blocked` only checks `insurance` and `license` — no W-9 /
   VAT / safety stats. Hard-coded `REQUIRED_CERT_TYPES_FOR_PAYMENT` tuple.
3. No lien-waiver tracking model at all.
4. No SOV summary endpoint (each work package's claimed/certified/approved
   roll-up).
5. No tax-ID / VAT validity check at all — even basic format validation by
   country is missing.

## PRD Definition-of-Done — gap analysis

| DoD item | State |
|---|---|
| All entities + migrations | ✅ done |
| Full prequalification cycle | ✅ submit / approve / reject endpoints exist |
| Document expiry auto-block | 🟡 partial — only 2 cert types hard-coded |
| OCR ≥90% accuracy on certs | ❌ stub (`ocr_extract_certificate_hook`) |
| Payment app workflow | ✅ submit→foreman→finance→paid |
| Retention accrual + release | ✅ ledger + accrue/release helpers |
| Rating engine auto-update | ❌ compute_rating exists but no event subscribers feed it |
| Subcontractor portal | 🟡 portal module exists but no real subcontractor view-models |

## Integration points actually wired vs paper-only

| Integration | Status |
|---|---|
| Contacts (contact_id FK) | ✅ |
| Procurement | ⚪ no path |
| Finance (payment.paid → AP) | 🟡 event emitted, no subscriber yet |
| Bid Management | ⚪ |
| Quality NCR → rating | ❌ paper-only |
| HSE incidents → rating | ❌ paper-only |
| Insurance / Bond | 🟡 Certificate table exists; no bond-specific flow |
| Customer Portal (sub view) | ❌ |
| Resource Planning | ❌ |

## Implementation plan (this pass)

1. New endpoint `POST /api/v1/subcontractors/agreements/{id}/sov` returning the
   Schedule-of-Values rollup (planned / claimed / certified / approved /
   remaining per work-package).
2. New `LienWaiver` model + `POST /payment-applications/{id}/lien-waiver`
   endpoint. Submitting a payment-app requires either a conditional waiver
   OR explicit waiver-not-required flag.
3. Subscribers in subcontractors module:
   - `qms.ncr.created` (or `ncr.created`) → if the NCR's metadata names a
     subcontractor_id, +1 to that sub's current-month ncr_count and recompute
     the rating.
   - `safety.incident.created` (or `hse.incident.created`) → same flow for hse.
4. Tax-id / VAT validator helper (`validate_tax_id(country, value)`)
   returning a structured `TaxIdValidation` (format-valid yes/no, country,
   reason). Used in both create + update endpoints and surfaced on dashboard.
5. Extend `REQUIRED_CERT_TYPES_FOR_PAYMENT` to be configurable via the
   subcontractor's metadata (tenant-level default + per-agreement override).
