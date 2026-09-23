# Design System — Terrn

**Document Status:** Canonical & Authoritative Master Specification  
**Scope:** Complete Visual Design System for Terrn Platform (`tern_poc` & Production)  
**Version:** 2.4.0  
**Last Updated:** August 2026  

---

## 1. Product Context & Design Philosophy

Terrn is a high-performance, client-side geospatial analytics platform capable of rendering massive vector and raster datasets at 60 FPS directly on the WebGL canvas with zero mandatory backend compute.

```
+---------------------------------------------------------------------------------------------------+
|                                       TERRN SPATIAL CANVAS                                        |
|                                                                                                   |
|  +-----------------------+     +----------------------------------+     +---+ +-----------------+ |
|  | TOPBAR (44px)         |     | MAP CANVAS (Deck.gl / WebGL)     |     | R | | RIGHT PANEL     | |
|  | Mark | Nav | Copilot  |     | 60 FPS Viewport                  |     | A | | (360px)         | |
|  +-----------------------+     | Floating Instruments             |     | I | | Style / Info /  | |
|  | LEFT PANEL (300px)    |     | Map Controls (40px)              |     | L | | Table / AI      | |
|  | Layers / Ingest       |     | Coordinates: var(--text-data)    |     |48p| | Frosted Glass   | |
|  +-----------------------+     +----------------------------------+     +---+ +-----------------+ |
+---------------------------------------------------------------------------------------------------+
```

### Core Design Principles

1. **The Map Canvas is the Product:** The WebGL canvas occupies 100% of the viewport. Panels, rails, toolbars, and modals are precision instrument chrome floating above the geographic data.
2. **Industrial & Utilitarian Aesthetics:** Function-first precision engineering. Every border, token, and glyph earns its position through utility. No decorative ornamentation, no extraneous shadows, no gratuitous animations.
3. **Restrained Color Discipline:** Neutrals carry the structural weight. Electric blue serves as the sole operational accent. Amber is strictly quarantined for live telemetry and data signals.
4. **Typographic Hierarchy as Instrument Readability:** Clear separation between human-readable UI text (Geist Sans) and high-density numeric spatial data (JetBrains Mono).
5. **Light Mode through Tint, Frosted Glass, and Precision Borders:** Never flat white paper; light mode achieves depth through background blue-gray tint (`#E4EBF5`), frosted glass (`backdrop-filter: blur(16px)`), and crisp borders (`#C8D4E6`), never fuzzy drop shadows.

---

## 2. Design Tokens & CSS Custom Property Architecture

All styling across the Terrn application originates from CSS custom properties defined in `src/core/styles/variables.css`. Direct hardcoded hex or RGB color literals outside `variables.css` are strictly banned and enforced via automated CI tests (`no-hardcoded-colors.spec.ts`).

### Master Token Definition

