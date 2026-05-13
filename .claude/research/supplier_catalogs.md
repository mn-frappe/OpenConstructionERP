# Research: supplier_catalogs (Module 11)

## Competitive landscape

| Platform | Key strengths in supplier mgmt |
|----------|--------------------------------|
| SAP Ariba | UNSPSC commodity codes, supplier qualification workflows, PEPPOL e-invoicing, configurable 3-way match tolerances, supplier scorecard with weighted criteria |
| Coupa | Procure-to-Pay workflow, line-level + header-level tolerances, exception routing, supplier risk scoring |
| Jaggaer | Spend analytics, supplier 360 view, KYC document expiry tracking |
| GEP SMART | AI-driven category management, supplier performance KPIs (OTD, quality, ESG) |
| Procore Supplier Mgmt | Construction-specific: warehouse staging, batch tracking, low-stock alerts |

## Top 5 user needs

1. **Standardised commodity classification** — pick UNSPSC code on each catalog item + vendor for spend analytics and EU public-procurement reporting.
2. **3-way match that doesn't break on tiny variances** — configurable per-tenant tolerance bands (price %, qty %, period days). 80% of invoices should auto-clear.
3. **Receive PEPPOL UBL 2.1 invoices** from EU suppliers electronically — no manual data entry.
4. **Supplier scorecards** — composite score (on-time delivery, quality, price, ESG) recomputed monthly so PMs can pick the best supplier.
5. **KYC compliance** — track W-9 / VAT / GST / TRN per region with expiry alerts; block PO creation if KYC lapsed.

## Top 5 industry features missing in current code

1. **UNSPSC/eClass codes** — catalog items have ``classification_ref`` (free text) but no validated commodity-code list, no dropdown, no spend-by-code reporting.
2. **Configurable tolerance bands** — match_invoice uses a single ``tolerance_pct`` parameter; production needs separate price-pct / qty-pct / period-days, configurable per tenant + per vendor.
3. **PEPPOL ingest** — no UBL XML parser; invoices today are created via the JSON API only.
4. **Supplier scorecard** — has a 1-5 ``rating`` field, but no multi-criteria weighted formula pulled from GR/NCR/audit history.
5. **KYC document tracking** — ``tax_id`` is a free string; no region-aware doc type (W-9 / VAT / GST / TRN), no expiry, no alert.

## Top 5 gaps in code (after gap-analysis)

1. ``supplier_catalogs_commodity_code`` table missing (UNSPSC + eClass seed).
2. ``supplier_catalogs_tolerance_profile`` per-tenant config missing — only the single query-param tolerance.
3. ``supplier_catalogs_kyc_document`` table missing — vendor.tax_id is too thin.
4. ``supplier_catalogs_scorecard`` aggregate table missing — Vendor.rating is a manual int.
5. No PEPPOL ingest endpoint (``POST /invoices/peppol``).

## Plan (implemented)

- New table ``commodity_code`` seeded with top UNSPSC segments (csv ships in module).
- New table ``tolerance_profile`` with price_pct / qty_pct / period_days fields.
- New table ``kyc_document`` with region-aware ``doc_type`` enum + ``expires_on`` + nightly expiry-alert job.
- New table ``scorecard`` with weighted formula recompute method.
- New endpoint ``POST /invoices/peppol`` accepts UBL 2.1 XML → VendorInvoice row + auto-match.
- Procurement extension: ``supplier_catalogs.po.*`` events bridge to a procurement-side adaptor that mirrors the new PO into the legacy table for any dashboard still pointed at ``oe_procurement_po``.
