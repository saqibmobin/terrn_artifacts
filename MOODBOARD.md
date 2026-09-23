# Terrn — Visual Design Moodboard

**Project:** Terrn Geospatial Analytics Platform (`tern_poc`)  
**Design Paradigm:** Industrial / Utilitarian Cartographic Workstation  
**Aesthetic Core:** Precision Instrument Cockpit floating over a 60 FPS WebGL Data Canvas  
**Date:** August 2026  

---

## 1. Visual Moodboard Collage & Atmosphere

![Terrn Visual Design Moodboard](./terrn_visual_moodboard.jpg)
*Figure 1.1: The Terrn Design Universe — 6-facet directional diamond mark, glowing amber spatial telemetry, floating frosted glass instrument panels, and 3D WebGL topographic vector flows.*

![Terrn UI Atmosphere & Spatial Canvas](./terrn_ui_atmosphere.jpg)
*Figure 1.2: UI Chrome Atmosphere — Frosted glass instruments (`rgba(10,22,40,0.96)`), 48px right rail, precision range sliders with knockout rings, and 60 FPS WebGL vector geometry.*

---

## 2. Core Aesthetic Pillars & Design Philosophy

```
+----------------------------------------------------------------------------------------------------+
|                                    TERRN AESTHETIC DNA MATRIX                                      |
+------------------------------------+----------------------------------+----------------------------+
| 1. THE SPATIAL VOID                | 2. FLOATING INSTRUMENTS          | 3. THE SACRED AMBER SIGNAL |
| Deep Navy (#050D1A) universe       | Frosted acrylic glass panels     | Quarantined exclusively    |
| where dense WebGL vectors,         | (blur 16px) with 1px precision   | for live telemetry,        |
| point clouds & 3D tiles glow.      | steel borders (#1E2A3A).         | coordinates & counts.      |
+------------------------------------+----------------------------------+----------------------------+
| 4. SWISS TYPOGRAPHIC DISCIPLINE    | 5. DIRECTIONAL LIGHT GEOMETRY    | 6. ZERO CONSUMER FLUFF     |
| Geist Sans geometric UI +          | 6-facet diamond simulating       | No drop shadows, no emojis |
| JetBrains Mono sacred telemetry.   | physical crystal refraction.     | in chrome, no transitions  |
| No serifs anywhere in brand.       | Amber centroid @ (19, 21).       | that drop 60 FPS frames.   |
+------------------------------------+----------------------------------+----------------------------+
```

### Pillar 1: The Spatial Void & 60 FPS Velocity
- The application canvas is an infinite, responsive geospatial coordinate space.
- The map canvas is not a background image — it is the **living product**.
- Vector tiles, bathymetric lines, dynamic heatmaps, and extruded 3D meshes render seamlessly with GPU-accelerated WebGL shaders.

### Pillar 2: Floating Glass Instrumentation (The Cockpit)
- Panels do not crowd or divide the map — they hover lightly above it like avionics instruments in an aircraft cockpit.
- **Dark Mode:** `background: rgba(10, 22, 40, 0.96); backdrop-filter: blur(16px); border: 1px solid #1E2A3A;`
- **Light Mode:** `background: rgba(243, 246, 251, 0.92); backdrop-filter: blur(16px); border: 1px solid #C8D4E6;`
- Depth is communicated through **tint + blur + crisp 1px borders** — never fuzzy, consumer drop shadows.

### Pillar 3: The Amber Signal Law (The Heartbeat)
- In aviation and radar systems, amber signifies an active signal. In Terrn, amber is **sacred**.
- `#F59E0B` (Dark Mode) / `#D97706` (Light Mode) is reserved **exclusively** for live spatial telemetry:
  - Coordinate readouts (`37.7749° N, 122.4194° W`)
  - Feature counts (`1,429,850 points`)
  - Live ingestion chips (`● Live`)
  - SQL query execution metrics & active slider values
- By banning amber from general buttons, hovers, or icons, its appearance immediately triggers subconscious recognition as live data.

