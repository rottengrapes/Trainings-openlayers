# Module 7 - MBTiles, Offline GIS & Wrap-up

Welcome to Module 7! Throughout this course, we have built web GIS applications that stream data across live networks from GeoServer, PostGIS, and tile servers. In this concluding module, we examine **offline Web GIS architectures**, tile archive containers (**MBTiles** and **PMTiles**), data distribution strategies, and review the end-to-end Web GIS technology stack.

---

## 1. Why Offline Web GIS?

### 1.1 Connected vs. Offline Architectures
Standard Web GIS applications assume continuous, high-bandwidth internet connectivity. Offline GIS is engineered for environments where network access is unreliable, intermittent, or completely absent:

```text
Connected Web GIS Architecture

Browser Client (OpenLayers)
             ↓  (HTTP / Internet)
  GeoServer / Cloud Tile Server
             ↓  (Database Queries)
    Enterprise PostGIS Database
```

versus:

```text
Offline Web GIS Architecture

Field Device (Tablet / Laptop)
             ↓
     Local File Storage / SQLite
             ↓
  Local Tile Reader / Service Worker
             ↓
    OpenLayers Engine (Browser Viewport)
```

### 1.2 Common Offline Use Cases

- **Field Data Collection:** Environmental surveys, forestry inventories, and agricultural soil sampling in remote wilderness areas.
- **Disaster Response & Emergency Management:** Search-and-rescue teams operating after earthquakes or hurricanes where telecommunications infrastructure is destroyed.
- **Maritime & Aviation:** Ships and aircraft operating outside terrestrial cellular networks where satellite bandwidth is cost-prohibitive.
- **Defense & High-Security Facilities:** Air-gapped command centers and secure facilities where external internet connectivity is strictly prohibited.

---

## 2. What are MBTiles?

### 2.1 The MBTiles Concept
**MBTiles** is an open specification created by Mapbox for packaging thousands or millions of map tiles into a single portable container. 

Instead of copying millions of individual image files across a filesystem (which causes file-system thrashing and slow transfer speeds), MBTiles stores the entire tile pyramid inside a single **SQLite 3 database** file (`.mbtiles`).

### 2.2 Tile Grid & Coordinate Pyramid

MBTiles structures data according to standard `{z}/{x}/{y}` slippy map tile coordinates:

- **`z` (Zoom level):** Pyramid level ($0$ = whole world in one tile; $18$ = building level).
- **`x` (Column):** Horizontal tile index from West to East.
- **`y` (Row):** Vertical tile index.

> **Note on TMS vs. XYZ Coordinate Conventions:** Standard web XYZ tiles count $y=0$ from the **North** (top-down). MBTiles follows the OSGeo **TMS (Tile Map Service)** specification, which counts $y=0$ from the **South** (bottom-up). Converting between them is straightforward:
>
> $$\text{tile\_row}_{\text{TMS}} = (2^z - 1) - \text{tile\_row}_{\text{XYZ}}$$

### 2.3 Internal Structure of an `.mbtiles` File

Inside the SQLite file, MBTiles utilizes two primary tables:

- **`metadata` Table:** Stores key-value configuration pairs:
  - `name`: Human-readable dataset title.
  - `format`: `png`, `jpg`, `webp`, or `pbf`.
  - `bounds`: Bounding box coordinates (`minLon, minLat, maxLon, maxLat`).
  - `minzoom` & `maxzoom`: Supported zoom range.
  - `json`: Vector layer schema definitions (for vector tiles).
- **`tiles` Table:** Stores the raw tile binaries:
  - `zoom_level` (INTEGER)
  - `tile_column` (INTEGER)
  - `tile_row` (INTEGER)
  - `tile_data` (BLOB): Binary PNG/JPEG image or gzipped MVT Protobuf.

---

## 3. Raster vs. Vector MBTiles

MBTiles packages both pre-rendered raster imagery and raw vector tiles:

| Architectural Aspect | Raster MBTiles | Vector MBTiles |
|---|---|---|
| **Underlying Content** | Bitmap images (`.png`, `.jpg`, `.webp`) | Mapbox Vector Tiles (`.pbf` / MVT) |
| **Styling Location** | Pre-rendered on server before export | Dynamic client-side styling in OpenLayers |
| **Typical File Size** | Large (often 1 GB – 50 GB+) | Compact (typically 10%–20% of raster size) |
| **Client Interactivity** | None (pixels only) | Full feature hover, click, and attributes |
| **Visual Quality** | Pixelated when zoomed past max zoom | Sharp, resolution-independent vector rendering |
| **OpenLayers Layer** | `ol/layer/Tile` with `ol/source/XYZ` | `ol/layer/VectorTile` with `ol/source/VectorTile` |

```text
Raster MBTiles:  [ Satellite / OSM Pixels ] ──▶ Displayed Directly on Canvas
Vector MBTiles:  [ Raw Points, Lines, Polys ] ──▶ Styled Dynamically by OpenLayers in Browser
```

---

## 4. How MBTiles Are Created & Hands-On CLI Commands

Creating MBTiles involves slicing GIS datasets into a multi-zoom tile pyramid and compressing them into an SQLite database:

```text
Raw Spatial Data (PostGIS / Shapefile / GeoTIFF)
                       ↓
         Tile Generation Software
                       ↓
              .mbtiles Archive
                       ↓
           Offline Field Application
```

### 4.1 Common Tile Generation Tools

- **Tippecanoe (Mapbox / Felt):** The industry standard CLI for creating vector MBTiles from GeoJSON. Automatically simplifies line vertices, drops minor features at low zoom levels, and balances tile density to prevent browser overdraw.
- **GDAL (`gdal_translate` & `gdaladdo`):** Powerful command-line utility for converting georeferenced raster imagery (GeoTIFFs, ECW) into raster MBTiles.
- **QGIS:** Features a built-in processing tool: **Generate XYZ tiles (MBTiles)**. Exports any configured QGIS map canvas into an MBTiles file.
- **MapTiler Engine:** Commercial desktop application offering a graphical workflow for generating high-performance raster and vector tile packages.

### 4.2 Commands to Generate Vector MBTiles (Tippecanoe)

#### 1. Installation:
```bash
# macOS (via Homebrew)
brew install tippecanoe

# Ubuntu / Debian
sudo apt-get install tippecanoe
```

#### 2. Basic Conversion from GeoJSON:
Convert a single Natural Earth GeoJSON file into vector MBTiles with zoom levels 0 through 14:
```bash
tippecanoe -o roads.mbtiles -z 14 -Z 0 ne_10m_roads.geojson
```

#### 3. Production Multi-Layer Generation:
Package multiple Natural Earth vector files (roads, populated places, and country boundaries) into a single `.mbtiles` archive with specific layer names and automated density optimization:
```bash
tippecanoe -o natural_earth_basemap.mbtiles \
  --maximum-zoom=14 \
  --minimum-zoom=2 \
  --layer=roads:ne_10m_roads.geojson \
  --layer=places:ne_10m_populated_places.geojson \
  --layer=countries:ne_10m_admin_0_countries.geojson \
  --drop-densest-as-needed \
  --extend-zooms-if-still-dropping \
  --force
```

#### Key Tippecanoe Flags Explained:
- `-o <file.mbtiles>`: The output MBTiles filename.
- `-z <max-zoom>` / `--maximum-zoom`: Highest zoom level generated (e.g., `14`).
- `-Z <min-zoom>` / `--minimum-zoom`: Lowest zoom level generated (e.g., `2`).
- `-l <name>:<file>` / `--layer`: Assigns a named vector layer identifier within each vector tile (`.pbf`).
- `--drop-densest-as-needed`: Dynamically simplifies and omits excess features at low zoom levels to guarantee tiles remain under the standard 500 KB browser limit.
- `--force`: Overwrites the output file if it already exists.

