# PDF Takeoff Stability Findings — India Fresh-Install Report

Date: 2026-05-13
Trigger: User report from India that PDF takeoff "does not perform as expected" on a fresh install.

## Pipeline Map

Upload hits `backend/app/modules/takeoff/router.py:upload_document` (`POST /api/v1/takeoff/documents/upload/`). The router validates extension, rate-limits per user, checks `%PDF-` magic bytes, reads the bytes into memory, and hands off to `TakeoffService.upload_document` (`backend/app/modules/takeoff/service.py:upload_document`). The service performs two parser passes via helpers `_count_pdf_pages` and `_extract_pdf_pages`: pdfplumber first (text + tables), pymupdf as fallback for text only. The extracted text and per-page table data are persisted as a `TakeoffDocument` row plus a copy of the PDF on disk under `~/.openestimator/takeoff_documents/`. A separate analysis endpoint (`POST /documents/{id}/analyze/`) ships the extracted text to an LLM provider for BOQ extraction. Table-row → BOQ-element conversion happens in `TakeoffService.extract_tables`. The `services/cv-pipeline/` directory documented in CLAUDE.md does not exist on disk yet — PaddleOCR is declared in `pyproject.toml` under `[cv]` extras but no Python code invoked it until this fix.

## Issues Found & Fixes

| # | Severity | File:Line | Issue | Fix |
|---|----------|-----------|-------|-----|
| 1 | **HIGH** | `service.py:upload_document` | No upload size cap — a 2 GB scanned PDF would be loaded entirely into RAM before failing. | Added `OE_TAKEOFF_MAX_UPLOAD_MB` (default 200 MB) with `413` response and remediation message. Tested. |
| 2 | **HIGH** | `service.py:upload_document` | No password-protected-PDF detection. pdfplumber raises an opaque error, user sees "Failed to parse PDF document". Common on Indian govt + bank PDFs. | New `_is_encrypted_pdf()` scans the trailer dict in the last 8 KB for `/Encrypt`. Returns structured 400 with instructions (Acrobat / qpdf). Short-circuits in both `_extract_pdf_pages` and `_count_pdf_pages`. Tested. |
| 3 | **HIGH** | `service.py:upload_document` | 0-byte uploads (interrupted drag-drop) silently created a TakeoffDocument with status=uploaded. | Pre-flight check at both router and service layer raises 400 "Uploaded file is empty". Tested. |
| 4 | **HIGH** | `service.py:_extract_pdf_pages` | **No OCR fallback for scanned PDFs**. Most Indian construction PDFs are printed→signed→scanned photocopies with NO embedded text layer. Both pdfplumber and pymupdf return empty strings, document persists with empty `extracted_text`, AI analysis endpoint returns 400 "Document has no extracted text". User stuck. | New `_ocr_pdf_pages()` rasterises every page via pymupdf at 200 DPI (`OE_TAKEOFF_OCR_DPI`) and runs PaddleOCR with multi-language auto-pick from `OE_TAKEOFF_OCR_LANGS` (default `en,hi,ta,te,ar,ch,fr,german,es`). Wired into `_extract_pdf_pages` only when both vector parsers yield empty text (so vector PDFs stay fast). When `[cv]` extra isn't installed, logs a structured INFO line telling the operator exactly which pip install to run, and persists the document with `status="needs_ocr"` so the user sees their file in the list. Tested (stubbed PaddleOCR absence). |
| 5 | **HIGH** | `service.py:extract_tables` | Number parsing was `float(qty_str.replace(",", "."))`. Indian "1,00,000" (1 lakh) became `1.00000`, "1,500" became `1.500`, "12,345.67" became `12.345.67` and crashed silently. Lakh / crore / decimal-comma / decimal-point / mixed all broken. | New `_parse_indian_number()` recognises: Indian lakh `1,00,000` → 100000; crore `1,23,45,678` → 12345678; US thousands `1,500` → 1500; US thousands+decimal `1,500.50` → 1500.5; German `1.500,50` → 1500.5; decimal-comma `12,5` → 12.5; trailing unit `1500mm` → 1500; feet-inches `5'-6"` → 5.5; currency prefixes `Rs. 1500 only` → 1500. 27 parametrised tests. |
| 6 | **MEDIUM** | `service.py:extract_tables` | Unit field passed through raw, so "SqM", "Sq.M", "Sq M", "m2" were treated as different units and downstream BOQ rollup fragmented. | New `_normalize_unit()` aliases ~50 locale variants to canonical OE units. Tested across Indian (RMt, NOS, SFT, CFT, CuM), DACH, and imperial. Unknown units pass through lowercased to preserve user data. |
| 7 | **MEDIUM** | `service.py` (no `status="needs_ocr"`) | When OCR isn't available there was no taxonomy to tell the user "your PDF is scanned, we couldn't read it, please install the OCR extra". Now there is. | New status value persisted; frontend can render an "Install OCR" hint next to such documents. |
| 8 | **LOW** | `service.py` | No env-tunable knobs for OCR DPI / lang / size limit — operators on RAM-constrained VPS had to fork code. | `OE_TAKEOFF_MAX_UPLOAD_MB`, `OE_TAKEOFF_OCR_DPI` (clamped to 72–600), `OE_TAKEOFF_OCR_LANGS`. Tested. |

## Indian-User Edge Case Coverage (per task spec A–H)

