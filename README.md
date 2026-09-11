# Beyond Ecological Suitability: Integrating Population Exposure and Landscape Context for Nipah Virus Spillover Risk Mapping in South Asia

## 🌐 Interactive Map

### **[→ Launch Interactive Risk Map ←](https://geospatialhealth.github.io/Nipah-Virus-High-Resolution-Risk-Mapping/)**

---

**Overview**

This map provides high-resolution spatial risk assessments for Nipah virus (NiV) transmission in South Asia (Bangladesh, Eastern India) and Southern India. The interactive web map allows researchers, public health officials, and decision-makers to explore risk patterns and identify high-priority areas for surveillance, preparedness, and intervention. The platform integrates ecological suitability modeling, ensemble risk surfaces, historical outbreak data, species occurrence and survey records, and uncertainty estimates into a single, browser-based visualization tool.

# Nipah Virus Risk Mapping — Bangladesh, Eastern & Southern India (Leaflet + GeoRaster)

A single-page, client-side web GIS viewer for interactive Nipah virus risk mapping across Bangladesh / Eastern and South India. This interactive web map visualizes a regional ecological niche model alongside population- and land-cover-informed ensemble risk surfaces and their uncertainty. The goal is to provide a fast, intuitive way to explore where environmental suitability and modeled risk signals are highest, and how these patterns relate to known outbreak locations, bat occurrence/survey data, and geographic context.

## **1) Purpose and scope**

- **Application type:** Single-page, client-side web GIS viewer
- **Primary function:** Interactive Nipah virus risk mapping for Bangladesh / Eastern South Asia and South India, plus a regional ecological niche (suitability) surface
- **Data types supported:**
  - **Raster:** GeoTIFFs (including Cloud Optimized GeoTIFFs: `*_cog.tif`)
  - **Vector:** GeoJSON/JSON polygons and points (`.geojson`, `.json`)

## **2) Runtime environment**

- **Execution model:** Runs entirely in the browser (no server-side code in this repo)
- **Network requirements:**
  - External CDNs for JS/CSS libraries
  - Local/relative `fetch()` for raster/vector files (e.g., `enm_cog.tif`, `outbreaks.json`)
    - Files must be hosted alongside the HTML (same origin) or be CORS-accessible
- **Browser APIs used:**
  - `fetch()` for loading rasters/vectors
  - Clipboard API: `navigator.clipboard.writeText()` with fallback to `document.execCommand('copy')`

## **3) Core libraries and versions (dependencies)**

Loaded via `<script>` tags / CDNs:

- **Leaflet 1.9.4** — map rendering, layer system, controls, popups
- **georaster 1.6.0** — parses GeoTIFF `ArrayBuffer`s in-browser
- **georaster-layer-for-leaflet 3.10.0** — renders parsed georasters as Leaflet layers
- **chroma-js 2.4.2** — color scales for continuous rasters
- **leaflet-draw 1.0.4** — interactive measurement/drawing tools (polyline, polygon, rectangle)

## **4) Map initialization and display stack**

- **Initial view:** `center=[17.329664, 82.880859]`, `zoom=6` (regional view)

### **Basemaps (tile layers)**
All basemaps are served from Esri ArcGIS Online; **Satellite Imagery is the default**:
- Esri World Street Map
- Esri World Imagery (default)
- Esri World Topographic Map

### **Custom Leaflet panes and z-index ordering**
A deliberate cartographic stack ensures rasters don't hide boundaries and vector points/markers stay on top. Each analysis raster is given its own pane so multiple rasters can be stacked predictably:

- `backgroundPane`: **250**
- `rasterPane`: **300**
- Per-raster panes `raster_<id>`: **310–315** (one each for `enm`, `enm_cv`, `basic_risk`, `blend_risk`, `risk_blend_cv`, `diff_blend_minus_basic`)
- `speciesRasterPane`: **320**
- `boundariesPane`: **350** (M-Region outline)
- `speciesVectorPane`: **360** (occurrence/survey points)
- Popups/tooltips render above all of the above (Leaflet defaults)

## **5) Data layers and configuration model**