---

### 4.3 Commands to Generate Raster MBTiles (GDAL)

For Natural Earth shaded relief, orthophotos, or digital elevation models (DEMs) stored as GeoTIFF files, use GDAL:

#### 1. Convert Natural Earth GeoTIFF to Raster MBTiles:
```bash
# Convert Natural Earth Shaded Relief GeoTIFF to an MBTiles container
gdal_translate -of MBTILES \
  -co TILE_FORMAT=PNG \
  -co ZOOM_LEVEL_STRATEGY=UPPER \
  NE1_50M_SR_W.tif natural_earth_relief.mbtiles
```

#### 2. Generate Pyramid Zoom Levels (Overviews):
A newly translated MBTiles file only contains the native resolution zoom level. Build lower zoom overviews using `gdaladdo`:
```bash
# Generate overview levels using bilinear interpolation
gdaladdo -r bilinear natural_earth_relief.mbtiles 2 4 8 16 32 64
```

#### 3. Compressing Photographic Imagery with JPEG:
When packaging large raster imagery where file size is critical, use JPEG compression:
```bash
gdal_translate -of MBTILES \
  -co TILE_FORMAT=JPEG \
  -co QUALITY=85 \
  NE1_50M_SR_W.tif natural_earth_relief_jpeg.mbtiles

gdaladdo -r average natural_earth_relief_jpeg.mbtiles 2 4 8 16 32
```

---

### 4.4 Inspecting MBTiles Metadata via SQLite CLI

Because `.mbtiles` is a standard SQLite database, you can inspect it directly using `sqlite3`:

```bash
# 1. View metadata table (format, bounds, zoom levels, layers)
sqlite3 city_basemap.mbtiles "SELECT * FROM metadata;"

# 2. Count total tiles per zoom level
sqlite3 city_basemap.mbtiles "SELECT zoom_level, count(*) AS tile_count FROM tiles GROUP BY zoom_level;"
```

---

### 4.5 File Size & Zoom Considerations
Because every zoom level quadruples the number of tiles ($4^z$), export bounds and zoom levels must be selected deliberately:

```text
Zoom Level 0: 1 tile
Zoom Level 5: 1,024 tiles
Zoom Level 10: 1,048,576 tiles
Zoom Level 15: 1,073,741,824 tiles (Over 1 billion tiles!)
```

> **Best Practice:** Limit high zoom levels (e.g., zoom 16–18) strictly to target cities or areas of interest, using lower zoom levels (e.g., zoom 0–12) for regional context.

---

## 5. Using MBTiles with OpenLayers

### 5.1 The Browser Security Sandbox
**OpenLayers cannot directly open a `.mbtiles` file from a user's hard drive.** Web browsers enforce strict security sandboxing that prevents client-side JavaScript from executing arbitrary SQLite file operations against local disk paths.

To consume MBTiles in OpenLayers, an intermediate tile provider is required:

```text
MBTiles (.mbtiles file)
          ↓
  Local Tile Reader / Service
          ↓
  HTTP Tile URL: http://localhost:8080/{z}/{x}/{y}.pbf
          ↓
  OpenLayers (XYZ / VectorTile Source)
```

### 5.2 Common Integration Approaches

- **Lightweight Local Tile Server:** A background binary (such as `mbtiles-server`, TileServer GL, or a small Python/Go daemon) runs locally, reads the SQLite file, and serves standard REST endpoints (`http://localhost:8080/tiles/{z}/{x}/{y}.png`).
- **Packaged Desktop Application (Electron):** Node.js runs with native OS filesystem access, using `better-sqlite3` to read tile blobs from the `.mbtiles` file and serving them to the OpenLayers frontend via custom protocols (`app://tiles/{z}/{x}/{y}`) or IPC channels.
- **Packaged Mobile Application (Capacitor / Cordova):** Native mobile plugins query the SQLite database stored in device flash memory and stream base64 or blob URLs to OpenLayers inside a Web View.
- **WebAssembly SQLite (`sql.js`):** Loads small MBTiles files into browser memory using WebAssembly. Suitable only for small datasets (< 100 MB) due to browser RAM limits.