```css
:root {
  /* Primary Brand Palette */
  --void:          #050D1A;
  --deep-navy:     #0A1628;
  --electric:      #2563EB;
  --arc-blue:      #60A5FA;
  --amber:         #F59E0B;
  --ice-white:     #F1F5F9;

  /* Secondary Neutral & Data Accents */
  --navy-mid:      #0F2240;
  --slate:         #1E2A3A;
  --steel:         #334155;
  --mist:          #94A3B8;
  --amber-light:   #FCD34D;
  --mark-navy:     #3B4D67;
  --mark-ice:      #C7D2E9;
  --accent-pink:   #D946EF;

  /* Status Colors */
  --green:         #10B981;
  --red:           #EF4444;

  /* Semantic Theme Aliases (Dark Mode Default) */
  --bg:            var(--void);
  --surface:       var(--deep-navy);
  --surface-raised: var(--navy-mid);
  --border:        var(--slate);
  --border-subtle: rgba(30, 42, 58, 0.8);
  --border-strong: var(--steel);
  --text-primary:  var(--ice-white);
  --text-secondary: var(--mist);
  --text-data:     var(--amber);
  --on-accent:     #F1F5F9;

  /* Focus & Interactive States */
  --hover-surface:       rgba(96, 165, 250, 0.07);
  --hover-surface-solid: #243244;
  --active-border:       rgba(37, 99, 235, 0.5);
  --focus-ring:          0 0 0 2px var(--electric);
  --focus-ring-inset:    inset 0 0 0 2px var(--electric);
  --panel-glass:         rgba(10, 22, 40, 0.96);
  --scrim:               rgba(5, 13, 26, 0.6);

  /* Typography */
  --sans: 'Geist', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  --mono: 'JetBrains Mono', ui-monospace, 'Cascadia Code', monospace;

  /* Chrome Layout Metrics */
  --topbar-h:      44px;
  --left-panel-w:  300px;
  --right-rail-w:  48px;
  --right-panel-w: 360px;

  /* Icon Size Hierarchy */
  --icon-xs:       12px;
  --icon-sm:       14px;
  --icon-md:       16px;
  --icon-lg:       18px;

  /* Border Radius Hierarchy */
  --radius-sm:     4px;
  --radius-md:     8px;
  --radius-lg:     12px;
  --radius-pill:   20px;

  /* Overlay Stacking */
  --z-modal:       2000;
  --z-overlay:     2100;
}
```

---

## 3. Color Architecture & Surface Hierarchy

Terrn utilizes a dual-theme surface system where elevation is expressed through tonal luminosity shifts and frosted-glass blurs rather than standard elevation shadows.

```
DARK MODE ELEVATION:
[ --void (#050D1A) Base ] ──► [ --surface (#0A1628) Panel ] ──► [ --surface-raised (#0F2240) Card ]

LIGHT MODE ELEVATION:
[ --bg (#E4EBF5) Base ] ──► [ --surface (#F3F6FB) Panel ] ──► [ --surface-raised (#FFFFFF) Card ]
```

### Color Token Reference Table

| Category | Token | Dark Hex | Light Hex | Semantic Role & Application |
|---|---|---|---|---|
| **Accent** | `--electric` | `#2563EB` | `#2563EB` | Primary CTAs, active selections, focus rings, lead highlights |
| **Accent** | `--arc-blue` | `#60A5FA` | `#60A5FA` | Hover highlights, active tab text, secondary interactive |
| **Data Signal** | `--amber` | `#F59E0B` | `#D97706` | **Live Data Only:** Coordinate readouts, counts, telemetry |
| **Data Signal** | `--amber-light` | `#FCD34D` | `#F59E0B` | Live pulse badge indicator accent (`● Live` chip) |
| **Data Signal** | `--accent-pink` | `#D946EF` | `#D946EF` | Raster layer geometry glyph data encoding only |
| **Status** | `--green` | `#10B981` | `#10B981` | Success state, healthy connection, valid SQL |
| **Status** | `--red` | `#EF4444` | `#EF4444` | Error state, destructive action, invalid query |
| **Text Fixed** | `--on-accent` | `#F1F5F9` | `#F1F5F9` | Foreground text/icons on solid `--electric` or `--red` fills |
| **Base Surface** | `--bg` | `#050D1A` (`--void`) | `#E4EBF5` | Application background / underlay canvas |
| **Panel Surface**| `--surface` | `#0A1628` (`--deep-navy`) | `#F3F6FB` | Sidebars, topbar, bottom toolbars, floating panels |
| **Card Surface** | `--surface-raised`| `#0F2240` (`--navy-mid`) | `#FFFFFF` | Elevated cards, input fields, dropdown backgrounds |
| **Border** | `--border` | `#1E2A3A` (`--slate`) | `#C8D4E6` | Structural borders, panel boundaries, dividers |
| **Subtle Border**| `--border-subtle`| `rgba(30,42,58,0.8)` | `rgba(200,212,230,0.7)` | Intra-card dividers, subtle table grid lines |
| **Strong Border**| `--border-strong`| `#334155` (`--steel`)| `#9DB0CC` | Floating chrome edges over map canvas, switch borders |
| **Primary Text** | `--text-primary` | `#F1F5F9` (`--ice-white`)| `#0F172A` | Primary headings, body copy, active labels |
| **Secondary Text**| `--text-secondary`| `#94A3B8` (`--mist`) | `#4A5A72` | Subtitles, inactive tabs, default icon fill, placeholders |
| **Data Text** | `--text-data` | `#F59E0B` | `#D97706` | Numeric counts, coordinates, SQL editor data outputs |

