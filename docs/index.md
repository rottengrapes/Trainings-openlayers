# OpenLayers Phase 2 Classroom Training

**8-hour practical session**

*From OpenLayers fundamentals to a complete Web GIS workflow*

## Purpose of the classroom session

This classroom session builds on the Phase 1 self-study. The objective is not to repeat the introductory material, but to use it in realistic Web GIS workflows: consuming services, connecting OpenLayers to GeoServer and PostGIS, styling and interacting with vector features, integrating CesiumJS, and completing an end-to-end practical exercise.

Participants are expected to arrive with the basic GIS and OpenLayers vocabulary covered in Phase 1. The first module is therefore used to identify gaps and then move quickly into implementation.

## Learning outcomes

By the end of the session, participants should be able to:

- Explain the role of Map, View, Layer, Source and Feature in an OpenLayers application.
- Work with projections, resolution, zoom and coordinate transformation.
- Choose between GeoJSON, WMS, WMTS, XYZ and vector tiles for common Web GIS use cases.
- Understand the PostGIS → GeoServer → OpenLayers enterprise architecture and data flow.
- Consume GeoServer WMS/WFS services from OpenLayers with CQL filtering and GetFeatureInfo.
- Create practical vector styles, including attribute-based styling and labels.
- Implement interactive digitizing (Draw, Modify, Snap) and desktop file Drag & Drop.
- Configure essential map controls (ScaleLine, ZoomSlider, FullScreen, MousePosition, OverviewMap).
- Understand CesiumJS and use ol-cesium to connect 2D and 3D views.
- Understand raster/vector MBTiles and their role in tiled/offline Web GIS.
- Troubleshoot common Web GIS problems using browser developer tools.
- Integrate the day's concepts into a small working Web GIS application.

## Prerequisites

- A working laptop with the Phase 1 OpenLayers project.
- Node.js and npm installed and verified.
- A code editor such as VS Code and a modern browser.
- The Phase 1 GeoJSON exercise available locally.
- Basic JavaScript knowledge.
- Basic GIS, CRS/projection and web-service concepts from Phase 1.
- Three questions or problems encountered during self-study.

## Training Datasets & Free Data Sources

