# Terrn Performance Audit & Comprehensive Optimization Guide
## Full-Stack Client-Side Geospatial Performance Engineering (Cross-Referenced with GeoLibre)

**Document Version:** 1.1.0  
**Date:** August 21, 2026  
**Target Codebase:** `/tern/tern_poc`  
**Reference Benchmark:** `/tern/geolibre`

---

## 1. Overview & Executive Summary

While the primary rendering lag and visual flickering in **Terrn** stem from Deck.gl’s interleaved WebGL overlay and lack of client-side vector tiling (detailed in [`/tern/RCA_and_Competitive_Analysis.md`](file:///tern/RCA_and_Competitive_Analysis.md)), a full-stack architectural audit reveals multiple secondary bottlenecks across:
1. **Reactivity & State Store Management**
2. **Feature Picking & Inspector Interaction**
3. **File Ingestion, Memory Footprint & Structured Cloning**
4. **Tabular Data Processing & DuckDB-WASM IPC Pipelines**
5. **Statistical Classification & Numeric Analysis**

This document cross-references each identified bottleneck directly against **GeoLibre's production implementations** to provide actionable engineering patterns for Terrn.

```
+----------------------------------------------------------------------------------------------------+
|                               TERRN VS. GEOLIBRE SUBSYSTEM ARCHITECTURE                            |
+----------------------------------------------------------------------------------------------------+
| SUBSYSTEM               | TERRN (/tern/tern_poc)                 | GEOLIBRE (/tern/geolibre)       |
+-------------------------+----------------------------------------+---------------------------------+
| 1. State Store          | JSON.parse(JSON.stringify) on slider   | Shallow immutability + versioned|
|                         | WeakMap rebuilds on every style update | derived re-evaluations (store.ts)|
|                         |                                        |                                 |
| 2. Feature Picking      | gl.readPixels() on raw mousemove       | Non-blocking R-Tree spatial     |
|                         | (CPU-GPU sync stall in base-map.ts)    | queryRenderedFeatures() (canvas)|
|                         |                                        |                                 |
| 3. File Ingestion       | Main thread string reading & clone     | Worker-first binary streaming + |
|                         | to geo.worker.ts (2x-3x RAM overhead)  | Transferable ArrayBuffers       |
|                         |                                        |                                 |
| 4. DuckDB Tabular Sync  | Main thread tableFromJSON & tableToIPC | Worker-side Arrow IPC streaming |
|                         | (1-3s UI freeze during file load)      | and direct SQL ingestion        |
|                         |                                        |                                 |
| 5. Numeric Breaks       | Main thread Array.sort() on 53k floats | Quantile interpolation + GPU    |
|                         | JS evaluation per feature on CPU       | step expressions (color-ramp.ts)|
+----------------------------------------------------------------------------------------------------+
```

---

## 2. Subsystem 1: State Management & Reactivity Optimizations

### 2.1 Terrn Bottlenecks
* **Deep Cloning on Style Drag:** In [`layer.store.ts:L367-L370`](file:///tern/tern_poc/src/features/layers/state/layer.store.ts#L367-L370):
  ```typescript
  public setLayerStyle(id: string, styleConfig: StyleConfig) {
    this.layersSig.set(this.layersSig.get().map(l =>
      l.id === id ? { ...l, styleConfig: JSON.parse(JSON.stringify(styleConfig)) } : l
    ));
    this.layersVersion++;
    this.notifyChange();
  }
  ```
  Dragging an opacity or radius slider emits 60+ input events/sec. Serializing and deserializing JSON objects continuously thrashes the V8 memory heap and triggers GC pauses.
* **O(N) Feature Identity Map Rebuilds:** In [`attribute-table-element.ts:L100-L112`](file:///tern/tern_poc/src/features/tabular/components/attribute-table-element.ts#L100-L112):
  `layersVersion` increments on **every style update, opacity tweak, and visibility toggle** ([`layer.store.ts:L353, L362, L370`](file:///tern/tern_poc/src/features/layers/state/layer.store.ts#L353)). Consequently, `ensureFeatureIdentity()` throws away and rebuilds a `WeakMap` across all **55,000+ features** on the main thread during simple styling tweaks.

### 2.2 How GeoLibre Implements It
* **Reference Implementation:** [`geolibre/GeoLibre/packages/core/src/store.ts:L1738-L1763`](file:///tern/geolibre/GeoLibre/packages/core/src/store.ts#L1738-L1763)
  ```typescript
  updateLayer: (id, patch) =>
    set((s) => {
      let layers = s.layers.map((l) => (l.id === id ? { ...l, ...patch } : l));
      // Re-derive expensive column joins ONLY when the raw geojson data changes
      if (patch.geojson !== undefined) {
        if (patch.joins === undefined && patch.virtualFields === undefined) {
          layers = layers.map((l) =>
            l.id === id && (l.joins?.length || l.virtualFields?.length)
              ? applyJoinsToLayer(l, layers)
              : l,
          );
        }
        layers = cascadeLayerJoinRefresh(layers, id);
      }
      return { layers, isDirty: true };
    })
  ```
  1. **Shallow Object Updates:** GeoLibre uses clean object spreads (`{ ...l, ...patch }`) with zero `JSON.parse(JSON.stringify)` overhead.
  2. **Gated Derivative Re-computation:** Heavy indexing or derived calculations run only if `patch.geojson !== undefined`.
  3. **Zustand Selectors:** UI components subscribe only to their specific slice of state (e.g. `useAppStore(s => s.layers[id].style)`), preventing whole-app re-renders on slider drag.

### 2.3 Cues for Terrn
1. **Shallow Patches:** Replace `JSON.parse(JSON.stringify(styleConfig))` with shallow immutable patches:
   ```typescript
   public setLayerStyle(id: string, patch: Partial<StyleConfig>) {
     this.layersSig.set(this.layersSig.get().map(l => {
       if (l.id !== id) return l;
       return { ...l, styleConfig: { ...l.styleConfig, ...patch } };
     }));
     this.layersStyleVersion++; // Bump style version only
     this.notifyChange();
   }
   ```
2. **Decouple Mutation Versions:** Separate `layersDataVersion` (bumped only on dataset add/remove/reload) from `layersStyleVersion`. In `attribute-table-element.ts`, gate `ensureFeatureIdentity()` strictly on `layersDataVersion`.

---

## 3. Subsystem 2: Feature Picking & Inspector Interaction

### 3.1 Terrn Bottlenecks
* **Synchronous GPU Stalls on `mousemove`:** In [`base-map.ts:L195-L204`](file:///tern/tern_poc/src/core/canvas/base-map.ts#L195-L204):
  ```typescript
  mapInstance.on('mousemove', (e) => {
    if (!isFeatureInfoModeActive || !deckOverlay) return;
    const hit = deckOverlay.pickObject({
      x: e.point.x,
      y: e.point.y,
      radius: 6,
    });
    const container = mapInstance!.getCanvasContainer();
    container.style.cursor = hit ? 'pointer' : 'crosshair';
  });
  ```
  Deck.gl's `pickObject()` renders an offscreen framebuffer with color-coded IDs and issues `gl.readPixels()`. Calling this on raw mouse movement events forces a synchronous CPU-GPU pipeline synchronization barrier, lagging cursor responsiveness over dense layers (53k points).

### 3.2 How GeoLibre Implements It
* **Reference Implementation:** [`geolibre/GeoLibre/packages/map/src/MapCanvas.tsx:L1540-L1553, L1897-L1917`](file:///tern/geolibre/GeoLibre/packages/map/src/MapCanvas.tsx#L1540-L1553)
  ```typescript
  const queryLayerIds = identifyStyleLayerIds(layer).filter((id) => map.getLayer(id));
  if (queryLayerIds.length === 0) {
    clearIdentifyResult();
    return;
  }
  const [feature] = map.queryRenderedFeatures(event.point, {
    layers: queryLayerIds,
  });
  if (feature) {
    const featureId = findFeatureId(layer, feature);
    selectFeature(featureId);
    showIdentifyPopup(
      createIdentifyPopupElement(layer.name, feature.properties ?? {}, featureId ?? feature.id),
    );
  }
  ```
  1. **In-Memory Spatial Queries:** GeoLibre uses MapLibre’s native `map.queryRenderedFeatures()`.
  2. **Zero GPU Readbacks:** It queries MapLibre’s in-memory R-Tree spatial index of visible tile geometries in CPU memory without triggering offscreen WebGL passes or `gl.readPixels()`.
  3. **Execution Speed:** Completes in **$< 1\text{ ms}$** even across dense point collections.

### 3.3 Cues for Terrn
1. **Switch to MapLibre Spatial Queries:** With native MapLibre layers, replace `deckOverlay.pickObject()` with `map.queryRenderedFeatures(e.point, { layers: [...] })`.
2. **Animation Frame Throttling:** If hover picking is needed for other overlays, wrap it in `requestAnimationFrame` to limit checks to the screen's refresh rate (16.6ms).

---

## 4. Subsystem 3: File Ingestion & Memory Management

### 4.1 Terrn Bottlenecks
* **Main-Thread Parsing & Structured Cloning:** In [`main.ts:L126-L163`](file:///tern/tern_poc/src/main.ts#L126-L163) and [`geo.worker.ts:L21-L71`](file:///tern/tern_poc/src/core/workers/geo.worker.ts#L21-L71):
  ```typescript
  const text = reader.result as string;
  geojson = JSON.parse(text);
  const bounds = await getGeoJsonBoundsAsync(geojson);
  ```
  When importing a 20–50 MB GeoJSON/Shapefile:
  1. `FileReader.readAsText()` allocates a full text string in RAM.
  2. `JSON.parse()` instantiates tens of thousands of heap objects.
  3. `getGeoJsonBoundsAsync(geojson)` passes `geojson` via `postMessage`, which triggers a **structured clone** of the entire object graph, doubling heap usage and freezing the main thread.

### 4.2 How GeoLibre Implements It
* **Reference Implementation:** [`geolibre/GeoLibre/packages/plugins/src/plugins/remote-file-formats.ts`](file:///tern/geolibre/GeoLibre/packages/plugins/src/plugins/remote-file-formats.ts) and [`geolibre/GeoLibre/packages/processing/src/wasm-client.ts:L670-L750`](file:///tern/geolibre/GeoLibre/packages/processing/src/wasm-client.ts#L670-L750)
  1. **Worker-First Processing:** File parsing, geometry conversion, and bounds calculation are delegated to background Web Workers and WebAssembly modules.
  2. **Zero-Copy Memory Transfer:** Binary buffers are passed across threads as **Transferable Objects** (`postMessage(buffer, [buffer])`), eliminating structured clone overhead.
  3. **Cloud-Native Format Support:** GeoLibre supports **FlatGeobuf** (`.fgb`), **PMTiles** (`.pmtiles`), and **GeoParquet** (`.parquet`). FlatGeobuf includes an embedded Hilbert R-tree index, allowing MapLibre to read feature bounding boxes directly into TypedArrays without instantiating millions of JS objects.

### 4.3 Cues for Terrn
1. **Worker-First Ingest Pipeline:** Move `FileReader`, `JSON.parse()`, and `shpjs` parsing inside `geo.worker.ts`.
2. **Transferable Buffers:** Read files as `ArrayBuffer` and transfer them directly to the worker with zero-copy semantics.
3. **Binary Formats:** Add native support for FlatGeobuf and PMTiles to bypass JSON parsing overhead for large vector layers.

---

## 5. Subsystem 4: Tabular Data & DuckDB Integration

### 5.1 Terrn Bottlenecks
* **Main-Thread Arrow Table Serialization:** In [`duckdb.ts:L270-L285`](file:///tern/tern_poc/src/services/duckdb.ts#L270-L285):
  ```typescript
  const props = features.map((f: any, idx: number) => {
    const p = f.properties ? { ...f.properties } : {};
    p.__row_idx = idx;
    return p;
  });
  const arrowTable = tableFromJSON(props);
  await conn.insertArrowFromIPCStream(tableToIPC(arrowTable, 'stream'), { name: tblName });
  ```
  Extracting properties, mapping 53k new objects, creating an Arrow table (`tableFromJSON`), and serializing to an IPC stream (`tableToIPC`) all run on the **main thread**, causing a 1–3s freeze on layer load.

### 5.2 How GeoLibre Implements It
* **Reference Implementation:** [`geolibre/GeoLibre/packages/plugins/src/plugins/maplibre-duckdb.ts`](file:///tern/geolibre/GeoLibre/packages/plugins/src/plugins/maplibre-duckdb.ts) and [`geolibre/GeoLibre/packages/core/src/joins.ts`](file:///tern/geolibre/GeoLibre/packages/core/src/joins.ts)
  1. **Background Table Registration:** DuckDB ingestion and table creation occur entirely within the DuckDB Web Worker.
  2. **Direct SQL Data Loading:** Instead of converting JSON objects to Arrow on the main thread, DuckDB loads data directly inside the worker:
     ```sql
     CREATE TABLE layer_table AS SELECT * FROM read_json_auto('...');
     ```
  3. **Virtual Row Materialization:** The UI only queries the visible page slice (e.g. `LIMIT 50 OFFSET 0`), keeping memory and IPC transfer minimal.

### 5.3 Cues for Terrn
1. **Move Arrow Serialization to Worker:** Move `tableFromJSON` and `tableToIPC` into the DuckDB Worker thread.
2. **Integer Feature Indexing:** Assign an integer `__id` to features at ingest time to replace `JSON.stringify` / `WeakMap` identity scans in `attribute-table-element.ts`.

---

## 6. Subsystem 5: Statistical Classification & Numeric Analysis

### 6.1 Terrn Bottlenecks
* **Synchronous Array Sorting on Main Thread:** In [`numeric.ts:L11-L30`](file:///tern/tern_poc/src/utils/numeric.ts#L11-L30):
  ```typescript
  export function computeNumericIntervals(values: number[], method: ..., breaksCount: number) {
    const sorted = [...values].sort((a, b) => a - b);
    ...
  }
  ```
  For 53,316 features, copying and sorting 53k numbers synchronously blocks the main thread during style classification. In addition, Terrn evaluates colors for all features on CPU via JS accessors ([`renderer.ts:L323`](file:///tern/tern_poc/src/features/layers/rendering/renderer.ts#L323)).

### 6.2 How GeoLibre Implements It
* **Reference Implementation:** [`geolibre/GeoLibre/packages/core/src/color-ramp.ts:L328-L349`](file:///tern/geolibre/GeoLibre/packages/core/src/color-ramp.ts#L328-L349) and [`geolibre/GeoLibre/packages/map/src/style-mapper.ts:L37-L106`](file:///tern/geolibre/GeoLibre/packages/map/src/style-mapper.ts#L37-L106)
  1. **Sampled Quantile Breaks:** [`createGraduatedClassBreaks()`](file:///tern/geolibre/GeoLibre/packages/core/src/color-ramp.ts#L328) calculates class lower bounds (stops).
  2. **GPU GLSL Shader Expressions:** GeoLibre maps these stops directly to MapLibre style expressions:
     ```json
     [
       "step",
       ["get", "population"],
       "#eff6ff",
       10, "#93c5fd",
       25, "#2563eb",
       50, "#1e3a8a"
     ]
     ```
  3. **Zero CPU Loop:** MapLibre compiles these expressions into GPU fragment shaders. The CPU spends **0ms** evaluating feature colors on frame redraws.

### 6.3 Cues for Terrn
1. **Reservoir Sampling / DuckDB Quantiles:** When $N > 5,000$, compute statistical breaks on a sample of 2,000–5,000 numbers or query DuckDB WASM's `approx_quantile()` in the worker.
2. **GLSL Paint Expressions:** Convert `StyleConfig` into native MapLibre `step` expressions so classification runs on the GPU.

---

## 7. Comprehensive Optimization Blueprint for Terrn

| Priority | Subsystem | Actionable Engineering Task | GeoLibre Reference |
| :--- | :--- | :--- | :--- |
| **P0** | **Vector Rendering** | Replace Deck.gl `GeoJsonLayer` with native MapLibre layers & `terrn-vt://` protocol | [`geojson-vt-protocol.ts:L71`](file:///tern/geolibre/GeoLibre/packages/map/src/geojson-vt-protocol.ts#L71) |
| **P0** | **State Reactivity** | Replace `JSON.parse(JSON.stringify)` with shallow patches; decouple data vs. style versions | [`store.ts:L1738`](file:///tern/geolibre/GeoLibre/packages/core/src/store.ts#L1738) |
| **P1** | **Feature Picking** | Replace `deckOverlay.pickObject()` on `mousemove` with `map.queryRenderedFeatures()` | [`MapCanvas.tsx:L1544, L1903`](file:///tern/geolibre/GeoLibre/packages/map/src/MapCanvas.tsx#L1544) |
| **P1** | **File Ingestion** | Worker-first parsing (`shpjs`, `JSON.parse`) with zero-copy Transferable ArrayBuffers | [`wasm-client.ts:L670`](file:///tern/geolibre/GeoLibre/packages/processing/src/wasm-client.ts#L670) |
| **P1** | **DuckDB Sync** | Offload `tableFromJSON` and Arrow IPC streaming into DuckDB Worker | [`maplibre-duckdb.ts`](file:///tern/geolibre/GeoLibre/packages/plugins/src/plugins/maplibre-duckdb.ts) |
| **P2** | **Classification** | Use reservoir sampling for quantile breaks; compile stops into GLSL `step` expressions | [`color-ramp.ts:L328`](file:///tern/geolibre/GeoLibre/packages/core/src/color-ramp.ts#L328) |
| **P2** | **Binary Formats** | Add native streaming support for FlatGeobuf and PMTiles | [`remote-file-formats.ts`](file:///tern/geolibre/GeoLibre/packages/plugins/src/plugins/remote-file-formats.ts) |