### The Amber Signal Rule

Amber is **strictly reserved** for live data and spatial signals:
- Map coordinate and elevation telemetry displays (`37.7749° N, 122.4194° W`)
- Layer feature count badges (`14,290 features`)
- Live real-time stream indicators (`● Live`)
- AI query execution token counts & spatial filter readouts
- Slider value readouts (`.slider-value`)

> [!CAUTION]
> **Strict Restriction:** NEVER use amber as a general UI button background, generic hover state, decorative border, or arbitrary icon color. Its rarity is what establishes immediate subconscious recognition as live data.

### Contrast & Accessibility (WCAG 2.1 AA/AAA)

| Surface Pairing | Contrast Ratio | WCAG Compliance | Context |
|---|---|---|---|
| `--ice-white` (`#F1F5F9`) on `--void` (`#050D1A`) | **14.8:1** | AAA | Dark Mode Body & Headings |
| `--mist` (`#94A3B8`) on `--deep-navy` (`#0A1628`) | **6.1:1** | AA | Dark Mode Secondary Text |
| `--amber` (`#F59E0B`) on `--deep-navy` (`#0A1628`) | **8.4:1** | AAA | Dark Mode Data Readouts |
| `--electric` (`#2563EB`) on `--void` (`#050D1A`) | **4.9:1** | AA (Large/UI) | Dark Mode Accent Controls |
| `#0F172A` on `#F3F6FB` | **15.4:1** | AAA | Light Mode Body & Headings |
| `#4A5A72` on `#E4EBF5` | **5.9:1** | AA | Light Mode Secondary Text |
| `#D97706` on `#FFFFFF` | **4.6:1** | AA | Light Mode Data Readouts |
| `#D97706` on `#F3F6FB` | **4.5:1** | AA | Light Mode Panel Data |

---

## 4. Logo Mark & Brand Geometry

The Terrn brand identity is anchored by a six-facet faceted diamond with directional ambient lighting simulation converging at an amber centroid.

```
       viewBox="0 0 38 42"
               (19, 0)
                 /\
                /  \
   Upper Left  / |  \  Upper Right
   #152F59    /  |   \ #60A5FA
             /   |    \
   (0, 21)  +----+-----+ (38, 21)
             \   |  . /
   Lower Left \  |   /  Lower Right
   #275292     \ |  /   #2563EB (Electric)
                \| /
                 \/
               (19, 42)
        Centroid @ (19, 21)
```

### Complete SVG Definition

```xml
<svg width="38" height="42" viewBox="0 0 38 42" fill="none" xmlns="http://www.w3.org/2000/svg">
  <!-- Left Spines & Facets -->
  <path d="M19 0L19 42L13 20.6912L19 0Z" fill="#3B4D67"/>
  <path d="M13.3255 21L19 42L0 21H13.3255Z" fill="#152F59"/>
  <path d="M13.3255 21L19 3.57628e-07L0 21H13.3255Z" fill="#275292"/>

  <!-- Right Spines & Facets -->
  <path d="M19 0L19 42L25 20.6912L19 0Z" fill="#F1F5F9"/>
  <path d="M24.6745 21L19 42L38 21H24.6745Z" fill="#2563EB"/>
  <path d="M24.6745 21L19 3.57628e-07L38 21H24.6745Z" fill="#60A5FA"/>

  <!-- Amber Centroid Point & Ring -->
  <circle cx="19" cy="21" r="2" fill="#F59E0B"/>
  <circle cx="19" cy="21" r="3.75" stroke="#F59E0B" stroke-opacity="0.4" stroke-width="0.5"/>

  <!-- Glowing Centroid Filter (≥ 32px) -->
  <g filter="url(#terrn_centroid_glow)">
    <circle cx="19" cy="21" r="4" fill="#F59E0B" fill-opacity="0.7"/>
  </g>
  <defs>
    <filter id="terrn_centroid_glow" x="11" y="13" width="16" height="16" filterUnits="userSpaceOnUse" color-interpolation-filters="sRGB">
      <feFlood flood-opacity="0" result="BackgroundImageFix"/>
      <feBlend mode="normal" in="SourceGraphic" in2="BackgroundImageFix" result="shape"/>
      <feGaussianBlur stdDeviation="2" result="effect1_foregroundBlur"/>
    </filter>
  </defs>
</svg>
```