---

## 6. Offline Web GIS Architecture

Designing a robust offline Web GIS requires coordinating storage, service workers, and local engines:

```text
                      Offline Client Device
                                │
        ┌───────────────────────┼───────────────────────┐
        ↓                       ↓                       ↓
  Service Worker            IndexedDB              Local SQLite
(Tile HTTP Cache)       (Survey Features)       (Pre-packaged MBTiles)
        │                       │                       │
        └───────────────────────┼───────────────────────┘
                                ↓
                        OpenLayers Engine
```

### 6.1 Storage Tiers in the Browser

- **Service Workers & Cache API:** Intercepts outgoing HTTP requests. When online, requests are cached; when offline, previously viewed tiles are served directly from the browser cache.
- **IndexedDB:** In-browser NoSQL database. Ideal for storing user-digitized survey features, GPS tracks, and attribute edits before synchronizing with the central server.
- **Local SQLite / Native Filesystem:** Used in packaged desktop/mobile wrappers to store gigabytes of base maps and aerial imagery.

---

## 7. PMTiles – Modern Tile Archives & CLI Commands

While MBTiles is the traditional standard, **PMTiles** is the modern cloud-native and serverless standard for web map tile distribution.

```text
Single PMTiles File (Hosted on S3 / CDN / Static Server)
                        ↓
            HTTP Byte-Range Requests
                        ↓
    Browser (pmtiles JavaScript client library)
                        ↓
         OpenLayers (VectorTile / Tile Source)
```

### 7.1 What is PMTiles?

**PMTiles** (developed by Protomaps) is a single-file tile archive format based on standard **HTTP Range Requests**:

- **Serverless Tile Delivery:** PMTiles files can be hosted on standard object storage (Amazon S3, Cloudflare R2, Google Cloud Storage, or a basic Nginx static server). **No GeoServer, Node.js server, or database daemon is required.**
- **HTTP Range Requests:** The browser's PMTiles client library reads the archive's internal directory header and issues HTTP requests fetching only the specific byte offsets for the desired tile:
  ```text
  GET /maps/city.pmtiles
  Range: bytes=1048576-1081344
  ```
- The storage server returns only the requested 32 KB tile.
- **Works in Browsers:** Because browsers natively support HTTP Range Requests, OpenLayers can read directly from a remote PMTiles archive over the web without server-side rendering pipelines.

---

### 7.2 Commands to Generate PMTiles

#### Installing the PMTiles CLI:
```bash
# macOS (Homebrew)
brew install pmtiles

# Or via npm globally
npm install -g pmtiles
```

#### Method A: Convert an Existing MBTiles File to PMTiles
If you already have an MBTiles SQLite file (vector or raster), convert it in seconds:
```bash
pmtiles convert natural_earth_basemap.mbtiles natural_earth_basemap.pmtiles
```

#### Method B: Generate PMTiles Directly from GeoJSON with Tippecanoe
Modern releases of Tippecanoe can write directly to `.pmtiles` without an intermediate MBTiles file:
```bash
tippecanoe -o natural_earth.pmtiles \
  --maximum-zoom=14 \
  --minimum-zoom=2 \
  --layer=roads:ne_10m_roads.geojson \
  --layer=places:ne_10m_populated_places.geojson \
  --layer=countries:ne_10m_admin_0_countries.geojson \
  --drop-densest-as-needed \
  --extend-zooms-if-still-dropping \
  --force
```

---

### 7.3 Inspecting & Verifying PMTiles

Use the `pmtiles` CLI to inspect archive headers and verify data integrity:

```bash
# 1. View metadata header, bounds, zoom levels, and format
pmtiles show city_map.pmtiles

# 2. Cryptographically verify the integrity of all internal tile offsets
pmtiles verify city_map.pmtiles
```

---

