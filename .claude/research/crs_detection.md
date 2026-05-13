# CRS Auto-Detection — Research Notes

**Date**: 2026-05-13
**Owner**: CAD/BIM ingest pipeline
**Goal**: Recognise the projected/geographic coordinate system of an uploaded
CAD/BIM file *without* asking the user, then surface a high-confidence EPSG
guess (with alternates) through the canonical JSON, ORM, and viewer.

This note is the design-time research that backs `crs_detector.py`. Numbers
below are taken from EPSG.io and the IOGP Geomatics Guidance Note 7-2
(November 2024), cross-checked against the EPSG database bundled with
`pyproj` 3.7.x (PROJ 9.4 / EPSG v11.011).

---

## 1. EPSG codes — what we cover

EPSG is the global registry of horizontal CRS definitions (currently
~6 400 entries). We do **not** ship the full registry — that's already in
`pyproj`. We ship a *region heuristic table* with the high-frequency
construction CRSs and let `pyproj` resolve names / units on demand.

### 1.1 Coverage matrix (Stage 2 detector)

| Region                | EPSG range                              | Heuristic bbox (m)                                             |
|-----------------------|-----------------------------------------|----------------------------------------------------------------|
| **India** UTM         | 32642 (43N), 32643 (43N), 32644 (44N), 32645 (45N), 32646 (46N) | x ∈ [166 000, 833 000], y ∈ [800 000, 3 800 000]              |
| **Germany** UTM       | 25832 (32N), 25833 (33N)                | x ∈ [166 000, 833 000], y ∈ [5 200 000, 6 100 000]            |
| **Germany** Gauss-Krüger | 31466–31469 (GK 2–5)                | x ∈ [2 500 000, 5 700 000], y ∈ [5 200 000, 6 100 000]        |
| **Switzerland**        | 2056 (LV95)                            | x ∈ [2 480 000, 2 840 000], y ∈ [1 070 000, 1 300 000]        |
| **Austria**            | 31256 (MGI/M28), 31257 (M31), 31258 (M34) | x ∈ [-200 000, 400 000], y ∈ [5 100 000, 5 500 000]        |
| **UK / Ireland**       | 27700 (OSGB36 BNG), 29903 (TM75 IG)    | x ∈ [0, 700 000], y ∈ [0, 1 300 000]                          |
| **France**             | 2154 (RGF93 Lambert-93)                | x ∈ [100 000, 1 200 000], y ∈ [6 000 000, 7 200 000]          |
| **Netherlands**        | 28992 (RD New)                         | x ∈ [-7 000, 300 000], y ∈ [289 000, 629 000]                 |
| **US State Plane**     | 2225 (CA Z1) … 2272 (NY LI) (top-10)   | per zone — feet *or* meters; very wide range, lower confidence |
| **US UTM**             | 32610–32619 (10N–19N WGS 84)           | x ∈ [166 000, 833 000], y ∈ [3 200 000, 5 500 000]            |
| **UAE / KSA**          | 32638 (38N), 32639 (39N), 32640 (40N)  | x ∈ [166 000, 833 000], y ∈ [2 200 000, 3 900 000]            |
| **Japan**              | 6669–6687 (JGD2011 zones I–XIX)        | x ∈ [-400 000, 400 000], y ∈ [-300 000, 600 000]              |
| **Brazil** SIRGAS UTM  | 31978 (18S) … 31985 (25S)              | x ∈ [166 000, 833 000], y ∈ [6 100 000, 10 000 000]           |
| **China** CGCS2000 GK  | 4513 (zone 13) … 4523 (zone 23)        | x ∈ [13 500 000, 23 500 000], y ∈ [1 800 000, 6 000 000]      |
| **Palestine / Levant** | 28191 (Pal 1923 Palestine Grid)        | x ∈ [100 000, 250 000], y ∈ [50 000, 350 000]                 |
| **WGS 84 geographic**  | 4326                                   | x ∈ [-180, 180], y ∈ [-90, 90]                                 |
| **Project-local**      | none (EPSG = unknown)                  | \|x\| < 10 000, \|y\| < 10 000, bbox area < 1 km²              |

That's **17 region groups** spanning **~80 EPSG codes**. The detector
ranks them by *bbox fit* (penalty = signed distance outside the region
window, scaled to bbox size) and returns the best match + top-3
alternates.

### 1.2 What "confidence" means

```
confidence = max(0, 1 - penalty)

penalty =
  0.0     if bbox is fully inside the heuristic window
  0.05    if bbox is within 5% of the window perimeter
  0.30    if bbox center is inside the window but corners spill
  0.70    if bbox center is outside but at least one corner overlaps
  1.00    if no overlap at all (this region is excluded)
```

So a tight UTM-43N bbox (e.g. 200 000–500 000 × 2 500 000–3 000 000)
returns `confidence ≈ 0.95`. A loose project-wide bbox that overlaps
two UTM zones returns `≈ 0.55` for each — and the alternates list shows
both.

---

## 2. DWG / DXF header fields

`ezdxf` exposes the header through `doc.header`. The fields we read:

| Field             | What it tells us                                                                    |
|-------------------|-------------------------------------------------------------------------------------|
| `$EXTMIN/$EXTMAX` | World-coord bounding box of model space — feeds the bbox heuristic                  |
| `$INSUNITS`       | Drawing units (0 = unitless, 1 = inches, 2 = feet, 4 = mm, 5 = cm, 6 = m, 7 = km)   |
| `$MEASUREMENT`    | 0 = imperial / 1 = metric — fallback when `$INSUNITS == 0`                          |
| `$LUNITS`         | Linear unit format (1 = scientific, 2 = decimal, 3 = engineering, 4 = architectural, 5 = fractional) — hint for "feet" if architectural |
| `$DWGCODEPAGE`    | Locale code page — a weak hint for region (e.g. "ANSI_1251" → ex-USSR)              |
| `$AUNITS`         | Angle units                                                                         |

