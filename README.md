# Dept 56 (Morbihan) — Lidar_Loop pipeline run

End-to-end PV-panel database for département 56 (Morbihan, Bretagne), produced
by [Lidar_Loop](../../Lidar_Loop). This folder holds every output of
that run, plus the viewer for inspecting the result.

## TL;DR

The pipeline takes IGN BDOrtho 20 cm imagery + IGN LiDAR-HD point clouds +
the national building cadastre, detects PV panels with a SegFormer-B0
segmentation model, and outputs a per-panel database with location, surface
area, tilt and azimuth.

**For dept 56:** 10,882 PV panels detected on 8,950 PV-positive buildings;
1,266 of those panels (11.6 %) have a real LiDAR-derived roof plane. The
rest sit on buildings outside IGN's current LiDAR-HD coverage.

## Source data (vintage)

```
BDOrtho 20 cm                  2024
Cadastre (Etalab)              January 2025
LiDAR-HD                       2025 acquisitions where flown
National PV registry           replaced May 2025
```

## Pipeline overview

The pipeline runs in four stages. All Python scripts live in
`/home/ennhirim/Bureau/Lidar_Loop`.

### Stage 0 — index PV-positive buildings against LiDAR-HD tiles
**Script:** `stage0_build_index.py --config config.56.json`

Reads two flat files:

- `presence/score_roof_mergePv__56.csv` — one row per Etalab building, with
  a PV-presence score (0 to 1) from an earlier model.
- `home/.../cadastre/batiments__56.shp` — Etalab cadastre polygons in EPSG:2154.

What it does:

1. Keeps only buildings with `Score > 0.50`. Result: 8,950 PV-positive
   buildings (out of 730,495 in the dept cadastre).
2. Queries IGN's WFS endpoint for every LiDAR-HD tile that intersects the
   dept bounding box (2,943 tiles returned for dept 56).
3. Spatial-joins each PV+ building centroid to a tile.
4. Writes `data/france_pv.sqlite` with two tables:
   - `building(building_id, dept, pv_score, cx, cy, polygon, tile_id)`
   - `tile(tile_id, dept, url, status, retries, ...)`

For dept 56:

- 1,099 of the 8,950 PV+ buildings land on a LiDAR-HD tile (12 %).
- 7,851 fall outside any IGN tile because **western Morbihan has not yet
  been flown** (Lorient, Quiberon, Belle-Île, Pontivy, Gourin, Golfe du
  Morbihan islands). A WFS probe of the western half returns zero tiles.
- 450 unique tiles host at least one PV+ building. These go into the queue.

### Stage 1 — download + segment each LiDAR tile
**Script:** `stage1_run.py --config config.56.json`

A pipelined download + process loop. State machine per tile, persisted in
`tile.status`:

```
pending → downloading → ready → processing → done
                                      ↓
                                  failed
```

For each `pending` tile, a worker:

1. Downloads the `.copc.laz` file from `data.geopf.fr/telechargement/...`
   (mean tile size ~96 MB).
2. Reads class-6 (building) points with `laspy`.
3. For every PV+ building polygon on that tile:
   - Clips the point cloud to the polygon.
   - Voxelizes onto a 1 m grid (highest Z per cell).
   - 2D Delaunay triangulation, filters by edge length and Z jump.
   - Region-grows triangles by normal similarity (cos 8°).
   - Merges near-coplanar regions (cos 3°).
   - Computes tilt, azimuth, normal, and the planar projection.
4. Writes one JSON per tile to `data/output/56/<tile_id>.json` with a list
   of `roof_regions` — each carrying its building_id, geometry, tilt and
   azimuth.
5. Deletes the LAZ buffer to keep disk usage bounded.

For dept 56:

- 448 of 450 tiles processed (2 failed on transient IGN errors).
- Wall-clock: ~30 min once IGN bandwidth opened up.
- Producing ~4,600 roof regions across 1,063 buildings (one building can
  carry several roof faces if the geometry is non-planar).

### Stage 2a — detect PV panels in the BDOrtho imagery
**Script:** `predict_panels.py --config config.56.json --department 56 --image-dir images_extracted --id-from-filename ...`

