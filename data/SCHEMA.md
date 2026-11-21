# Data contract — `data/` (doc v4)

Everything the app reads comes from this directory. Pipelines write it; the
app never computes. This file is the single source of truth for the contract
between this repo and `tipping-point-app` — the app's `docs/app-contract.md`
only points here and lists what the app does with each item.

**Case ids.** Canonical case keys are `rs_2024` and `wa_2025` — they name the
data directories (`data/rs_2024/`, `data/wa_2025/`), the manifest keys,
config.yaml and the `<caso>` filename stems. Event ids `rs-2024-05` /
`wa-2025-12` remain ONLY as each manifest node's `event_id` and the
`events.json` entry ids. `now` is both the key and the directory for the
live bundle. The app's `caseAlias()` maps both directions, so asset paths are
always written out explicitly — the app never derives a path from a key.

**Write order.** Every manifest is written LAST of its unit:
`data/now/manifest.json` after each live run's files, `data/manifest.json`
after everything else, `metrics.json` via atomic merge. Readers key on a
non-null `updated_at`; a null/absent `updated_at` means a run is in progress.
The app must never glob — every file it reads is either a fixed name below or
listed in the manifest.

CRS rules (v4):
- **COGs and GeoJSON: EPSG:4326** (COGs may also keep the NISAR product's
  native CRS).
- **PNGs for MapLibre image sources: EPSG:3857.** A 4326 PNG stretched over
  lon/lat corner coordinates misaligns at these latitudes; `live/render.py`
  warps class grids to Web Mercator before encoding, and each manifest layer
  carries `bounds` = the 3857 image extent in lon/lat `[w, s, e, n]`.

## Layout

```
data/
├── events.json                  # case index the app loads first
├── manifest.json                # the contract the app reads — written LAST
├── metrics.json                 # ONE file: per-case blobs keyed by case id
├── SCHEMA.md                    # this file
├── raw/                         # downloads — gitignored, never app-read
├── <case-id>/                   # one dir per case (rs_2024, wa_2025)
│   ├── cog/                     # rasters, clipped to AOI, keep each <= ~20 MB
│   ├── png/                     # pre-rendered EPSG:3857 PNGs (one per layer/day)
│   ├── geojson/                 # polygons/points, EPSG:4326
│   ├── series/                  # time series JSON for charts
│   └── nisar_products.json      # granule ids used (→ metrics collections)
└── now/                         # live bundle, overwritten every 3 h
    ├── manifest.json            # schema tipping-point/now@2 — written LAST
    ├── summary.json             # headline numbers for the side panel
    ├── alerts.json              # alert candidates for the bot/side panel
    ├── prev_classes.npz         # internal state (upgrade detection)
    ├── cog/                     # today's + D+1 TPI COGs (EPSG:4326)
    ├── geojson/critical_areas.geojson
    └── png/<layer>_d<k>.png     # tpi_{slide,flood}_d0..d3
```

## File naming

Stable app-read names, per case directory — the app reads exactly these:

- `series/imerg_smap_daily.json`
- `geojson/{priority_zones,permanent_water,flood_confirmed,landslide_candidates,places,municipal_boundaries,landslide_catalog}.geojson`
- `png/<layer-id>_<YYYYMMDD>.png` — EPSG:3857, `YYYYMMDD` = the day the layer
  depicts (the layer's `date` field), `<layer-id>` = the manifest layer `id`
  (incl. `_d<k>` forecast suffixes)

Dated pipeline outputs keep the `<caso>_<camada>_<AAAAMMDD>.*` stem under
`cog/`, `geojson/` and `series/` — e.g. `rs_2024_tpi_slide_20240502.tif`,
`wa_2025_swot_20251210.json`. `<caso>` is the config.yaml case key
(`rs_2024`, `wa_2025`) — the same id that names the directory.

`now/` forecast PNGs are `png/<layer>_d<k>.png` (d0 = today, d1..d3 =
forecast days).

## data/events.json

The case index, schema `tipping-point/events-index@1`:

```json
{ "schema": "tipping-point/events-index@1",
  "events": [
    { "id": "rs-2024-05", "title": "...", "bbox": [w, s, e, n],
      "reference_date": "2024-05-02", "layers": [] } ] }
```

Entry `id` is the hyphen **event id** — it equals the manifest node's
`event_id`, NOT the underscore manifest case key (which names the directory
and `<caso>` stems). `layers[]` is refreshed at publish time (runtime only;
the app uses it for layer hints).

## data/manifest.json (root)

```json
{ "updated_at": "2026-11-15T09:00:00Z",
  "cases": {
    "rs_2024": {
      "event_id": "rs-2024-05",
      "mode": "replay",
      "bounds": [-52.5, -30.5, -50.5, -29.0],
      "forecast_source": "perfect-forecast",
      "layers": [
        { "id": "tpi_slide", "date": "2024-05-02",
          "png": "rs_2024/png/tpi_slide_20240502.png",
          "cog": "rs_2024/cog/rs_2024_tpi_slide_20240502.tif",
          "asset": "projects/<GEE_PROJECT>/assets/tipping-point/rs_2024/tpi_slide_series",
          "legend": ["Baixo", "Moderado", "Alto", "Crítico"],
          "bounds": [-52.6, -30.6, -50.4, -28.9] },
        { "id": "tpi_slide_d1", "date": "2024-05-03",
          "png": "rs_2024/png/tpi_slide_d1_20240503.png",
          "cog": "rs_2024/cog/rs_2024_tpi_forecast_d1_20240502.tif",
          "asset": null,
          "legend": ["Baixo", "Moderado", "Alto", "Crítico"],
          "bounds": [-52.6, -30.6, -50.4, -28.9],
          "forecast": true },
        { "id": "r_series", "date": "2024-05-02",
          "png": null, "cog": "rs_2024/cog/rs_2024_r_series_20240502.tif",
          "asset": "projects/<GEE_PROJECT>/assets/tipping-point/rs_2024/r_series" }
      ],
      "metrics": "metrics.json#rs_2024",
      "data_dates": { "imerg": "2024-05-02", "smap": "2024-04-30" },
      "products": { "places": "rs_2024/geojson/places.geojson", "...": "..." },
      "_null_reasons": { "flood_confirmed": "no NISAR coverage — event predates launch (Jul 2025)" }
    },
    "wa_2025": { "event_id": "wa-2025-12", "...": "..." },
    "now": { "...": "copied verbatim from data/now/manifest.json" }
  } }
```

- Case keys are the canonical underscore ids (`rs_2024`, `wa_2025`, `now`).
- Every asset path is a repo-relative path under `data/` (which syncs to the
  app's `public/data/`): `rs_2024/png/...`, `now/summary.json`. Never
  derived from the key or `event_id`.
- Per-case node fields: `event_id`, `mode` (`replay` | `live`), `bounds`
  `[w, s, e, n]`, `layers[]`, `metrics` (`metrics.json#<case>` fragment into
  the single root metrics file), `data_dates`, `products`,
  `forecast_source`, `_null_reasons`.
- `data_dates`: the freshest date of each input actually used —
  `{imerg, smap, gfs_issued?, nisar_pre?, nisar_post?, ...}`. In the `now`
  bundle `smap` is the SPL4SMGP granule stamp as ISO UTC
  (`"2026-11-10T22:30:00Z"`, the raw granule id when unparseable), with
  `smap_granule` (file name, no CMR suffix), `smap_age_days` (whole days on
  the America/Sao_Paulo calendar) and `smap_label` (pt-BR
  `"SMAP de dd/mm, N dias"`).
- `forecast_source`: `"perfect-forecast"` for replay cases (D+1..D+3 =
  observed IMERG as forecast), `"noaa-gfs"` for the live bundle. The app must
  render forecast days differently (never like observed).
- `data_basis`: provenance/latency honesty label. Replay cases carry
  `"hindcast — IMERG_V07 + SMAP SPL4SMGP as archived in GEE (no real-time
  latency modelled); not as-available"` — the frames answer "what TPI would
  have said on day D with the archive we have now", NOT "what was visible at
  D". The lead-time series (live-acceptance A5) is the only as-available
  view: IMERG Early ~4 h latency + SMAP at its real L4 latency, no D+k
  data.
- The `now` node is copied from `data/now/manifest.json` by
  `tools/build_manifest.py` (schema `tipping-point/now@2`).
- Per-case `data/<case>/manifest.json` files are not part of the contract —
  the root manifest is authoritative (the app only falls back to them for
  legacy synced data).

### Layer fields

`{id, date, png, cog, asset, legend, bounds, forecast?, issued?,
forecast_source?, kind?, upper_bound?}`:

- `id` — one of the layer ids below; repeated ids with different `date`s are
  a per-day series.
- `date` — `YYYY-MM-DD`, the day the layer depicts.
- `png` — EPSG:3857 pre-render for the MapLibre image source; `null` on
  COG-only layers (`bounds`/`legend` then unused).
- `cog` — download/reproducibility raster (EPSG:4326 or NISAR native); may be
  `null` on PNG layers, required on COG-only layers.
- `asset` — full GEE asset id backing the layer
  (`projects/<GEE_PROJECT>/assets/tipping-point/...`), or `null` for
  file-only layers. `asset` is the primary source (served via the app
  server); `png`/`cog` stay as fallback — see "GEE assets".
- `legend` — pt-BR class names in index order, pinned exactly as
  `live/config.py LABELS`: `["Baixo", "Moderado", "Alto", "Crítico"]` for
  class values 0–3. The app maps by index to its own label names — order
  must never change.
- `bounds` — the PNG extent in lon/lat `[w, s, e, n]` (image-source corners).
- `forecast: true` — forecast layers only (`<layer>_d<k>` ids, k = 1..3).
- `issued` — `YYYY-MM-DD` issue day of a real issued forecast (L22 GFS
  frames); the depicted `date` is issued + k.
- `forecast_source` — `"noaa-gfs"` for the issued-GFS replay frames,
  `"perfect-forecast"` for the copied-observation upper bound, `"noaa-gfs"`
  live.
- `kind` — `"observed"` | `"forecast"` when the index entry declares it;
  `"nisar_extra"` on NISAR-derived overlay layers added after publish
  (e.g. `nisar_coherence_loss`), which also carry:
- `label` — display name (`nisar_extra` layers).
- `legend` — for `nisar_extra` a list of `{label, color}` objects, not the
  pinned class-name list (which stays fixed for the 0–3 class layers).
- `attribution` — source/credit line (`"NISAR (NASA/ISRO), ASF DAAC —
  Beta product"`).
- `note` — free-text method/caveat blurb (`nisar_extra` layers).
- `upper_bound: true` — on `perfect-forecast` frames only: they are the
  labelled upper bound of predictability, not an issued forecast.

### Layer ids

`tpi_slide`, `tpi_flood` — classified TPI geomean (class legend above);
forecast variants `tpi_slide_d1..d3`, `tpi_flood_d1..d3` carry
`forecast: true`. Reference/validation layers: `hillshade` (NASADEM terrain
shade), `nisar_pre`, `nisar_post` (NISAR GCOV backscatter),
`opera_validation` (3-class NISAR-vs-OPERA DSWx-S1 like-for-like agreement
map: 1=TP both, 2=FP NISAR-only, 3=FN OPERA-only, 0=nodata; PNG palette TP
#1a9850 / FP #f46d43 / FN #4575b4, nodata transparent; legend/class areas in
`data/<case>/opera_validation.json`, definition = `validation_method`),
`night_lights_pre`, `night_lights_post` (VIIRS Black Marble composites),
`t_ml` (XGBoost-modeled hazard; same class legend). NISAR-derived extra
overlays (`kind: "nisar_extra"`): `nisar_coherence_loss` (GUNW 12-day
coherence-loss candidate polygons — candidates, never confirmed) and
`nisar_coherence_event` (event-pair coherence, grayscale).

COG-only score layers — for geotiff.js range reads; `png: null`, EPSG:4326,
still listed in `layers[]`: `r_series`, `s_series` (per-day R/S score
series, one band per day named `YYYYMMDD`), `t_slide`, `t_flood` (static
terrain-score rasters). Files `<caso>_r_series_<AAAAMMDD>.tif` etc. under
`cog/`. `tpi_slide`/`tpi_flood` remain the CLASS renders — different layers
from the score series.

**Date-label convention:** a band/layer named `YYYYMMDD` is labeled by the
**end** of its accumulation window — the 3-day rain sum filtered
`[end − 3 days, end)`. A layer labeled `20240503` therefore covers rain
from Apr 30 through May 2 and a TPI peak on that label is *before or on*
the event date (2024-05-02). The series reads correctly when interpreted
as "conditions at the end of day D".

### products map

Non-raster discovery, per case node (D3). Every value is a repo-relative
path or `null`:

```json
"products": {
  "municipal_boundaries": "rs_2024/geojson/municipal_boundaries.geojson",
  "landslide_catalog":    "rs_2024/geojson/landslide_catalog.geojson",
  "places":               "rs_2024/geojson/places.geojson",
  "priority_zones":       "rs_2024/geojson/priority_zones.geojson",
  "permanent_water":      "rs_2024/geojson/permanent_water.geojson",
  "flood_confirmed":      "rs_2024/geojson/flood_confirmed.geojson",
  "landslide_candidates": "rs_2024/geojson/landslide_candidates.geojson",
  "swot_reference":       "rs_2024/series/swot_reference.json",
  "swot_series":          "rs_2024/series/swot.json"
},
"_null_reasons": {
  "flood_confirmed": "no NISAR coverage — event predates launch (Jul 2025)"
}
```

`products` values are strictly `string | null`. A `null` product carries its
reason under the case node's `_null_reasons` (a sibling of `products`, same
convention as metrics.json) — never inside the map itself.
`flood_confirmed` is only non-null when a NISAR or
otherwise validated flood mask exists. Landslide outputs are always
**candidates**, never "confirmed" — no ground-truth confirmation step exists.
`municipal_boundaries` (IBGE for RS, US Census TIGER for WA) and
`landslide_catalog` (GLC for WA, UFRGS/Cemaden inventory for RS) are
reference layers — never index inputs, credited in the footer.

## Series files

### `series/imerg_smap_daily.json`

The chart series per event:

```json
{ "event": "rs-2024-05",
  "reference_date": "2024-05-02",
  "rain_p95_mm": 45.0,
  "tpi_threshold": 0.6,
  "days": [
    { "date": "2024-05-01", "offset": -1, "kind": "observed",
      "imerg_mm": 12.4, "smap": 0.71, "tpi": 0.55, "river_m": null },
    { "date": "2024-05-03", "offset": 1, "kind": "forecast",
      "imerg_mm": 8.0, "smap": 0.73, "tpi": 0.62, "river_m": 4.1 }
  ] }
```

- `offset` — days relative to `reference_date` (0 = event day).
- `kind` — `"observed"` | `"forecast"` | `"hindcast"`; replay forecast days
  (perfect forecast) are `"forecast"`.
- `smap`, `tpi` — 0–1. `river_m` — SWOT reach level, `null` when no reach.
- `rain_p95_mm` — the p95 daily-rain climatology used for the rain axis;
  `tpi_threshold` — the Alto cut (config.yaml `tpi_classes`).

### `series/imerg_vs_gauge.json`

Per-station daily pairs for the IMERG-vs-gauge validation check (rs_2024,
ruling L19 — INMET gauges validate IMERG, never feed the index). Written by
`tools/imerg_vs_gauge.py`: gauge daily mm summed from the INMET hourly
archive on the CSV's own UTC `Data` day; `imerg_mm` is the GPM_3IMERGDF V07
daily *Final* value of the 0.1° cell containing the station, fetched per
day via GES DISC OPeNDAP ascii subsets (`.npy`-cached under gitignored
`data/raw/imerg_vs_gauge/`). A station-day is skipped (`stations[].skipped[]`
carries the reason) when > `max_missing_hours` (default 2) of its 24
hourly values are missing, when the station has no rows that UTC day, or
when IMERG has no value.

```json
{ "source": "...", "gauge_source": "...",
  "imerg_product": "GPM_3IMERGDF V07 daily Final",
  "window": {"start": "2024-04-20", "end": "2024-05-15"},
  "event_total_window": {"start": "2024-04-27", "end": "2024-05-06"},
  "max_missing_hours": 2,
  "caveat": "point gauge vs 0.1 deg cell",
  "stations": [
    { "id": "A801", "name": "...", "lat": -30.05, "lon": -51.17,
      "imerg_cell": {"i": 1288, "j": 599, "lon": -51.15, "lat": -30.05},
      "n_days": 26, "r": 0.98, "bias_pct": 5.5,
      "event_total_gauge_mm": 382.6, "event_total_imerg_mm": 425.3,
      "event_total_ratio": 1.112,
      "skipped": [{"date": "...", "reason": "..."}],
      "days": [{"date": "2024-04-20", "gauge_mm": 0.0,
                "gauge_hours": 24, "imerg_mm": 0.0}] } ] }
```

### `series/<caso>_swot_<AAAAMMDD>.json`

Dated SWOT river-level series per reach (pipeline output, e.g.
`rs_2024_swot_20240502.json`); the publisher merges it into
`imerg_smap_daily.json`'s `river_m`.

### `series/swot.json`

Per-reach SWOT water-surface-elevation chart series built by
`tools/wa_swot_series.py --case <case>` from PO.DAAC Hydrocron
(SWOT L2 HR RiverSP); `products.swot_series` points here:

```json
{ "case": "rs_2024",
  "source": "SWOT L2 HR RiverSP via PO.DAAC Hydrocron",
  "window": ["2024-04-01", "2024-06-30"],
  "event_dates": { "start": "2024-04-29", "peak": "2024-05-05" },
  "quality_legend": {"0": "good", "1": "suspect", "2": "degraded", "3": "bad"},
  "reaches": [
    { "reach_id": "64120300041", "name": null,
      "lon": -52.021467, "lat": -29.9419171,
      "series": [
        { "time": "2024-05-06T11:46:38Z", "wse_m": 14.2538,
          "wse_u_m": 0.10084, "quality": 2 } ],
      "dropped_fill": 0, "dropped_out_of_window": 0 } ],
  "_null_reasons": { "64120300024": "no valid SWOT pass in window" } }
```

- `window` / `event_dates` come from the event's `swot.window` /
  `swot.event_dates` when present, else `window` and
  `start_date`/`reference_date` in `events/<event>.json`.
- `reaches[].name` is the SWORD `river_name` (`null` when `no_data`);
  `lon`/`lat` the reach centroid.
- `series[]` entries carry UTC `time`, `wse_m`, `wse_u_m` (uncertainty,
  `null` at fill), `quality` (SWORD `reach_q`, `null` at fill). Fill
  rows are dropped and counted in `dropped_fill`; observations outside
  `window` in `dropped_out_of_window`.
- A reach with no Hydrocron timeseries keeps `series: []` and a
  `_null_reasons` entry keyed by reach id — values are never invented.

## GeoJSON feature shapes

All FeatureCollections are EPSG:4326. `priority_zones` features carry:

```json
{ "rank": 1, "name": "Canoas", "tpi": 0.78,
  "class": "Critical", "hazard": "landslide",
  "area_km2": 3.4, "people": 12400 }
```

`rank` (1 = worst), `name` (nearest place), `tpi` (0–1),
`class` (`Low|Moderate|High|Critical`), `hazard`, `area_km2`, `people`
(population in zone or `null` + reason). Other products
(`flood_confirmed`, `landslide_candidates`, `permanent_water`, `places`,
`municipal_boundaries`, `landslide_catalog`) are plain FeatureCollections;
landslide polygons are always `landslide_candidates`.

## data/metrics.json

ONE root file, per-case blobs keyed by case id. Each blob holds the flat v4
key set plus `_null_reasons` and `collections`; the manifest case node points
at its blob via the `metrics` field (`"metrics.json#rs_2024"`).

```json
{ "cases": {
    "rs_2024": {
      "event": "rs-2024-05",
      "iou": 0.0, "f1": 0.0, "cloud_pct": 0.0,
      "landslide_hit_pct": 0.0, "hand_validation": 0.0,
      "people_critical": 0, "people_flooded": 0, "flooded_km2": 0.0,
      "flood_source": "opera",
      "night_light_drop_pct": 0.0,
      "lead_time_mean_h": 0.0, "lead_time_median_h": 0.0,
      "warned_before_pct": 0.0,
      "rf_auc": 0.0, "rule_auc": 0.0, "feature_importance": {},
      "forecast_source": "perfect-forecast",
      "collections": { "...": "..." },
      "_null_reasons": { "...": "..." }
    },
    "wa_2025": { "event": "wa-2025-12", "...": "same flat key set" }
  } }
```

- `flood_source` — flat string identifying the input behind the flood numbers:
  `"nisar"` for a NISAR GCOV flood-mask asset, `"opera"` for a validated OPERA
  DSWx product, `"observed"` for other observed inputs; `null` with a reason
  when no flood number is emitted. A null update never clobbers a written
  source (`tp.metrics.merge_metrics`). Mock cases use `wa_2025: "nisar"` and
  `rs_2024: "opera"`; runtime values describe the input actually used.
- `forecast_source` — `"perfect-forecast"` for replay cases; the live now
  bundle uses `"noaa-gfs"`.
- `collections` — `{ "<dataset>": "<collection id/version>" }` for the inputs
  that produced the numbers (e.g. `NISAR_L2_GCOV_BETA_V1`,
  `OPERA_L3_DSWX-HLS_V1`, IMERG V07, SMAP L4, GPW epoch). General rule:
  record what was actually used.
- `_null_reasons` — `{ "<key>": "<why>" }` inside each case blob; any key
  written `null` must have a reason. `tp.metrics.merge_metrics` merges
  atomically into the root file keyed by case: never drops keys or cases,
  never overwrites a written non-null value.
- `withdrawn` — `{ "<key>": { "value": <old>, "reason": "<decision id + why>" } }`
  inside each case blob: a published value retired by a ruling (e.g. L32/L33
  for the v1 ML scores), kept for audit; the live key then carries the v2
  replacement (or `null` + `_null_reasons.<key>`). Nested paths appear as
  dotted names (e.g. `landslide_ml.t_ml`).
- `_sources` — `{ "<key>": "<exact source file#key.path>" }` inside each
  case blob: where each replaced value was copied from.
- `iou`/`f1`/AUCs are 0–1 ratios; `*_pct` keys are 0–100 (the app normalizes
  `> 1` values to 0–1 for display).
- Flood validation vs OPERA DSWx (wa_2025), one key set:
  `iou`/`f1`/`precision`/`recall` + `confusion {tp,fp,fn}` = like-for-like
  (NISAR new flood vs OPERA DSWx-S1 new flood = post water AND NOT pre water,
  cloud-free on both dates), described by `validation_method`;
  `iou_raw`/`f1_raw`/`precision_raw`/`recall_raw` + `confusion_raw` = the
  all-water comparison (`validation_method_raw`, inflated FN from permanent
  water, see `opera_fn_diagnosis`); `hls_iou`/`hls_f1`/`hls_confusion` =
  optical DSWx-HLS all-water, mostly cloud (`hls_method`, `cloud_pct`,
  `cloud_pct_source`). `tools/validation_metrics.py` writes only `*_raw` and
  `hls_*`; the headline keys are never overwritten by the all-water run.
  `confusion_km2`/`confusion_raw_km2`/`hls_confusion_km2` carry the same
  counts as geodesic areas (px × true geodesic pixel area at the AOI-mean
  latitude — see `validation_area_note`: a 30 m nominal lattice pixel is
  ~600 m² at ~48°N, not 900 m²).
- rs_2024 optical cloud (`tools/validation_metrics.py` task B): `cloud_pct`
  (100 − `cloud_free_pct`), `hls_scenes` (count), `hls_scene_dates` (UTC dates
  of those HLS L30/S30 scenes over the AOI) and `cloud_pct_source` (product,
  Fmask bits, scene count + dates, union rule) — what the app cites for
  'clouds hid X%'.
- `hand_validation` is the HAND<5 m zone IoU in 0–1 against MERIT Hydro 'hnd'
  at the selected radius (unit: fraction; before the G2 schema fix it was an
  ambiguous 0–100 percent, which the app read as "98%"). The unambiguous keys:
  `hand_validation_iou` (IoU 0–1), `hand_validation_r` (Pearson r, -1..1),
  `hand_validation_radius_m` (best radius = highest IoU, ties -> smallest;
  reported only, `thresholds.hand_proxy_radius_m` is NOT rewritten),
  `hand_validation_radii` {radius_m: iou 0-1}; `hand_validation_by_radius`
  {radius_m: {iou, r, r_5k, n_px, low_proxy_pct,
  low_merit_pct}} and `hand_validation_definition` (method text). The radius
  actually served by the TPI is published alongside:
  `hand_validation_served_radius_m` (= `thresholds.hand_proxy_radius_m`,
  currently 1500), `hand_validation_served_iou`, `hand_validation_served_r`
  (the same-radius entries of `hand_validation_by_radius`) and
  `hand_validation_note` (one-line caveat). Everything
  is computed locally on ONE common grid: MERIT 'hnd' plus the three proxy
  bands (NASADEM z − focalMin(radius)) downloaded together via
  `getDownloadURL` (NPY, 90 m, EPSG:4326, clipped to the case AOI, tiled over
  its bounding box). `r` covers all pixels valid on both bands, `r_5k` a
  5000-px sample (seed 1); `low_*_pct` is each HAND<5 m zone's share of the
  compared pixels. MERIT is a reference only — never an index input. No valid
  comparison produces null plus `_null_reasons`.
  (`tools/validation_metrics.py` Task C compares the same proxy against the
  NISAR flood mask — a different question — and publishes `hand_nisar_*`
  keys in `data/<case>/validation_metrics.json`, never `hand_validation*`.)
- `rf_auc` is the XGBoost model AUC — the key name stays for contract
  stability; public text says "XGBoost". Detailed sub-results may nest under
  extra sections (e.g. `landslide_ml`); the keys above stay flat.
- `imerg_vs_gauge` — nested IMERG-vs-INMET-gauge validation summary
  (rs_2024; L19, validation-only — gauges never feed the index):
  `{source, imerg_product, window, n_stations, n_station_days,
  pearson_r_median, bias_pct_median, event_total_ratio_median, caveat,
  stations[]}` where `bias_pct` is 100·(ΣIMERG − Σgauge)/Σgauge over paired
  UTC days and `event_total_*` sums Apr 27–May 6. Written by
  `tools/imerg_vs_gauge.py`; per-day pairs in
  `series/imerg_vs_gauge.json`. Caveat: point gauge vs 0.1° cell.

Writers (merge via `tp.metrics.merge_metrics` into the case's blob in the
root file, never `write_text` on the file): `04_validation.py` (iou, f1,
cloud_pct), `gee/09_impact.js` /
`gee/impact.py` (people_critical, people_flooded, flooded_km2,
night_light_drop_pct — merge its CSV printout), `gee/04_hand_vs_merit.js` /
`gee/hand.py` (hand_validation), `08_landslide_hit.py` (landslide_hit_pct),
`ml/lead_time.py` (lead_time_*, warned_before_pct), `ml/evaluate.py`
(rf_auc = learned-model AUC, rule_auc = manual slope rule,
feature_importance; XGBoost stands in for the doc's in-GEE Random Forest —
see ml/README.md), `07_forecast.py` (forecast_source).

### Impact bookkeeping — `gpw_year`, `gpw_product`, `population_total`

`gpw_year` 2020 (GPW v4.11 count, latest epoch, nearest both events);
`population_total` (persons) = sum of the case GPW COG over the case AOI
(`population_total_scope`); `people_critical` must be smaller. rs
`lead_time_mean_h` is exact from `lead_time_hist` (`lead_time_mean_note`).
Producer `tools/fill_impact_lead_keys.py`.

### "Did the alert come before?" — `flood_warned*` (wa_2025), `slide_warned*` (rs_2024)

Producer `tools/warned_before.py` (Modo 3 row 1). Share of the damage that the
TPI already flagged **Alto or higher (>= 0.4)** — and Crítico (> 0.6) — in the
frames available **24-48 h before the damage anchor**; each damage location
takes the ~90 m TPI cell that contains it; max over the window frames.

| key | unit | compares |
|---|---|---|
| `flood_warned_pct` (wa) | % of NISAR-mapped new-flood-candidate **area** (27.9 km²) | Alto+ in the L22 issued-GFS TPI_flood frames from 00Z runs 24-48 h before the anchor (headline) |
| `flood_warned_observed_pct` | same | Alto+ in observed TPI_flood bands dated anchor-2d..anchor-1d (latency ignored = upper bound) |
| `flood_warned_critical_pct` | same | Crítico share of the headline window |
| `slide_warned_pct` / `_observed_pct` / `_critical_pct` (rs) | % of the 16,027 inventory **points** | same rules with TPI_slide |
| `flood_warned` / `slide_warned` | object | `definition`, `anchor_utc` + `anchor_source`, `observed`, `forecast_gfs`, `best_issued_gfs_before_anchor`, `by_issue_gfs[]`, `by_date_observed[]` (per-date curve), WA `explanation`; WA `mapped_flood_km2` is an alias of `confirmed_flood_km2` (NISAR outputs are candidates, never confirmed — key name kept for contract stability) |

Anchors (L37): the documented event **ONSET**, not the peak —
`*_warned.anchor_basis = "onset"`, `*_warned_anchor_{basis,utc,source}` flat.
rs = 2024-04-29 00:00 BRT (03:00Z; first deaths of the event, MetSul laudo +
press); wa = 2025-12-08 00:00 PST (08:00Z; onset day, WA EMD SEOC sitrep).
`*_warned.sensitivity[]` repeats the window + per-lead numbers for other
anchors (rs: 2024-04-30 first landslide deaths, 2024-05-01 00Z previous fixed
dating; wa: King County FWC update 2025-12-09 00:35Z). Per-lead flat keys
`{slide|flood}_warned_{gfs|observed}_{24|48|72}h[_critical]_pct` +
`_gfs_{h}h_issue`: the 00Z run of the UTC date (onset − h) — issued at least
h hours before the onset. rs `flood_warned_pct` is null with a reason (no RS
flood map). `no_tpi_pct` = locations on TPI no-data, kept in the denominator.

`forecast_lead_time_onset` (both cases): the L22/L28b forecast lead time
re-anchored on the onset — issue day warned if any frame depicting a date <=
the onset day has an AOI-max TPI (slide or flood) > 0.6 anywhere in the AOI
(not at the damage locations); `lead_h` = onset − issue 00Z; `issue_days[]`,
`sensitivity[]`. The older `forecast_lead_time` block keeps its own anchor.

### EMSR720 flood reference — `flood_ems*` (rs_2024)

Producer `tools/rs_ems_validation.py`. Copernicus EMS Rapid Mapping
activation **EMSR720** (GLIDE FL-2024-000063-BRA, activated 2024-05-03) is the
RS 2024 flood reference map (gap G1; licence "Copernicus EMS — free use with
attribution: © European Union, Copernicus Emergency Management Service").
Files: `rs_2024/geojson/flood_ems.geojson` (union of all `5-Flood` polygons —
observedEventA 'Flooded area' + 'Flood trace' + DEL_MONIT 'Maximum flood
extent' — dissolved per AOI+product+image date, EPSG:4326, props `aoi`,
`product`, `image_date_utc`, `sensor`, `source`, `status`, `notations`,
`permanent_water_excluded`, `area_km2`; BLP `hydrographyA` permanent/reference
water subtracted), `rs_2024/geojson/flood_ems_aoi.geojson` (the five EMS AOI
extents), `rs_2024/geojson/landslide_ems.geojson` (`6-Mass Movement`
polygons — scored by `tools/rs_ems_landslide.py`, next section),
`rs_2024/flood_ems_meta.json` (activation,
AOIs, products with versions/image dates/sensors/download URLs, licence).

**Coverage limit (applies to every flood_ems key):** EMSR720 mapped only five
small Taquari-valley AOIs (Guapore, Encantado, Roca Sales, Das Antas Dam,
Santa Tereza — ~51.6–51.9 W, 29.0–29.3 S). Porto Alegre, Guaíba, Canoas,
Eldorado do Sul and São Leopoldo — the metropolitan flood epicentre — were
**not** mapped by any EMS activation, so scores are Taquari-valley-only
evidence, not AOI-wide.

| key | unit | definition |
|---|---|---|
| `flood_ems_iou` / `_f1` / `_precision` / `_recall` | fraction | headline cell-confusion score: EMS flood rasterized centre-in (`all_touched=False`) on the ~90 m TPI grid vs TPI_flood **Alto+ (> 0.4)** in the **observed** band dated **2024-05-06** — the EMS image date with the most AOI coverage (all five AOIs, Pleiades Neo optical GRA) |
| `flood_ems_flooded_km2` | km² | EMS-mapped flood area on that headline date (rasterized cell km²) |
| `flood_ems_flagged_before_onset_pct` / `_critical_pct` | % of the EMS flood footprint | already Alto+ / Crítico in the best of the GFS 00Z runs issued 2024-04-26/27/28 (frames depicting dates ≤ the onset day, max over frames; onset 2024-04-29 03:00Z, L37 — `tools/warned_before.py` evaluate/by_lead logic). Also reports the observed band dated 2024-04-29 |
| `flood_ems` | object | `definitions`, `flagged_before_onset` (anchor, denominator km², per-issue %, `by_lead`), `rows[]` — every image-date × rasterization (center / area-weighted ≥50 % coverage) × threshold (alto_plus / critico) × source (observed band E / GFS run issued E−1, frames depicting ≤ E) with tp/fp/fn, IoU, F1, precision, recall, km², AOI list — plus `licence`, `coverage_limit`, `scope` |
| `flood_ems_method` | text | one-paragraph method + caveats (imagery-mapped extent at a moment; misses flood under cloud/vegetation; 'Flood trace' = earlier water; EMS date E pairs loosely with frame conventions — observed band E = rain E−3..E−1, GFS frame depicting E = rain E−2..E) |

`flood_ems*` never touches `flooded_km2` / `flood_warned_pct` — those keep
their existing semantics (`flood_warned_pct` stays null pending the ASF
route; `flood_ems` is the reference-map validation, a different question).

### EMSR720 landslide reference — `landslide_ems*` + `scar_ems*` (rs_2024)

Producer `tools/rs_ems_landslide.py` (reuses `tools/rs_ems_validation.py`
loaders/AOI logic and `tools/warned_before.py` ANCHORS/frame conventions).
The 497 member polygons of `rs_2024/geojson/landslide_ems.geojson`
(photo-interpreted `6-Mass Movement`, imagery 2024-05-06..05-15, ~6.4 km²) are
the landslide reference. **The same coverage limit applies: scored ONLY inside
the five EMS AOI extents — Taquari-valley-only evidence, and the slides are
undated** (post-event imagery shows scars, not failure times; all scored
against the L37 onset 2024-04-29 03:00Z).

| key | unit | definition |
|---|---|---|
| `landslide_ems_n` / `landslide_ems_km2` | count / km² | EMS mass-movement member polygons (geodesic area) |
| `landslide_ems_warned_observed_pct` / `_critical_pct` | % of the 497 polygons | ANY ~90 m TPI_slide cell the polygon touches is Alto+ (> 0.4) [Crítico > 0.6] in the observed band dated **2024-04-29** (all information before the 04-29 03Z onset) |
| `landslide_ems_warned_gfs_{24,48,72}h_pct` / `_critical_pct` | % of the 497 polygons | same any-cell-touched rule vs the GFS 00Z run issued 2024-04-28/27/26 (max over d1..d3 `_slide` frames depicting dates ≤ the onset day) — the `slide_warned_*` by-lead convention, polygons instead of points |
| `scar_ems_hit_{100,500}m_pct` | % of the 497 polygons | ≥1 OPERA DIST scar candidate within r m (polygon-to-polygon, UTM 22S; candidates restricted to those whose representative point falls inside the AOI extents — 150 of 2,849) |
| `scar_ems_precision_{100,500}m_pct` | % of the 150 in-AOI candidates | within r m of an EMS polygon |
| `scar_ems_baseline` | object | random-placement null (science-review D2): same count (150) of placements per seed on steep terrain — T_slide > 0 = NASADEM slope ≥ 15° on the ~90 m grid (the scar product's terrain gate, from the case T_slide COG; the 1″ `steep15_1as.tif` mask it used is a gitignored static asset) — inside the AOI extents, buffered to circles with areas drawn from the in-AOI candidate distribution; `seeds` (200), per-metric `mean` + `ci95` (2.5/97.5 percentiles), `method`, `slope_source`, `lift_precision` |
| `scar_ems_lift_{100,500}m` | ratio | observed `scar_ems_hit_*m_pct` / baseline mean |
| `landslide_ems` | object | `tpi_warned` (per-date observed curve + per-lead GFS rows, all four variants: `{any_cell, centroid_cell} × {alto_plus, critico}`, count + area-weighted), `scar` (observed/baseline/lift), `inventory_16027` (published point-inventory numbers + `same_lead_order`), `definitions`, `licence`, `coverage_limit`, `scope` |
| `landslide_ems_method` | text | one-paragraph method + caveats |

`landslide_ems_warned_*` differs from `slide_warned_*` in BOTH the damage
reference (497 EMS polygons in five small AOIs vs 16,027 inventory points
AOI-wide) and the cell rule (any-cell-touched vs the point's own cell — more
permissive); the order comparison lives in `landslide_ems.inventory_16027`.

### Alarm precision — `alarm_*` (both cases)

Producer `tools/alarm_precision.py` (imports `tools/warned_before.py`
ANCHORS / frame conventions). Unit = **municipality x GFS 00Z run issue
day** over every replay issue day that exists as
`<case>/cog/<case>_tpi_gfs_<D>.tif` (rs_2024: 138 IBGE municipios x 15 days,
2024-04-23..05-07; wa_2025: 7 US Census TIGER counties intersecting the AOI x
20 days, 2025-12-01..12-20 — the published
`wa_2025/geojson/municipal_boundaries.geojson` holds places/county
subdivisions, NOT counties; the county polygons come from TIGERweb
State_County MapServer layer 1, BAS 2026, cached under
`~/.cache/lead-analysis/alarm-prec/wa_counties.geojson`).

ALARM(muni, D) = any ~90 m TPI_slide cell whose centre falls inside the
municipality polygon has TPI > 0.6 (`critical` cut) / > 0.4 (`alto_plus`),
max over the `d1..d3 _slide` bands depicting D+1..D+3 (the TPI-frame
convention; the `depict_d0_d2` variant re-reads the same bands as D..D+2
under the forecast_rain COG `d<j>` = D+j-1 convention — alarm masks
identical, only the hit-overlap test shifts). HIT(muni, D) = the
municipality holds >= 1 mapped landslide AND the depicted window overlaps
the trigger window — the inventories are undated, so every slide is assumed
inside it (rs: 2024-04-29..05-02; wa: 2025-12-08..12-12; L37,
`docs/conventions-dates.md`).

| key | unit | definition |
|---|---|---|
| `alarm_precision_critical_pct` / `_n_alarms` / `_n_hits` / `_false_alarms` | % / counts | alarmed units that are hits, Crítico cut |
| `alarm_precision_alto_plus_pct` / `_n_alarms` / `_n_hits` / `_false_alarms` | % / counts | same at the Alto+ cut |
| `alarm_pod_before_onset_pct` | % of slide municipalities | >= 1 alarmed hit unit issued (00Z) before the onset (`pod_before_window_end_pct` in the object relaxes to any in-window run) |
| `alarm_lead_h_median` | h | onset - issue 00Z, median over alarmed hit units issued before the onset (all-issue median + IQRs in the object) |
| `alarm_baseline_pct` | % of all units | share of muni x D units that are hits — what a uniformly random alarm scores |
| `alarm_lift` | ratio | precision / baseline (critical cut) |
| `alarm_precision` | object | `unit`, `municipality_source`, `n_municipalities`, `issue_days`, `trigger_window`, `dating_assumption`, `definitions`, `flat_keys_*` map, and `references.<ref>` (`catalog`, rs also `ems`) each with `n_slides`, `slide_municipalities`, `variants.depict_{d1_d3,d0_d2}.cuts.<cut>` = the full statistic block incl. `by_issue_day` table |

All flat keys mirror `references.catalog.variants.depict_d1_d3` only
(`flat_keys_cut` names which cut each mirrors). rs_2024 scores two
references separately (catalog = 16,027 inventory points; ems = the 497
EMSR720 polygons, Taquari-valley-only coverage).

## data/now/ — the live bundle

Moved from repo-root `now/` so `scripts/sync-data.mjs` copies it with
everything else. Overwritten every 3 h by `live/run_now.py` via
`.github/workflows/now.yml`; all files are written before
`manifest.json`, which is written LAST — a consumer keying on a non-null
`updated_at` never reads a half-updated set.

### `data/now/manifest.json` — schema `tipping-point/now@2`

```json
{ "schema": "tipping-point/now@2",
  "mode": "live", "updated_at": "2026-11-15T09:00:00Z",
  "bounds": [-58.0, -34.0, -48.0, -22.5],
  "data_dates": { "imerg": "...", "smap": "2026-11-10T22:30:00Z",
    "smap_granule": "SMAP_L4_SM_gph_20261110T223000_Vv8013_001.h5",
    "smap_age_days": 5, "smap_label": "SMAP de 10/11, 5 dias",
    "gfs_issued": "..." },
  "layers": [
    { "id": "tpi_slide_d0", "date": "2026-11-15",
      "png": "png/tpi_slide_d0.png", "cog": "cog/tpi_slide_today.tif",
      "asset": "projects/<GEE_PROJECT>/assets/tipping-point/now/tpi_slide_d0",
      "legend": ["Baixo", "Moderado", "Alto", "Crítico"],
      "bounds": [-58.0, -34.0, -48.0, -22.5], "forecast": false },
    { "id": "tpi_slide_d1", "forecast": true,
      "asset": "projects/<GEE_PROJECT>/assets/tipping-point/now/tpi_slide_d1",
      "...": "..." }
  ],
  "products": { "critical_areas": "geojson/critical_areas.geojson",
                "summary": "summary.json", "alerts": "alerts.json" },
  "nisar_passes": [ { "date": "...", "track": "...", "note": "..." } ] }
```

Paths inside this file are relative to `data/now/`. Layers are
`tpi_{slide,flood}_d0` (today) plus `_d1..d3` with `forecast: true`.

### `data/now/places.json` — "search for city"

Written every Now run by `live/places.py` (CI step after P(trigger) and alerts;
`products.places` in the Now manifest). One record per IBGE municipality in
`geojson/municipal_boundaries.geojson` (RS, SC, PR + the SP/MS municipalities
inside the bbox), sorted by uf then name, minified JSON (~0.6 MB):

```json
{ "schema": "tipping-point/places@1", "generated_at": "...", "now_updated_at": "...",
  "dates": {"d0": "YYYY-MM-DD", "d1": "...", "d2": "...", "d3": "..."},
  "classes": ["Baixo", "Moderado", "Alto", "Crítico"], "people_method": "...",
  "places": [{"id": "<IBGE code>", "name": "...", "uf": "RS",
              "centroid": [lon, lat], "bbox": [w, s, e, n],        // 4 dp
              "pop": 0,                                            // GPW v4.11 2020 (unadjusted), native 30", cell centres
              "flood": {"cls": [d0..d3 0-3|null], "max": [d0..d3 TPI|null]},
              "slide": {"cls": [...], "max": [...]},
              "p_trigger": {"24h": {"max": 0.0, "p90": 0.0} | null, "48h": ..., "72h": ..., "7d": ...},
              "alert": false, "alert_level": "Alto"|"Crítico"|null,
              "alert_hazard": "flood"|"landslide"|"flood+landslide"|null,
              "people_high": 0}]}                                   // High+ (TPI > 0.4), slide or flood, today or D+1
```

TPI per day = max over the municipality's 0.01° cells (a municipality smaller
than a cell takes the cell under its representative point); class = class of
that max. P(trigger) = max and p90 of the municipality's cells (null outside
Brazilian land / when the product is absent). `people_high` = High+ covered
fraction of each native 30" GPW cell × its count. d0..d3 come from the
internal `places_tpi.npz` written by `run_now` (not published).

### `data/now/summary.json`

```json
{ "generated_at": "...",
  "critical_km2": { "slide": 0.0, "flood": 0.0, "any": 0.0, "any_today_or_d1": 0.0 },
  "alert_km2":    { "slide": 0.0, "flood": 0.0, "any": 0.0, "any_today_or_d1": 0.0 },
  "critical_km2_def": "...", "alert_km2_def": "...",
  "stats_domain": "Brazilian land in the bbox (...)", "stats_domain_km2": 0.0,
  "people_critical": 0, "people_alert": 0, "people_exposed": 0,
  "forecast_source": "noaa-gfs",
  "classes_km2": { "landslide": { "Baixo": 0.0, "Moderado": 0.0, "Alto": 0.0, "Crítico": 0.0 },
                   "flood": { ... } },
  "data_dates": { "imerg": "...", "smap": "...", "smap_granule": "...",
    "smap_age_days": 5, "smap_label": "...", "gfs_issued": "..." } }
```

A8 (2026-11-15): `critical_km2` = class **Crítico only** (TPI > 0.6);
`alert_km2` = Alto or Crítico (TPI > 0.4, "High or above"); `people_critical`
= people in Crítico cells today or D+1, `people_alert` / `people_exposed` = in
Alto+ cells. All class / critical / alert / nodata areas cover **Brazilian
land only** (`stats_domain`: IBGE outline minus MCD12Q1 water); the map layers
still cover the whole bbox. `response/summary.json` and the priority-zone
properties follow the same split (`critical_km2` / `people_critical` Crítico,
`alert_km2` / `people_alert` Alto+).

People (S03a): Every people number is GPW v4.11 2020 population count (the UNADJUSTED count, not the UN-adjusted one) on its **native 30″ grid**, each cell weighted by the fraction of its area covered by the mapped area (exact polygon ∩ cell intersection; unions de-duplicate overlaps). Now: R1 DSWx-S1 and R2 NISAR polygons, R3 summary, priority zones and the Now `summary.json` `people_*` (merged from R3) all use it (`tp.response_summary.people_geom`, `people_method`). **Exception, not yet unified:** the replay cases (`people_critical` in metrics/series/GFS layer stats, WA `people_flooded`, `scar_people`) use GPW resampled by Earth Engine to ~1 km (0.00898°), which undercounts by ~11–13 %: RS 2024-05-02 Crítico area holds 1.49 M people on that grid vs 1.68 M on the native grid (2024-04-30: 1.17 M vs 1.30 M). Treat case people as a lower bound; re-run on the native grid after submission.

Replay case nodes: `products.flood_candidates` is the contract key for the
NISAR flood polygons (candidates, never confirmed); `products.flood_confirmed`
is a deprecated alias for one cycle (`_deprecated` says so).

### `data/now/alerts.json`

```json
{ "generated_at": "...",
  "quiet_hours": false,
  "rule": "upgrade to Alto/Crítico today or D+1; quiet 23h-6h unless Crítico",
  "candidates": [
    { "class": 3, "label": "Crítico", "centroid": [-51.2, -29.9],
      "pixels": 42, "hold": false } ] }
```

`candidates[]` pinned exactly as `live/alerts.py` emits it: `class` int
(2 = Alto, 3 = Crítico), `label` pt-BR class name, `centroid` `[lon, lat]`,
`pixels` int, `hold` = quiet-hours (23h–6h) held flag (only set on Alto
upgrades; Crítico is never held). These are alert *candidates* — the bot owns
sending and rate limiting.

`prev_classes.npz` is internal state for upgrade detection between runs —
not app-read.

### `data/now/response/` — flood response (L27, R1: OPERA DSWx-S1)

Written by `pipeline/scripts/13_response_dswx.py` (`tp/response_dswx.py`),
order: `flood_dswx.geojson`, `dswx_state.npz`, `dswx_meta.json` LAST.
Not yet wired into `data/now/manifest.json` (R3 hands that node to lead-live).

- `response/flood_dswx.geojson` — EPSG:4326 FeatureCollection, ≤ 5 MB.
  Polygons of flood **candidates** (new water vs the per-pixel DSWx-S1
  baseline). Feature props: `source` `"opera-dswx-s1"`, `acquired` (UTC ISO,
  newest obs in the polygon), `granules` (DSWx-S1 GranuleURs), `area_km2`,
  `people` (GPW 2020 sum, or `null` when GPW is unavailable), `status`
  `"candidate"` (never "confirmed"), `change` `"new"|"persisting"` (vs the
  previous run's candidate mask), `confidence` `"standard"|"lower"` (lower =
  MCD12Q1 cropland 12/14, e.g. rice paddies), `landcover` (majority IGBP
  class name). Only cells inside Brazil (IBGE outline) and outside MCD12Q1
  permanent wetland/water (11/17, dilated 1 cell) become polygons.
- Static inputs (committed): `data/now/static/mcd12q1/lc_type1_2025.tif`
  (MCD12Q1.061 2025 LC_Type1, EPSG:4326 0.005°, uint8 COG, nodata 255; built
  by `tools/convert_mcd12q1.py`) and `data/now/static/brazil_outline.geojson`
  (IBGE malhas v3 BR, clipped to the Now bbox + 0.5°).
- `response/dswx_state.npz` — internal rolling state, not app-read. Grid
  EPSG:4326 at 0.0025° over `domains.now.bbox` (4600×4000): `water_n`/`obs_n`
  uint16 baseline counts (halved at 60 obs), `cur` uint8 latest obs (0 dry,
  1 water, 255 none), `cur_t` uint16 hours since 2024-01-01Z, `cur_g` uint16
  granule index into `gids`, `prev_flood` uint8, `meta` JSON string.
- `response/dswx_meta.json` — `source`, `status`, `generated_at`,
  `latest_acquired`, `latency_h` (median CMR insert − acquisition of this
  run's granules), `n_granules` (this run), `n_failed`, `n_granules_total`,
  `flooded_km2`, `new_km2`, `people_flooded` (headline: Brazil, standard
  confidence only, summed over the mapped polygons ≥ `params.min_area_km2`
  so it equals what the map draws), `sub_mmu_km2` (in-Brazil candidate
  pixels in smaller clusters: not mapped, not in any headline),
  `receded_km2`, `lower_confidence_km2` (also polygon sums; "lower" =
  MCD12Q1 cropland OR JRC GSW seasonal/occasional water OR no supporting rain — occurrence ≥
  `params.gsw_occurrence_min_pct` % or seasonality ≥
  `params.gsw_seasonality_min_months` months; polygons carry
  `lower_reason` [list]), `seasonal_downgraded_km2` /
  `seasonal_downgraded_people` (what the GSW rule moved standard → lower in
  the mapped polygons = change to the headline), `seasonal_flagged_pixel_km2`
  / `seasonal_flagged_pixel_people` (same incl. sub-MMU specks),
  `seasonal_water_source`,
  `norain_downgraded_km2` / `norain_downgraded_people` (moved standard →
  lower by the rain-support rule: 8-day IMERG Early total < 10 mm locally
  AND everywhere within 50 km — a direction-agnostic upstream proxy, no flow
  routing — AND JRC occurrence 0; unknown rain never downgrades),
  `norain_flagged_pixel_km2`, `rain_rule` (window, n_files, threshold_mm,
  radius_km, local_median_mm), `rain_source`, `all_candidates_km2` /
  `all_candidates_people` (standard + lower, shown alongside the
  standard-only headline),
  `lower_confidence_new_km2`, `lower_confidence_people`,
  `excluded_water_wetland_km2`, `outside_brazil_km2`, `raw_candidate_km2`
  (before land cover / Brazil filtering),
  `unbaselined_water_km2` (water with < `min_obs` baseline obs — never a
  candidate), `observed_km2`, `coverage_pct`, `features`, `grid`, `params`,
  `bootstrap`, `people_source`, `landcover_source`,
  `brazil_outline_source`, `_null_reasons` (every null key explained;
  a failed baseline bootstrap nulls the km2/people keys and says so).

#### R2 — NISAR GCOV radar flood candidates (`pipeline/scripts/14_response_nisar.py`)

- `response/flood_nisar.geojson` — candidate polygons (`flood_type` open-water |
  bright, `confidence`, `people`, `status: "candidate"`).
- `response/nisar_meta.json` — `latest_acquired`, `latency_h`, `pairs[]`,
  `flooded_km2` / `open_water_km2` / `bright_km2` / `people_flooded`
  (after the HAND gate), plus the gate breakdown: `pre_hand_flooded_km2` /
  `pre_hand_people` (open-water candidates with no HAND gate),
  `hand_excluded_km2` / `hand_excluded_people` (removed by HAND > 15 m),
  `hand_source` {file, sha256_12, res_deg, res_m_approx} (prefers the 90 m
  proxy `data/static/south_brazil/terrain/hand_90m.tif`, static-v1, else the
  legacy ~1 km `hand_m.tif`), `method`, `_null_reasons`.

#### Landslide scar candidates (`pipeline/scripts/16_response_dist.py`)

OPERA DIST-ALERT-HLS (LP DAAC, 30 m MGRS tiles), newest granule per tile in
the window; candidate = VEG-DIST-STATUS in {5,6,8} (≥ 50 % vegetation loss,
provisional/confirmed/finished) AND first detection (VEG-DIST-DATE) in the
window AND slope ≥ 15° (1" NASADEM mask `data/now/static/nasadem/
steep15_1as.tif`, static-v1) AND not MCD12Q1 cropland/water; 3 px … 1 km².
Always `status: "candidate"`.

- Now: `response/landslide_dist.geojson` (props `class, kind:
  "landslide_candidate", status, date` (first detection), `area_km2, people,
  municipality, code, uf`) and `response/dist_meta.json` (`window,
  latest_acquired` — DIST lags ~1-2 weeks —, `n_tiles, n_tiles_failed,
  features, candidates_km2, people, excluded{loss/flat/cropland_or_water km²,
  too_small/too_large/outside_brazil polygons}, params, caveat,
  _null_reasons`). Clipped to the IBGE Brazil outline.
- Replay rs_2024: `rs_2024/geojson/landslide_candidates.geojson` (manifest
  `products.landslide_candidates`) + metrics `scar_*` (candidates n/km²/people,
  inventory hit rate within 100/500 m, precision proxy within 500 m,
  `scar_sensitivity` table).
- Replay wa_2025: `wa_2025/geojson/landslide_candidates.geojson` + `scar_*`.
  The Brazil static masks do not cover WA, so gates come from the gitignored
  `static/cache/wa/` cache: slope > 15° from the case T_slide COG (GEE
  NASADEM, 90 m export) and MCD12Q1 tile h09v04
  (`tools/convert_mcd12q1.py --bbox`); people from the case GPW count COG;
  municipalities = US Census TIGER county subdivisions (`name` only — no
  IBGE `code`/`uf`). Window 2025-12-05 … 2026-02-28 (the storm plus the
  first cloud-free looks, ~12 weeks).

#### R3 — response summary + priority zones (`pipeline/scripts/15_response_summary.py`)

Runs after the TPI job, R1 and R2. Inputs: `response/flood_dswx.geojson`,
`response/flood_nisar.geojson` (+ metas), `geojson/critical_areas.geojson`,
`summary.json`, GPW count `static/gpw/gpw_popcount_2020_30sec.tif`, IBGE
municipality boundaries (IBGE API at runtime). Every `*_km2` is a geodesic
area of the unioned polygons (sources unioned first, never double-counted);
people are GPW sums on one centre-sampled 0.0025° grid. Headline counts
standard-confidence flood only; R1 "lower" (cropland) is `*_lower`.

- `response/summary.json` — schema `tipping-point/response@1`:
  `people_flooded`, `people_flooded_lower`, `people_critical`,
  `people_exposed` (flood ∪ critical), `flooded_km2`, `flooded_km2_lower`,
  `critical_km2`, `exposed_km2`, `overlap_km2` (DSWx ∩ NISAR),
  `sources.{opera-dswx-s1,nisar-gcov,tpi-critical}` (features, km2, people,
  latest_acquired, latency_h, status), `nisar` (pair granules),
  `priority_zones`, `people_source` (label, never a local path),
  `_null_reasons`.
- `response/priority_zones.geojson` and the app copy
  `geojson/priority_zones.geojson` (same features) — IBGE municipalities
  ranked by exposed people then area: `name, code, uf, rank,
  people_flooded, people_critical, flooded_km2, critical_km2, sources,
  status: "candidate"` + app props `tpi, class, hazard, area_km2, people,
  label_lon, label_lat`.
- `response/manifest_node.json` — fragment the now job merges into
  `now/manifest.json`; `people_exposed` / `people_critical` / `response`
  are merged into `now/summary.json` without dropping keys.

## data/tipping_point.sqlite — queryable mirror (stdlib sqlite3)

Built by `tools/build_sqlite.py` (`pipeline/tp/sqlite_store.py`) from the files
above, after the manifests are written; rebuilt from scratch each run, < 50 MB
(the tool fails above that). Missing inputs leave a table empty and are listed in
`build_notes`; values are never invented. Since L53 (Neon → SQLite) the `docs`
table carries every app-facing JSON/GeoJSON by path, so a server can read one
local file instead of fetching each JSON (PNG/COG frames stay files on the
fallback branch).

| table | columns | from |
|---|---|---|
| `events` | event_id, case_id, title, reference_date, west, south, east, north | `events.json` |
| `nisar_passes` | granule, acquired, path, direction | `now/manifest.json` `nisar_passes` |
| `metrics` | case_id, key, value, unit, updated_at, source | `metrics.json` per replay case (nested values as JSON text, bools 0/1, `_null_reasons.<key>` rows carry the reasons) + `case_id='now'` scalars from `now/summary.json` (`critical_km2.<hazard>`, `response.<key>`) |
| `series` | case_id, date, var, value | `<case>/series/imerg_smap_daily.json`, long format (numeric vars only) |
| `zones` | run_id, municipality, code, uf, hazard, area_km2, people, class, rank | `now/geojson/priority_zones.geojson` (run_id = now `updated_at`), `<case>/geojson/priority_zones.geojson` (run_id = case) |
| `response_candidates` | run_id, source, hazard, area_km2, people, municipality, confidence, status, acquired, lon, lat | `now/response/flood_dswx`, `flood_nisar`, `landslide_dist` and `rs_2024/geojson/landslide_candidates` (lon/lat = representative point; all `status = candidate`) |
| `runs` | run_id, kind (now / replay), updated_at, verified (0/1), data_dates (JSON) | `now/manifest.json`, root `manifest.json`; **one row per Now run** — earlier `now` rows are carried over from the previous database (CI downloads it from the fallback branch before rebuilding) |
| `build_notes` | item, note | build time + missing inputs |
| `docs` | path (PRIMARY KEY, data-root-relative, exactly the path the app requests, e.g. `manifest.json`, `now/manifest.json`, `ml/ml_card_v2.json`, `rs_uruguai_2026/geojson/upstream_basin.geojson`), bytes, sha256 (of the raw file), encoding (`gzip`), body (gzip of the raw bytes, mtime 0) | every `*.json` / `*.geojson` under `data/` except `raw/` and `static/` dirs (same rule as the fallback publisher); files > 25 MB are skipped and noted in `build_notes` |

Units come from the key suffix (`_km2` km², `_pct` %, `_h` h, `_mm` mm, `_m` m,
`people*` persons); `unit` is null otherwise.

## GEE assets

The rasters also live under the GEE asset namespace
`projects/<GEE_PROJECT>/assets/tipping-point/` so the app server can read
them server-side (geotiff.js/ee range reads). Tree:

```
tipping-point/
├── static/<domain>/             # domain = south_brazil | wa_2025
│   ├── clim_rain                # ImageCollection, `doy` property;
│   │                            #   bands p90|p95|p99 = 3-day-sum mm percentiles,
│   │                            #   int16 mm*10 (config.yaml climatology.scale)
│   ├── slope                    # degrees
│   └── hand                     # metres
├── <case>/                      # rs_2024, wa_2025
│   ├── tpi_slide_series, tpi_flood_series, r_series, s_series
│   │                            # multiband, one band per day named YYYYMMDD
│   ├── t_slide, t_flood         # single-band static terrain scores
│   ├── landslide_candidates_hls
│   └── ...                      # further ingested rasters — names in data/<case>/assets.json
└── now/
    ├── imerg_early, smap_l4, geosfp                            # input collections
    └── tpi_slide_d0..d3, tpi_flood_d0..d3, r_d0..d3, s_d0..d3   # single-day images
```

- Every derived asset carries the image properties `data_date`
  (YYYY-MM-DD) and `updated_at` (ISO-8601) — readers can check freshness
  without the manifest.
- Climatology layout (approved): per-case `static/<case>/clim_rain`
  collections cover the case's event window (~25 doys over the case AOI);
  the live now mode uses a rolling ~75-doy window over the `domains.now`
  bbox in config.yaml at `static/south_brazil/clim_rain`. Percentile bands
  are stored int16 mm*10 (~0.9 GB total). The read interface is unchanged —
  `doy` image property, `p90|p95|p99` bands — with consumers dividing by
  `climatology.scale` (`tp.config.clim_scale()`) before scoring.
- Reader rule: a manifest layer's `asset` is the PRIMARY source (served via
  the app server); `png`/`cog` remain the fallback, unchanged.
- Extra ingested rasters beyond the names above are recorded by asset id in
  `data/<case>/assets.json` (per case, written by the gathering ingest tool).

## Fallback branch (no GCS)

There is no Cloud Storage bucket anywhere in this pipeline (ruling L10).
`tools/export_fallback.py --now|--case` writes fallback files **locally**
into `data/` — PNGs via `getThumbURL` (EPSG:3857), JSON numbers via
`reduceRegion().getInfo()`, GeoJSON via `reduceToVectors().getInfo()`, and
COGs only via `getDownloadURL` when the download is under 10 MB (larger
rasters are skipped with a logged reason). The live workflow then publishes
`data/` to the orphan branch `fallback` of this repo (force-pushed each run
so history does not grow), and the app reads it via
`DATA_BASE_URL=https://raw.githubusercontent.com/r2l2pj/tipping-point/fallback/data/`.
The layout on that branch is exactly the `data/` tree above, with manifests
written last (`data/now/manifest.json` last in its group, `data/manifest.json`
last overall).

## Who writes what

| File | Writer |
|---|---|
| `data/events.json` | hand-maintained; `layers[]` refreshed by `tools/publish_case.py` |
| `data/manifest.json` | `tools/build_manifest.py` — merges per-case nodes + `data/now/manifest.json`; written LAST |
| `data/metrics.json` | `pipeline/tp/metrics.py::merge_metrics` — atomic merge into the per-case blob keyed by case id; contributors listed above |
| `data/<case>/cog/<caso>_tpi_{slide,flood}_<AAAAMMDD>.tif` | `python -m gee.tpi` (`gee/01_tpi_index.js`, `gee/07_tpi_wa.js` — multiband, one band per day) |
| `data/<case>/cog/<caso>_{r,s}_series_<AAAAMMDD>.tif`, `<caso>_t_{slide,flood}_<AAAAMMDD>.tif` | `python -m gee.tpi` — score series + static terrain scores (COG-only geotiff.js layers) |
| `data/<case>/cog/<caso>_tpi_forecast_d{1,2,3}_<AAAAMMDD>.tif` | `pipeline/scripts/07_forecast.py` / `tp/forecast.py` (perfect forecast) |
| `data/<case>/cog/<caso>_water_nisar_<AAAAMMDD>.tif` | `pipeline/scripts/02_flood_mask.py` / `tp/flood_mask.py` (NISAR GCOV, wa) |
| `data/<case>/cog/<caso>_landslide_candidates_<AAAAMMDD>.tif` | `pipeline/scripts/03_landslide_coh.py` (coherence) / `09_hls_scar.py` / `tp/hls_scar.py` (wa HLS optical) |
| `data/<case>/cog/<caso>_hazard_slide_series_*.tif`, `hazard_classes.tif` | `ml/predict.py` (XGBoost — source of the `t_ml` layer) |
| `data/<case>/geojson/<dated>.geojson` | `pipeline/scripts/06_polygonize.py` / `tp/polygonize.py` |
| stable `geojson/*.geojson`, `png/<layer>_<YYYYMMDD>.png`, `series/imerg_smap_daily.json` | `tools/publish_case.py` / `tp/bundle.py` — PNGs via `live/render.layer_png_3857` + `tpi.classify` |
| `geojson/municipal_boundaries.geojson`, `geojson/landslide_catalog.geojson` | lead-gathering fetchers (IBGE / US Census TIGER; GLC / UFRGS-Cemaden) |
| `data/<case>/series/<caso>_swot_<AAAAMMDD>.json` | `pipeline/scripts/05_swot.py` / `tp/swot.py` |
| `data/<case>/series/swot.json` | `tools/wa_swot_series.py --case <case>` (Hydrocron per-reach WSE chart series) |
| `data/<case>/nisar_products.json` | `pipeline/scripts/01_get_nisar.py` / `tp/nisar.py` (granule ids → `collections`) |
| `data/now/*` | `live/run_now.py` (static grid `static/sul_static.npz` from `live/build_static.py`) |
| `projects/<GEE_PROJECT>/assets/tipping-point/**` | `python -m gee.tpi` (static/<case>), `live/run_now.py` (now/), publish ingest (`tools/publish_case.py`) |
| `data/<case>/assets.json` | lead-gathering ingest tool — records extra ingested asset ids per case |
| `data/<case>/cog/<caso>_opera_validation_<AAAAMMDD>.tif`, `png/opera_validation_<AAAAMMDD>.png`, `opera_validation.json` | `tools/wa_opera_layer.py` (wa_2025) — TP/FP/FN agreement grid via getDownloadURL |
| `data/raw/**` | fetch scripts (NISAR, OPERA DSWx, HLS, MODIS NRT) — gitignored |
| `simular.json` | `tools/build_simular.py` — an **app-side fixture** (its Telegram `/simular` consumer was removed, L55), NOT a contract product |

## Conventions

- Nodata: rasters carry a written nodata value (uint8 class grids use 255;
  the `opera_validation` grid uses 0);
  PNG renders mask nodata transparent; GeoJSON omits empty geometries.
- `data/raw/` is gitignored — never commit downloads; commit only clipped
  outputs.