### 7.4 Serving PMTiles Locally for Development

The `pmtiles` CLI includes a zero-configuration local HTTP tile server with built-in CORS support:

```bash
# Serve the PMTiles archive locally on port 8080
pmtiles serve city_map.pmtiles --cors=* --port=8080
```
Tiles are immediately available at standard XYZ endpoints:
```text
http://localhost:8080/city_map/{z}/{x}/{y}.pbf
```

---

### 7.5 Consuming PMTiles Directly in OpenLayers

OpenLayers can consume PMTiles directly using the official `pmtiles` npm package (`npm install pmtiles`) via custom tile loading functions:

```javascript
import VectorTileLayer from 'ol/layer/VectorTile.js';
import VectorTileSource from 'ol/source/VectorTile.js';
import MVT from 'ol/format/MVT.js';
import * as pmtiles from 'pmtiles';

// 1. Initialize PMTiles instance pointing to remote S3 or local URL
const p = new pmtiles.PMTiles('https://r2.example.com/maps/city_map.pmtiles');

// 2. Wrap PMTiles byte fetching inside an OpenLayers VectorTile source
const pmtilesSource = new VectorTileSource({
  format: new MVT(),
  tileLoadFunction: (tile, url) => {
    tile.setLoader((extent, resolution, projection) => {
      const [z, x, y] = tile.getTileCoord();
      p.getZxy(z, x, y).then((res) => {
        if (res && res.data) {
          const format = tile.getFormat();
          const features = format.readFeatures(res.data, {
            extent: extent,
            featureProjection: projection
          });
          tile.setFeatures(features);
        }
      });
    });
  }
});

const pmtilesLayer = new VectorTileLayer({
  source: pmtilesSource
});

map.addLayer(pmtilesLayer);
```

---

## 8. MBTiles vs. PMTiles vs. XYZ

| Feature | MBTiles | PMTiles | Traditional XYZ Tiles |
|---|---|---|---|
| **Storage Format** | SQLite database file | Single archive file | Millions of separate files in folders |
| **File Portability** | Single portable file | Single portable file | Poor (unzipping millions of files is slow) |
| **Server Requirement** | Requires tile server or SQLite reader | **No server required** (static S3/R2 storage) | Requires standard web server |
| **Browser Direct Access** | Requires middleware/Electron | **Direct via HTTP Range Requests** | Direct via standard HTTP GET |
| **Offline Desktop/Mobile**| Excellent | Excellent | Moderate (cache management complex) |
| **Dynamic Vector Styling**| Yes (with Vector MBTiles) | Yes (with Vector PMTiles) | Yes (with MVT tiles) |
| **Update Mechanism** | Replace entire `.mbtiles` file | Replace single `.pmtiles` file | Overwrite individual tile files |

---

## 9. Offline Data Limitations & Challenges

While offline mapping provides essential resilience, systems engineers must plan for significant trade-offs:

- **Storage Limits:** A multi-zoom vector or raster tile package can quickly exceed mobile device flash memory limits.
- **Stale Data:** Without live connections, field users operate on static snapshots. Infrastructure changes or new survey edits made by other teams are not visible until synchronization.
- **Two-Way Synchronization Complexity:** When field users digitize or edit features offline, reconciling those edits with the central PostGIS database upon reconnecting requires conflict resolution rules (e.g., *last-write-wins* vs. *manual review*).
- **Search & Geocoding Constraints:** Address geocoding and road routing engines typically rely on massive database indexes. Providing offline routing requires embedded routing engines (e.g., Valhalla or OSRM) bundled locally.

---

## 10. Choosing the Right Web GIS Delivery Method

Throughout this course, we have covered diverse spatial protocols and data structures. Use this decision matrix when designing real-world Web GIS systems:

| Requirement / Scenario | Recommended Solution | Architecture Path |
|---|---|---|
| Small vector dataset (< 5,000 features) | **GeoJSON** | Local file / API $\rightarrow$ `VectorSource` |
| Large vector dataset (> 20,000 features) | **Vector Tiles (MVT)** | Tile Server $\rightarrow$ `VectorTileLayer` |
| Enterprise spatial database with live queries | **PostGIS** | PostgreSQL $\rightarrow$ Spatial SQL $\rightarrow$ GeoServer |
| Centralized enterprise map publishing | **GeoServer** | PostGIS $\rightarrow$ GeoServer $\rightarrow$ WMS / WFS |
| Server-rendered maps with dense labels | **WMS (`ImageWMS`)** | GeoServer SLD $\rightarrow$ OpenLayers ImageLayer |
| Fast global base maps | **XYZ / WMTS** | Cached Tile Service $\rightarrow$ OpenLayers TileLayer |
| In-browser raster analysis & elevation | **GeoTIFF + WebGL** | Cloud-Optimized GeoTIFF $\rightarrow$ `WebGLTileLayer` |
| Interactive 3D terrain, buildings & meshes | **3D Tiles + CesiumJS** | Cesium ion / 3D Tiles $\rightarrow$ `ol-cesium` |
| Serverless cloud-hosted tile archive | **PMTiles** | Cloud Storage (S3/R2) $\rightarrow$ HTTP Range Requests |
| Fully offline field mapping | **MBTiles / PMTiles** | Local SQLite / Archive $\rightarrow$ OpenLayers |

---

## 11. Complete Web GIS Architecture Review

The diagram below synthesizes the complete enterprise Web GIS ecosystem covered across Modules 1 through 7:

```text
                          DATA STORAGE TIER
                                  │
          ┌───────────────────────┼───────────────────────┐
          ↓                       ↓                       ↓
   PostGIS Database           Flat Files            Raster Archives
  (Spatial Tables)       (GeoJSON, Shapefiles)     (GeoTIFF, Elevation)
          │                       │                       │
          └───────────────────────┼───────────────────────┘
                                  ↓
                        MIDDLEWARE & SERVICES
                                  │
             ┌────────────────────┴────────────────────┐
             ↓                                         ↓
         GeoServer                            Tile Storage / CDN
     (WMS, WFS, WMTS)                        (XYZ, MVT, PMTiles)
             │                                         │
             └────────────────────┬────────────────────┘
                                  ↓ (HTTP / Web Services)
                        CLIENT APPLICATION TIER
                                  │
                    ┌─────────────┴─────────────┐
                    ↓                           ↓
             OpenLayers (2D)             CesiumJS (3D)
         (Vector, Raster, Styling,        (Terrain, 3D Tiles,
          Overlays, Interactions)          ol-cesium Bridge)
                    │                           │
                    └─────────────┬─────────────┘
                                  ↓
                             END USERS
                     (Browser, Mobile, Desktop)

OFFLINE WORKFLOW:
PostGIS / Files ──▶ Tile Generator ──▶ MBTiles / PMTiles ──▶ Local OpenLayers
```

---

### Reference resources

