# Fresh-install stability findings (2026-05-13)

Real-world install test executed in a clean isolated venv on Windows 11 + Python 3.13.9.
Goal: reproduce the India-user complaint that DWG and PDF conversion underperform on a fresh install.

## Test harness
- venv: `C:\Users\Artem Boiko\AppData\Local\Temp\oce-cleantest-venv`
- DB:   `C:\Users\Artem Boiko\AppData\Local\Temp\oce-cleantest.db` (SQLite)
- Port: 8001 (8000 in use by dev backend, untouched)
- Install: `pip install -e backend/` (no extras)

## Phase 1 — install

- **PASS**: `pip install -e .` completed with exit code 0. Wheel built and installed cleanly.
- Version detected: `3.0.0`
- 60 transitive deps resolved, no conflict warnings.
- Wall time: ~6 min on Windows (cold pip cache, mostly numpy/pandas/duckdb compile).

## Phase 2 — alembic upgrade head

- **FAIL** (but pre-existing, by design): `alembic upgrade head` on a freshly-created empty DB
  exits 1 with `OperationalError: no such table: main.oe_users_user`.
- Root cause: the base table `oe_users_user` is **not created by any alembic migration**.
  Per `main.py:1611-1619` it is created by SQLAlchemy `Base.metadata.create_all` on first
  app boot. Migration `v2918_risk_owner_user_id` then tries to add a FK to it.
- Status: **NOT a regression** — the documented workflow is "boot the app first, alembic
  runs as a fast-forward on subsequent boots." But it is a footgun for the universal
  Postgres-style workflow. **Deferred** — fixing would require moving the user-table DDL
  into a migration (large blast radius, touches every fresh-install path).

## Phase 3 — first boot

- **PASS**: backend bound on `:8001`. Boot time ≈ 120 s (88 modules + 30 catalogues +
  3 demo projects + vector backfill). `GET /api/health` returns 200 with `database: ok`,
  `version: 3.0.0`, `modules_loaded: 88`.
- No tracebacks in the boot log.
- The Qdrant + sentence-transformers warnings are expected (optional extras).

## Phase 4 — auth

- `POST /api/v1/users/auth/register/` → 201, first user auto-promoted to `admin` (bootstrap path).
- `POST /api/v1/users/auth/login/` → 200, JWT issued.

## Phase 5 — DWG / DXF takeoff

### Bug 1 (CRITICAL) — every layer reported `visible: false` after DXF upload

**Reproduction**: upload any DXF (including the existing 587-byte R14 fixture in
`backend/data/dwg_uploads/*.dxf`, or a freshly-built file via `ezdxf.new('R2010')`).
The response payload contains the entities, but `latest_version.layers[*].visible`
is `false` for every layer. The frontend renders an empty canvas because layer-toggle
state derives from this flag.

**Root cause**: `backend/app/modules/dwg_takeoff/dxf_processor.py:194-201`.
ezdxf 1.4.x changed `Layer.is_off` / `Layer.is_frozen` from properties to *methods*.
`bool(layer.is_off)` of a bound method always returns `True`. Verified directly:

```python
>>> doc = ezdxf.readfile(...)
>>> layer = doc.layers.get('0')
>>> layer.is_off
<bound method Layer.is_off of LAYER(#1)>
>>> bool(layer.is_off)
True              # ← BUG. Should be False unless layer is actually off.
>>> layer.is_off()
False             # ← correct.
```

**Fix shipped** (in this commit-set):
- File: `backend/app/modules/dwg_takeoff/dxf_processor.py:178-220`
- Detect callable vs attribute and dispatch: `raw_off() if callable(raw_off) else raw_off`.
- Works on both legacy property API and 1.4+ method API.

**Verification**: re-uploaded the Hindi Devanagari DXF and the R14 fixture; layers
now correctly report `visible: true`.

**Blast radius**: minimal — single function, two boolean toggles. No downstream
schema changes. All existing tests pass.

### Bug 2 (minor) — extents calc never includes TEXT/MTEXT/INSERT anchors

**Reproduction**: an all-text DXF (no LINE/CIRCLE/POLYLINE) produces the fallback
extents `(0,0)-(1000,1000)` because the extents loop looks for the key
`insertion_point`, but the serializer writes the anchor under `insert`.

