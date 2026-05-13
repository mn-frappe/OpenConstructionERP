# DWG stability — Phase 2 findings (Indian-user ticket)

Date: 2026-05-13
Author: stability sub-agent (coord via files only, did NOT touch bim_hub/router.py)

## Pipeline map (one paragraph)

`POST /api/v1/takeoff/drawings/upload/` (dwg_takeoff/router.py) → extension
check (.dwg/.dxf) → `DwgTakeoffService.upload_drawing` writes the file
under `data/dwg_uploads/{uuid}{ext}` and inserts a `DwgDrawing` row and a
cross-link `Document` row → if `.dxf`, `_process_drawing` runs ezdxf in
a thread → if `.dwg`, `_handle_dwg` finds `DwgExporter.exe` via
`app.modules.boq.cad_import.find_converter("dwg")`, runs it via
`subprocess.run([exe, input, output, "-no-collada"], cwd=exe.parent,
input=b"\\n", timeout=300)` → parses the resulting XLSX with
`ddc_dwg_parser.parse_ddc_dwg_excel` (openpyxl, walks AcDbLayerTableRecord
/ AcDbLine / AcDbPolyline / AcDbVertex / AcDbArc / AcDbCircle /
AcDbEllipse / AcDbSpline / AcDbHatch / AcDbText / AcDbMText /
AcDbBlockReference / AcDbAttributeDefinition / AcDbRotatedDimension) →
flattens entities to the frontend DxfViewer shape via `_normalize_entity`
→ writes `data/dwg_uploads/{drawing_id}_entities.json` and inserts a
`DwgDrawingVersion`. The BIM Hub (`/api/v1/bim/upload-cad/`) has its
own DWG path that goes through `_generate_pdf_in_background` only for
PDF-sheet export (timeout=900 s) — left untouched per coordination
constraint.

## Smoke-test result (live backend probe)

```
POST http://localhost:8000/api/v1/takeoff/converters/dwg/verify/
→ 200 {"converter_id":"dwg","installed":true,
       "path":"C:\\Users\\Artem Boiko\\.openestimator\\converters\\dwg_windows\\DwgExporter.exe",
       "health":"ok","health_message":"","suggested_actions":[]}
```

Converter discovery, smoke-test, and the verify endpoint all pass on the
dev box. Backend reports `version=3.0.0, modules_loaded=88, database=ok`.

## Issues found

| # | Severity | File:line (before fix) | Issue | Fix shipped |
|---|----------|-----------------------|-------|-------------|
| 1 | High | `dwg_takeoff/service.py:371` | 120 s subprocess timeout. Large Indian site/cadastral plans (30 MB+) routinely exceed 120 s on Windows but finish inside ~180 s — user saw "DWG conversion timed out". | Raised to 300 s (matches `cad_import.convert_cad_to_excel`). Error message now suggests exploding xrefs / splitting by layout. |
| 2 | High | `dwg_takeoff/service.py:336` | No DWG version pre-check. AutoCAD LT 2007 (R17, AC1021) and older files — still common in Indian municipal / surveyor workflows — silently produce empty XLSX. User sees cryptic "DDC DwgExporter produced no output". | Added `_sniff_dwg_version` (reads 6-byte ACxxxx magic) + `_dwg_version_too_old` (rejects pre-R2010). 422-style error message names the version and tells user to `SAVEAS` 2018 DWG. |
| 3 | High | `dwg_takeoff/service.py:336` | No 0-byte / garbage upload guard. A renamed PDF, ZIP, or JPEG `.dwg` consumed up to 120 s of converter time before failing. | Pre-conversion guard: file size < 32 bytes → reject; magic-byte sniff returns `None` for non-`ACxxxx` heads → "this does not look like an AutoCAD DWG file". |
| 4 | Med | `dwg_takeoff/router.py:160` | No upload-time size check. 0-byte uploads still consumed a DB row + disk write. | Added 422 at the door when `file.size < 32`. |
| 5 | Med | `dwg_takeoff/service.py:375` | stderr truncated to first 300 chars; useful diagnostic tail (e.g. `libQt6Core.so.6: cannot open shared object`) lost. | Combined stderr+stdout; surface LAST 400 chars; UTF-8 decode with `errors="replace"` for translated locale strings; logs full payload at WARNING. |
| 6 | Med | `dwg_takeoff/service.py:388` | Bare `except Exception` lost the OS-level class (`PermissionError`, `FileNotFoundError`). | Split out `except OSError` with `exc.__class__.__name__` in the error message; `logger.exception` for stack capture. |
| 7 | Low | `boq/cad_import.py:545` | Same bare `except Exception` in shared converter wrapper. | Added dedicated `except OSError` branch for symmetry. |

