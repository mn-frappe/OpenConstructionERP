# Module 17 — Customer & Partner Portal — research + gap analysis

## Industry context

Reference platforms: Procore Customer Portal, Aconex (project information
collaboration), Newforma Project Center, PlanGrid (Autodesk Construction
Cloud) for stakeholder portal flows. Subcontractor-side portals: Textura
(Oracle), GCPay. Investor / building-user portals: Yardi Voyager, RealPage.

## Top 5 user-need findings

1. **Single portal per relationship type, not per project** — buyers want one
   portal account that sees everything they're entitled to across all
   projects. The current model satisfies this via per-resource access rules.
2. **Issue / ticket intake is by far the most-used portal feature** — every
   external user wants a "report a problem" form that does not require them
   to phone or email. The ticket should appear in the dispatcher's queue
   indistinguishable from a phone-call ticket except by its `source` tag.
3. **Change-order visibility with redacted internal commentary** — the buyer
   should see the executed change order (contractor amount, approved amount,
   time impact) but not internal markup, internal notes, or pre-decision
   workflow.
4. **Every download / view / sign event is audited** — for legal disclosure,
   buyers want a per-document timeline showing who in their organisation
   accessed it and when. Current model has the audit table; routes are
   incomplete.
5. **Notification preferences** — email vs in-portal-only toggle. By default
   most users want email for "actionable" events (approval needed, payment
   due) but only in-portal for FYI.

## Top 5 industry-standard features

1. Magic-link login (Procore uses this for guests, Aconex for two-factor).
2. Row-level access rules (per-document / per-project).
3. Buyer-side ticket intake.
4. Read-only change-order viewing with redaction.
5. Multilingual + timezone support.

## Top 5 gaps in current code

1. There is no portal-side "create ticket" endpoint — the portal user has
   nowhere to file a service request.
2. There is no `me/change-orders` endpoint — buyers cannot see executed COs.
3. Document download endpoint not present (only an "audit log a view" entry
   point on /me/document-access). Real document streaming with audit is the
   user-visible action.
4. Notification email preference not stored on PortalUser.
5. No portal-side dashboard endpoint — the buyer's home screen has no data
   source.

## PRD Definition-of-Done — gap analysis

| DoD item | State |
|---|---|
| Models + migrations | ✅ |
| Magic-link login | ✅ |
| RLS via access rules | ✅ |
| Document audit log | ✅ data model, partial endpoints |
| Notifications feed | ✅ |
| External ticket intake | ❌ |
| Change-order viewing | ❌ |
| Subcontractor portal (sub view) | ❌ |
| Email preference | ❌ |

## Integration points actually wired vs paper-only

| Integration | Status |
|---|---|
| Service (ticket intake) | ❌ |
| Subcontractor Mgmt (sub view) | ❌ |
| Documents (download) | ❌ |
| Changeorders | ❌ |
| Payment applications (sub side) | ❌ |
| Bids | ❌ |
| Notifications | ✅ |

## Implementation plan (this pass)

1. Add `notification_email_opt_in` boolean to `PortalUser` (new alembic patch
   migration) + expose on `PortalUserResponse` and `PortalUserPatch`.
2. New endpoint `POST /api/v1/portal/me/tickets` — portal user files a
   ServiceTicket. Required:
   - user must have a `service_contract` access rule (RLS gate).
   - the contract_id in the body must match a rule the user has.
   - on success, a real `ServiceTicket` row is created with `source="portal"`
     and `reported_by = "portal:{portal_user_id}"`.
3. New endpoint `GET /api/v1/portal/me/change-orders` — list executed change
   orders the user has access to (via `change_order` resource_type rules) with
   *redacted* fields (no internal `metadata`, no `submitted_by`, only the
   buyer-facing approved figures).
4. New endpoint `GET /api/v1/portal/me/tickets` — list tickets the user can
   see (via `service_contract` rules), only tickets reported_by them.