AutoCAD also stores an *optional* dictionary entry
`ACAD_PROJECTION_GEODATA` (introduced in AutoCAD 2010, Civil 3D).
When present it embeds an XML-ish blob with **the actual coordinate
system definition** including an EPSG-equivalent name. We parse it
opportunistically — if found it bypasses the bbox heuristic and
returns `detection_method = "dwg_geodata"` with `confidence = 1.0`.

The bulk of construction DWGs in the wild do **not** carry that
dictionary (estimated <5%), so the bbox heuristic remains the primary
path.

---

## 3. IFC header — IfcProjectedCRS / IfcMapConversion

The IFC4 schema introduced `IfcProjectedCRS` (was missing in IFC2x3).
The header line we care about:

```
IFCPROJECTEDCRS('EPSG:25832','ETRS89 / UTM zone 32N','EPSG:6258','Transverse Mercator','UTM zone 32N',$,#42);
                ^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^
                Name         Description             GeodeticDatum MapProjection       MapZone
```

We parse this with a regex (no IfcOpenShell):

* `Name` is canonical "EPSG:25832" in the well-authored case.
* If absent we fall back to grepping for `EPSG:\d+` anywhere in the
  header section.
* `Description` is often human-readable ("ETRS89 / UTM zone 32N") —
  used as the display name when EPSG is parseable.

`IfcMapConversion` (also IFC4) holds the
`Eastings/Northings/OrthogonalHeight/XAxisAbscissa/XAxisOrdinate/Scale`
transform from model space → map space. We surface the eastings/
northings as the **bbox origin** for the heuristic, in case
`IfcProjectedCRS` is missing.

IFC2x3 files (still ~40% of the wild population — Allplan / older
Tekla) carry **no CRS** at all. They go straight to the bbox heuristic.

---

## 4. Region heuristic — design

`detect_from_bbox` is the universal fallback. It:

1. Discards obvious degenerate inputs (zero-area bbox, NaN coords).
2. Iterates the region table, scoring each region (see §1.2).
3. Picks the highest scorer; emits up to 3 alternates by next score.
4. Reads the EPSG name via `pyproj.CRS.from_epsg(code).name` so the
   string is always live (not a stale copy).

**WGS 84 lat-lon** is a special case: if bbox falls inside
`[-180, 180] × [-90, 90]` AND units are not declared as "m"/"ft", we
return EPSG:4326. (A 200 m × 200 m project in metres also fits that
window — units check disambiguates.)

**Project-local** is the catch-all: when |x|, |y| < 10 000 AND the bbox
area is under 1 km², we return `epsg=None`, `name="Project-local
(unknown CRS)"`, `confidence=0.4`. The frontend asks the user to pick.

---

## 5. Pipeline integration points

| Step                  | File                                                         | What we add                                                            |
|-----------------------|--------------------------------------------------------------|------------------------------------------------------------------------|
| DXF parse             | `app/modules/dwg_takeoff/dxf_processor.py`                   | Read `$EXTMIN/$EXTMAX/$INSUNITS`, return `crs` field                   |
| DWG parse             | `app/modules/dwg_takeoff/service.py` (`_handle_dwg`)         | Call `detect_from_dwg_header`, persist on `DwgDrawingVersion`          |
| IFC/RVT process       | `app/modules/bim_hub/ifc_processor.py` (`process_ifc_file`)  | Call `detect_from_ifc`, return `crs` in result dict                    |
| BIMModel persist      | `app/modules/bim_hub/router.py` (background CAD processor)   | Copy result `crs` onto `BIMModel.crs_epsg / crs_name / crs_confidence` |
| Canonical JSON schema | top-level `crs` field (optional, additive)                   |                                                                        |
| Frontend viewer       | `frontend/src/features/bim/BIMPage.tsx`                      | Info chip + "Set CRS" affordance                                       |

---

## 6. Why pyproj is justified

`pyproj` is a thin Python wrapper around PROJ (a C library that ships
under MIT). For us it's a *read-only EPSG dictionary lookup* —
we never call the actual reprojection routines. Pulling the wheel
costs ~3 MB and adds zero runtime dependencies. The alternative
(hard-coding 80 EPSG names ourselves) creates a stale-data hazard
every time IOGP updates the registry — not worth the savings.

If `pyproj` is unavailable at runtime, the detector falls back to a
ship-with-us minimal name table (the 17 regions listed in §1.1). This
keeps `pip install openconstructionerp[core]` clean for ops who don't
ship CAD/BIM features.

---

## 7. Future improvements (deferred)

1. **GeoPDF detection** — if a PDF takeoff document carries a GeoTIFF
   tagging layer we could read the LGI dict and pull the CRS.
   `pikepdf` exposes this; `pdfplumber` doesn't.
2. **Reprojection on the fly** — once we know the source CRS, the
   viewer can place the model on a WMS basemap (OSM tiles or
   government ortho). Out of scope for this pass.
3. **Per-tenant defaults** — orgs that always work in one CRS can pin
   it and skip the prompt.
4. **Verify against ground-truth WMS** — fetch the OSM bbox for the
   region and confirm the model lands on land. Out of scope.