### Brand Logo Rules

1. **Single Mark Across Both Themes:** The left facets (`#152F59`, `#275292`) provide high contrast against dark backgrounds. The right spine (`#F1F5F9`) gently recedes on light surfaces. This gives the mark an authentic three-dimensional physical presence responding to ambient light.
2. **Glow Threshold:** At sizes $\ge 32\text{px}$, render the full centroid dot, ring, and Gaussian glow filter. At sizes $< 32\text{px}$, drop the Gaussian blur filter to prevent anti-aliasing fuzziness while retaining the solid dot and ring.
3. **Horizontal Lockup:** `[SVG Mark (22px)]` + `8px Gap` + `1px Vertical Divider in var(--border)` + `8px Gap` + `Geist Sans 500 Wordmark "Terrn"`.

---

## 5. Typography Architecture

Terrn relies strictly on two high-character typeface families, self-hosted via modern WOFF2 formats to ensure offline operation and zero latency.

- **Primary UI & Display:** **Geist Sans** (`wght@300..700`) — Developer-tool geometry, clean letterforms, tight numerical tabular support.
- **Data, Code & Coordinates:** **JetBrains Mono** (`wght@400..600`) — Dedicated to GIS telemetry, SQL commands, coordinate readouts, and table cells.
- **Serifs:** Absolutely forbidden anywhere in the system.

### Type Scale Matrix

| Role | Family | Weight | Size | Line Height | Tracking | Target Usage |
|---|---|---|---|---|---|---|
| **Display / Hero** | Geist Sans | 700 (Bold) | 48px | 1.1 | -2.5px | Welcome screens, marketing hero |
| **H1** | Geist Sans | 600 (SemiBold) | 32px | 1.2 | -1.2px | View headers, modal titles |
| **H2** | Geist Sans | 500 (Medium) | 22px | 1.3 | -0.6px | Panel headers, section dividers |
| **H3** | Geist Sans | 500 (Medium) | 16px | 1.4 | -0.3px | Group headers, card titles |
| **Body (Standard)**| Geist Sans | 400 (Regular) | 14px | 1.5 | 0 | General UI text, descriptions |
| **Body (Small)** | Geist Sans | 400 (Regular) | 12px | 1.4 | 0 | Secondary labels, hints |
| **Eyebrow / Label**| Geist Sans | 600 (SemiBold) | 11px | 1.0 | +0.1em CAPS | Section eyebrows, input headers |
| **Rail Caption** | Geist Sans | 600 (SemiBold) | 8px | 1.0 | +0.04em CAPS| Rail stacked icon captions (`STYLE`, `AI`) |
| **Data Mono** | JetBrains Mono | 400 / 500 | 13px | 1.4 | 0 | Coordinates, feature counts, tables |
| **Data Small** | JetBrains Mono | 500 (Medium) | 11px | 1.2 | 0 | Slider readout values, badge counts |

---

## 6. Spatial Grid, Spacing & Layout Architecture

Terrn operates on a strict **4px modular grid**. Every margin, padding, gap, and dimension must be an exact multiple of 4px.

### Spacing Scale