**Root cause**: `dxf_processor.py:265-266` checks `gd["insertion_point"]` but
`_serialize_entity` stores it as `gd["insert"]` (lines 109/118/125 of the same file).

**Fix shipped**: accept both keys; fall back to `insertion_point` for backwards
compat with any cached data. File: `dxf_processor.py:263-269`.

## Phase 6 — PDF takeoff

- `POST /api/v1/takeoff/documents/upload/` accepts valid PDFs (PDF-1.2 through 1.7),
  returns 201 with parsed page count.
- Magic-byte check correctly rejects non-PDF blobs (400 "missing %PDF- header").
- Corrupt PDF (good header, bad body) → 400 "Failed to parse PDF document".

### India-specific PDF cases tested

| Case | Result |
|------|--------|
| PDF 1.2 (old encoding) | **PASS** — 201 |
| PDF with valid header but garbage body | **PASS** (rejected, 400) |
| Plain ASCII PDF | **PASS** — 201 |

Not exercised (no fixture available, would need a real Indian-language scanned PDF):
Devanagari/Tamil/Telugu OCR, scanned-photocopy PDFs (no vector layer), mm-vs-inches
imperial mixing. These rely on `paddleocr` from the `[cv]` extra which is **not
installed by default** — and that's the correct behaviour because the CV stack is
~800 MB. Behaviour without the extra: upload + page count works, OCR endpoints
degrade gracefully (separate code path).

## India-specific DWG cases tested

| Case | Result |
|------|--------|
| AutoCAD R14 (AC1014, 1997 format) | **PASS** — parsed, ready, 5 entities |
| DXF with Devanagari layer names (`दीवारें`, `दरवाजे`) | **PASS** — UTF-8 round-trip OK |
| DXF with Devanagari TEXT body (`कमरा १`) | **PASS** — parsed, layer counts correct |
| INSUNITS=4 (mm, India default) | **PASS** — units field reports `mm` |
| Layer visibility (every-layer-invisible bug) | **FIXED** (Bug 1 above) |
| UTM Zone 43N coordinate system | Not exercised — coordinate-system reprojection is not implemented in the ezdxf path. DXF has no native CRS field; would need a `.prj` sidecar or user override. **Deferred** — non-blocking for an estimation workflow where world-coords are unusual. |

## Files modified

| File | Lines | Reason |
|------|-------|--------|
| `backend/app/modules/dwg_takeoff/dxf_processor.py` | 178-220 | Bug 1 — ezdxf 1.4 callable methods |
| `backend/app/modules/dwg_takeoff/dxf_processor.py` | 263-269 | Bug 2 — `insert`/`insertion_point` key mismatch |
| `.claude/research/install_stability_findings.md` | new | This report |

## Items not fixed (deferred, with rationale)

1. **alembic upgrade head on fresh empty DB** — pre-existing design choice. Fixing
   would require moving `oe_users_user` DDL into a migration (large blast radius).
   Workaround: boot the app once (auto-creates tables via SQLAlchemy), then alembic.

2. **New 18-module wave tables on first boot** — `app/main.py:1626-1678` imports
   model modules before `Base.metadata.create_all`, but the wave-1+ modules
   (service, subcontractors, equipment, portal, resources, contracts, crm, carbon,
   property_dev, bid_management, variations, schedule_advanced, hse_advanced,
   daily_diary, qms, supplier_catalogs, bi_dashboards) are missing from the
   import block. They DO get imported by the module loader after `create_all`,
   so their tables only appear on the **second** boot. Alembic migrations
   v3010-v3026 cover them, but the first-boot user who hasn't run alembic will
   see "no such table" errors when those modules' endpoints are first hit.
   **Recommended fix**: append the 17 missing model imports to the create_all
   preamble. Not landed in this pass to keep blast radius minimal — module
   models pull in service / repository chains that may have side effects.

3. **DWG → Excel pipeline (DDC DwgExporter)** — only exercised by the `.dwg`
   path, which requires an installed converter binary (`DwgExporter.exe`).
   Not present on this test machine; tested-once via offline-readiness probe
   (returns `ready: false` with a clear "install dwg2data" message). Behaviour
   is correct: graceful degradation, not a silent failure.
