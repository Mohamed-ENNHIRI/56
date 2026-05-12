# Rooftop PV detection — Brittany pilot

Web presentation of detected rooftop photovoltaic panels for two Brittany
départements, derived from aerial imagery and LiDAR.

## What you'll see

Open `index.html` (served over HTTP, e.g. via `python3 -m http.server`) and
pick a département. The interactive map then displays:

- Every detected panel as a coloured polygon (the colour encodes the panel's
  azimuth; gray means the orientation could not be measured for that panel).
- Department and commune boundaries on top of an IGN orthophoto basemap.
- The LiDAR-HD tiles used during the analysis, so the un-flown areas are
  visible as blank regions.

Click any panel to see its surface, tilt, azimuth, and confidence. Click any
commune to see its name, INSEE code, and panel count.

## Results

| Metric | **Ille-et-Vilaine (35)** | **Morbihan (56)** |
|---|---:|---:|
| Detected panels                 | **12,591** | **10,882** |
| Buildings carrying ≥1 panel     |  10,636    |   8,950    |
| Total inclined surface          | **694,021 m²** | **564,504 m²** |
| Panels with measured tilt + az  | **87.4 %** | **11.6 %** |
| Aerial imagery vintage          | 2023       | 2024       |
| Cadastre vintage                | April 2023 | January 2025 |

The detection model is identical for both departments. The large gap in
"panels with measured tilt + az" reflects **IGN LiDAR-HD coverage**, not
model performance: Ille-et-Vilaine is almost fully flown, Morbihan has been
flown only on its eastern third (Vannes → Vilaine valley → Pénestin and a
smaller pocket near Mauron). When IGN publishes the missing tiles for
western Morbihan, the Morbihan numbers will catch up automatically.

## How it works in one paragraph

For each PV-positive building identified by an upstream presence-detection
model, we crop a 20 cm aerial image. A fine-tuned SegFormer-B0 segmentation
model produces a binary panel mask, which we vectorise into georeferenced
panel polygons. Each polygon is then matched to a roof plane extracted from
the IGN LiDAR-HD point cloud, giving the true 3D inclined surface, tilt and
azimuth. Where LiDAR is not yet available, we still output the panel
polygon and its 2D surface — only the plane metadata is missing.

## What's in this folder

```
56_pv/
├── index.html              the web presentation (open this)
├── README.md               this file
├── 35/
│   ├── panels_assigned__35.geojson    detected panels with surface + plane info
│   ├── panels__35.csv                 same as above, in tabular form
│   ├── admin_35.geojson               département + commune boundaries
│   └── lidar_tiles_35.geojson         IGN LiDAR-HD tile coverage
└── 56/
    └── … same four files for Morbihan
```

The two GeoJSON files (`panels_assigned__XX.geojson` and `admin_XX.geojson`)
can be opened directly in QGIS, ArcGIS, or any GIS tool that reads GeoJSON.

## Why these two departments

Ille-et-Vilaine (35) and Morbihan (56) were processed first as a Brittany
pilot. Two adjacent departments were chosen so we could compare a fully
LiDAR-covered area (35) with a partially-covered one (56). The same
approach scales to any French département with comparable input data.

## Sources

- IGN BDOrtho 20 cm orthophotos
- IGN LiDAR-HD point clouds
- Cadastre Etalab (building footprints)
- Building-level PV-presence scores (upstream model)