Takes 257,124 per-building BDOrtho JPGs (one per cadastre building with a
sidecar `.aux.xml` carrying the EPSG:2154 georeference), filters to the
8,950 PV+ buildings by filename stem, and runs SegFormer-B0 fine-tuned on
the rooftop PV task (checkpoint `segformer_b0_cfg21_best.pth`,
val IoU 0.876).

What it does for each image:

1. Resize to 128×128, normalise, forward through the model.
2. Threshold the sigmoid output at 0.5 to get a binary mask.
3. Vectorise connected components into polygons via
   `rasterio.features.shapes` with the .aux.xml affine, putting the polygon
   coordinates in EPSG:2154.
4. Drop polygons under 1 m².
5. Spatial join against the cadastre to attach `building_id`.
6. Optionally save the binary mask as PNG to `data/panel_segmentation/masks/`.

Output: `data/panel_segmentation/panels__56.geojson` with one feature per
detected panel polygon, carrying `panel_id`, `building_id`, `confidence`,
`source_image`.

For dept 56:

- 8,950 input JPGs → 10,882 detected panels in ~150 s on CUDA (~60 img/s).
- One building can carry multiple panels (panels are split where the
  segmentation mask has disconnected components).

### Stage 2b — assign each panel to its roof plane
**Script:** `stage2_panels.py --config config.56.json --departments 56`

Joins the panel polygons (from stage 2a) to the roof regions (from
stage 1) so every panel inherits a tilt and azimuth, and surface area
becomes a true 3D inclined area.

Per panel:

1. Find the roof region whose horizontal footprint **contains** the panel
   centroid → `region_match_method = "contains"`.
2. If no region contains it, pick the one with the largest area
   **overlap** → `"overlap"`.
3. If no overlap, fall back to the **nearest** region by distance →
   `"nearest"` (with a 5 m cap).
4. If the panel sits on a building with no roof regions at all (e.g.
   building outside LiDAR coverage), keep the row with
   `region_match_method = "none"`, tilt and azimuth NULL, surface area =
   2D mask area (flat-assumption fallback).

Surface formula:
```
surface_m2 = 2D_mask_area / cos(tilt_radians)     # default when tilt is known
surface_m2 = 2D_mask_area                         # when tilt is NULL
```

Outputs in `data/panel_segmentation/`:

- `panels_assigned__56.geojson` — same geometry as stage 2a, properties
  enriched with `tilt_degrees`, `azimuth_degrees`, `surface_m2`,
  `region_match_method`, `building_link_method`, `lon`, `lat`,
  `source_tile`, ...
- `panels__56.csv` — same fields as a flat CSV for downstream analysis.
- `panel` table inside `data/france_pv.sqlite` — same rows, queryable
  with SQL.

For dept 56:

- 10,882 panel rows.
- 992 `contains` + 206 `overlap` + 68 `nearest` = **1,266 panels with tilt
  and azimuth (11.6 %)**.
- 9,616 panels with `region_match_method = "none"` (i.e. no LiDAR cover
  for that building, mostly in western Morbihan).
- Total 2D mask area: 557,463 m². Total inclined surface: 564,504 m².

## Why the 11.6 % tilt coverage is not a model limitation

The model detects panels purely from BDOrtho imagery, which covers the
whole department. So the 10,882 detected panels are real and complete for
2024 imagery. The 11.6 % figure is about **the tilt+azimuth metadata**, not
the panel detection.

Tilt and azimuth come from the LiDAR-HD point cloud. IGN has flown the
**eastern third of Morbihan** (around Vannes, Vilaine estuary, Pénestin,
Allaire, Questembert, plus a smaller pocket near Mauron/Mohon). Western
Morbihan, the Quiberon peninsula, Belle-Île, the Golfe islands and Lorient
have **zero LiDAR-HD tiles** in the current IGN acquisition — confirmed by
direct WFS probe.

When IGN publishes the missing tiles, re-running stage 1 (which is
idempotent against the SQLite queue) plus stage 2b will fill in the
tilt/azimuth of the remaining 9,616 panels automatically.

## Folder layout

