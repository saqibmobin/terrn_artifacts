# Terrn Voice System Review & PR 31–35 Compliance Audit

**Status:** Approved Reference & Review  
**Date:** 2026-09-18  
**Scope:** Architecture of [`VOICE.md`](file:///tern/tern_poc/VOICE.md), enforcement test harness [`src/core/styles/ui-text.spec.ts`](file:///tern/tern_poc/src/core/styles/ui-text.spec.ts), and exhaustive compliance audit of Pull Requests 31 through 35.

---

## Executive Summary

Terrn’s voice system, codified in [`VOICE.md`](file:///tern/tern_poc/VOICE.md), establishes a normative, engineering-grade standard for how the product speaks across all user-facing surfaces. Unlike conventional product copy guides that exist as disconnected marketing guidelines, Terrn’s voice system is directly derived from the industrial/utilitarian product philosophy in [`DESIGN.md`](file:///tern/tern_poc/DESIGN.md) (*"an expert tool speaking to an expert"*) and is enforced via automated AST and literal extraction tests in [`src/core/styles/ui-text.spec.ts`](file:///tern/tern_poc/src/core/styles/ui-text.spec.ts).

Between PR 31 and PR 35, the entire user-facing surface area—including styling controls, file ingestion errors, chrome tooltips, attribute table interactions, destructive modals, and documentation—was systematically refactored. This reduced legacy voice violations in [`src/core/styles/ui-text.baseline.json`](file:///tern/tern_poc/src/core/styles/ui-text.baseline.json) from **141 down to 0**, creating a completely clean, unified interface voice.

---

# Part 1: Comprehensive Review of `VOICE.md`

## 1. Architectural Foundations & Strengths

### A. The Authority and Inheritance Model (§0)
Terrn cleanly partitions responsibility across three governing documents:
```
DESIGN.md ────────────► How Terrn looks      (tokens: color, typography, spacing, motion)
docs/design/
  icons-and-buttons.md ► How Terrn is operated (icons, button variants, interactive controls)
VOICE.md ─────────────► How Terrn speaks     (voice, tone, mechanics, glossary, words per surface)
```
This separation of concerns prevents cross-cutting ambiguity: `icons-and-buttons.md` owns a button’s interactive variant, while `VOICE.md` owns its text label.

### B. Bounded Voice Traits (§2)
Rather than abstract platitudes (e.g., *"be clear"*), the specification defines four core traits bounded by their failure thresholds:
* **Precise** *(not pedantic)*: Exact counts, real object names, and correct units. Never more detail than the next action requires.
* **Direct** *(not curt)*: Lead with the verb or outcome. Cut words that do not change meaning, but keep the one that indicates what to do next.
* **Calm** *(not cold)*: Plain-spoken failure reporting. No exclamation marks, no alarmism, no apology theater, and no blame.
* **Peer** *(not gatekeeping)*: Speak to an analyst who knows GIS. Use standard GIS terminology; never invent grandiose marketing labels, and never over-explain concepts the analyst already understands.

### C. Domain Polysemy Disambiguation (§7 Glossary)
Geospatial tooling often conflates terminology between desktop GIS (QGIS, ArcGIS), web applications, and database engines. `VOICE.md` strictly resolves these:
* **`layer` vs. `file`:** A *file* on disk becomes a *layer* once rendered on the map.
* **`feature` vs. `row`:** Spatial entities on the canvas or inspector are *features*; lines in the attribute table grid or raw parse counts are *rows*.
* **`attribute` vs. `column`:** Named properties of features are *attributes*; vertical columns in the tabular grid are *columns*.
* **`add` vs. `import` vs. `remove`:** User action is *add*; parse counts are *import*; unmounting layers from the canvas is *remove* (never *delete*, because disk files are untouched).

### D. Mechanical Discipline & Punctuation Rigor (§6)
* **Em / En Dash Ban (`—`, `–`):** Dashes are prohibited in UI copy and user documentation. The rationale is twofold: (1) Readers associate em dashes with machine-generated text, eroding credibility in precision software; (2) In spatial data, negative values are frequent, making class ranges like `-12.5 – -3.2` completely unreadable. Terrn mandates `to` (`-12.5 to -3.2`).
* **Unicode Ellipsis (`…` U+2026):** Restricted strictly to active background progress (`Adding roads.geojson…`) or text truncation. ASCII `...` is banned.
* **Sentence Case Everywhere:** Eliminates title-casing across buttons, titles, tabs, labels, options, and documentation headings.

### E. Error Decoupling Architecture
`VOICE.md` (§5.3) dictates that errors must follow: `What happened` + `why` (if known) + `what to do`. In code, this is operationalized via [`IngestError`](file:///tern/tern_poc/src/shared/parsers/ingest-error.ts#L10). Generic exceptions (JSON syntax errors, WebGL crashes) are intercepted and translated into helpful user guidance in [`src/main.ts`](file:///tern/tern_poc/src/main.ts#L209-L210), preventing raw technical stack traces from polluting toasts.

---

## 2. Gaps, Nuances & Critical Observations

### A. Accessibility of `title === aria-label` (§5.1)
* **The Rule:** `title`, when present, must match `aria-label` exactly.
* **Critique:** While matching prevents conflicting accessible names, modern accessibility standards (WCAG / W3C) generally advise against native HTML `title` attributes on interactive elements. Native tooltips cannot be styled or resized, exhibit arbitrary OS hover delays, fail on touch devices, and can cause certain screen readers to announce the label redundantly.
* **Recommendation:** Migrate from native `title` to custom accessible tooltips (`role="tooltip"`) or omit `title` when `aria-label` is already present.

### B. Screen Reader Annunciation of Punctuation
* **Middle Dot (`·` §6.2):** Joining peer facts on a single line (`Roads 2024 · 1,204 features`) often leads screen readers (VoiceOver, NVDA) to vocalize *"Roads 2024 middle dot 1,204 features"*.
* **Recommendation:** In UI templates, wrap middle dots in `<span aria-hidden="true"> · </span>` or separate items with CSS borders.

### C. Internationalization (i18n) Readiness
* **Pluralization:** [`formatCount()`](file:///tern/tern_poc/src/utils/format.ts#L17-L19) relies on English binary pluralization (`n === 1 ? singular : plural`). Expanding to Slavic, Arabic, or Asian languages will break under this helper. Adopting `Intl.PluralRules` is advised for future localization.
* **Sentence Case Constraints:** Casing rules assume English conventions; languages such as German capitalize all nouns, requiring locale-aware capitalization handling.

### D. Static vs. Runtime Dynamic Copy
* [`src/core/styles/ui-text.spec.ts`](file:///tern/tern_poc/src/core/styles/ui-text.spec.ts) inspects string literals and template placeholders. Dynamic runtime messages (e.g. ternary expressions, dynamic DuckDB query rejections) bypass static extraction. Defensive runtime assertions in development mode can help catch dynamic drift.

---

# Part 2: Audit of Pull Requests 31 through 35

Below is the exhaustive catalog of content refactors executed across PRs 31 to 35, detailing the original text, the replacement, the UI context, user impact, and specific `VOICE.md` compliance.

---

## PR 31: Styling Panel Text Refactoring
> **Commit:** `22c00d2` · **Branch:** `fix/voice-styling-panel`  
> **Key Files:** [`styling-panel-element.ts`](file:///tern/tern_poc/src/features/styling/components/styling-panel-element.ts), [`point-style-element.ts`](file:///tern/tern_poc/src/features/styling/components/point-style-element.ts), [`line-style-element.ts`](file:///tern/tern_poc/src/features/styling/components/line-style-element.ts), [`polygon-style-element.ts`](file:///tern/tern_poc/src/features/styling/components/polygon-style-element.ts), [`label-style-element.ts`](file:///tern/tern_poc/src/features/styling/components/label-style-element.ts), [`basemap-sandwich-element.ts`](file:///tern/tern_poc/src/features/styling/components/basemap-sandwich-element.ts).

### 1. Panel Header & Idle State
* **Before:**
  * Header: `Layer Styling`
  * Empty State: `Click the <strong>Paintbrush</strong> icon next to a layer in the Layers list to begin.`
* **After:**
  * Header: `Style`
  * Empty State: `To style a layer, click the paintbrush on its card.`
* **Context:** Header of the right inspector pane when configuring layer styles, and the guidance displayed when no layer is selected.
* **Review:**
  * *User understanding:* Replaces clunky instructions with a direct, task-oriented sentence (*"To style a layer..."*). Directs the user to the card itself rather than the abstract "Layers list".
  * *Voice compliance:* Enforces §5.1 (Tabs/titles: noun, 1–2 words), §5.3 (Empty state: clear guidance), and §6.1 (Sentence case).

### 2. Operational Mode Dropdown
* **Before:**
  * Title: `Operational Mode`
  * Options: `Cluster Mode`, `3D Bubble Mode`, `3D Extrusion Mode`
* **After:**
  * Title: `Mode`
  * Options: `Clusters`, `3D columns`, `3D extrusion`, `Graduated`, `Categorized`, `Heatmap`
* **Context:** Selects the rendering mode for points, lines, or polygons.
* **Review:**
  * *User understanding:* Removes the redundant suffix *"Mode"* from each dropdown item. Renaming *"3D Bubble"* to *"3D columns"* accurately reflects the extruded cylinder geometry rendered by WebGL.
  * *Voice compliance:* Enforces §4.4 (No inflated words), §5.1 (Select options: sentence-case nouns), and §7 (Glossary: `clusters`).

### 3. Attribute Dropdown Harmonization (The "One Input, Six Names" Fix)
* **Before:**
  * Section titles: `Categorized Points/Lines/Polygons`, `Graduated Step Buckets`, `Graduated Line Thickness`, `Continuous Value Ramp Gradient`.
  * Field labels: `Classification Key`, `Unique Attribute Key`, `Numeric Attribute Key`, `Numeric Column Field`.
  * Select placeholders: `-- Choose Field --`, `-- Select Text Column --`.
* **After:**
  * Section titles: `Categorized`, `Graduated`, `Graduated width`, `Gradient`.
  * Field labels: `Attribute`.
  * Select placeholders: `Choose attribute`.
* **Context:** Selecting a feature property to drive data-driven visualizations across all geometry types.
* **Review:**
  * *User understanding:* Solves severe cognitive inconsistency. Previously, users encountered six different names for the same action depending on geometry type. "Attribute" provides an intuitive, GIS-standard mental model.
  * *Voice compliance:* Directly resolves §7 Glossary (*"A named property of a feature: attribute; don't use: field, column, key, property"*), §5.1 (Field labels ≤ 3 words), and §4.4 (No inflated words).

### 4. Classification Method & Class Breakers
* **Before:**
  * Method label: `Classification Mode Matrix` or `Classification Mode`.
  * Method options: `Quantiles`, `Equal Interval`, `Standard Deviation`.
  * Breakers label: `Range Size Breakers`, `Interval Classes`, or `Steps count`.
* **After:**
  * Method label: `Method`.
  * Method options: `Quantile`, `Equal interval`, `Standard deviation`.
  * Breakers label: `Classes`.
* **Context:** Statistical binning configuration for graduated choropleths.
* **Review:**
  * *User understanding:* Replaces academic jargon (*"Mode Matrix"*, *"Range Size Breakers"*) with terms universal to desktop GIS analysts (*"Method"*, *"Classes"*).
  * *Voice compliance:* Enforces §7 Glossary (`method`, `classes`), §2 (Peer trait), and §6.1 (Sentence case).

### 5. Line Stroke, Caps, Joins & Dash Array
* **Before:**
  * Labels: `Stroke Properties`, `Line Thickness (px)`, `Parallel Track Offset (px)`.
  * Dash array: Label `Dash Array Editor (e.g. 5,5)`, placeholder `Solid (leave blank)`.
  * Caps & joins: `Cap & Join Parameters`, options `Round Cap`, `Butt Cap`, `Miter Join`, etc.
* **After:**
  * Labels: `Stroke`, `Width (px)`, `Offset (px)`.
  * Dash array: Label `Dash pattern (blank for solid)`, placeholder `e.g. 5,5`.
  * Caps & joins: `Caps and joins`, options `Round cap`, `Butt cap`, `Miter join`, etc.
* **Context:** Stroke geometry and line styling parameters.
* **Review:**
  * *User understanding:* Moving the instructional hint *"blank for solid"* into the label allows the placeholder to show syntax examples (`e.g. 5,5`), preserving guidance after the user begins typing.
  * *Voice compliance:* Enforces §5.1 (Placeholders: hint or example, never the only label), §4.4, and §6.1.

### 6. Labels & Collision Avoidance
* **Before:**
  * Section title: `Layer Typography & Labels`.
  * Switch: `Render Typography Labels` / `Enable text rendering on canvas`.
  * Overlap label: `Forced Spatial Collision Avoidance`.
  * Alignment: `Top Centered`, `Bottom Centered`, `Left Centered`, `Right Centered`.
* **After:**
  * Section title: `Labels`.
  * Switch: `Show labels`.
  * Overlap label: `Avoid label overlap`.
  * Alignment: `Top`, `Bottom`, `Left`, `Right`.
* **Context:** Map text labeling and label placement controls.
* **Review:**
  * *User understanding:* Replaces verbose engineering labels with concise functional descriptions. *"Avoid label overlap"* communicates immediate utility over technical implementation.
  * *Voice compliance:* Directly matches §2 (Direct trait: *"Avoid label overlap"* vs *"Forced Spatial Collision Avoidance"*), §4.4, and §5.1.

### 7. Basemap Sandwich / Draw Order
* **Before:**
  * Title: `Basemap Sandwich Placement`.
  * Options: `Below Labels (Standard Sandwich)`, `Above All Map Layers (On Top)`.
  * Description: `Places this layer underneath Carto basemap street and boundary labels for professional map composition.`
* **After:**
  * Title: `Draw layer`.
  * Options: `Below labels`, `Above labels`.
  * Description: `Places the layer under basemap labels so text stays readable.`
* **Context:** Z-index ordering relative to the basemap's vector label hierarchy.
* **Review:**
  * *User understanding:* "Basemap sandwich" was an internal development metaphor that confused users. "Draw layer: Below labels / Above labels" describes the physical outcome clearly.
  * *Voice compliance:* Strictly satisfies §7 Glossary (banned: `sandwich`, use: `below labels / above labels`), §6.6 (Banned words), and §4.1 (User's model, not engine's).

---

## PR 32: Ingest Messages & Error Decoupling
> **Commit:** `7715af1` · **Branch:** `fix/voice-ingest-messages`  
> **Key Files:** [`main.ts`](file:///tern/tern_poc/src/main.ts), [`ingest-error.ts`](file:///tern/tern_poc/src/shared/parsers/ingest-error.ts), [`src/shared/parsers/index.ts`](file:///tern/tern_poc/src/shared/parsers/index.ts), [`geo-worker-client.ts`](file:///tern/tern_poc/src/core/workers/geo-worker-client.ts), [`duckdb.ts`](file:///tern/tern_poc/src/services/duckdb.ts).

### 1. Ingest Error Toast Architecture
* **Before:**
  ```ts
  showToast(err.message ?? `Failed to load ${name}`, 'error');
  ```
  *(Exposed unhandled exceptions such as `Unexpected token < in JSON at position 0` directly in toasts).*
* **After:**
  ```ts
  const reason = err instanceof IngestError ? err.message : "Check that it's a valid, undamaged file and try again.";
  showToast(`Couldn't add ${name}. ${reason}`, 'error');
  ```
* **Context:** Global ingest catch block in [`src/main.ts`](file:///tern/tern_poc/src/main.ts).
* **Review:**
  * *User understanding:* Always names the file, provides a clear reason, and offers a way forward. If an unhandled parser exception occurs, it displays a polite, actionable recovery prompt rather than an alarming stack trace.
  * *Voice compliance:* Enforces §4.1 (User's model), §4.2 (Name the object), §4.5 (Way forward in every error), and §5.3 (Toast structure).

### 2. Parser Rejection Messages ([`src/shared/parsers/index.ts`](file:///tern/tern_poc/src/shared/parsers/index.ts))
* **KML Empty:**
  * *Before:* `throw new Error("No features found in KML file. Make sure it is valid XML/KML.");`
  * *After:* `throw new IngestError("It contains no features. Check that it's valid KML.");`
* **GPX Empty:**
  * *Before:* `throw new Error("No tracks or waypoints found in GPX file.");`
  * *After:* `throw new IngestError("It contains no tracks or waypoints.");`
* **CSV Header Failure:**
  * *Before:* `throw new Error("CSV file is empty or lacks headers.");`
  * *After:* `throw new IngestError("It's empty or has no header row.");`
* **CSV Missing Lat/Lon:**
  * *Before:* `throw new Error("Could not identify Latitude or Longitude columns in CSV. Ensure your headers contain 'lat' and 'lon' keys.");`
  * *After:* `throw new IngestError("It has no latitude and longitude columns. Name them lat and lon, or latitude and longitude.");`
* **CSV No Valid Coordinates:**
  * *Before:* `throw new Error("Could not parse any valid coordinate rows from CSV.");`
  * *After:* `throw new IngestError("None of its rows have valid coordinates.");`
* **Unsupported Format:**
  * *Before:* `throw new Error(`Unsupported format: .${ext}`);`
  * *After:* `throw new IngestError("Terrn can't read this file type. Add GeoJSON, KML, GPX, CSV, a zipped Shapefile, or GeoTIFF.");`
* **Review:**
  * *User understanding:* Because the toast template provides the prefix `"Couldn't add roads.kml."`, each parser message starts with `"It"`, creating seamless grammatical flow (*"Couldn't add roads.kml. It contains no features. Check that it's valid KML."*). The unsupported format error explicitly lists every valid format.
  * *Voice compliance:* Implements §2 (Calm, Direct, Precise), §4.5 (Way forward), §6.3 (Impersonal pronouns), and §7 (Glossary: `add`, not `load` or `upload`).

### 3. Partial-Import Notification
* **Before:**
  `${features.length} of ${total} rows imported — ${skipped} rows malformed and skipped`
* **After:**
  `${features.length.toLocaleString('en-US')} of ${formatCount(total, 'row')} imported. ${formatCount(skipped, 'malformed row')} skipped.`  
  *(Outputs: `1,204 of 1,210 rows imported. 6 malformed rows skipped.` or `... 1 malformed row skipped.`)*
* **Context:** Sticky toast notifying users of skipped malformed rows during CSV parsing.
* **Review:**
  * *User understanding:* Clear outcome summary. Proper pluralization (`1 malformed row` vs `6 malformed rows`) eliminates mechanical-sounding copy.
  * *Voice compliance:* Enforces §4.3 (Exact counts and correctly pluralized via [`formatCount()`](file:///tern/tern_poc/src/utils/format.ts#L17-L19)), §6.2 (Dash ban; period used for secondary fact), and §6.4 (Thousands separators).

### 4. Background Processing Timeout
* **Before:** `showToast('Background processing timed out, using standard import', 'info');`
* **After:** `showToast('This file is taking longer than usual. The page may pause while Terrn finishes it.', 'info');`
* **Context:** Notification when Web Worker bounds calculation exceeds 15 seconds and falls back to main-thread execution.
* **Review:**
  * *User understanding:* Translates backend concurrency details into what the user will actually experience (*"The page may pause..."*).
  * *Voice compliance:* Satisfies §4.1 (Speak in user's model, not engine's).

### 5. DuckDB Table Sort Failure
* **Before:** `showToast('Sort unavailable — schema query failed', 'error');`
* **After:** `showToast("Can't sort the attribute table. Remove the layer and add it again.", 'error');`
* **Context:** Failure during DuckDB PRAGMA table query execution when sorting attribute columns.
* **Review:**
  * *User understanding:* The user is informed of the specific failure and provided a concrete recovery path.
  * *Voice compliance:* Enforces §4.1, §4.5, and §6.2 (No em dashes).

---

## PR 33: Tooltips, Aria-Labels & Chrome Text
> **Commit:** `626a097` · **Branch:** `fix/voice-tooltips`  
> **Key Files:** [`index.html`](file:///tern/tern_poc/index.html), [`layer-list-element.ts`](file:///tern/tern_poc/src/features/layers/components/layer-list-element.ts), [`legends-element.ts`](file:///tern/tern_poc/src/features/layers/components/legends-element.ts), [`feature-info-element.ts`](file:///tern/tern_poc/src/features/inspector/components/feature-info-element.ts), [`right-panel-controller.ts`](file:///tern/tern_poc/src/features/panel/right-panel-controller.ts).

### 1. Title / `aria-label` Parity & Sentence Casing
* **Before:**
  * `title="Zoom In" aria-label="Zoom in"` / `title="Zoom Out" aria-label="Zoom out"`
  * `title="Reset View" aria-label="Reset map view"`
  * `title="Feature Info" aria-label="Feature info"`
  * `title="Layer Style" aria-label="Layer style"`
  * `title="Expand panel" aria-label="Expand info panel"`
* **After:**
  * `title="Zoom in" aria-label="Zoom in"` / `title="Zoom out" aria-label="Zoom out"`
  * `title="Reset map view" aria-label="Reset map view"`
  * `title="Feature info" aria-label="Feature info"`
  * `title="Layer style" aria-label="Layer style"`
  * `title="Expand panel" aria-label="Expand panel"` / `title="Collapse panel" aria-label="Collapse panel"`
* **Context:** Map viewport controls, right rail buttons, and panel collapse triggers.
* **Review:**
  * *User understanding:* Eliminates discrepancies between mouse hover tooltips and screen-reader announcements. Sentence casing feels calm and modern.
  * *Voice compliance:* Directly enforces §5.1 (`title`, when present, must equal `aria-label`) and §6.1 (Sentence case everywhere).

### 2. State Toggles vs. Verbose Actions
* **Before:**
  * Opacity: `title="Layer opacity" aria-label="Toggle opacity control"`
  * Layer list visibility: `title="Toggle visibility" aria-label="Toggle layer visibility"`
  * Legend visibility: `title="Toggle visibility" aria-label="Toggle ${layer.name} visibility"`
* **After:**
  * Opacity: `title="Layer opacity" aria-label="Layer opacity"`
  * Layer list visibility: `title="Layer visibility" aria-label="Layer visibility"`
  * Legend visibility: `title="${layer.name} visibility" aria-label="${layer.name} visibility"`
* **Context:** Layer eye icons and opacity sliders.
* **Review:**
  * *User understanding:* Eliminates the redundant verb "Toggle". State is already conveyed by `aria-pressed="true|false"` and icon artwork (`eye` vs `eye-slash`).
  * *Voice compliance:* Enforces §5.1 (*"Toggle / switch label: Name the thing being switched, not the verb. State is shown by the control"*).

### 3. Toolbar Verbs & Modal Header
* **Before:**
  * Ingest button: `title="Upload spatial data">Upload</button>`
  * Remove button: `<button ...>Clear</button>`
  * Modal header: `Ingest Spatial Data`
* **After:**
  * Ingest button: `title="Add files as layers">Add data</button>`
  * Remove button: `<button ...>Remove</button>`
  * Modal header: `Add data`
* **Context:** Left panel layer toolbar and data ingestion dialog.
* **Review:**
  * *User understanding:* "Upload" falsely suggested that data leaves the client machine for a remote server. "Add data" accurately reflects adding local datasets to the map canvas.
  * *Voice compliance:* Enforces §7 Glossary (`add` is the ingestion verb; `remove`, not `clear` or `delete`).

### 4. Inspector Prompt & Mobile Overlay
* **Feature info idle state:**
  * *Before:* `Enable <strong>Feature Info</strong> mode then click any vector feature on the map.`
  * *After:* `Turn on <strong>Feature info</strong>, then click a feature on the map.`
* **Mobile block overlay:**
  * *Before:* `Please switch to a larger screen to use the app.`
  * *After:* `Open Terrn on a larger screen to use it.`
* **Context:** Feature inspector pane and the mobile screen overlay (<768px).
* **Review:**
  * *User understanding:* Clear, direct instruction. Strips unnecessary politeness filler (*"Please"*) in favor of an actionable directive (*"Open Terrn on a larger screen..."*).
  * *Voice compliance:* Enforces §6.6 (banned word: `please`), §6.1 (feature names are not proper nouns), and §2 (Direct trait).

---

## PR 34: Table, Legend, Dialog & Settings Text
> **Commit:** `53309d1` · **Branch:** `fix/voice-panels-dialogs`  
> **Key Files:** [`attribute-table-element.ts`](file:///tern/tern_poc/src/features/tabular/components/attribute-table-element.ts), [`confirm-copy.ts`](file:///tern/tern_poc/src/utils/confirm-copy.ts), [`legends-element.ts`](file:///tern/tern_poc/src/features/layers/components/legends-element.ts), [`layer-list-element.ts`](file:///tern/tern_poc/src/features/layers/components/layer-list-element.ts), [`settings-panel-element.ts`](file:///tern/tern_poc/src/features/settings/components/settings-panel-element.ts).

### 1. Attribute Table Search, Buttons & Pagination
* **Search input:**
  * *Before:* `placeholder="Search all columns..."`
  * *After:* `placeholder="Search values"`
* **Selection & Column Buttons:**
  * *Before:* `Clear Selection`, `title="Toggle column visibility"`
  * *After:* `Clear selection`, `title="Show or hide columns"`
* **Minimize / Maximize Button:**
  * *Before:* `title="Minimize/Maximize attribute table" aria-label="Minimize/Maximize table"`
  * *After:* Dynamic state: `title="Minimize table"` or `title="Maximize table"`
* **Tab Close Button:**
  * *Before:* `title="Close tab"`
  * *After:* `title="Close ${layer.name}" aria-label="Close ${layer.name}"`
* **Pagination Readout:**
  * *Before:* `Showing ${start} to ${end} of ${filteredCount} features`
  * *After:* `Showing ${start.toLocaleString('en-US')} to ${end.toLocaleString('en-US')} of ${formatCount(filteredCount, 'feature')}`
* **Context:** Bottom drawer attribute table controls.
* **Review:**
  * *User understanding:* "Search values" clarifies that search queries row data. Tab close buttons explicitly name the layer being dismissed. Slashes (*"Minimize/Maximize"*) are replaced with the explicit active state.
  * *Voice compliance:* Enforces §5.1 (Placeholders: no trailing ellipsis), §6.2 (No `/` as "or"; no ASCII `...`), §4.2 (Name the object), and §4.3 (Pluralization and number formatting).

### 2. Destructive Layer Removal Confirmation ([`src/utils/confirm-copy.ts`](file:///tern/tern_poc/src/utils/confirm-copy.ts))
* **Single layer ($n=1$):**
  * *Before:* Title `Remove layer`, Body `Remove this layer? This can't be undone.`, Button `Remove`.
  * *After:* Title `Remove "${layerNames[0]}"?`, Body `It will be removed from the map. This can't be undone.`, Button `Remove layer`.
* **Multiple layers ($n>1$):**
  * *Before:* Title `Clear all layers`, Body `Remove all 2 layers? This can't be undone.`, Button `Remove all`.
  * *After:* Title `Remove all 2 layers?`, Body `They'll be removed from the map. This can't be undone.`, Button `Remove 2 layers`.
* **Context:** Confirmation dialog triggered when clearing layers from the left toolbar.
* **Review:**
  * *User understanding:* Exceptional safety UX. The confirm button repeats the explicit action and count (*"Remove 2 layers"*), preventing accidental confirmation when scanning buttons. Naming the individual layer prevents removing the wrong dataset.
  * *Voice compliance:* Implements §4.2 (Name the object), §5.3 (Destructive confirmation: title is a question naming the object, button repeats verb and count), and §6.2 (Question marks allowed only in confirmation titles).

### 3. Legend Class Ranges
* **Before:** `label: `${fmt(lower)} – ${fmt(upper)}`` *(Used en dash `–`)*
* **After:** `label: `${fmt(lower)} to ${fmt(upper)}`` *(Example: `-12.5 to -3.2`)*
* **Context:** Numeric class break labels in map legends.
* **Review:**
  * *User understanding:* Resolves visual ambiguity. In GIS, negative values (e.g. elevation, temperature) produced confusing strings like `-12.5 – -3.2`. Using `to` ensures clarity.
  * *Voice compliance:* Directly enforces §6.4 (*"Ranges use `to`, never a dash... A dash beside a negative value is unreadable"*).

### 4. Settings & Privacy Panel Copy
* **Crash Monitoring:**
  * *Before:* `Always on — helps us fix errors you hit (crash diagnostics only).`
  * *After:* `Always on. Crash reports help us fix errors you run into.`
* **Eyebrow & Tier Text:**
  * *Before:* `PRIVACY & ANALYTICS`, `opt out in Settings`
  * *After:* `Privacy and analytics`, `opt out in settings`
* **Context:** Telemetry settings panel and privacy disclosure.
* **Review:**
  * *User understanding:* Replaces machine-like em dashes with clean sentences. Removes uppercase styling on eyebrows.
  * *Voice compliance:* Enforces §6.2 (No em dashes), §6.1 (Sentence case; feature names are not proper nouns), and §6.3 (`we` is permitted because it represents a company privacy commitment).

---

## PR 35: User Documentation Rewrite
> **Commit:** `1a4c901` · **Branch:** `fix/voice-user-docs`  
> **Key Files:** [`tutorial-getting-started.md`](file:///tern/tern_poc/docs/tutorial-getting-started.md), [`howto-ingest-data.md`](file:///tern/tern_poc/docs/howto-ingest-data.md), [`howto-query-data.md`](file:///tern/tern_poc/docs/howto-query-data.md), [`howto-style-layers.md`](file:///tern/tern_poc/docs/howto-style-layers.md), [`howto-extend-parsers.md`](file:///tern/tern_poc/docs/howto-extend-parsers.md), [`reference-parsers.md`](file:///tern/tern_poc/docs/reference-parsers.md).

### 1. Document Titles & Headings
* **Before:** Title-cased headings across all markdown guides:
  * `# Tutorial: Getting Started with Terrn`
  * `## Step 2: Load a geospatial file`
  * `## Step 4: Style the layer by a data field`
  * `## Classification Mode Matrix`
* **After:** Sentence-cased headings:
  * `# Tutorial: getting started with Terrn`
  * `## Step 2: Add a geospatial file`
  * `## Step 4: Style the layer by an attribute`
  * `## Method`
* **Context:** Structural headings across tutorials and how-to articles.
* **Review:**
  * *User understanding:* Consistent capitalization between in-app UI and documentation reduces cognitive friction.
  * *Voice compliance:* Enforces §6.1 (*"Sentence case everywhere: buttons, titles, tabs, labels, options, tooltips, doc headings"*).

### 2. UI Reference Exactness in Steps
* **Before:**
  * Tutorial steps directed users to click *"Upload spatial data"*, open *"Layer Styling"*, adjust *"Field"*, and set *"Breaks"*.
* **After:**
  * Tutorial steps explicitly instruct users to click **Add data**, open the **Style** tab, choose **Attribute**, and adjust **Classes**.
* **Context:** Step-by-step procedures in tutorials.
* **Review:**
  * *User understanding:* Eliminates documentation drift. Users are never left searching for non-existent buttons or renamed dropdowns.
  * *Voice compliance:* Satisfies §5.4 (*"All docs: Refer to UI text exactly as rendered, in bold: `Click **Add data**.` If the UI label changes, the doc changes in the same commit"*).

### 3. Em-Dash Elimination Across Docs
* **Before:**
  * `...run an analytical SQL query — all in the browser, no server required.`
  * `- [Reference: Parsers](reference-parsers.md) — all supported formats`
* **After:**
  * `...run an analytical SQL query.`
  * `- [Reference: Parsers](reference-parsers.md): every supported format`
* **Context:** Document introductions, lists, and reference links.
* **Review:**
  * *User understanding:* Sentences are concise and factual. Marketing hype (*"all in the browser, no server required"*) was removed ahead of planned backend features.
  * *Voice compliance:* Enforces §6.2 (Em-dash ban) and §10 (Decisions Log).

### 4. Technical Accuracy Corrections
* **Port Correction:** Updated local dev server URLs from Vite’s default `5173` to Terrn’s configured `3000`.
* **Pagination Sync:** Corrected documented pagination text from `Showing 1-50 of 12,450 rows` to match live UI output: `Showing 1 to 50 of 12,450 features`.
* **Parser Architecture:** Updated [`docs/howto-extend-parsers.md`](file:///tern/tern_poc/docs/howto-extend-parsers.md) and [`docs/reference-parsers.md`](file:///tern/tern_poc/docs/reference-parsers.md) to document [`IngestError`](file:///tern/tern_poc/src/shared/parsers/ingest-error.ts#L10) and user-facing error message contracts.
* **Review:**
  * *User understanding:* Eliminates setup friction and maintains trust with developers.
  * *Voice compliance:* Enforces §5.4 (Reference documentation must be exhaustive and accurate).

---

# Baseline Reduction Summary

| PR # | Surface Area | Violations Cleared | Impact on Baseline |
|---|---|---|---|
| **31** | Style Panel | **99** | Baseline reduced from 141 to 42 |
| **32** | Ingest & Errors | **3** | Baseline reduced from 42 to 39 |
| **33** | Tooltips & Chrome | **24** | Baseline reduced from 39 to 15 |
| **34** | Panels, Dialogs & Table | **10** | Baseline reduced from 15 to 5 |
| **35** | Documentation | *(Docs)* | Aligned all tutorials, guides, and README |
| *R0* | *Copilot Removal* | **5** | *Final 5 legacy AI violations cleared (Baseline at 0)* |

### Conclusion
The execution across PRs 31–35 represents a model migration of product language. By tying normative voice standards ([`VOICE.md`](file:///tern/tern_poc/VOICE.md)) directly to automated unit test enforcement ([`ui-text.spec.ts`](file:///tern/tern_poc/src/core/styles/ui-text.spec.ts)), Terrn has successfully transitioned from inconsistent UI copy to an interface that speaks like the precision cartographic instrument it is designed to be.
