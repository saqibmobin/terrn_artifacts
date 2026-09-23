# Layer Type Icons — Design Specification

**Status:** Approved Specification · **Authoritative Design**  
**Date:** 2026-09-22  
**Context:** Terrn Geospatial UI · Layer Panel Cards (`.layer-type-icon`)  
**Design Authority:** `docs/design/icons-and-buttons.md` (§2.4, §4.5, §4.6)

---

## 1. Executive Summary & Problem Space

In the Terrn left panel, every layer card renders a 28×28px type indicator chip (`.layer-type-icon`) displaying an SVG glyph at **14×14px** (`--icon-sm`).

Because Lucide (Terrn's canonical icon family) lacks native cartographic primitives, the codebase previously borrowed general-purpose Lucide icons as stand-ins (`src/features/layers/rendering/geometry-icon.ts`):

| Layer Type | Legacy Stand-In | Visual Metaphor | Failure Mode |
|---|---|---|---|
| **Polygon** | `cube` (`lucide:box`) | 3D isometric shipping box | Conveys volumetric 3D extrusion rather than a 2D planar boundary, parcel, or zone. |
| **LineString** | `code` (`lucide:code`) | Developer angle brackets `< >` | Conveys code syntax/editing rather than a vector road, river, contour, or path. |
| **Raster** | `cloud` (`lucide:cloud`) | Weather cloud / Cloud storage | Conveys remote sync or meteorology rather than a continuous pixel grid, DEM, or satellite GeoTIFF. |
| **Point** | `map-pin` (`lucide:map-pin`) | Destination needle / Address pin | Conveys an isolated point-of-interest search result rather than coordinate observation features (sensors, incidents, trees). |
| **Tabular Data** | *(None)* | *(Unrepresented)* | Non-spatial tables (CSVs, Parquet) lacked a dedicated layer-type classification. |

This specification establishes a cohesive 5-tier layer icon system drawn to Lucide's exact construction rules, benchmarked against GIS cartographic standards, and optically tuned for the 14px chip.

---

## 2. Design System Constraints & Construction Rules

Per `docs/design/icons-and-buttons.md` (§2.1, §4.5, §4.6) and `DESIGN.md`:

1. **Canvas & ViewBox:** Strict `viewBox="0 0 24 24"`.
2. **Live Area:** 20×20 unit live area (`x, y ∈ [2, 22]`), maintaining a 2px outer breathing margin.
3. **Stroke Weight & Geometry:** Constant `stroke-width="2"`, `fill="none"`, `stroke-linecap="round"`, `stroke-linejoin="round"`.
   - *Documented Exception:* Solid filled dots for the Point icon use Lucide's def-level attribute override (`fill: 'currentColor'`, `stroke: 'none'`).
4. **Scale Baseline (14px):** Must maintain high silhouette contrast at 14×14px inside `.layer-type-icon` with zero muddy stroke collisions.
5. **Color Binding (Ingest Color Exception):** In accordance with `icons-and-buttons.md §3` rule 4 and `geometry-icon.ts`, the glyph stroke (and fill for points) carries the layer's **ingest color** (RGB) in dark mode, darkened/saturated in light mode (`filter: brightness(0.5) saturate(1.2)`).

---

## 3. Approved Icon Catalog

### 3.1 Point (`point`)
*Replaces: `map-pin`*

- **Visual Concept:** Tri-point scatter triad (`Option 4A.1`).
- **Cartographic Semantics:** Represents discrete vector coordinate observations (sampling stations, GPS fixes, trees, incidents) across a spatial extent.
- **Construction:** Three solid circular dots arranged in a balanced triangular constellation. Uses $r = 2.5$ (5px drawn diameter), which scales to ~2.9px at 14px—providing sharp, punchy contrast against both dark and light card chips.

#### Canonical Registry Definition (`src/shared/icons.ts`)
```typescript
point: [
  ['circle', { cx: 12, cy: 6, r: 2.5, fill: 'currentColor', stroke: 'none' }],
  ['circle', { cx: 6, cy: 17, r: 2.5, fill: 'currentColor', stroke: 'none' }],
  ['circle', { cx: 18, cy: 15, r: 2.5, fill: 'currentColor', stroke: 'none' }],
],
```

#### SVG Markup
```xml
<svg viewBox="0 0 24 24" fill="none" aria-hidden="true" focusable="false">
  <circle cx="12" cy="6" r="2.5" fill="currentColor" />
  <circle cx="6" cy="17" r="2.5" fill="currentColor" />
  <circle cx="18" cy="15" r="2.5" fill="currentColor" />
</svg>
```

---

### 3.2 LineString (`line`)
*Replaces: `code`*

- **Visual Concept:** Survey Terrace (`Option L2`).
- **Cartographic Semantics:** An authentic vector polyline consisting strictly of 3 straight line segments (zero curves): horizontal base approach $\to$ diagonal ascent $\to$ horizontal high shelf plateau.
- **Topology:** True to GIS vector topology where a polyline is mathematically defined as straight segments between vertices. Diverges completely from Esri Calcite's rigid, symmetrical Z-zig-zag (`line-24`).
- **Coordinate Geometry:** `(3, 17) → (9, 17) → (15, 7) → (21, 7)`.
  - Base segment: 6px horizontal span at $y=17$
  - Ascent segment: dx=6, dy=-10 (59° incline)
  - Shelf segment: 6px horizontal span at $y=7$

#### Canonical Registry Definition (`src/shared/icons.ts`)
```typescript
line: [
  p('M3 17h6l6-10h6'),
],
```

#### SVG Markup
```xml
<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true" focusable="false">
  <path d="M3 17h6l6-10h6" />
</svg>
```

---

### 3.3 Polygon (`polygon`)
*Replaces: `cube`*

- **Visual Concept:** Asymmetric 5-Vertex GIS Boundary (`Option 1A`).
- **Cartographic Semantics:** Irregular closed planar polygon representing cadastral parcels, zoning envelopes, municipal districts, or watershed catchments.
- **Benchmarking:** Calcite `polygon-24` and font-gis `geom/polygon-o`.
- **Coordinate Geometry:** `(5, 6) → (18, 4) → (20, 16) → (12, 20) → (4, 15) → Z`.
  - Width: 16px (x: 4 to 20)
  - Height: 16px (y: 4 to 20)
  - Optical Center: (11.8, 12.2) — perfectly centered on the 24 grid
- **14px Performance:** Maximum internal negative space. No internal facets or micro-details, ensuring an unmistakable boundary silhouette at small sizes.

#### Canonical Registry Definition (`src/shared/icons.ts`)
```typescript
polygon: [
  p('M5 6l13-2 2 12-8 4-8-5z'),
],
```

#### SVG Markup
```xml
<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true" focusable="false">
  <path d="M5 6l13-2 2 12-8 4-8-5z" />
</svg>
```

---

### 3.4 Raster (`raster`)
*Replaces: `cloud`*

- **Visual Concept:** Satellite Imagery & Terrain Peaks (`Option R2`).
- **Cartographic Semantics:** Aerial / satellite imagery, digital elevation models (DEM), and continuous gridded fields.
- **Distinction from Tabular Data:** Features an outer spatial frame, terrain elevation peaks, and an orbital satellite/sun sensor dot. Strictly avoids spreadsheet rows and columns, eliminating any visual confusion with data tables.
- **Coordinate Geometry:**
  - Outer frame: `rect x="3" y="3" width="18" height="18" rx="2"`
  - Terrain ridgeline: `m3 17 5-5 4 4 5-5 4 4` (spans cleanly across x: 3 to 21)
  - Sensor dot: `circle cx="8" cy="8" r="1.5"` (`fill="currentColor"`)

#### Canonical Registry Definition (`src/shared/icons.ts`)
```typescript
raster: [
  ['rect', { x: 3, y: 3, width: 18, height: 18, rx: 2 }],
  p('m3 17 5-5 4 4 5-5 4 4'),
  ['circle', { cx: 8, cy: 8, r: 1.5, fill: 'currentColor', stroke: 'none' }],
],
```

#### SVG Markup
```xml
<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true" focusable="false">
  <rect x="3" y="3" width="18" height="18" rx="2" />
  <path d="m3 17 5-5 4 4 5-5 4 4" />
  <circle cx="8" cy="8" r="1.5" fill="currentColor" stroke="none" />
</svg>
```

---

### 3.5 Tabular Data (`table`)
*New Layer Type for Non-Spatial Tables*

- **Visual Concept:** Canonical Data Table Grid (`Option T1`).
- **Cartographic Semantics:** Non-spatial datasets (standalone CSVs, Parquet tables, survey databases without geometry columns).
- **Registry Alignment:** Reuses the canonical `table` definition already present in `src/shared/icons.ts:160` (Lucide `table`).

#### Canonical Registry Definition (`src/shared/icons.ts`)
```typescript
// Reuses existing definition in src/shared/icons.ts
table: [
  p('M12 3v18'),
  ['rect', { width: 18, height: 18, x: 3, y: 3, rx: 2 }],
  p('M3 9h18'),
  p('M3 15h18'),
],
```

#### SVG Markup
```xml
<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true" focusable="false">
  <rect x="3" y="3" width="18" height="18" rx="2" />
  <path d="M12 3v18M3 9h18M3 15h18" />
</svg>
```

---

## 4. Full Layer Stack Visual Simulation

In the layer panel, all 5 layer types exhibit completely distinct silhouettes, line weights, and structural shapes with zero perceptual overlap:

```
┌──────┬────────────────────────────────────────────────────────┐
│ [●●●]│ weather_stations.geojson                       53,316  │  Point (3 solid observation dots)
├──────┼────────────────────────────────────────────────────────┤
│ [_/-]│ transit_corridors.geojson                         145  │  LineString (Survey terrace traverse)
├──────┼────────────────────────────────────────────────────────┤
│ [ ⬠ ]│ municipal_parcels.geojson                       2,299  │  Polygon (Planar boundary envelope)
├──────┼────────────────────────────────────────────────────────┤
│[[⏶•]]│ sentinel2_ortho.tif                            Raster  │  Raster (Satellite imagery & terrain)
├──────┼────────────────────────────────────────────────────────┤
│[[⊞] ]│ census_demographics.csv                         Table  │  Tabular (Non-spatial spreadsheet)
└──────┴────────────────────────────────────────────────────────┘
```

---

## 5. Implementation Roadmap (When Transitioning to Code)

When the project transitions from design to replacement, the following changes will be executed:

### Step 1: Update Icon Registry (`src/shared/icons.ts`)
Add `point`, `line`, `polygon`, and `raster` to the `DEFS` dictionary:

```typescript
// original · cartographic point observation cluster
point: [
  ['circle', { cx: 12, cy: 6, r: 2.5, fill: 'currentColor', stroke: 'none' }],
  ['circle', { cx: 6, cy: 17, r: 2.5, fill: 'currentColor', stroke: 'none' }],
  ['circle', { cx: 18, cy: 15, r: 2.5, fill: 'currentColor', stroke: 'none' }],
],
// original · survey terrace polyline
line: [
  p('M3 17h6l6-10h6'),
],
// original · benchmark calcite:polygon-24
polygon: [
  p('M5 6l13-2 2 12-8 4-8-5z'),
],
// original · satellite imagery & terrain raster
raster: [
  ['rect', { x: 3, y: 3, width: 18, height: 18, rx: 2 }],
  p('m3 17 5-5 4 4 5-5 4 4'),
  ['circle', { cx: 8, cy: 8, r: 1.5, fill: 'currentColor', stroke: 'none' }],
],
```

### Step 2: Update Layer Geometry Resolver (`src/features/layers/rendering/geometry-icon.ts`)
1. Update `GeometryIconName`:
   ```typescript
   export type GeometryIconName = 'raster' | 'polygon' | 'line' | 'point' | 'table';
   ```
2. Update resolution logic:
   ```typescript
   export function geometryIconName(layer: LayerItem): GeometryIconName {
     if (layer.type === 'raster') return 'raster';
     if (layer.type === ('table' as any) || layer.geometryType === 'None') return 'table';

     const geomTypes = new Set<string>();
     try {
       if (layer.data && layer.data.features) {
         layer.data.features.slice(0, 10).forEach((f: { geometry?: { type?: string } }) => {
           if (f.geometry && f.geometry.type) geomTypes.add(f.geometry.type);
         });
       }
     } catch {
       // Malformed feature data — fall through to default point icon
     }

     if (geomTypes.has('Polygon') || geomTypes.has('MultiPolygon')) return 'polygon';
     if (geomTypes.has('LineString') || geomTypes.has('MultiLineString')) return 'line';
     return 'point';
   }
   ```
3. Update `geometryIcon()` template to set `style="color: ${geometryStroke(layer)}"`:
   ```typescript
   export function geometryIcon(layer: LayerItem): SVGTemplateResult {
     const stroke = geometryStroke(layer);
     return svg`<svg
       class="layer-icon"
       viewBox="0 0 24 24"
       fill="none"
       stroke=${stroke}
       style="color: ${stroke};"
       stroke-width="2"
       stroke-linecap="round"
       stroke-linejoin="round"
       aria-hidden="true"
       focusable="false"
     >${iconShapes(geometryIconName(layer))}</svg>`;
   }
   ```

### Step 3: Update Design System Documentation (`docs/design/icons-and-buttons.md`)
Add the four original icons to the canonical registry table in §6.5 with provenance and benchmark notes.

### Step 4: Verification Suite
Run unit tests to enforce three-emitter parity (`icon()`, `iconMarkup()`, `iconElement()`):
```bash
npm test -- src/shared/icons.spec.ts src/features/layers/rendering/geometry-icon.spec.ts
```

---

## 6. Artifact & Interactive Showcase Reference

The interactive visual testing showcase is maintained at:
`/home/saqib/.gemini/antigravity-cli/brain/1a019f29-2035-485c-a35d-d4c2ec353cd6/scratch/icon-showcase.html`