The script uses separate configuration objects for analysis rasters and vector overlays.

### **A) Raster layers (`layerConfigs`)**
- **Total raster layers:** **6** (`totalLayers = 6`)

#### **2 continuous layers (1 km)**
- `enm` → `enm_cog.tif` — **Ecological Niche Model (1km)**
  - 7-color ramp (gray → dark red), perceptual `lch` interpolation
  - Symbolized with a **±1 standard-deviation stretch** about the mean
  - Legend: continuous gradient bar ("Low Suitability" → "High Suitability")
  - Default raster shown on load
- `enm_cv` → `enm_cv.tif` — **ENM Coefficient of Variation (1km)**
  - 5-color ramp, `lch` interpolation, ±1 SD stretch
  - Legend: continuous gradient bar ("Low Uncertainty" → "High Uncertainty")

#### **2 discrete risk layers**
- `basic_risk` → `risk_basic.tif` — **Baseline Risk (ENM + Pop)**
- `blend_risk` → `risk_blend.tif` — **Landcover Weighted Risk (ENM + Pop + LC)**
  - Discrete palette keyed by integer classes `{0, 1, 2, 3, 4, 6, 9}` (No Risk → Very High Risk)
  - `noDataValue: 15`
  - Legend: checkbox list to toggle individual class visibility

#### **2 discrete diagnostic layers**
- `risk_blend_cv` → `risk_blend_cv.tif` — **Ensemble CV**
  - Discrete classes `{0, 2, 3, 4, 5, 6, 7, 8}` (No Variability → Extreme CV), `noDataValue: 255`
- `diff_blend_minus_basic` → `diff_blend_minus_basic.tif` — **Normalized Difference Surface**
  - Discrete diverging classes `{1…7}` (Large Decrease → No Change → Large Increase), `noDataValue: 15`

Each layer also carries an `infoDescription` string that is surfaced in the on-map description box (see §7).

### **B) Vector overlays**

#### **Species & survey data (`speciesSurveyConfigs`)**
- **GBIF Bat Species Occurrences** — `gbif_bats.json`
  - Point layer rendered as `circleMarker`s, **colored by species** (`Binomial`)
  - Popup fields: full scientific name, country, year collected, coordinates, GBIF link
  - Legend lists each species with a per-species toggle and occurrence count
- **Bat Survey & Phylogeographic Data** — `survey.json`
  - Point layer rendered with **diamond** markers
  - Popup fields: species, district/state/country, collection date, bats sampled (total/male/female), RT-PCR and IgG ELISA results, sample types, test methods, precision, notes, study, source

#### **Outbreak locations**
- **Nipah Outbreak Locations (2001–2026)** — `outbreaks.json`
  - Point layer with a custom concentric-circle SVG marker (always above boundaries)
  - Popup fields: location/admin name, country, year, reference, coordinates
  - On by default

#### **M-Region boundary**
- **M-Region Boundary** — `m_region.json`
  - Outline-only polygon (black stroke, transparent fill), **non-interactive**
  - On load, all features are dissolved into a single geometry used internally for point-in-polygon checks (`pointInMRegion`)
  - On by default

## **6) Rendering logic for rasters (important implementation detail)**

### **Continuous rasters**
- Builds `chroma.scale(config.colors).mode('lch').domain(stretchDomain)`
- For layers with `useSdStretch`, `stretchDomain` is computed as **mean ± 1 SD** of valid pixels (`computeSdDomain`); otherwise the configured `domain` is used
- Per pixel: returns `scale(value).hex()` for valid values, clamped to the stretch domain; returns `null` (transparent) for nodata/NaN

### **Discrete rasters (classed risk)**
- Pixel values are rounded: `Math.round(value)`
- Each class can be toggled via `hiddenClasses[layerId][classValue]`
- If a class is hidden, the pixel returns `null` → transparent
- Enables interactive "masking" of classes without reloading the raster
- A broad nodata guard rejects sentinel values (`255`, `15`, `-9999`, `-32768`, `-3.4e38`, magnitudes `> 1e10`) plus each raster's own `noDataValue`