```
4px   (1x) ──► Tight icon gaps, inline badge padding
8px   (2x) ──► Button vertical padding, input inline gaps
12px  (3x) ──► Panel horizontal padding, section gaps
16px  (4x) ──► Standard card padding, modal content padding
24px  (6x) ──► Ingest dropzone padding, card section separation
32px  (8x) ──► Section dividers, major dialog separation
48px (12x) ──► Empty state padding, large container margins
64px (16x) ──► Splash / onboarding layout margins
```

### Border Radius Hierarchy

- `--radius-sm` (`4px`): Dense toolbar buttons, inline badges, color preview swatches, scrollbar thumbs.
- `--radius-md` (`8px`): Standard buttons, inputs, selects, segmented controls, cards, tooltips.
- `--radius-lg` (`12px`): Floating panels, modal containers, notification sheets.
- `--radius-pill` (`20px`): Chips, live status tags, switch tracks, progress bars.

### Fixed Layout Chrome Tokens

| Token | Dimension | Role in Application |
|---|---|---|
| `--topbar-h` | `44px` | Top navigation bar containing logo, project title, telemetry, and quick actions |
| `--left-panel-w` | `300px` | Layer inventory, ingest drawer, and layer order manager |
| `--right-rail-w` | `48px` | Vertical icon tool rail (Style, Info, Table, Copilot, Settings) |
| `--right-panel-w`| `360px` | Active instrument inspector panel docked beside right rail |

---

## 7. Interactive Component Hierarchy

The Terrn component system is defined through pure, lightweight CSS classes and isolated Web Components designed for zero frame drops.

### 7.1 Button System (5 Variants Only)

```
[ .btn-primary ]  ──► Solid --electric background, --on-accent text, brightness(1.08) hover
[ .btn-ghost ]    ──► Transparent background, --border edge, --hover-surface wash
[ .btn-danger ]   ──► Destructive ghost structure, red wash on hover
[ .btn-icon ]     ──► Square icon wrapper (22px, 26px, 36px, 40px), --hover-surface wash
[ .btn-link ]     ──► Text-only inline action, underline on hover
```

#### Button Sizes & Touch Minimums

| Class | Dimensions | Typography | Icon Target | Touch / Target Usage |
|---|---|---|---|---|
| `.btn-sm` | `4px 10px` | 11px / 500 | `--icon-xs` (12px) | Table toolbars, dense panel actions |
| `.btn-md` | `6px 12px` | 12px / 500 | `--icon-sm` (14px) | Default modal & card actions |
| `.btn-lg` | `36×36px` | — | `--icon-lg` (18px) | Topbar action buttons (Touch min 36px) |
| `.btn-xl` | `40×40px` | — | `--icon-lg` (18px) | Map controls, tool rail (Touch min 40px) |

#### Button Modifiers & States

- `.btn-primary.danger`: Solid red confirmation button for irreversible destruction.
- `.btn-ghost.accent`: Toolbar lead action highlighting secondary workflow.
- `.btn-icon.active`: Selected toggled state (`color-mix(in srgb, var(--electric) 15%, transparent)` + `--arc-blue`).
- `.btn-icon.stacked`: Vertical icon + uppercase 8px label layout for the 48px right rail.
- `.btn.loading`: Content hidden, centered 14px spinning ring indicator.
- `.focus-inset`: Uses `box-shadow: var(--focus-ring-inset)` for buttons in `overflow: hidden` containers.

### 7.2 Controls Vocabulary

#### 1. Range Slider (`.slider`)
A unified cross-browser range input supporting both WebKit and Gecko engines:
- **Track:** 4px height, `--border` fill, `--radius-sm`.
- **Thumb:** 12×12px, `--electric` fill with a **2px `--surface-raised` ring** ensuring visibility across both dark and white surfaces.
- **Value Readout:** Sits in `.slider-value` using `--mono`, 11px, `--text-data` (Amber).