| Case | Status |
|------|--------|
| **A. Scanned (raster-only) PDFs** | **Handled.** New `_ocr_pdf_pages` rasterises at 200 DPI (env-tunable) and runs PaddleOCR. When `[cv]` extra missing, persists `status="needs_ocr"` with operator-facing install hint. |
| **B. Multi-language OCR** | **Handled architecturally.** PaddleOCR is initialised with the first lang that successfully loads from `OE_TAKEOFF_OCR_LANGS=en,hi,ta,te,ar,ch,fr,german,es`. Note: PaddleOCR doesn't bundle every lang pack offline — the first call after install needs network to fetch weights (see Limitations below). |
| **C. Unit detection (mm/m/inch, decimal-comma, lakh/crore)** | **Handled.** `_parse_indian_number` + `_normalize_unit` cover all listed cases; tested. |
| **D. Title block / legend extraction** | **Not yet handled.** Out of scope for this stability pass — requires the cv-pipeline YOLO zone-detection model that doesn't exist yet. Tracked as architectural item 1 below. |
| **E. Large multi-page PDFs (memory cap, parallelism)** | **Partially handled.** Upload size cap added (200 MB default). OCR pages are processed serially inside one call. Parallel OCR would need a celery worker; deferred. |
| **F. Empty / corrupt / password-protected PDFs** | **Handled.** 0-byte → 400. Password → structured 400 with remediation. Magic-byte check already present. Corrupt PDF that passes magic but fails parsers → existing 400 with server-side stack. |
| **G. Font encoding (Type 3 fonts, custom encodings)** | **Handled by existing fallback chain.** pdfplumber → pymupdf is the standard mitigation. When both return empty text, OCR fallback kicks in. |
| **H. PDF/A-1a compatibility** | **Handled.** Both pdfplumber and pymupdf read PDF/A natively, OCR fallback catches the rare case where they can't. |

## Tests

- New: `backend/tests/unit/test_takeoff_pdf_stability.py` — **71 tests**, covering encryption detect, upload caps, OCR tuning env, Indian numbering (27 parametrised), unit normalisation (19 parametrised), upload gates, scanned-PDF → needs_ocr flow, end-to-end `extract_tables` with Indian-locale data.
- Existing: `backend/tests/unit/test_takeoff_error_logging.py` — **10 tests, all still pass** (no regressions).
- Broader sweep: `pytest -k "takeoff or quantit or pdf"` → **123 passed, 0 failed** (4 modules; 1 unrelated `hse_advanced` failure pre-exists and is out of scope).

## PaddleOCR Language Coverage

**Languages declared in default `OE_TAKEOFF_OCR_LANGS`:**
- `en` — English (latin) — covers global default.
- `hi` — Hindi (Devanagari) — covers Hindi, Marathi, Sanskrit, Nepali.
- `ta` — Tamil.
- `te` — Telugu.
- `ar` — Arabic — covers MENA construction PDFs.
- `ch` — Chinese (simplified).
- `fr` — French — Maghreb francophone construction.
- `german` — German — DACH market.
- `es` — Spanish — LATAM market.

**Indian languages NOT yet in default list (but PaddleOCR supports them):** Gujarati, Bengali, Kannada, Malayalam, Punjabi, Urdu (RTL). Operators can add via `OE_TAKEOFF_OCR_LANGS="...,gujarati,bengali,kannada,malayalam"`. Adding all 22 Indian official languages by default would push the model-download requirement to 2+ GB which conflicts with the lightweight-install principle. Recommended for a follow-up: a `/api/v1/takeoff/ocr/languages` endpoint listing supported codes + per-language download size, so the user can opt-in.

**Limitation:** PaddleOCR fetches model weights lazily on first use (network-blocked installs will fail at OCR-time, not import-time). For air-gapped Indian govt deployments we should ship a `make download-ocr-models` Makefile target — flagged in architectural item 3 below.

## Top 3 Architectural Improvements for v3.1 (Out of Scope Here)

1. **CV pipeline service** (`services/cv-pipeline/` per CLAUDE.md). Today the OCR fallback is in-process inside the FastAPI worker; on a real Indian 30-page scanned PDF this can block a request for 30+ s. Move to a celery worker + a YOLO11 zone-detection model that separates title block / legend / drawing area / tables so OCR runs only on the high-signal regions. Estimated 5–10× speedup and an order-of-magnitude better quality of extracted line-items.

2. **Pre-trained symbol detection model.** PaddleOCR gives us text; what Indian estimators actually need is symbol recognition (doors, windows, MEP fittings) on the drawing area. A small YOLO11 model trained on RSI/IS-962-flavoured symbol sets would unlock click-to-count workflows that the frontend `TakeoffPage` already has UI for.

3. **Offline OCR model bundle.** Many Indian govt deployments are air-gapped or behind aggressive proxies. Ship a `make download-ocr-models LANGS="en hi ta te gu ben"` Makefile target plus an env `OE_TAKEOFF_OCR_MODEL_DIR` that points PaddleOCR at the pre-fetched directory; document the offline-install path in README.

## Files Touched

- `backend/app/modules/takeoff/service.py` — refactored: added `_max_upload_bytes`, `_ocr_dpi`, `_ocr_langs`, `_is_encrypted_pdf`, `_parse_indian_number`, `_normalize_unit`, `_ocr_pdf_pages`; rewrote `_extract_pdf_pages` (OCR fallback wired); rewrote `_count_pdf_pages` (encryption short-circuit); rewrote `upload_document` (pre-flight gates + `status="needs_ocr"`); rewrote table parsing in `extract_tables`.
- `backend/app/modules/takeoff/router.py` — added 0-byte rejection in `upload_document` endpoint.
- `backend/tests/unit/test_takeoff_pdf_stability.py` — new test module, 71 tests.