### **Performance/quality settings**
GeoRasterLayer uses:
- `opacity: 1.0` (default; adjustable per layer)
- `resolution: 256` (render sampling; lower = faster, higher = sharper/heavier)
- A dedicated `pane: 'raster_<id>'` per layer

## **7) UI controls and interactions**

### **Layer Control (top-left)**
A custom Leaflet control containing:
- **Basemap** radio buttons (Esri Street / Satellite / Topographic; Satellite default)
- **Analysis Layers** checkboxes — all six rasters can be toggled independently and stacked (`enm` checked by default)
- **Species & Survey Data** checkboxes (GBIF occurrences, survey data)
- **Overlays** checkboxes (Nipah Outbreak Locations, M-Region Boundary — both on by default)
- **Per-raster opacity sliders** (0–100%, default 100%) calling `layer.setOpacity(opacity)`

### **Legend (bottom-right)**
Updates dynamically for the active layers:
- **Discrete risk/diagnostic:** checkbox list by class (toggle visibility)
- **Continuous:** gradient bar + low/high labels
- **Species:** per-species toggles with counts (GBIF), diamond marker (survey)
- Outbreak-marker key when outbreaks are shown

### **Description box (top-center)**
A floating, resizable panel that shows the `infoDescription` for the topmost active analysis raster (model details, predictor importance, validation statistics, etc.). Hidden when no analysis raster is active.

### **Measurement / drawing tools (leaflet-draw, top-right)**
- Enabled drawing: **polyline**, **polygon**, and **rectangle**
- Disabled: circle, marker, circlemarker
- On creation:
  - Polyline: sums segment distances; popup shows **km**
  - Polygon: geodesic area via `L.GeometryUtil.geodesicArea`; popup shows **km²**
  - Rectangle: approximate bounding-box area; popup shows **km²**
- Includes edit/remove support for drawn shapes

### **Click-to-query**
- Clicking the map queries the active analysis raster at that point:
  - `enm` / `enm_cv` show coordinates only (cell values not meaningful to display)
  - Discrete layers show the class label; other continuous layers show the numeric cell value
- Vector features (occurrences, survey sites, outbreaks) open attribute popups on click

### **Right-click coordinate copy**
- `contextmenu` handler copies `lat,lng` to 6 decimals
- Clipboard API if available; fallback textarea copy
- Temporary on-screen notification

## **8) Layer behavior**

- Analysis rasters are **checkboxes**, so several can be displayed at once; the "current" layer for click-query and the description box is the topmost active one in render order.
- The M-Region boundary is drawn in black and sits above the rasters but below the vector points.
- Toggling a discrete class re-renders only that raster's color function (no re-fetch).

## **9) Loading strategy and sequencing**

- Immediately starts loading **all rasters**: `Object.keys(layerConfigs).forEach(loadLayer)`
- Then loads outbreak points, the M-Region boundary, and the species/survey layers
- Tracks raster progress with `loadedCount / totalLayers`
- First displayed raster is explicitly **`enm`** (added in its own load callback, not first-loaded or alphabetical)

## **10) Error handling and diagnostics**

- `fetch()` failures throw with an HTTP status / "Not found" and log errors to the console
- Missing optional files (e.g., outbreaks, boundary, species) fail gracefully with a console message and do not block the rest of the map
- `updateStatus()` logs to console (CSS exists for `.loading` / `.error`, but no visual widget is wired up)

## **11) Technical assumptions / constraints (implicit specs)**

- Raster and vector files must be reachable via browser `fetch()`:
  - Opening via `file://` often breaks due to browser security/CORS rules
  - Serve via HTTP(S) (local server or hosted site)
- GeoTIFFs must be compatible with in-browser parsing (projection and bounds are read from the file)
- Discrete "risk" rasters are assumed to contain integer-like class values (or values very close to integers)
- Expected data files in the same directory as the HTML:
  - Rasters: `enm_cog.tif`, `enm_cv.tif`, `risk_basic.tif`, `risk_blend.tif`, `risk_blend_cv.tif`, `diff_blend_minus_basic.tif`
  - Vectors: `gbif_bats.json`, `survey.json`, `outbreaks.json`, `m_region.json`