```
bretagne/56/
├── README.md                                ← this file
├── home/nerotb/.../entrepot_3/56/
│   ├── cadastre/batiments__56.{shp,dbf,...}     input: Etalab cadastre
│   └── tar_files/images_NN__56.tar.gz           input: per-building JPGs
├── presence/
│   ├── score_roof_mergePv__56.csv               input: PV-presence scores
│   └── score_roof_mergePv__56.{shp,dbf,...}     same as CSV in shapefile form
├── images_extracted/                            stage 2a working dir
│   ├── 056100032.jpg  +  056100032.jpg.aux.xml  (~754k JPGs + sidecars)
│   └── ...
└── data/
    ├── france_pv.sqlite                         queue + final panel table
    ├── output/56/<tile_id>.json                 stage 1 outputs (per tile)
    ├── laz_buffer/                              transient LAZ download buffer
    ├── logs/
    │   ├── stage0.log
    │   ├── stage1_56.stdout
    │   ├── stage1.log
    │   └── predict_panels_56.stdout
    └── panel_segmentation/
        ├── panels__56.geojson                   stage 2a output (raw panels)
        ├── panels_assigned__56.geojson          stage 2b output (with planes)
        ├── panels__56.csv                       same as above, CSV form
        ├── admin_56.geojson                     dept + 249 commune boundaries
        ├── lidar_tiles_56.geojson               IGN LiDAR-HD tile coverage
        ├── masks/<building_id>_mask.png         binary masks (~8,950 PNGs)
        └── viewer_56.html                       interactive web map
```

## Viewing the result

Open `data/panel_segmentation/viewer_56.html`. It auto-loads the four
sibling files when served over HTTP (Live Server, `python -m http.server`,
any local server). Under plain `file://` browsers block the auto-load, so
drag the four files into the drop zone instead.

The viewer renders:

- Each panel as an azimuth-coloured polygon (gray when tilt is unknown).
- Communes as polygons, **green outline if they contain ≥1 panel with
  LiDAR-derived tilt, yellow otherwise** — quick visual proxy for IGN
  coverage.
- The department boundary as a thick red outline.
- The 1 km × 1 km LiDAR-HD tiles as faint overlays (translucent green for
  tiles the pipeline used, translucent blue for available-but-unused).
- Click any panel for the full attribute card. Click any commune for its
  name, INSEE code, and panel count.

## How to re-run

Configuration lives at `/home/ennhirim/Bureau/Lidar_Loop/config.56.json`.

```bash
cd /home/ennhirim/Bureau/Lidar_Loop

# Stage 0 — idempotent, replaces dept-56 rows in SQLite each run
python3 stage0_build_index.py --config config.56.json --departments 56

# Stage 1 — resumable, picks up at `pending`/`ready` tiles
python3 stage1_run.py        --config config.56.json --departments 56

# Stage 2a — needs the torch venv with CUDA
/usr/bin/python3 predict_panels.py \
    --config config.56.json --department 56 \
    --image-dir /home/ennhirim/Bureau/bretagne/56/images_extracted \
    --id-from-filename \
    --segformer-py "/home/ennhirim/Bureau/backup__ (2)/seg_models/pv_panel_models/vit_model/segformer_model.py" \
    --checkpoint   "/home/ennhirim/Bureau/backup__ (2)/seg_models/experiments/finetune_grid_search/checkpoints/segformer_b0_cfg21_best.pth" \
    --save-masks  /home/ennhirim/Bureau/bretagne/56/data/panel_segmentation/masks \
    --output-dir  /home/ennhirim/Bureau/bretagne/56/data/panel_segmentation

# Stage 2b
python3 stage2_panels.py     --config config.56.json --departments 56
```

Stage 1 is the only one that takes hours. The other three together run in
under five minutes.

## Run summary (for the record)

| Stage | Started      | Finished     | Wall-clock | Result |
|-------|--------------|--------------|------------|---|
| 0     | 2026-05-11 11:02 | 2026-05-11 11:03 | 7 s | 8,950 PV+ buildings, 450 tiles |
| 1     | 2026-05-11 11:09 | 2026-05-11 12:09 | ~1 h, with IGN throttling | 448/450 tiles, 2 failed |
| 2a    | 2026-05-11 12:15 | 2026-05-11 12:18 | 150 s | 10,882 panels detected |
| 2b    | 2026-05-11 13:40 | 2026-05-11 13:41 | 13 s | 10,882 rows, 11.6 % with tilt |