#### 2. Native Select (`.select` / `.select-sm`)
Retains the native `<select>` element for full keyboard accessibility, type-ahead, and OS mobile pickers while styling the closed trigger:
- **Appearance:** `appearance: none`, `--surface-raised` background.
- **Dropdown Indicator:** `--select-chevron` data URI token rendering Lucide chevron at stroke-width 2, rotated 90°.
- **Border:** `--border` transitioning to `--arc-blue` on hover and `--focus-ring` on focus.

#### 3. Segmented Control (`.segmented`)
Mutually exclusive tab switches (e.g. Basemap Picker: Vector / Satellite / Dark):
- Container: `--surface-raised` background, `--border` stroke, 3px inner padding.
- Item: `.segmented-item` with active state turning solid `--electric` with `--on-accent` text.

#### 4. Tab Strip (`.tabs`)
- Horizontal navigation bar where active item `.tab-item.active` displays a **2px `--electric` border rule on the content-facing edge** (bottom for top strips, top for `.tabs-below`).

#### 5. Custom Boolean Switch (`<terrn-switch>`)
Standardized custom Web Component for all boolean settings:
- **Track:** 34×20px, `--radius-pill`, `--border-strong` toggling to `--electric` when active.
- **Knob:** 14×14px circle in `--text-primary` smoothly translating $14\text{px}$.
- **A11y:** `role="switch"`, `aria-checked`, keyboard accessible via Enter/Space.

#### 6. Interactive Chips (`.chip`) vs Static Badges (`.badge`)
- `.chip`: Real `<button>`, `--radius-pill`, `--border` outline, interactive hover.
- `.badge`: Static `<span>`, JetBrains Mono 11px, electric tint wash, **strictly no hover state**.

---

## 8. Iconography & Symbol System

### 8.1 The Single Family Rule: Lucide

Terrn standardizes exclusively on **Lucide v1.27.0** (ISC / MIT license) rendered at:
```xml
viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"
```

1. **Stroke Consistency:** All icons are drawn with `stroke-width: 2` at every size. Per-icon stroke overrides are banned.
2. **CurrentColor Only:** Icons inherit color from the parent control's CSS token. (Only layer geometry swatches carry ingest stroke colors).
3. **No Intrinsic Sizes:** Registry SVGs carry no `width` or `height` attributes; dimensions are governed by `--icon-*` tokens.
4. **Icons are NEVER Amber:** Amber is reserved for live telemetry. Icons are `--text-secondary` at rest, `--text-primary` or `--arc-blue` on hover, `--red` for destructive actions.
5. **No Text Glyphs or Emoji:** Banned characters (`×`, `✕`, `▲`, `▼`, `▾`, `●`, `■`, `➔`, emoji) must never be used in UI chrome.

### 8.2 Icon Size Hierarchy

| Token | Dimension | Applied Locations |
|---|---|---|
| `--icon-xs` | `12px` | Dense table headers, sub-menus, `.btn-icon.btn-sm`, select indicators |
| `--icon-sm` | `14px` | Layer inventory cards, pane headers, legend items, `.btn-icon.btn-md` |
| `--icon-md` | `16px` | Modal headers, toast notifications, dialog actions |
| `--icon-lg` | `18px` | Topbar instruments, 48px right rail, 40px map tool controls |
| *(Decorative)*| `32px / 48px` | Empty state watermarks (32px @ 0.3 opacity), mobile blocking screen (48px) |

---

## 9. Motion & 60 FPS Performance Contract

Because Terrn executes intensive WebGL shader pipelines and client-side DuckDB-WASM operations, UI transitions must never cause layout recalculations or drop frames.

### Motion Tokens

| Purpose | Duration | Easing Curve | Application |
|---|---|---|---|
| **Micro-interaction** | `150ms` | `ease` | Button hover, select highlight, switch toggle |
| **Panel Slide** | `280ms` | `cubic-bezier(0.16, 1, 0.3, 1)` | Left layer panel & right inspector open/close |
| **Modal Entry** | `200ms` | `cubic-bezier(0.16, 1, 0.3, 1)` | Modal dialog scale + fade in |
| **Toast / Alert** | `250ms` | `cubic-bezier(0.16, 1, 0.3, 1)` | Toast notification slide up |
| **Overlay Scrim** | `200ms` | `ease` | Modal background dimming fade |

