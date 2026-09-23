# Terrn: Defect-to-Solution Mapping & Architectural Remediation Guide
## Conceptual Fixes for All Defects Cataloged in `Terrn_Detailed_Issues_Catalog.md`, `RCA_and_Competitive_Analysis.md`, and `Terrn_Performance_Optimization_Guide.md` (Cross-Referenced with GeoLibre)

**Document Version:** 1.1.0  
**Date:** August 21, 2026  
**Target Codebase:** `/tern/tern_poc`  
**Reference Benchmark:** `/tern/geolibre`  
**Scope:** Exhaustive 1:1 conceptual mapping of every defect, bottleneck, and architectural gap identified across all audit documents to its precise resolution (no code dumps).

---

## Table of Contents
1. [Core WebGL & Map Rendering Defects (DEF-01 to DEF-07)](#1-core-webgl--map-rendering-defects)
2. [State Management & Reactivity Defects (DEF-08 to DEF-10)](#2-state-management--reactivity-defects)
3. [Feature Picking & Map Interaction Defects (DEF-11 to DEF-12)](#3-feature-picking--map-interaction-defects)
4. [File Ingestion & Memory Overhead Defects (DEF-13 to DEF-15)](#4-file-ingestion--memory-overhead-defects)
5. [Tabular Data & DuckDB Integration Defects (DEF-16 to DEF-17)](#5-tabular-data--duckdb-integration-defects)
6. [Statistical Classification Defects (DEF-18)](#6-statistical-classification-defects)
7. [Advanced Enablers: Binary Formats & 3D Camera Transform Compatibility](#7-advanced-enablers-binary-formats--3d-camera-transform-compatibility)
8. [Comprehensive Master Summary Matrix](#8-comprehensive-master-summary-matrix)

---

## 1. Core WebGL & Map Rendering Defects

### DEF-01: Depth Buffer Z-Fighting & Flickering on Zoom/Pan
* **Root Cause in Terrn:** `MapboxOverlay({ interleaved: true })` places Deck.gl custom layers into MapLibre’s WebGL draw loop ([`base-map.ts:L154`](file:///tern/tern_poc/src/core/canvas/base-map.ts#L154)). 2D polygon fills, boundary strokes, and points share the same depth buffer at elevation $Z = 0.0$. Changing near/far projection clipping planes on zoom/pan cause depth precision rounding fluctuations, leading to high-frequency Z-fighting.
* **How It Can Be Fixed:**
  * **Eliminate Interleaved 2D Depth Testing:** Remove Deck.gl for 2D vector rendering.
  * **Adopt Native 2D Painter's Algorithm:** Render vector layers directly as native MapLibre style layers (`fill`, `line`, `circle`). In native 2D rendering, layers are drawn sequentially from back to front, completely bypassing depth buffer contention and eliminating flickering.
* **GeoLibre Pattern:** GeoLibre uses MapLibre native layers for all 2D vector data ([`layer-sync.ts:L1901`](file:///tern/geolibre/GeoLibre/packages/map/src/layer-sync.ts#L1901)), keeping Deck.gl strictly for specialized 3D elevation point clouds.

---

### DEF-02: Stencil Buffer & Scissor Mask Contamination (Tile Boundary Cropping)
* **Root Cause in Terrn:** MapLibre renders basemaps tile-by-tile, setting WebGL stencil buffers to clip drawing to tile squares ([`renderer.ts:L650`](file:///tern/tern_poc/src/features/layers/rendering/renderer.ts#L650)). Interleaved Deck.gl draws execute without resetting these stencil masks, so Deck.gl’s global geometry gets clipped along arbitrary tile boundaries.
* **How It Can Be Fixed:**
  * **Integrate Within MapLibre's Tile Pipeline:** When vector data is registered as native MapLibre layers or vector tiles, MapLibre manages its own stencil lifecycle automatically per tile, ensuring feature geometries are never clipped by lingering WebGL state.
* **GeoLibre Pattern:** Decomposing layers into native MapLibre sub-layers (`layer-${id}-fill`, `layer-${id}-line`) ensures draw calls follow MapLibre's built-in WebGL state isolation.

---

### DEF-03: WebGL State Pollution & Layer Compositing Bleed-Through
* **Root Cause in Terrn:** Deck.gl shaders modify WebGL blend equations and depth masks during execution and return control to MapLibre without restoring the expected WebGL state ([`base-map.ts:L154`](file:///tern/tern_poc/src/core/canvas/base-map.ts#L154)). Subsequent MapLibre layers (such as road labels) fail to blend correctly.
* **How It Can Be Fixed:**
  * **Single WebGL Pipeline Ownership:** Managing all standard layers inside MapLibre’s style specification ensures MapLibre maintains consistent WebGL blend modes, depth writes, and scissor boxes across all layer passes.
* **GeoLibre Pattern:** Managing all styling inside MapLibre prevents foreign WebGL state leakage into basemap text and symbol layers.

---

### DEF-04: Fragile `beforeId` Resolution & Satellite Basemap Ordering Inversion
* **Root Cause in Terrn:** `getBeforeId` looks for `dark-labels-layer` or `light-labels-layer` ([`renderer.ts:L257`](file:///tern/tern_poc/src/features/layers/rendering/renderer.ts#L257)). On satellite view, no label layer exists, so `getBeforeId` returns `undefined`. Deck.gl groups all layers into an opaque custom unit at the top of the stack, inverting polygon and point order.
* **How It Can Be Fixed:**
  * **Explicit Native Style Tree Placement:** Give every sub-layer a distinct, predictable identifier (`layer-${id}-fill`, `layer-${id}-line`, `layer-${id}-circle`). Use MapLibre’s native `moveLayer(id, beforeId)` API to place each sub-layer deterministically in the style tree regardless of the active basemap.
* **GeoLibre Pattern:** GeoLibre explicitly moves each individual native sub-layer based on user stack ordering and basemap sandwich preferences ([`layer-sync.ts:L415`](file:///tern/geolibre/GeoLibre/packages/map/src/layer-sync.ts#L415)).

---

### DEF-05: Main-Thread CPU Polygon Tessellation (`earcut` in Deck.gl)
* **Root Cause in Terrn:** Deck.gl’s composite `GeoJsonLayer` parses multi-polygon rings and executes CPU-side triangulation (`earcut`) synchronously on the main JavaScript thread ([`renderer.ts:L715`](file:///tern/tern_poc/src/features/layers/rendering/renderer.ts#L715)), locking the UI during file load and style edits.
* **How It Can Be Fixed:**
  * **Worker-Pool Slicing & Native WebGL Triangulation:** Move polygon tessellation off the main thread. MapLibre’s internal Web Worker pool handles geometry processing in background threads, keeping the main UI event loop free.
* **GeoLibre Pattern:** For $\le 50k$ features, MapLibre's internal worker pool slices polygons; for $> 50k$ features, client-side vector tiling indexes polygons without main-thread locking.

---

### DEF-06: Unindexed Full-Dataset Vertex Processing (No Viewport Culling or LOD)
* **Root Cause in Terrn:** Deck.gl uploads all 53,316 points and 2,299 complex polygons to WebGL as unindexed arrays ([`renderer.ts:L651`](file:///tern/tern_poc/src/features/layers/rendering/renderer.ts#L651)). The GPU transforms every vertex on every frame even when zoomed into a single street or zoomed out over the whole country.
* **How It Can Be Fixed:**
  * **Dynamic Client-Side Vector Tiling:** Slice large GeoJSON datasets into vector tiles on the fly (`GeoJSONVT` for polygons/lines and `Supercluster` for points). MapLibre requests and renders only the specific tiles visible in the viewport at the current zoom level, applying automatic Level of Detail (LOD) simplification.
* **GeoLibre Pattern:** GeoLibre registers a custom protocol (`geolibre-gjvt://`) via `@maplibre/vt-pbf` ([`geojson-vt-protocol.ts:L17`](file:///tern/geolibre/GeoLibre/packages/map/src/geojson-vt-protocol.ts#L17)), streaming only visible viewport tiles to the GPU.

---

### DEF-07: CPU-Bound JavaScript Style Accessor Loops per Feature
* **Root Cause in Terrn:** Terrn evaluates styling rules (`getPointFillColor`, `getPointSize`, `getPolygonFillColorValue`) via JavaScript callback functions executed sequentially for all 53k+ features on every state change or redraw ($200,000+$ calls per update) ([`renderer.ts:L286`](file:///tern/tern_poc/src/features/layers/rendering/renderer.ts#L286)).
* **How It Can Be Fixed:**
  * **GPU GLSL Shader Expressions:** Compile style configurations (categories, color ramps, size ranges) into MapLibre GL style expressions (`match`, `step`, `interpolate`). The MapLibre engine compiles these into GPU fragment and vertex shaders, executing color and size calculations directly on the GPU in parallel.
* **GeoLibre Pattern:** GeoLibre translates all styling into declarative MapLibre expressions ([`style-mapper.ts:L37`](file:///tern/geolibre/GeoLibre/packages/map/src/style-mapper.ts#L37)), reducing CPU style evaluation time to $0\text{ ms}$.

---

## 2. State Management & Reactivity Defects

### DEF-08: Continuous Deep-Cloning (`JSON.parse(JSON.stringify)`) on Style Sliders
* **Root Cause in Terrn:** In `layer.store.ts`, dragging a slider or adjusting opacity serializes and deserializes the entire `StyleConfig` object tree via `JSON.parse(JSON.stringify)` on every input event (60+ times/sec) ([`layer.store.ts:L368`](file:///tern/tern_poc/src/features/layers/state/layer.store.ts#L368)), creating heavy garbage collection churn.
* **How It Can Be Fixed:**
  * **Shallow Immutable Updates:** Replace full JSON serialization with shallow object spreads (`{ ...layer, styleConfig: { ...layer.styleConfig, ...patch } }`), mutating only the specific visual property changed.
* **GeoLibre Pattern:** GeoLibre applies lightweight shallow patches in its store ([`store.ts:L1738`](file:///tern/geolibre/GeoLibre/packages/core/src/store.ts#L1738)) without deep stringification.

---

### DEF-09: O(N) WeakMap Discard and Rebuild on Every Style & Visibility Change
* **Root Cause in Terrn:** `layersVersion` increments on every style update ([`layer.store.ts:L370`](file:///tern/tern_poc/src/features/layers/state/layer.store.ts#L370)). Whenever `layersVersion` changes, the Attribute Table invalidates its cache and loops over all 55,000+ features to rebuild an identity `WeakMap` ([`attribute-table-element.ts:L100`](file:///tern/tern_poc/src/features/tabular/components/attribute-table-element.ts#L100)), freezing the UI during slider drags.
* **How It Can Be Fixed:**
  * **Decouple Mutation Versions:** Separate `layersDataVersion` (incremented only when dataset features/geometry change) from `layersStyleVersion` (incremented on visual tweaks). Gate the Attribute Table’s `WeakMap` indexing strictly on `layersDataVersion`.
* **GeoLibre Pattern:** GeoLibre gates expensive derived re-indexing to execute only when raw feature geometry or dataset arrays are explicitly replaced ([`store.ts:L1752`](file:///tern/geolibre/GeoLibre/packages/core/src/store.ts#L1752)).

---

### DEF-10: Monolithic Global Change Notification Cascade (`onLayersChange`)
* **Root Cause in Terrn:** The application relies on a single global `onLayersChange` subscriber list in `main.ts` ([`main.ts:L337-L379`](file:///tern/tern_poc/src/main.ts#L337-L379)) that fires indiscriminately on any state mutation, triggering full Deck.gl layer regeneration, toolbar DOM queries, and DuckDB table iteration simultaneously.
* **How It Can Be Fixed:**
  * **Fine-Grained Targeted Reactive Signals:** Replace global coarse callbacks with granular per-layer signals or targeted subscriptions so that editing one layer's opacity does not trigger recalculations in unrelated toolbar or tabular components.
* **GeoLibre Pattern:** GeoLibre uses Zustand selectors (`useAppStore(s => s.layers[id].style)`), so only components observing the specific slice of state re-render.

---

## 3. Feature Picking & Map Interaction Defects

### DEF-11: Synchronous GPU Pipeline Stalls via `gl.readPixels()` on `mousemove`
* **Root Cause in Terrn:** On every mouse movement, `deckOverlay.pickObject()` renders an offscreen picking framebuffer and issues a synchronous `gl.readPixels()` ([`base-map.ts:L197`](file:///tern/tern_poc/src/core/canvas/base-map.ts#L197)), forcing the CPU to block until the GPU pipeline flushes.
* **How It Can Be Fixed:**
  * **In-Memory Spatial Indexing:** Replace Deck.gl picking with MapLibre’s native `map.queryRenderedFeatures()`. This checks the in-memory R-Tree spatial index of tile geometries in CPU memory without triggering GPU offscreen passes or pipeline readbacks.
* **GeoLibre Pattern:** GeoLibre performs all hover and click inspection using `map.queryRenderedFeatures()` ([`MapCanvas.tsx:L1544, L1903`](file:///tern/geolibre/GeoLibre/packages/map/src/MapCanvas.tsx#L1544)), executing in $< 1\text{ ms}$ with zero frame drops.

---

### DEF-12: Synchronous Main-Thread Grid Clustering Fallback
* **Root Cause in Terrn:** When clustering is enabled, zooming to an uncached zoom level immediately executes `clusterFeatures()` synchronously on the main thread ([`renderer.ts:L579`](file:///tern/tern_poc/src/features/layers/rendering/renderer.ts#L579)), running an $O(N)$ grid-binning loop over 53k features before dispatching to the worker.
* **How It Can Be Fixed:**
  * **Native MapLibre Clustering:** Enable MapLibre's built-in GeoJSON source clustering (`cluster: true`), or use `Supercluster` inside the vector tile worker protocol. All clustering calculations run in background worker threads without blocking the UI.
* **GeoLibre Pattern:** GeoLibre delegates point clustering to `Supercluster` within the tile protocol worker ([`geojson-vt-protocol.ts:L92`](file:///tern/geolibre/GeoLibre/packages/map/src/geojson-vt-protocol.ts#L92)).

---

## 4. File Ingestion & Memory Overhead Defects

### DEF-13: Main-Thread Text Reading & Heap Object Proliferation
* **Root Cause in Terrn:** `FileReader.readAsText()` and `JSON.parse()` execute on the main thread for 20–50 MB files ([`main.ts:L126`](file:///tern/tern_poc/src/main.ts#L126)), creating hundreds of thousands of heap objects and freezing the browser.
* **How It Can Be Fixed:**
  * **Worker-First Ingestion:** Delegate file reading and parsing directly to a Web Worker. The worker decodes the file, extracts bounding boxes, and prepares binary geometry structures off the main thread.
* **GeoLibre Pattern:** GeoLibre parses files and runs spatial conversions inside WebAssembly/Web Worker modules ([`wasm-client.ts:L670`](file:///tern/geolibre/GeoLibre/packages/processing/src/wasm-client.ts#L670)).

---

### DEF-14: Structured Clone Memory Duplication via Web Worker `postMessage`
* **Root Cause in Terrn:** Passing the full parsed GeoJSON object tree from the main thread to `geo.worker.ts` via `postMessage` triggers a deep structured clone ([`geo-worker-client.ts:L16`](file:///tern/tern_poc/src/core/workers/geo-worker-client.ts#L16)), duplicating memory in RAM and causing potential Out-Of-Memory crashes.
* **How It Can Be Fixed:**
  * **Transferable ArrayBuffers (Zero-Copy):** Read file data as raw `ArrayBuffer` and transfer ownership to the worker using Transferable Objects (`postMessage(buffer, [buffer])`), achieving zero-copy memory transfer.
* **GeoLibre Pattern:** GeoLibre transfers raw binary buffers and MVT protobuf byte arrays across thread boundaries with zero structured cloning overhead.

---

### DEF-15: Synchronous In-Thread File Format Decoding (Shapefiles & GeoTIFFs)
* **Root Cause in Terrn:** `parseShapefile` (`shpjs`) and `parseGeoTIFF` (`geotiff`) are executed directly on the main UI thread ([`main.ts:L172-L200`](file:///tern/tern_poc/src/main.ts#L172-L200)), freezing ingest spinner animations while unzipping archives and decoding raster tiles.
* **How It Can Be Fixed:**
  * **Offloaded Multi-Format Worker Decoders:** Execute `shpjs`, `geotiff`, `togeojson`, and CSV parsing entirely inside background Web Workers. The main thread receives only the structured metadata and rendered canvases/buffers.
* **GeoLibre Pattern:** GeoLibre executes all Shapefile and TIFF decoding inside WebAssembly background pipelines.

---

## 5. Tabular Data & DuckDB Integration Defects

### DEF-16: Main-Thread Apache Arrow Table Construction & IPC Serialization
* **Root Cause in Terrn:** For a 53k-feature layer, `tableFromJSON` and `tableToIPC` execute synchronously on the main thread to register DuckDB tables ([`duckdb.ts:L283`](file:///tern/tern_poc/src/services/duckdb.ts#L283)), causing a 1.5–3.0 second UI freeze during layer ingestion.
* **How It Can Be Fixed:**
  * **In-Worker DuckDB Loading:** Move Arrow table creation and IPC streaming into the DuckDB Web Worker, or use DuckDB’s native SQL `read_json_auto()` directly inside the worker thread.
* **GeoLibre Pattern:** In GeoLibre, DuckDB table instantiation and spatial indexing execute entirely within the background worker thread ([`maplibre-duckdb.ts`](file:///tern/geolibre/GeoLibre/packages/plugins/src/plugins/maplibre-duckdb.ts)).

---

### DEF-17: Linear Property Extraction and Object Remapping
* **Root Cause in Terrn:** Helpers in `duckdb.ts` ([`duckdb.ts:L159-L164`](file:///tern/tern_poc/src/services/duckdb.ts#L159-L164)) repeatedly iterate through the `features` array and instantiate mapped object arrays for properties, distinct value lookups, and schema validation.
* **How It Can Be Fixed:**
  * **Direct Worker SQL Queries & Integer Feature IDs:** Assign an integer `__id` to features at ingest time and query distinct values directly via SQL (`SELECT DISTINCT col FROM table LIMIT 256`), eliminating redundant JS property array remapping on the main thread.
* **GeoLibre Pattern:** GeoLibre queries attribute metadata directly from DuckDB worker tables or reads typed property columns on demand.

---

## 6. Statistical Classification Defects

### DEF-18: Synchronous Main-Thread Float Array Sorting for Quantiles
* **Root Cause in Terrn:** Calculating graduated quantile breaks maps and sorts 53,316 floating-point numbers synchronously on the main thread whenever classification settings change ([`numeric.ts:L11`](file:///tern/tern_poc/src/utils/numeric.ts#L11)).
* **How It Can Be Fixed:**
  * **Reservoir Sampling & Worker SQL Aggregations:** When feature count exceeds 5,000, calculate quantile stops on a representative sample of 2,000–5,000 values (accurate within 0.1%), or delegate the quantile calculation to DuckDB WASM's background query engine (`approx_quantile`).
* **GeoLibre Pattern:** GeoLibre calculates quantile break stops using interpolated sampling ([`color-ramp.ts:L328`](file:///tern/geolibre/GeoLibre/packages/core/src/color-ramp.ts#L328)) and applies the resulting stops as GPU `step` expressions.

---

## 7. Advanced Enablers: Binary Formats & 3D Camera Transform Compatibility

### 7.1 Cloud-Native Binary Format Streaming (FlatGeobuf / PMTiles / GeoParquet)
* **Gap in Terrn:** Terrn relies exclusively on text-based GeoJSON and zipped Shapefiles, requiring millions of JavaScript objects to be instantiated in heap memory.
* **How It Can Be Fixed:**
  * Support **FlatGeobuf (`.fgb`)**: Packed Hilbert R-tree indexed binary vector format that can be mapped directly into TypedArrays without intermediate JS objects.
  * Support **PMTiles (`.pmtiles`)**: Single-file vector tile archives that stream only requested tile bytes for active zoom levels and bounding boxes without loading entire multi-gigabyte files into RAM.
* **GeoLibre Pattern:** GeoLibre natively supports streaming PMTiles, FlatGeobuf, and GeoParquet ([`remote-file-formats.ts`](file:///tern/geolibre/GeoLibre/packages/plugins/src/plugins/remote-file-formats.ts)).

### 7.2 Scoped 3D & Deck.gl Camera Transform Compatibility
* **Gap in Terrn:** When 3D mode is enabled, MapLibre camera projection and Deck.gl viewport transforms can lose depth precision or cause altitude mismatches.
* **How It Can Be Fixed:**
  * Synchronize camera altitude, projection transforms, and `nearZ`/`farZ` depth planes between MapLibre and Deck.gl so 3D extrusions and point clouds maintain exact depth precision on pitched views.
* **GeoLibre Pattern:** GeoLibre implements a dedicated camera transform compatibility bridge ([`map-transform-compat.ts:L1-L76`](file:///tern/geolibre/GeoLibre/packages/map/src/map-transform-compat.ts#L1-L76)) that re-aliases `nearZ`/`farZ` getters live per frame.

---

## 8. Comprehensive Master Summary Matrix

| Defect ID | Category | Specific Failure in Terrn | Conceptual Architectural Resolution | GeoLibre Blueprint |
| :--- | :--- | :--- | :--- | :--- |
| **DEF-01** | Rendering | Polygons & points flicker on zoom/pan | Native MapLibre 2D layers via Painter's Algorithm | [`layer-sync.ts:L1901`](file:///tern/geolibre/GeoLibre/packages/map/src/layer-sync.ts#L1901) |
| **DEF-02** | Rendering | Features cropped into tile rectangles | Encapsulate within MapLibre native tile stencil state | [`layer-sync.ts:L2051`](file:///tern/geolibre/GeoLibre/packages/map/src/layer-sync.ts#L2051) |
| **DEF-03** | Rendering | Road labels bleed through polygons | Single WebGL pipeline ownership under MapLibre | [`style-mapper.ts:L37`](file:///tern/geolibre/GeoLibre/packages/map/src/style-mapper.ts#L37) |
| **DEF-04** | Rendering | Polygon/point sub-layer order inverts | Discrete sub-layer IDs with native `moveLayer()` | [`layer-sync.ts:L415`](file:///tern/geolibre/GeoLibre/packages/map/src/layer-sync.ts#L415) |
| **DEF-05** | Rendering | Multi-second UI freeze on polygon load | Worker-pool slicing & native WebGL triangulation | [`types.ts:L703`](file:///tern/geolibre/GeoLibre/packages/core/src/types.ts#L703) |
| **DEF-06** | Rendering | 15–25 FPS lag on 53k+ features | Dynamic client-side vector tiling (`GeoJSONVT` / MVT) | [`geojson-vt-protocol.ts:L17`](file:///tern/geolibre/GeoLibre/packages/map/src/geojson-vt-protocol.ts#L17) |
| **DEF-07** | Rendering | 200k+ CPU JS accessor calls/tick | Compile `StyleConfig` into GPU GLSL shader expressions | [`style-mapper.ts:L37`](file:///tern/geolibre/GeoLibre/packages/map/src/style-mapper.ts#L37) |
| **DEF-08** | State Store | `JSON.parse(JSON.stringify)` GC churn | Shallow immutable store patches | [`store.ts:L1738`](file:///tern/geolibre/GeoLibre/packages/core/src/store.ts#L1738) |
| **DEF-09** | State Store | Slider drag rebuilds 55k `WeakMap` loop | Decouple `layersDataVersion` from `layersStyleVersion` | [`store.ts:L1752`](file:///tern/geolibre/GeoLibre/packages/core/src/store.ts#L1752) |
| **DEF-10** | State Store | Global `onLayersChange` redraw cascade | Targeted granular reactive signals / selectors | [`store.ts:L3`](file:///tern/geolibre/GeoLibre/packages/core/src/store.ts#L3) |
| **DEF-11** | Interaction | `gl.readPixels()` GPU stalls on hover | MapLibre in-memory R-Tree spatial index (`queryRenderedFeatures`) | [`MapCanvas.tsx:L1544, L1903`](file:///tern/geolibre/GeoLibre/packages/map/src/MapCanvas.tsx#L1544) |
| **DEF-12** | Interaction | 200–400ms UI stall during zoom | In-worker `Supercluster` point clustering | [`geojson-vt-protocol.ts:L92`](file:///tern/geolibre/GeoLibre/packages/map/src/geojson-vt-protocol.ts#L92) |
| **DEF-13** | Ingestion | Main-thread string allocation & parse | Worker-first streaming file ingestion | [`wasm-client.ts:L670`](file:///tern/geolibre/GeoLibre/packages/processing/src/wasm-client.ts#L670) |
| **DEF-14** | Ingestion | Double RAM usage across `postMessage` | Zero-copy Transferable ArrayBuffers | [`wasm-client.ts:L748`](file:///tern/geolibre/GeoLibre/packages/processing/src/wasm-client.ts#L748) |
| **DEF-15** | Ingestion | In-thread Shapefile/GeoTIFF decoding | Dedicated background worker decoders | [`shapefile-zip-multipatch.test.ts`](file:///tern/geolibre/GeoLibre/tests/shapefile-zip-multipatch.test.ts#L16) |
| **DEF-16** | Tabular | 1.5–3.0s UI freeze during file load | Worker-side DuckDB table loading and Arrow streaming | [`maplibre-duckdb.ts`](file:///tern/geolibre/GeoLibre/packages/plugins/src/plugins/maplibre-duckdb.ts) |
| **DEF-17** | Tabular | Linear property extraction in JS | Direct worker SQL queries & integer `__id` indexing | [`joins.ts`](file:///tern/geolibre/GeoLibre/packages/core/src/joins.ts) |
| **DEF-18** | Numeric | Array sorting stutter on style edit | Reservoir sampling & GPU GLSL `step` expressions | [`color-ramp.ts:L328`](file:///tern/geolibre/GeoLibre/packages/core/src/color-ramp.ts#L328) |
