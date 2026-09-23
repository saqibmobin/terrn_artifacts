# Terrn: Architectural Proposals, Edge Cases & Risk Analysis
## Critical Engineering Review of Enhanced Remediations for Spatial Clustering and Raster Ingestion

**Document Version:** 1.0.0  
**Date:** August 21, 2026  
**Target Codebase:** `/tern/tern_poc`  
**Referenced Catalogs:** 
- [`Terrn_Detailed_Issues_Catalog.md`](file:///tern/Terrn_Detailed_Issues_Catalog.md)
- [`Terrn_Defects_to_Solutions_Mapping.md`](file:///tern/Terrn_Defects_to_Solutions_Mapping.md)

---

## Executive Summary

While the architectural remediations outlined in [`Terrn_Defects_to_Solutions_Mapping.md`](file:///tern/Terrn_Defects_to_Solutions_Mapping.md) correctly identify the high-level strategies needed to stabilize Terrn (specifically offloading to background workers and adopting native spatial indexing), translating these concepts into high-performance browser execution introduces critical failure modes.

This document details:
1. **Conceptual enhancements** to the existing proposals for **Point Clustering** and **GeoTIFF Raster Ingestion**.
2. **Exhaustive edge-case and risk analyses** for each proposal (failure modes, hardware limitations, and visual artifacts).
3. **Mandatory engineering safeguards and fallback strategies** required to ensure zero-regression stability.

---

## 1. Point Clustering Architecture

### Target Defect Reference
* **Primary Defect:** [`Terrn_Detailed_Issues_Catalog.md: DEF-11`](file:///tern/Terrn_Detailed_Issues_Catalog.md#L225-L243) (*Synchronous Main-Thread Grid Clustering Fallback*)
* **Mapping Guide Counterpart:** [`Terrn_Defects_to_Solutions_Mapping.md: DEF-12`](file:///tern/Terrn_Defects_to_Solutions_Mapping.md#L119-L125)
* **Underlying File:** [`src/features/layers/rendering/renderer.ts:L574-L580`](file:///tern/tern_poc/src/features/layers/rendering/renderer.ts#L574-L580)

---

### Proposal 1.1: Frustum-Constrained Spatial Scoping (Viewport Bounding)

#### The Concept
Building a hierarchical spatial index (via `Supercluster`) is only half the equation. Querying must be strictly decoupled from total layer scale. Rather than querying clusters across the global extent (`[-180, -85, 180, 85]`), cluster extraction must be constrained strictly to the camera's active view frustum, returning only the hundred or thousand centroids visible on screen.

#### Identified Risks & Failure Modes
1. **Edge "Pop-in" Artifacts on Fast Pan / Fling:**
   - *Failure Mechanism:* If queries are clipped strictly to the camera viewport (`map.getBounds()`), any high-velocity camera movement (trackpad fling, momentum panning) will instantly expose unrendered, blank canvas at the edges before the subsequent query can resolve.
2. **Antimeridian (±180° Longitude) Inversion:**
   - *Failure Mechanism:* When viewing the Pacific Ocean or crossing the International Date Line, the bounding box coordinates invert (e.g. West: $170^\circ$, East: $-170^\circ$). Standard spatial range trees fail or evaluate the query as an empty set, causing all clusters over the Pacific to vanish completely.
3. **Multi-World Wrapping at Low Zoom Levels:**
   - *Failure Mechanism:* At zoom levels $0 \le Z \le 3$, modern map engines render multiple horizontal copies of the world. A single bounding-box query resolves only for the central copy ($X \in [-180, 180]$), leaving duplicate world tiles on the left and right completely devoid of clustered points.

#### Mandatory Engineering Safeguards & Fallbacks
* **Hysteresis Buffer (Frustum Padding):** Expand the queried bounding envelope by a deterministic padding ratio of $30\% \text{ to } 50\%$ beyond the active viewport. This buffer provides an offscreen visual runway that absorbs momentum panning before new queries are dispatched.
* **Antimeridian Split-Query Logic:** Detect when $\text{west} > \text{east}$. Automatically split the envelope into two discrete bounding boxes: $[\text{west}, \text{south}, 180, \text{north}]$ and $[-180, \text{south}, \text{east}, \text{north}]$, merging the resulting centroid arrays prior to rendering.
* **Coordinate Clamping at Low Zoom:** When $Z < 3$, disable viewport bounding entirely and query the canonical global envelope $[-180, -85, 180, 85]$ once, delegating horizontal tile wrapping to the WebGL shader layer.

---

### Proposal 1.2: Temporal Latching & Perceptual Persistence (Stale-While-Revalidate)

#### The Concept
Asynchronous worker handoffs introduce an inevitable latency gap (5–30 ms) while the background thread computes clusters for a newly reached zoom level. To prevent visual flickering, the rendering pipeline must implement **temporal latching**: retaining and rendering the cluster hierarchy of the previous zoom level until the worker yields the new discrete cluster tree.

#### Identified Risks & Failure Modes
1. **"Cluster Jumping" & Disorientation on Rapid Zoom:**
   - *Failure Mechanism:* When a user scrolls the wheel aggressively (e.g., jumping from Zoom 4 to Zoom 14 in 300 ms), latching onto Zoom 4 clusters causes massive country-level cluster bubbles to float over individual street blocks. When the worker finally catches up, these bubbles abruptly explode and snap into hundreds of individual points, creating an unsettling visual jolt.
2. **Out-of-Order Worker Response Race Conditions:**
   - *Failure Mechanism:* If a user zooms rapidly through $Z_5 \to Z_6 \to Z_7$, three distinct worker queries are dispatched. If the $Z_6$ query takes longer than $Z_7$ (due to a denser point distribution), $Z_6$ might resolve *after* $Z_7$, overwriting fresh zoom data with obsolete cluster trees.
3. **Feature Picking & Inspector Desynchronization:**
   - *Failure Mechanism:* In Feature Info mode ([`Terrn_Detailed_Issues_Catalog.md: DEF-10`](file:///tern/Terrn_Detailed_Issues_Catalog.md#L203-L223)), the user sees the latched (stale) cluster circles, but the internal camera matrix is already at the new zoom level. Clicking a latched cluster results in picking misses or displays the wrong underlying records.

#### Mandatory Engineering Safeguards & Fallbacks
* **Zoom-Delta Threshold Bypass:** Measure the delta between the requested zoom and the latched zoom ($\Delta Z = |Z_{\text{current}} - Z_{\text{latched}}|$). If $\Delta Z > 2.5$, bypass temporal latching, temporarily fade out cluster centroids, and display a subtle ambient loading pulse to avoid visual ghosting.
* **Monotonic Request IDs & Cancellation:** Tag every worker query with a monotonically increasing integer sequence ID. When a worker resolves, discard any payload whose request ID is lower than the latest dispatched ID (`currentSequenceId`). Terminate pending queries using an `AbortController` signal when zoom transitions exceed $100\text{ ms}$.
* **Synchronized Interaction Latching:** In the feature-picking handler, point queries against the cluster tree must explicitly use the zoom level and spatial snapshot of the *currently rendered latched layer*, not the live camera zoom, ensuring $100\%$ picking fidelity during asynchronous transitions.

---

### Proposal 1.3: Phased Decoupling (Two-Speed Remediation Strategy)

#### The Concept
Migrating Terrn's entire rendering pipeline to native MapLibre vector tiles (`geojson-vt-protocol.ts`) is a high-cost architectural refactor. Decouple remediation into two distinct phases: an **immediate non-blocking stopgap** within the existing Deck.gl layer, followed by a **strategic migration** to vector tiles.

#### Identified Risks & Failure Modes
1. **Technical Debt Calcification:**
   - *Failure Mechanism:* If Phase 1 (patching the Deck.gl cache fallback) makes clustering "good enough" for 50k points, development momentum stalls. Phase 2 (migrating to native MapLibre) is deprioritized, leaving the root WebGL state contamination defects ([`DEF-01 to DEF-04`](file:///tern/Terrn_Detailed_Issues_Catalog.md#L41-L96)) permanently in the codebase.
2. **Throwaway Code Overhead:**
   - *Failure Mechanism:* Engineering hours spent building custom Deck.gl cluster caching, spatial bounding, and picking bridges are completely discarded once vector tiles are introduced.

#### Mandatory Engineering Safeguards & Fallbacks
* **Strict Milestone Gating:** Treat Phase 1 strictly as a hotfix (bounded to $< 20$ lines in [`renderer.ts`](file:///tern/tern_poc/src/features/layers/rendering/renderer.ts)).
* **Automated CI Regression Tests:** Enforce automated Puppeteer performance benchmarks asserting that main-thread frame execution times during zoom remain $< 16.6\text{ ms}$ under penalty of CI build failure.

---

## 2. GeoTIFF Raster Ingestion & Presentation Architecture

### Target Defect Reference
* **Primary Defect:** [`Terrn_Detailed_Issues_Catalog.md: Section 4.3`](file:///tern/Terrn_Detailed_Issues_Catalog.md#L279-L293) (*Synchronous In-Thread File Format Decoding*)
* **Mapping Guide Counterpart:** [`Terrn_Defects_to_Solutions_Mapping.md: DEF-15`](file:///tern/Terrn_Defects_to_Solutions_Mapping.md#L145-L152)
* **Underlying File:** [`src/shared/parsers/index.ts:L198-L275`](file:///tern/tern_poc/src/shared/parsers/index.ts#L198-L275)

---

### Proposal 2.1: Decoupling Immutable Ingestion from Mutable GPU Presentation

#### The Concept
Decompressing GeoTIFFs, extracting metadata, and normalizing coordinates is an **immutable, one-time operation**. Applying color ramps, adjusting contrast cutoffs, and changing opacity is an **interactive, mutable presentation operation**. 
The worker should output raw physical data (elevation, radiance); dynamic color mapping must be evaluated live on the GPU via a native WebGL fragment shader, eliminating the need to re-decode files when styles change.

#### Identified Risks & Failure Modes
1. **Floating-Point Texture Filtering Incompatibility:**
   - *Failure Mechanism:* Storing raw elevation as 32-bit floating-point textures (`R32F`) in WebGL2 requires linear texture filtering support (`OES_texture_float_linear`). This extension is **optional** in WebGL2 and is unsupported on certain mobile chipsets, older integrated Intel GPUs, and Apple Silicon Safari under specific power-saving profiles.
   - *Visual Symptom:* Without linear filtering, zooming into an elevation model results in harsh, jagged nearest-neighbor pixel blocks rather than smooth gradients.
2. **NoData Sentinel Value Bleed & Edge Halos:**
   - *Failure Mechanism:* Elevation models frequently store void/water areas as sentinel numbers (e.g. `-9999` or `-3.4028235e+38`). If raw textures are sampled across border boundaries, standard GPU hardware interpolation will interpolate between valid elevation (e.g. `100.0`) and `-9999.0`, creating dark or discolored ringing halos along coastlines.
3. **Severe VRAM Bloat:**
   - *Failure Mechanism:* Raw 32-bit float textures consume 4 bytes per pixel ($16\text{ bytes/px}$ for 4-band rasters). An $8000 \times 8000$ single-band raster consumes $256\text{ MB}$ of dedicated GPU VRAM. In multi-layer GIS sessions, multiple rasters will rapidly exceed browser VRAM quotas, triggering WebGL context loss.

#### Mandatory Engineering Safeguards & Fallbacks
* **Hardware Capability Sniffing with Graceful Degradation:** During initialization, probe for `OES_texture_float_linear`. If absent, automatically execute manual 4-tap bilinear interpolation inside the fragment shader or fall back to an 8-bit quantized texture pipeline.
* **Strict Alpha NoData Masking:** Include the dataset's declared NoData value as a uniform in the shader. Any pixel matching NoData within an epsilon threshold must execute `discard` or output an alpha value of `0.0` before linear filtering or color ramp assignment occurs.
* **Resolution Pyramids (Overview Sub-sampling):** For rasters exceeding $2048 \times 2048$, the background worker must generate downsampled overview levels (mipmaps) during decompression, uploading only the mip level corresponding to the current screen resolution.

---

### Proposal 2.2: Zero-Copy Ownership Handoff Across Thread Boundaries

#### The Concept
Offloading decompression to a worker fails if sending the decoded image back to the main thread triggers deep object cloning. The worker must wrap pixel data in an opaque, GPU-backed visual handle (a transferable `ImageBitmap`) and relinquish ownership directly to the main thread via `postMessage(message, [transferables])`.

#### Identified Risks & Failure Modes
1. **Buffer Neutering & Loss of Inspector Queryability:**
   - *Failure Mechanism:* In JavaScript, transferring an `ArrayBuffer` permanently **detaches (neuters)** the underlying memory in the worker thread. If the user subsequently uses the Feature Inspector to click on the raster and query the exact elevation value, the worker no longer holds the data to answer the query.
2. **GPU Surface Memory Leaks (VRAM Exhaustion):**
   - *Failure Mechanism:* An `ImageBitmap` retains GPU memory allocations that are **not** reliably collected by the V8 JavaScript garbage collector. If a user repeatedly toggles layers or loads new rasters without explicitly invoking `.close()`, VRAM leaks accumulate rapidly, causing browser tab crashes.

#### Mandatory Engineering Safeguards & Fallbacks
* **Dual-Buffer Strategy (Decoupled Query Buffer):** In the worker, convert the raw decoded measurements into an internal `SharedArrayBuffer` (if cross-origin isolation is active) or retain the raw typed array in the worker's persistent cache while creating the `ImageBitmap` from a temporary transferable copy. The worker retains queryability for attribute inspections.
* **Explicit Lifecycle Management (`dispose` Hook):** Every raster layer in `layer.store.ts` must implement an explicit disposal contract. When a raster layer is removed, reloaded, or styled, the application must invoke `bitmap.close()` synchronously on the previous bitmap handle before allocating a new one.

---

### Proposal 2.3: Discretized Quantization (LUT-Based Classification)

#### The Concept
Continuous physical data must not be transformed into display pixels by evaluating heavy non-linear formulas (like `Math.pow()` or exponential functions) across millions of pixels. Instead, the continuous measurement space should be quantized onto a precomputed, discrete 256-entry 32-bit Look-Up Table (LUT), reducing color mapping to an $O(1)$ indexed memory lookup.

#### Identified Risks & Failure Modes
1. **Contour Banding (Posterization Artifacts):**
   - *Failure Mechanism:* Mapping continuous floating-point terrain into only 256 discrete bins introduces visible stepping ("contour banding") across broad, gradual geographic slopes (e.g. floodplains or bathymetric basins).
2. **Contrast Blowout on Skewed / Heavy-Tailed Distributions:**
   - *Failure Mechanism:* Standard linear normalization ($\frac{\text{val} - \text{min}}{\text{max} - \text{min}}$) collapses completely when a dataset contains spatial outliers (e.g. 99% of values are between 10 and 50, but a cloud reflection artifact is at 5,000). Two hundred fifty-four of the 256 bins will be empty, rendering the entire map as an unintelligible flat monochromatic surface.

#### Mandatory Engineering Safeguards & Fallbacks
* **Dithered Quantization / Expanded LUT:** Expand the discrete LUT to 1024 entries (`Uint32Array(1024)`) or apply a lightweight Blue Noise or Bayer spatial dither matrix in the worker to break up visible contour banding.
* **Histogram-Equalized / Percentile Cutoffs:** Before quantizing into the LUT, compute a fast cumulative histogram in the worker. Clamp the minimum and maximum scaling values to the 2nd and 98th percentiles rather than absolute minimum and maximum values, preventing extreme outliers from destroying visual dynamic range.

---

## 3. Comprehensive Master Risk & Safeguard Matrix

| Domain | Proposed Conceptual Remediation | Primary Failure Mode / Risk | Manifested Defect | Mandatory Engineering Safeguard |
| :--- | :--- | :--- | :--- | :--- |
| **Point Clustering** | Frustum-Constrained Viewport Scoping | Velocity panning exposes unrendered edges; Pacific Ocean antimeridian inversion | Visual edge popping; missing data over $\pm 180^\circ$ | Apply a 30–50% hysteresis buffer; implement antimeridian split queries. |
| **Point Clustering** | Temporal Latching (Stale-While-Revalidate) | Multi-level zoom jumps display grossly oversized clusters; out-of-order worker returns | "Cluster jumping" disorientation; stale zoom race conditions | Bypass latching when $\Delta Z > 2.5$; enforce monotonic sequence IDs with `AbortController`. |
| **Point Clustering** | Phased Decoupling Strategy | Stopgap hotfix creates inertia; team abandons root WebGL fixes | Perpetual technical debt; Z-fighting ([DEF-01](file:///tern/Terrn_Detailed_Issues_Catalog.md#L41-L56)) remains | Enforce strict milestone gating with automated CI frame-rate regression gates. |
| **Raster Ingestion** | Decoupled GPU Presentation (Float Textures) | Optional `OES_texture_float_linear` absent on older/mobile GPUs; NoData bleed halos | Blocky nearest-neighbor rendering; coastline ring artifacts | Sniff hardware capabilities with 8-bit fallback; implement strict shader NoData alpha discards. |
| **Raster Ingestion** | Zero-Copy Ownership Handoff | Detached memory prevents hover queries; V8 fails to GC GPU surface handles | Inability to inspect pixel values; catastrophic VRAM tab crashes | Retain queryable typed array in worker; enforce explicit `.close()` calls on obsolete bitmaps. |
| **Raster Ingestion** | Discretized Quantization (LUTs) | 256 discrete levels produce contour steps; spatial outliers wash out contrast | Posterization banding on plains; flat monochromatic rendering | Expand LUT to 1024 entries with dither; apply 2nd/98th percentile histogram equalization. |

---

## 4. Verification & Gate Criteria

Before merging remediations based on these proposals into `/tern/tern_poc`, the following automated gates must pass:

1. **Gestural Continuity Gate:**
   - Drive continuous zoom sweeps ($Z_4 \to Z_{14}$) via Puppeteer at $60\text{ FPS}$. Zero empty frames or cluster flickers permitted.
2. **Memory Leak Gate:**
   - Ingest and toggle 10 GeoTIFF raster layers in sequence. Total VRAM allocation must stabilize, with $100\%$ of detached `ImageBitmap` handles confirmed closed via DevTools heap snapshots.
3. **Antimeridian Integrity Gate:**
   - Center map viewport on $[-180.0, 0.0]$ at zoom level 6. Validate that point clusters crossing the date line render contiguously across both sides of the canvas seam.