### The Specific Property Transition Rule

> [!IMPORTANT]
> **Strict Performance Rule:** NEVER write `transition: all`. Layout-triggering properties (`width`, `height`, `margin`, `padding`, `top`) recalculate DOM layout trees and stall GPU frame swaps. Always enumerate exact properties:
> ```css
> /* CORRECT */
> transition: background 0.15s ease, color 0.15s ease, border-color 0.15s ease, box-shadow 0.15s ease;
> 
> /* STRICTLY FORBIDDEN */
> transition: all 0.15s ease;
> ```

### Reduced Motion Guarantee

All motion, keyframes, and transitions are instantly zeroed when the user system prefers reduced motion:
```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

---

## 10. Automated Enforcement & CI Test Suite

The integrity of this Design System is strictly enforced by automated Vitest suites in `src/core/styles/` and Playwright E2E suites:

1. **`src/core/styles/no-hardcoded-colors.spec.ts`**: Verifies that zero raw hex, rgb, or hsl color literals exist outside `variables.css`.
2. **`src/core/styles/design-system.spec.ts`**: Enforces the ban on text glyphs and emoji, validates that all inline SVG paths match registry hashes, guarantees no `transition: all`, and validates that all `<select>` controls use `.select`.
3. **`src/core/styles/button-classes.spec.ts`**: Ensures button class names form a closed vocabulary with zero unrecognized variants or un-prefixed sizes.
4. **`src/shared/icons.spec.ts`**: Validates the Lucide icon registry contracts, stroke invariants, and SVG output parity.
5. **`e2e/icon-sizing.spec.ts` & `e2e/visibility-toggle-state.spec.ts`**: Real browser rendering tests verifying no 0×0 collapsed SVGs and valid computed styles.

---

## 11. Decisions Log & Architectural Record

| Date | Architectural Decision | Core Rationale |
|---|---|---|
| **2026-06-30** | Typography upgrade from Inter to Geist Sans | Inter has become generic SaaS default; Geist Sans delivers geometric precision suited for developer/GIS tooling. |
| **2026-06-30** | Light Mode surface stack (`#E4EBF5` → `#F3F6FB` → `#FFFFFF`) | Eliminates flat "sheet of paper" look; mirrors dark mode depth by making the canvas background the most colored surface. |
| **2026-06-30** | No box-shadows on Light Mode chrome | Box shadows read as consumer/decorative; crisp borders (`#C8D4E6`) and frosted glass read as precision cartography tools. |
| **2026-06-30** | Amber darkened to `#D97706` in Light Mode | `#F59E0B` fails WCAG AA contrast against light backgrounds; `#D97706` maintains AA compliance while preserving data signal semantics. |
| **2026-07-25** | Settings moved from fixed modal to 48px Right Rail | Eliminates overlay collisions over map controls and unifies rail tab state machine. |
| **2026-07-28** | Lucide adopted as single icon family | Replaced Heroicons because Lucide natively targets `stroke-width: 2` on a 24px grid with 1,600+ spatial-ready icons. |
| **2026-07-28** | Calcite UI rejected for direct artwork vendoring | Blocked due to proprietary Esri Master License Agreement; retained solely as a concept benchmark. |
| **2026-07-28** | Absolute stroke-width 2 rule | Abolished all per-icon stroke overrides; if an icon appears thin, its token size is stepped up. |
| **2026-07-29** | Unified `.select` recipe with `--select-chevron` token | Eliminated 27 fragmented select styles while preserving native accessibility and keyboard handling. |
| **2026-08-27** | Master Visual Design System document consolidated at `/tern/` | Unified architectural reference across core engine, MCP server, and web client. |

---

*Authored by the Terrn Core Engineering & Design Systems Team.*