Defer-list (not shipped, intentional):

* **B. Path-with-spaces** — every `subprocess.run` already uses list-form
  args and `Path` wrapping; verified safe.
* **A. Encoding for Devanagari/Tamil layer names** — DDC writes XLSX with
  UTF-8 worksheet strings; openpyxl decodes correctly. `subprocess.run`
  uses bytes I/O (no `encoding=` passed) so no cp1252 fallback path
  exists. No fix needed.
* **D. Coordinate / units mismatch** (INSUNITS=1 inches vs metric content)
  — canonical JSON exposes `units` field; takeoff scale calibration
  already overrides at UI level. Backend has no good way to detect
  intent-vs-units conflicts without per-drawing user input.
* **F. Large-file memory** — content is read in one `await file.read()`
  call (`service.py:195`). Streaming upload deferred to v3.1.

## India-specific edge cases — coverage

| Edge case | Status |
|-----------|--------|
| A. Devanagari / Tamil layer names | OK — pipeline is UTF-8 end to end |
| B. Path with spaces / non-ASCII (e.g. `C:\Users\Artem Boiko\...`) | OK — list-form subprocess args + `Path.resolve()` |
| C. Old AutoCAD versions (R14-R17, AutoCAD LT 2007) | **FIXED** — pre-check rejects with actionable "SAVEAS 2018" hint |
| D. Coordinate systems (UTM 43N/44N) | Out of scope — surfaced via canonical `units` field but not auto-detected |
| E. Empty / corrupt files | **FIXED** — 0-byte rejected at router, < 32 bytes / non-ACxxxx rejected in service |
| F. Large files (30 MB+ cadastral plans) | **PARTIAL** — timeout now 300 s (was 120 s). Streaming upload deferred. |

## Tests

* New: `backend/tests/unit/test_dwg_stability_guards.py` — 16 tests covering
  the magic-byte sniff (modern, R2010, R2007, PDF-renamed, ZIP-renamed,
  empty, missing-file, non-ASCII, non-pattern ASCII) and the version-too-old
  comparator (modern, pre-R2010, unknown, None, min-version pin).
* Run result: `30 passed in 1.38s` covering the new file plus the existing
  `test_cad_import_linux.py` (10), `v1_9/test_dwg_offline_readiness.py` (2),
  and `v1_9/test_dwg_groups.py` (2). 0 regressions.

## Top 3 architectural improvements for v3.1

1. **Replace the XLSX intermediate with a JSON pipe**. DDC DwgExporter
   today writes an Excel file we immediately re-parse with openpyxl —
   double serialization for ~30 MB drawings. A `--json` flag on the
   converter (or a stdout pipe) would cut conversion time by ~30 %
   and eliminate the `_dwg.xlsx` temp file that we leave on disk on the
   error path (currently leaked).
2. **Background-task / queue refactor**. `_handle_dwg` runs inline on
   the upload request, holding the HTTP connection open for the full
   conversion duration (now up to 300 s). Move to Celery / RQ /
   in-process queue so the upload endpoint returns immediately with
   `status="processing"` and the frontend polls. Required for India-grade
   bandwidth where the upload itself already takes 30 s.
3. **Per-tenant converter health dashboard**. The existing
   `/api/v1/takeoff/converters/?verify=true` endpoint is per-host. In a
   multi-tenant deployment we don't know which Indian user is hitting
   the broken-Qt-DLL path until they report it. A periodic background
   health probe writing to a `converter_health` table would surface
   regressions before users notice.