### Pillar 4: Typographic Rigor
- **Geist Sans (UI & Display):** High-density geometric sans engineered specifically for developer and analyst tools.
- **JetBrains Mono (Data & Tables):** Strict tabular numbers where character widths never jitter during real-time streaming updates.

---

## 3. Color & Material Palette

```
+----------------------------------------------------------------------------------------------------+
| BRAND ACCENTS                     SURFACE ELEVATION (DARK)           SURFACE ELEVATION (LIGHT)     |
+-----------------------------------+----------------------------------+-----------------------------+
| [■] Electric Blue   #2563EB       | [■] Void (Base)     #050D1A      | [■] Light Void (Base) #E4EBF5|
| [■] Arc Blue        #60A5FA       | [■] Deep Navy       #0A1628      | [■] Panel Surface     #F3F6FB|
| [■] Amber (Live)    #F59E0B       | [■] Navy Mid (Card) #0F2240      | [■] Raised Card (Pure)#FFFFFF|
| [■] Amber Light     #FCD34D       | [■] Slate (Border)  #1E2A3A      | [■] Border Line       #C8D4E6|
| [■] Emerald Success #10B981       | [■] Steel (Strong)  #334155      | [■] Border Strong     #9DB0CC|
| [■] Crimson Danger  #EF4444       | [■] Mist (Text 2)   #94A3B8      | [■] Slate Text (2)    #4A5A72|
| [■] Raster Magenta  #D946EF       | [■] Ice White (Text)#F1F5F9      | [■] Primary Text (1)  #0F172A|
+-----------------------------------+----------------------------------+-----------------------------+
```

### Materiality & Tactility

| Texture / Material | Visual Behavior & Formula | Intent & Perception |
|---|---|---|
| **Frosted Glass (Dark)** | `rgba(10, 22, 40, 0.96)` + `blur(16px)` + `1px #1E2A3A` | Industrial instrument glass hovering above cartographic layers |
| **Frosted Glass (Light)**| `rgba(243, 246, 251, 0.92)` + `blur(16px)` + `1px #C8D4E6` | Clean architectural glass with subtle cool-blue underlay |
| **Laser Focus Rings** | `box-shadow: 0 0 0 2px #2563EB` on `:focus-visible` | Tactile keyboard navigation with zero ambiguity |
| **Knockout Slider Ring**| 12px thumb with `2px solid var(--surface-raised)` | Thumb reads as an integrated mechanical knockout disc |
| **Rotated Chevron** | `lucide:chevron-right` rotated 90° via token data URI | Structural icon consistency between buttons and dropdowns |

---

## 4. Emotional Tone & Brand Personality

### What Terrn IS:
- **A Precision Workstation:** Like high-end IDEs (VS Code), CAD software (AutoCAD), professional cartography (QGIS, Mapbox Studio), and avionics instruments.
- **Serious & Trustworthy:** Design quality signals high numerical integrity and DuckDB-WASM execution speed.
- **Data-Dense & Efficient:** Compact 4px spacing rhythm where every square pixel maximizes map canvas visibility.

### What Terrn IS NOT:
- ❌ **Not a Casual Consumer App:** No pastel bubble buttons, no rounded cartoon mascots, no gradient fluff.
- ❌ **Not a Cluttered Legacy GIS Tool:** No 1990s bevels, no crowded 4-row toolbars, no modal dialog traps.
- ❌ **Not a Generic SaaS Template:** No default Inter fonts, no purple gradients, no standard flat-white light modes.

---

## 5. Architectural References & Related Docs

- **Canonical Master Design System:** [`/tern/DESIGN.md`](file:///tern/DESIGN.md)
- **Interactive Visual Design Studio:** [`/tern/visual-design-system.html`](file:///tern/visual-design-system.html)
- **Icon & Button Specification:** [`/tern/tern_poc/docs/design/icons-and-buttons.md`](file:///tern/tern_poc/docs/design/icons-and-buttons.md)
- **CSS Token Implementation:** [`/tern/tern_poc/src/core/styles/variables.css`](file:///tern/tern_poc/src/core/styles/variables.css)