- **Mapbox MBTiles Specification:** [https://github.com/mapbox/mbtiles-spec](https://github.com/mapbox/mbtiles-spec)
- **PMTiles Specification & Documentation:** [https://docs.protomaps.com/pmtiles/](https://docs.protomaps.com/pmtiles/)
- **Tippecanoe Tile Generation Tool:** [https://github.com/felt/tippecanoe](https://github.com/felt/tippecanoe)
- **OpenLayers VectorTileLayer API:** [https://openlayers.org/en/latest/apidoc/module-ol_layer_VectorTile-VectorTileLayer.html](https://openlayers.org/en/latest/apidoc/module-ol_layer_VectorTile-VectorTileLayer.html)

### Practical activity

**Your Task:**

Apply the architectural concepts from throughout the course to evaluate the following real-world system scenarios.

#### Scenario 1 — High-Traffic Municipal Web Portal
A city government requires an interactive public web map displaying parcels, zoning classifications, roads, and building footprints. The portal will experience over 50,000 daily visitors.
- **Questions to Answer:**
  1. What data storage tier would you select?
  2. Would you serve the road and parcel layers as raw GeoJSON, dynamic WFS, server-rendered WMS, or Vector Tiles? Why?
  3. How would you prevent database overload during traffic spikes?

#### Scenario 2 — Wilderness Field Survey Application
An environmental agency conducts vegetation surveys in a remote national park with zero cellular coverage. Field technicians carry Android tablets running a Web GIS application to record rare plant locations.
- **Questions to Answer:**
  1. How should the base map imagery be packaged and stored on the tablets?
  2. When a field technician digitizes a new plant location, where should the feature coordinates be saved on the device?
  3. What workflow should occur when the technician returns to headquarters and reconnects to Wi-Fi?

#### Scenario 3 — 3D Urban Planning & Shadow Analysis
An architectural firm needs an interactive web viewer to evaluate the shadow impact of a proposed 40-story skyscraper on surrounding parks across different seasons.
- **Questions to Answer:**
  1. Which visualization engine should be selected (pure OpenLayers or CesiumJS via `ol-cesium`)?
  2. In what format should the citywide 3D buildings and elevation data be streamed?
  3. How does this architecture handle high frame rates for dense urban models?

---

#### Hands-On Architecture Diagram Exercise
In your project notes, sketch or draft an architectural diagram for a Web GIS project of your choice, identifying:
```text
Data Source ──▶ Processing / Tile Generation ──▶ Web Service / Archive ──▶ OpenLayers / Cesium
```

---

#### Example Implementation (`main.js`):
The following production snippet demonstrates how OpenLayers integrates with local tile sources or cloud-native PMTiles using standard XYZ / VectorTile protocols:

```javascript
import Map from 'ol/Map.js';
import View from 'ol/View.js';
import TileLayer from 'ol/layer/Tile.js';
import VectorTileLayer from 'ol/layer/VectorTile.js';
import OSM from 'ol/source/OSM.js';
import XYZ from 'ol/source/XYZ.js';
import VectorTileSource from 'ol/source/VectorTile.js';
import MVT from 'ol/format/MVT.js';
import { fromLonLat } from 'ol/proj.js';
import 'ol/ol.css';

// 1. Online Base Layer: OpenStreetMap
const onlineBaseLayer = new TileLayer({
  source: new OSM(),
  visible: true
});

// 2. Offline / Local Raster Tile Layer (Served via local tile reader / daemon)
// Example endpoint exposed by a local mbtiles server:
const localOfflineRasterLayer = new TileLayer({
  source: new XYZ({
    url: 'http://localhost:8080/services/offline_basemap/tiles/{z}/{x}/{y}.png',
    maxZoom: 16
  }),
  visible: false // Toggled when offline
});

// 3. Offline Vector Tile Layer (High performance, client-styled)
const offlineVectorTileLayer = new VectorTileLayer({
  source: new VectorTileSource({
    format: new MVT(),
    url: 'http://localhost:8080/services/city_vectors/tiles/{z}/{x}/{y}.pbf',
    maxZoom: 14
  }),
  visible: false
});

// 4. Initialize Map
const map = new Map({
  target: 'map',
  layers: [
    onlineBaseLayer,
    localOfflineRasterLayer,
    offlineVectorTileLayer
  ],
  view: new View({
    center: fromLonLat([73.8567, 18.5204]),
    zoom: 12
  })
});

// 5. Connectivity Detection & Layer Switcher
window.addEventListener('offline', () => {
  console.warn('Network connection lost! Switching to offline tile source.');
  onlineBaseLayer.setVisible(false);
  localOfflineRasterLayer.setVisible(true);
  offlineVectorTileLayer.setVisible(true);
});

window.addEventListener('online', () => {
  console.log('Network restored. Reverting to online services.');
  onlineBaseLayer.setVisible(true);
  localOfflineRasterLayer.setVisible(false);
  offlineVectorTileLayer.setVisible(false);
});
```