Throughout this course, all hands-on exercises and practical modules standardize on **[Natural Earth](https://www.naturalearthdata.com/)** vector and raster datasets. Natural Earth is public domain, globally consistent, lightweight, and comes with rich, clean attributes.

### 1. Primary Course Vector Datasets (Natural Earth)
Download the following vector packages (1:10m or 1:50m scale) to use across Modules 2 through 8:

| Dataset | Geometry | Attributes of Interest | Direct Download |
|---|---|---|---|
| **Populated Places** (`ne_10m_populated_places`) | Points | `NAME`, `ADM0NAME`, `POP_MAX`, `FEATURECLA` (Admin-0 capital, city) | [Download ZIP (1.6 MB)](https://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/cultural/ne_10m_populated_places.zip) |
| **Roads & Highways** (`ne_10m_roads`) | LineStrings | `name`, `type` (Major Highway, Secondary), `scalerank`, `expressway` | [Download ZIP (15.5 MB)](https://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/cultural/ne_10m_roads.zip) |
| **Admin 0 Countries** (`ne_10m_admin_0_countries`) | Polygons | `NAME`, `CONTINENT`, `POP_EST`, `GDP_MD`, `ISO_A2` | [Download ZIP (5.3 MB)](https://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/cultural/ne_10m_admin_0_countries.zip) |
| **Admin 1 States & Provinces** (`ne_10m_admin_1_states_provinces`) | Polygons | `name`, `admin`, `type_en`, `postal` | [Download ZIP (14.2 MB)](https://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/cultural/ne_10m_admin_1_states_provinces.zip) |
| **Rivers & Lakes** (`ne_10m_rivers_lake_centerlines`) | LineStrings | `name`, `scalerank`, `featurecla` | [Download ZIP (1.8 MB)](https://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/physical/ne_10m_rivers_lake_centerlines.zip) |

> **Converting Shapefiles to GeoJSON:** Natural Earth downloads contain ESRI Shapefiles (`.shp`). To convert them into `.geojson` for OpenLayers or Tippecanoe, use either `ogr2ogr` or [Mapshaper (Web GUI)](https://mapshaper.org/):
> ```bash
> ogr2ogr -f GeoJSON -t_srs EPSG:4326 ne_10m_populated_places.geojson ne_10m_populated_places.shp
> ogr2ogr -f GeoJSON -t_srs EPSG:4326 ne_10m_roads.geojson ne_10m_roads.shp
> ogr2ogr -f GeoJSON -t_srs EPSG:4326 ne_10m_admin_0_countries.geojson ne_10m_admin_0_countries.shp
> ```

---

### 2. Primary Course Raster Datasets
For raster tiling (GDAL, MBTiles/PMTiles in Module 7) and GeoServer raster stores:

- **Natural Earth Cross-Blended Hypsometric Tints with Shaded Relief (GeoTIFF):** [Download 1:50m Raster (21.7 MB)](https://www.naturalearthdata.com/http//www.naturalearthdata.com/download/50m/raster/NE1_50M_SR_W.zip)
- **Natural Earth Gray Shaded Relief (GeoTIFF):** [Download 1:50m Shaded Relief (10.3 MB)](https://www.naturalearthdata.com/http//www.naturalearthdata.com/download/50m/raster/SR_50M.zip)

---

### 3. Recommended Free Vector & Raster Data Portals

When building your own Web GIS projects after this training, bookmark these free data repositories:

| Resource | Data Type | Best For | Link |
|---|---|---|---|
| **Natural Earth** | Vector & Raster | Global cartographic basemaps, boundaries, cultural and physical vectors | [naturalearthdata.com](https://www.naturalearthdata.com/downloads/) |
| **GeoBoundaries** | Vector (Boundaries) | Authoritative open political boundaries (ADM0 through ADM5) | [geoboundaries.org](https://www.geoboundaries.org/) |
| **Geofabrik OSM Extracts** | Vector (OSM) | Pre-packaged OpenStreetMap regional extracts (.osm.pbf, Shapefile) | [download.geofabrik.de](https://download.geofabrik.de/) |
| **Overpass Turbo** | Vector (OSM) | Interactive query builder to filter and export OSM features directly as GeoJSON | [overpass-turbo.eu](https://overpass-turbo.eu/) |
| **USGS EarthExplorer** | Raster (Satellite / DEM) | Free Landsat, Sentinel-2, and SRTM global elevation models | [earthexplorer.usgs.gov](https://earthexplorer.usgs.gov/) |
| **OpenTopography** | Raster (Elevation) | High-resolution Digital Elevation Models (COP30, NASADEM, LiDAR) | [opentopography.org](https://opentopography.org/) |
| **NASA Earthdata / GIBS** | Raster (Environmental) | Near real-time satellite imagery, atmospheric, and climate data | [earthdata.nasa.gov](https://www.earthdata.nasa.gov/) |
| **World Bank Open Data** | Spatial / Tabular | Country-level indicators (GDP, healthcare, energy) joinable to country vectors | [data.worldbank.org](https://data.worldbank.org/) |

## Full-day schedule

| Time | Duration | Module | Primary focus | Output |
|---|---:|---|---|---|
| 09:00–09:45 | 45 min | OpenLayers Fundamentals & Architecture | Map, View, projections, layers, sources, resolution | Modify and explain application structure |
| 09:45–10:45 | 60 min | GIS Data & Web Services | GeoJSON, WMS, WMTS, XYZ, vector tiles | Choose an appropriate service/data mechanism |
| 10:45–12:30 | 105 min | GeoServer & PostGIS Enterprise Integration | Spatial SQL, publishing, WMS/WFS, CQL filtering, GetFeatureInfo | Enterprise spatial data pipeline |
| 12:30–13:30 | 60 min | Lunch Break | — | — |
| 13:30–14:30 | 60 min | Vector Styling & Feature Interaction | Styles, labels, thematic classification, selection, popups | Interactive styled vector layer |
| 14:30–15:30 | 60 min | Map Interactions & Controls | Draw, Modify, Drag & Drop, ScaleLine, FullScreen, MousePosition | Digitizing & control suite application |
| 15:30–16:30 | 60 min | CesiumJS + OpenLayers Integration | 3D globe, terrain, camera, entities, ol-cesium | 2D/3D workflow |
| 16:30–17:15 | 45 min | MBTiles + Wrap-up | Raster/vector MBTiles, tiling, offline concepts | Understand tiled/offline workflow |
| 17:15–18:00 | 45 min | Integrated Practical Exercise & Q&A | End-to-end implementation and troubleshooting | Working mini Web GIS |