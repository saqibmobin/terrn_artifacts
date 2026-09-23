# Kepler.gl Architecture Review — Independent Verification and Terrn Adoption Analysis

**Date:** 2026-09-18
**Kepler.gl:** `/tern/kepler.gl` @ v3.3.0-alpha.12
**Terrn:** `/tern/tern_poc` @ `ca31c7a`, branch `feat/r1-benchmark-harness`
**Reviewed against:** `kepler_data_ingestion_and_rendering_architecture.md` (baseline),
`kepler_adoption_recommendations_for_terrn.md` (initial recommendation),
`docs/plan/sprint-11/Terrn_Performance_Implementation_Plan.md` (the plan),
`docs/plan/sprint-11/Terrn_Performance_Solutions_Review.md` (the review).

Every claim below was checked against source at the paths given, or executed
against Terrn's installed dependencies. Where I could not confirm a claim in
the prior documents, I say so and give what the code actually does.

---

## 0. Summary

The initial recommendation document is **directionally useful but structurally
inverted**. It proposes copying six small utility functions out of kepler.gl,
four of which are real and two of which are not in the codebase as described.
Meanwhile the one pattern kepler.gl genuinely has that Terrn genuinely lacks —
**a single schema-discovery pass that every consumer reads from** — is not
mentioned in either prior document.

Three findings change work already scheduled:

1. **Terrn has three independent, mutually inconsistent type-inference paths.**
   Demonstrated below: a field can be `Float64` in DuckDB while the styling UI
   believes it is a string, and a field can exist in DuckDB while being
   invisible to every styling dropdown. This is a robustness defect of the same
   class as DEF-14, and it is *not* fixed by R2 as currently scoped.
2. **DEF-14's coercion table in the review is subtly wrong.** The silently
   coerced value is `NaN`, not `null` (`nullCount: 0`). This matters: DuckDB
   treats `NaN` as a non-null Float64, so `quantile_cont` — which R6 adopts for
   classification — is poisoned by it, whereas NULLs would be ignored.
3. **A new, uncataloged ingest defect**: `parseCSV` silently selects the wrong
   latitude/longitude column on common header layouts, plotting data at wrong
   locations with no error. Verified with two reproductions.

One recommendation should be **dropped** (§1.3) and one **kept** (§1.2). See the
correction notice below — an earlier revision of this review wrongly told you to
drop both.

The **baseline architecture document is accurate** — ~20 structural claims
checked, three location slips, no substantive errors (§6). Its most useful
contribution to Terrn is something it does not itself draw attention to: kepler
runs **two** parallel ingestion processors whose schema logic has already
drifted to the point that one of them extracts no fields at all. That is exactly
what Terrn's worker import boundary will produce in R10 unless an equivalence
spec is added (§6.4).

> ### ⚠️ Correction — 2026-09-18, second pass
>
> **§1.2 of this review was wrong.** I reported that kepler.gl "does not
> contain" the WKB BLOB-detection code the recommendation document cites. It
> does: `src/processors/src/data-processor.ts:527-539`. The recommendation
> document's **line numbers were right and its file path was wrong** — it names
> `src/duckdb/src/table/duckdb-table.ts`. Git history confirms the block was
> introduced in `src/processors/` (commit `221b243c`) and never lived in
> `src/duckdb/`.
>
> **Cause:** I scoped the confirming grep to `src/duckdb/`, the directory the
> citation named, and read a directory-scoped miss as a repository-wide absence.
> A repo-wide grep finds it immediately.
>
> **Effect:** the recommendation's §3.2 is **valid and should be kept**, not
> dropped. §1.2 below is rewritten. My accompanying claim that kepler "uses the
> type system, not byte sniffing" was also a false dichotomy — it uses both, in
> one cascade.
>
> §1.3 (the scale-utils → MapLibre-expression item) still stands as a drop, but
> its reasoning was overstated and is corrected below. Confidence levels for
> every remaining claim are restated in the Appendix.

---

## 1. Verification of the initial recommendation document

### 1.1 Claims that hold

| Claim | Status | Actual location |
|---|---|---|
| `CSV_NULLS` / `cleanUpFalsyCsvValue` | ✅ Confirmed | `src/processors/src/data-processor.ts:42` (regex), `:213` (function). A second copy at `src/duckdb/src/processors/data-processor.ts:18, :163`. The doc cites `#L42` for the function; that line is the regex. |
| `detectDelimiter`, `SUPPORTED_DELIMITERS` | ✅ Confirmed | `src/processors/src/data-processor.ts:44, :57` |
| `castBigIntColumnsToFloat64` | ✅ Confirmed | `src/processors/src/data-processor.ts:585-589` — but see §1.4, kepler's *better* pattern is elsewhere |
| `getGeoArrowMetadataFromSchema` | ✅ Confirmed | `src/processors/src/data-processor.ts:468` |
| `compactArrowTable` | ✅ Confirmed | `src/utils/src/arrow-data-container.ts:144` |
| `findPointFieldPairs` | ✅ Confirmed | `src/table/src/kepler-table.ts:888` |

### 1.2 Claim that holds, at a different path — §3.2, "DuckDB WKB BLOB detection"

*Rewritten after the correction above. The original text of this section
asserted the code was absent; it is not.*

The recommendation document cites
`src/duckdb/src/table/duckdb-table.ts:527-540`. That file is 365 lines, so the
path is wrong — but the **line numbers are almost exactly right for the file the
code is actually in**: `src/processors/src/data-processor.ts:527-539`, inside
`arrowSchemaToFields`. The quoted snippet matches the source closely. Treat this
as a transcription slip in the citation, not an invented recommendation.

What the code actually does is richer than either document conveys. It is a
**priority cascade** over DuckDB's declared column types, not a standalone
sniffer (`data-processor.ts:505-560`):

| Order | Condition | Resulting type |
|---|---|---|
| 1 | suggestion `JSON` (from `st_asgeojson`) | `geojson` |
| 2 | suggestion `GEOMETRY`, or GeoArrow metadata on the field | `geoarrow` |
| 3 | `geo` schema metadata names the column (GeoParquet) | `geoarrow` + metadata set |
| 4 | suggestion `BLOB` → **try `parseSync(data, WKBLoader)` on row 0** | `geoarrow` + WKB metadata |
| 5 | suggestion `VARCHAR` + analyzer says GEOMETRY | WKB/WKT-as-varchar |
| 6 | suggestion `VARCHAR` + h3 detected | `h3` |
| 7 | otherwise | kepler's own field inference |

The `fieldTypeSuggestions` argument **is** the DuckDB declared-type map:
`duckdb-table.ts:270-271` passes `tableDuckDBTypes`, which comes from
`getDuckDBColumnTypes` → `PRAGMA table_info` (`duckdb-table-utils.ts:30-64`).

So the declared type system and the byte sniffing are **the same mechanism**:
declared types are consulted first, and WKB sniffing is a documented last-resort
branch for exactly one case, stated in its own comment — *"When arrow wkb column
saved to DuckDB as BLOB without any metadata, then queried back."*

**For Terrn**, both halves are worth taking, with one caveat each:

- **Take the cascade shape.** Ask the engine what it declared before guessing.
  `castDuckDBTypesForKepler` (`duckdb-table-utils.ts:129-155`) shows the payoff:
  `BIGINT`/`UBIGINT`/`HUGEINT`/`DECIMAL` become `CAST(col AS DOUBLE)` **in SQL**
  (see §1.4), and `GEOMETRY` becomes `ST_AsWKB(col)`.
- **Caveat:** `ST_AsWKB` is a spatial-extension function and Terrn descoped
  spatial (ADR-008). The type-directed and BLOB-sniffing branches work without
  spatial; the `GEOMETRY` branch does not.
- The BLOB fallback is what makes R13's "add query result as layer" possible
  without declaring geometry columns by hand, which is the use the
  recommendation document proposed. That use is sound.

### 1.3 Claim that does not hold — §4.1, "scale utils → MapLibre expressions"

The document's action is: *"Incorporate this directly into Terrn's
`style-compiler.ts` (R8) to compile MapLibre `step` and `interpolate`
expressions from pre-computed breaks."*

**Kepler.gl compiles no style expressions from classification** — but my first
pass overstated this as "no MapLibre layer path at all," which is wrong. The
precise position:

- Kepler **does** have a live native map-layer seam I initially missed:
  `generateMapboxLayers` / `updateMapboxLayers`
  (`src/layers/src/mapbox-utils.ts`), wired into `MapContainer` and called on
  render (`map-container.tsx:536, :1242-1255`). It adds GeoJSON sources, calls
  `source.setData`, adds native style layers, and toggles visibility with
  `setLayoutProperty` (`mapbox-utils.ts:128`).
- But **no built-in layer uses it.** `Layer.overlayType` returns
  `OVERLAY_TYPE_CONST.deckgl` (`base-layer.ts:289-290`) and nothing in
  `src/layers/` overrides it. It is an extension point for custom layers,
  shipped wired-up and unexercised.
- And when it *does* run, `updateLayerConfig` (`mapbox-utils.ts:113-126`)
  handles a config change by `removeLayer` **then** `addLayer` — a wholesale
  rebuild. `setLayoutProperty` for visibility is the only property it sets
  individually. **There is no paint-property diffing anywhere in kepler**
  (`setPaintProperty` appears nowhere in the current tree; repo-wide grep).

So the recommendation's specific action has no basis in kepler code: kepler's
scale utilities produce **d3 scale closures** (`getScaleFunction`, `:162`) for
deck.gl accessors and are never converted to MapLibre expressions.

What `data-scale-utils.ts` actually produces is **d3 scale closures**
(`getScaleFunction`, `:162`) for deck.gl accessors — exactly the per-feature-JS
architecture R8 exists to remove. Two further problems with adopting it:

- `getQuantileDomain` (`:56`) does a **full main-thread sort of every value**
  with no sampling. R6 explicitly moves this to DuckDB `quantile_cont`. Adopting
  kepler's version is a regression against Terrn's own plan.
- The review already identified the right reference for expression compilation:
  **GeoLibre**, which does have one (`core/vector-color.ts:325, 340` — `match`
  for categorized, `step` for graduated), and which the review marks "**Take**".

**Recommendation: drop this item**, and keep GeoLibre as R8's reference — now
for a grounded reason rather than my original wrong one. GeoLibre diffs each
paint key and calls `setPaintProperty` only for what changed
(`layer-sync.ts:3609-3648`, per the review); kepler's native path rebuilds the
layer wholesale. GeoLibre is strictly the better model for R8's per-layer sync.

The one thing worth lifting from kepler's scale utils is a data *shape*, not
logic — see §2.6.

### 1.4 Claim that is real but mis-framed — §2.2, BigInt casting

`castBigIntColumnsToFloat64` exists, but it is kepler's *fallback* path for
Arrow tables arriving from files. For its **DuckDB** path kepler does the cast
**in SQL** (`CAST(col AS DOUBLE)`, `duckdb-table-utils.ts:147`), never in JS.

The recommendation tells Terrn to "adopt this in `duckdb.ts` when converting
DuckDB query results to Arrow". That would put a per-column
`new Float64Array(col.length)` allocation and a per-value `Number()` loop on the
**main thread** — precisely the class of work the whole Sprint 11 plan is
removing. Do the cast in the projection list instead; it costs nothing on the
JS side and runs inside the DuckDB worker.

### 1.5 Claim that is real but not currently reachable — §5.1, the 255-layer cap

The cap is real, and I confirmed it in Terrn's own installed deck.gl rather than
relying on kepler's comment:

```js
// node_modules/@deck.gl/core/dist/passes/pick-layers-pass.js:136-143
a = byLayer.size + 1;
if (a <= 255) { ... }
else { log.warn('Too many pickable layers, only picking the first 255')(); a = 0; }
```

Three corrections to the framing:

- The index encoded in the alpha channel is the **pickable layer index**, not the
  Arrow record-batch count. Batches matter in kepler only because
  `GeoArrowScatterplotLayer` emits one sublayer per batch.
- deck.gl **warns** (`log.warn`); it does not fail silently. Kepler's silence is
  a property of its own layer explosion, not of deck.gl.
- **Terrn cannot reach this today.** `createDeckLayers`
  (`renderer.ts:516`) passes plain `layer.data.features` arrays to every deck
  layer — never an `arrow.Table`. Terrn emits roughly 1–3 deck layers per Terrn
  layer, so the cap needs ~100+ simultaneous layers.

Keep it as a note against R10/R13 (if Arrow ever reaches deck.gl), not as
scheduled work.

### 1.6 Claim that adds nothing — §5.2, MapLibre v6 transform isolation

The plan already solves this, and better. R3 moves to `@deck.gl/maplibre`, which
the review verified supports MapLibre 4–6 through public APIs (review §9, §1
last row). Kepler's arrangement — React wrappers passing `viewState` down — is
not transferable to Terrn's imperative, interleaved sandwich, and is not needed
once R3 lands.

---

## 2. What is genuinely worth adopting

Ranked by value to Terrn, not by how easy it is to copy.

### 2.1 One schema-discovery pass — the pattern both prior documents miss

**This is the highest-value finding in this review.**

Kepler has exactly one place where a dataset's field types are decided, and one
type list that every consumer reads:

- infer once — `getSampleForTypeAnalyze` (`src/common-utils/src/data-type.ts:39`)
  → `getFieldsFromData` → `Field[]`
- **coerce the data to match** — `parseRowsByFields`
  (`src/processors/src/data-processor.ts:195`), so the declared type is true by
  construction, not by hope
- everything downstream (layer creation, scales, filters, the data table, the
  DuckDB cast list) reads that one `Field[]`

**Terrn has three separate inference passes that can and do disagree:**

| # | Where | Samples | Rule |
|---|---|---|---|
| 1 | `getLayerSchema` (`layer.store.ts:30`) | first **100 features** | JS `typeof`; conflict → `'string'` |
| 2 | `tableFromJSON` (`duckdb.ts:283`) | **first value** per column | apache-arrow inference, silent coercion (DEF-14) |
| 3 | `getLayerGeometryType` (`layer.store.ts:9`) | first **50 features** | geometry type vote |

Pass 1 feeds every styling dropdown — all four style panels read `layer.fields`
(`point-style-element.ts:154`, `line-`:135, `polygon-`:148, `label-`:100), and
the numeric modes gate on it (`fields.filter(f => f.type === 'number')`,
`point-style-element.ts:389, 438, 489, 526, 566`). Pass 2 is what the attribute
table, SQL and (from R6) classification actually read.

**Demonstrated divergence.** Executed against Terrn's installed `apache-arrow`
21.1.0, with `getLayerSchema` transcribed verbatim:

```
input: feature 0  -> { x: 1 }             (number)
       features 1..99 -> { x: 'abc' }     (string)
       features 100+  -> { x: 'abc', late_field: 'hello' }

getLayerSchema ->  [{"name":"x","type":"string"}]
DuckDB/Arrow  ->   x:Float64, __row_idx:Float64, late_field:Dictionary<Int32,Utf8>
x values rows 0..2 ->  [ 1, NaN, NaN ]
```

Three distinct user-visible failures in one file:

- **`x` is hidden from every numeric styling mode** (Bubble, Graduated, Gradient,
  Heatmap weight, Extrusion height) because pass 1 called it a string — while
  DuckDB holds it as `Float64` and SQL treats it as numeric.
- **`x`'s values are destroyed** — `'abc'` became `NaN` in the store.
- **`late_field` is invisible to the styling UI entirely.** It is in the
  attribute table and queryable in SQL, but it appears in no dropdown, because
  it first occurs at feature 100 and pass 1 stops at 100.

**Adopt:** collapse to one inference pass whose output is the single declared
schema, and coerce values to it. In Terrn's plan this belongs in **R2** (where
the explicit Arrow schema is already being built) and moves into
`ingest.worker.ts` in **R10**. Concretely, R2's explicit per-column schema scan
should *produce* `SchemaField[]` rather than sitting beside it, and
`getLayerSchema` should be deleted rather than left as a second opinion.

This is cheap to do inside R2 — the R2 scope already walks every value of every
property to decide `Float64`/`Bool`/`Utf8`. Emitting `SchemaField[]` from that
same walk is nearly free, and removes the sampling limit at the same time.

### 2.2 Token-boundary coordinate pairing — fixes a verified, uncataloged defect

Terrn's `parseCSV` (`shared/parsers/index.ts:69-101`) detects lat/lon by exact
alias match, then falls back to a substring scan:

```ts
if (latIndex === -1 || lonIndex === -1) {
  for (let i = 0; i < headers.length; i++) {
    if (headers[i].includes('lat')) latIndex = i;                    // no break
    if (headers[i].includes('lon') || headers[i].includes('lng')) lonIndex = i;
  }
}
```

Two bugs: the fallback **overwrites an index that was already correctly set by
the exact-match pass**, and `includes('lat')` matches any word containing "lat".
Last match wins, since neither branch breaks.

**Verified by executing the function's logic verbatim:**

| Headers | Chosen lat | Chosen lon |
|---|---|---|
| `lat, latency, longitude_deg` | **`latency`** ❌ | `longitude_deg` |
| `lat, plate_number, lon_deg` | **`plate_number`** ❌ | `lon_deg` |
| `pickup_lat, pickup_lon, dropoff_lat, dropoff_lon` | `dropoff_lat` (arbitrary) | `dropoff_lon` |
| `x, y, value` (projected metres) | `y` | `x` → all rows fail the ±180/±90 check |

In rows 1 and 2 Terrn plots points using a non-coordinate column as latitude.
Values outside ±90 are silently dropped into the "malformed rows skipped" toast;
values that happen to land in range are **drawn at the wrong place with no
warning**. That is the DEF-14 failure class — silently wrong data — in the
ingest path, and it is in no existing catalog.

**Adopt** kepler's `findPointFieldPairs` (`kepler-table.ts:888`) approach:

- match on a **token boundary**, `(^|[#_&@.\- ])lat([#_&@.\- ]|$)`, so `latency`
  and `translation` cannot match
- **pair** candidates by shared trimmed prefix (`foundMatchingFields`), so
  `pickup_*` and `dropoff_*` become two distinct pairs
- return **all** pairs rather than one

Terrn need not build multi-layer-from-one-CSV (kepler makes one `PointLayer` per
pair). The minimum correct behavior is: match on token boundaries, and when more
than one pair is found, pick deterministically and say which was used — or ask.

**Where:** R2 already owns CSV correctness (M5, `parseFloat` coercion). Add this
there; R10 then copies correct logic into the worker, which is exactly the
sequencing R2's rationale already states.

### 2.3 Null sanitization before inference

`cleanUpFalsyCsvValue` (`data-processor.ts:213`) turns `''`, `null`, `NULL`,
`NaN`, `/N` into real `null` **before** type analysis, so one empty cell cannot
force a numeric column to string.

Terrn's `parseCSV` does the opposite: `properties[h] = isNaN(num) ? val : num`
(`parsers/index.ts:118`), so `''` stays the empty string and `'NULL'` stays a
5-character string. Combined with §2.1, one blank cell in the first 100 rows
flips a numeric field to `'string'` and removes it from every numeric styling
mode.

**Adopt** — small, and it directly serves R2's stated goal. Note the review's
verified case `1, ""` → `0` (Float64): sanitizing to `null` first is what
prevents that, so this is a prerequisite for R2's fix, not an optional extra.

### 2.4 Delimiter sniffing

Terrn's `parseCsvRows` (`parsers/index.ts:17`) is a correct RFC-4180 state
machine but hard-codes `,` as the only separator (`else if (c === ',')`, `:33`).
A semicolon- or tab-delimited export — the European Excel default — parses as a
single column, fails lat/lon detection, and raises *"It has no latitude and
longitude columns"*, which misdescribes the problem.

`detectDelimiter` (`data-processor.ts:57`) tries `, \t ; |` on the first line and
takes the highest column count ≥ 2. Terrn's state machine would need the
separator parameterized — a one-line change — plus the sniffer.

**Adopt** in R2 or R10. Low cost, removes a whole class of "Terrn can't open my
file" reports. Worth a VOICE.md check on the error text either way: the current
message names the wrong cause.

### 2.5 Streaming ingest with progress

Kepler reads files in batches (`readFileInBatches`,
`file-handler.ts:285`) and wraps the iterator in `makeProgressIterator`
(`:173`), which reports `{rowCount, rowCountInBatch, percent}` from
`batch.bytesUsed / file.size`.

Terrn reads the whole file with a single `FileReader.readAsText` and one
`JSON.parse` (`main.ts:121-137`), with no progress signal of any kind. R1's
benchmark includes a **50 MB GeoJSON**; R10's stated outcome is "dropping a 50 MB
GeoJSON doesn't freeze the UI". Moving that work into a worker removes the
freeze but still leaves the user with an unexplained multi-second wait.

**Adopt the progress contract, not the loaders.gl machinery.** The ingest worker
R10 introduces should post `{ bytesRead, total }` back; the ingest dialog shows
determinate progress. This is a small addition to R10's message protocol that is
much more expensive to retrofit later. It also needs a DESIGN.md/VOICE.md pass,
which §4 of the plan currently only anticipates for R6, R10's table error state
and R11's cancel control — add it to R10's row.

### 2.6 The `ColorBreak` shape for R6

Kepler stores a classification break as
`{ range: [lo, hi], inputs: [formattedLo, formattedHi], label: string }`
(`data-scale-utils.ts:32-42`, produced by `getThresholdLabels`, `:180`), so the
legend renders from the same record the map was styled from.

R6 already says "legends read the stored values" and folds in **L22** (legend
swatch drops alpha). Adopting the shape — raw range, formatted inputs, label,
and resolved RGBA together in one stored record — makes legend/map drift
structurally impossible rather than merely fixed once. That is the strongest
version of R6 item 2, and it also closes M13 (hex sanitization at edit time,
since the resolved RGBA is computed exactly where the `HEX6` check belongs).

**Adopt the shape. Do not adopt the computation** (§1.3).

---

## 3. What Terrn should deliberately *not* adopt

Worth recording, because each looks attractive in kepler's architecture diagram
and each conflicts with a decision Terrn has already made.

| Kepler pattern | Why not |
|---|---|
| **`DataContainerInterface`** (`utils/src/data-container-interface.ts`) — row vs Arrow storage behind one accessor | Terrn's plan makes **DuckDB the single client-side store** (review §0.4), which is a cleaner answer to the same problem. A DataContainer seam would add a second storage abstraction whose only job is to hide a choice Terrn has already made. Kepler needs it because it supports row arrays *and* Arrow *and* DuckDB simultaneously; Terrn will support one. |
| **GPU filtering** (`DataFilterExtension`, `MAX_GPU_FILTERS = 4`, `table/src/gpu-filter-utils.ts`) | Terrn has **no map filtering feature at all** — only attribute-table text search (`grid-state.ts:85`), and none is on the roadmap (`TODOS.md:222-251`). Building for it now is speculative. See §5 for the strategic caveat. |
| **d3 scale functions** for visual channels | Incompatible with R8's expression compiler; see §1.3. |
| **Auto-layer discovery** (`findDefaultLayerProps`, `base-layer.ts:442`) creating Point/Arc/H3 layers automatically | Kepler's model is "drop a table, get layers". Terrn's is "drop a file, get *a* layer" with an explicit style panel. Adopting auto-discovery would change the product, not the architecture. The *coordinate-pairing* half is worth taking (§2.2); the layer-creation half is not. |
| **Redux `visState` + `postMergeUpdater`** | Terrn's Lit signals + `layer.store.ts` is deliberate (ADR-005). No reason to revisit. |
| **`compactArrowTable`** | Not reachable today (§1.5). |

---

## 4. Corrections to existing plan documents

### 4.1 The DEF-14 coercion table records the wrong value

The review (§DEF-14) and the recommendation doc both state:

| Input | Stored in DuckDB |
|---|---|
| `1`, `"N/A"`, `3` | `1`, **`null`**, `3` (Float64) |

Executed against Terrn's installed `apache-arrow` 21.1.0:

```
raw value      : NaN
typeof         : number
Object.is NaN  : true
=== null?      : false
nullCount      : 0
JSON.stringify : null      <-- the source of the error
```

The stored value is **`NaN`**, a non-null Float64. The table reads `null`
because `JSON.stringify(NaN)` is `"null"`. The other three rows of the review's
table reproduce exactly as written — the review is otherwise accurate.

**Why this matters, beyond pedantry:**

- **R2's tests would be written wrong.** The plan says "Expected values are
  written by hand" for each row of the coercion table. A test asserting `null`
  fails; a test round-tripping through JSON passes misleadingly. It must assert
  `Number.isNaN`.
- **R6 is affected.** `NaN` is not `NULL` in DuckDB: aggregates do not skip it.
  `quantile_cont` — which R6 adopts for classification breaks — returns `NaN`
  once any `NaN` is in the column, as do `min`, `max` and `stddev_pop`. So today
  a single `"N/A"` in a numeric column silently destroys the whole
  classification, and `WHERE col IS NULL` will not find the bad rows. With
  correct `NULL`s the aggregates skip them cleanly.
- It strengthens R2's ordering: the explicit schema must produce **`null`**, not
  `NaN`, for unparseable numerics. Worth stating explicitly in R2's scope, since
  "objects and arrays become JSON strings" is specified but the numeric-failure
  sentinel is not.

**Suggested edit:** correct the table in the review, and add to R2's validation
bullet: *"assert `Number.isNaN` for the current behavior and `null` for the
fixed behavior; add a DuckDB-side assertion that `quantile_cont` over a column
with one unparseable value returns a real number."*

### 4.2 R2's scope should absorb the schema unification

R2 currently fixes the **Arrow** side of type inference only. As shown in §2.1,
the UI side (`getLayerSchema`) is an independent second opinion that will still
disagree afterwards — arguably *more* visibly, since R2 makes the Arrow side
correct while leaving the 100-feature sampler wrong.

**Suggested edit to R2 scope:** add a fourth bullet —

> - **One schema.** The explicit per-column scan that builds the Arrow schema
>   also emits `SchemaField[]`. Delete `getLayerSchema`; `layer.fields` is
>   assigned from the scan. Removes the 100-feature sampling limit and the
>   UI/store type disagreement.
>   *Validation:* a fixture whose 150th feature introduces a property, and one
>   whose first feature's type differs from the rest — both must produce
>   identical types in `layer.fields` and in the DuckDB table.

This also retires an invariant risk: nothing today enforces that the two agree,
and nothing would notice if they drifted further.

### 4.3 R2 should absorb the CSV coordinate-detection fix

§2.2 is a verified silent-wrong-data defect, which is exactly R2's stated
theme ("Fixes for silently wrong data"). It sits beside M5 in the same function.
Fixing it in R2 also satisfies R2's own rationale for pulling M5 forward: *"Fix
it before R10 moves CSV parsing into the worker, so the worker copies correct
logic."*

**Suggested edit:** add to R2 scope and validation —

> - **CSV coordinate columns.** Match lat/lon on token boundaries, not
>   substrings, and never let the fallback overwrite an exact match.
>   *Validation:* headers `lat, latency, longitude_deg` and
>   `lat, plate_number, lon_deg` must select `lat`; `pickup_lat, pickup_lon,
>   dropoff_lat, dropoff_lon` must be deterministic and reported.

### 4.4 R1 is unaffected

I found nothing in kepler.gl that changes R1. The benchmark protocol (review §5)
is sound and the long-task gate is the right call. One observation: kepler's
`percent`-based progress (§2.5) would give R1's "drop → layer visible" row a
natural instrumentation point, but that is an R10 concern, not R1.

---

## 5. Strategic note for G1

This is the one place where kepler.gl's architecture argues with Terrn's plan
rather than supporting it, and it belongs in the G1 decision record.

Kepler.gl's entire data model exists to make **filtering and time animation
cheap**: `MAX_GPU_FILTERS = 4` channels updated as shader uniforms, so a range
slider over a million points re-renders without touching CPU geometry
(`table/src/gpu-filter-utils.ts`). That is deck.gl's structural advantage, and
it is the thing MapLibre is worst at — a data-driven paint or filter change
triggers tile relayout in MapLibre's worker (`requiresRelayout`, verified in
the review at `maplibre-gl-dev.js:23670-23693`).

Terrn's G1 currently measures **recolor latency** (category color change,
graduated 5 → 7, opacity drag). That is the right measurement for the features
Terrn has today. But the decision it gates is long-lived, and the review already
names the hybrid fallback (§6.2).

**Suggested addition to G1's exit criteria:** record explicitly whether
per-feature filtering or time animation is a plausible 12-month product
direction. If it is, the spike should add one row — *"apply a numeric range
filter over the 53k-point layer, 3 s drag"* — because that is the scenario where
MapLibre's relayout cost compounds per tick rather than per settle, and it is
the scenario where kepler's architecture is straightforwardly better. Today's
roadmap (`TODOS.md:222-251`) lists no filter feature, so on current evidence
**MapLibre remains the right call** — but the absence should be a recorded
finding rather than an unexamined assumption, since it is the single fact that
would flip G1.

---

## 6. Audit of the baseline architecture document

`kepler_data_ingestion_and_rendering_architecture.md` was checked claim by
claim against source. **It holds up well.**

**Method, stated plainly:** each row below was resolved to a file and line, and
the surrounding code read, before being marked correct.

*Updated 2026-09-21:* the three rows previously marked ◐ have since been traced
end to end (`vis-state-updaters.ts` read in full, §6.2), and kepler's own
processor suite has been installed and run (§6.3). Both confirmed the rows —
and surfaced one nuance the baseline document flattens, recorded in §6.2.

| Claim | Resolved at |
|---|---|
| `getKeplerLoaders` resolves loaders lazily per file | `loader-registry.ts:186-196` |
| `KeplerCSVLoader`, `JSONLoader`, `NDJSONLoader`, `KMLLoader`, `GPXLoader`, `TCXLoader`, `GeoArrowLoader`, `ParquetArrowLoader` | `loader-registry.ts:37-92` |
| Format routing to `processArrowBatches` / `processKeplerglJSON` / `processGeojson` / `processRowObject` | `file-handler.ts:358-377` |
| `makeProgressIterator` reports byte offsets and row counts | `file-handler.ts:67, :173-189` |
| `processGeojson` uses `@mapbox/geojson-normalize`, extracts `_geojson`, pads missing properties with `null` | `data-processor.ts:7, :351-387` |
| `MAX_GPU_FILTERS = 4`; range filters take 1 channel, time-interval filters 2 | `default-settings.ts:1319`; `gpu-filter-utils.ts:16-18` |
| `filteredIndex` on the CPU path | `kepler-table.ts:214-217` |
| `formatLayerData` / `renderLayer` lifecycle | `base-layer.ts:670, :675` |
| `computeDeckLayers` → `renderDeckGlLayer` | `reducers/src/layer-utils.ts:422, :283` |
| Three-tier sandwich; `@vis.gl/react-maplibre` or `react-map-gl/mapbox-legacy` | `map-container.tsx:7-8, :1346, :1432, :1557` |
| `KeplerGlobeView` for globe mode | `deckgl-layers/src/globe/globe-view.ts:326` |
| `postMergeUpdater` → `addDefaultLayers` / `addDefaultTooltips` / `findMapBounds` — **but all three are conditional, see §6.2** | `vis-state-updaters.ts:3649-3770` |
| `getLayerHoverProp` maps picked index back to the container | `reducers/src/layer-utils.ts:221-265` |
| `TripLayer` passes `animationConfig.currentTime` as a prop **and** an `updateTriggers` key | `trip-layer.ts:869, :886-891` |
| `compactArrowTable` exists to protect picking from batch explosion | `arrow-data-container.ts:130-144` |

Three location slips, none affecting substance:

1. **§2.1.1 conflates two delimiter mechanisms.** `KeplerCSVLoader` guesses via
   `delimitersToGuess: [',','\t',';','|']` (`kepler-csv-loader.ts:10, :33`), a
   parser option. `detectDelimiter` (`data-processor.ts:57`) is a separate
   function used on the raw-string path (`:156`). Same delimiter set, different
   code. For Terrn this is useful rather than pedantic: Terrn hand-rolls its
   parser (`parsers/index.ts:17`), so `detectDelimiter` is the applicable one.
2. **§2.1.1 places the JSON streaming paths in the loader registry.**
   `['$', '$.features', '$.datasets']` are in `file-handler.ts:52-55`; the
   registry only decides which loader to import.
3. **§3.4 says split viewports filter through `splitMaps[mapIndex].layers`.**
   The actual call is `getMapLayersFromSplitMaps(splitMaps, mapIndex)`
   (`reducers/src/layer-utils.ts:544`).

### 6.2 `vis-state-updaters.ts` read end to end (2026-09-21)

5,743 lines, ~90 exported updaters. The ingestion path resolves as the baseline
document describes, with **one nuance the document flattens**: it presents
auto-layer creation, auto-tooltips and auto-centering as an unconditional
four-step pipeline. All three are **option-gated**:

| Step | Gate | Line |
|---|---|---|
| `addDefaultLayers` | `!newLayers.length && options.autoCreateLayers !== false` | `:3671-3675` |
| `addDefaultTooltips` | `options?.autoCreateTooltips !== false` **and** `fieldsToShow[dataId]` empty | `:3699-3705` |
| `findMapBounds` → fit | `newLayers.length && options.centerMap` | `:3757-3766` |

`findMapBounds` also does not mutate state directly — it dispatches a
`ACTION_TASK_FIT_BOUNDS` task via `withTask` (`:3761-3765`).

The chain closes cleanly: `addDefaultLayers` (`:4365`) → `findDefaultLayer`
(`reducers/src/layer-utils.ts:73`) → iterates **every** entry in
`state.layerClasses`, calling each class's static `findDefaultLayerProps(dataset,
previous)` and threading the accumulator so a later class sees what earlier ones
claimed — then instantiates and calls `setInitialLayerConfig`. Arc and line
layers are created with `isVisible: false` ("arcs tend to be too musy",
`:94-95`).

The document also elides real complexity in `postMergeUpdater`:
`updateAllLayerDomainData`, a **second** domain pass for polygon filters that
need centroids from the first pass (`:3714-3742`), `updateAnimationDomain`, and
a recursive `applyMergersUpdater` for `layerMergers`.

**Why the nuance matters for Terrn.** Kepler makes these steps opt-out because
`addDataToMap` is a public programmatic API. Terrn's ingest is always
user-initiated, so it needs no such options — which is further support for §3's
position that auto-layer discovery is a **product** decision, not an
architectural one to copy.

### 6.3 Kepler's processor suite installed and run (2026-09-21)

`yarn install --immutable` (21 workspaces; no git submodules despite the
bootstrap script, so nothing in the checkout was mutated), then the
data-processor, file-handler and loader-registry tape suites run under
`babel-register`:

```
# tests 833
# pass  833
```

Note kepler's folder naming misleads: the real data-processor tests are in
`test/node/utils/data-processor-test.js`, **not** `test/node/processors/`.

What the run confirms behaviorally, rather than by shape:

| Review claim | Confirming assertions |
|---|---|
| **§2.1** one inference pass yields rich types | `getFieldsFromData` asserts **seven** distinct types — timestamp, integer, real, string, boolean, h3, geojson (tests 1-12). Terrn's `getLayerSchema` yields four, from a 100-feature sample, with no test at all. |
| **§2.1** one complete field descriptor per field | `processGeojson` asserts `name, id, displayName, format, fieldIdx, type, analyzerType, valueAccessor` **per field** (tests 624-642) |
| **§2.3** falsy values become real nulls | "first row second value should be null (empty)", "second row first value should be null (empty)" (tests 184, 186) |
| **§2.4** delimiter sniffing is mature, not a sketch | 18 assertions (140-150, 174-180): comma/tab/semicolon/pipe, CRLF, ambiguous cases, surrounding whitespace, **and quoted fields containing other delimiters** (147, 148, 149) |
| **§2.5** progress iterator reports per batch | `makeProgressIterator` batch assertions (739-741) |
| `getSampleForTypeAnalyze` | test 665 |

Two consequences for Terrn's plan:

- **§2.4 is upgraded from Medium to High confidence.** `detectDelimiter` is not
  a two-line heuristic; it is tested against exactly the cases that break naive
  sniffers — a tab-delimited file whose quoted fields contain commas. Terrn's
  hand-rolled RFC-4180 parser (`parsers/index.ts:17`) must apply the sniffer
  *outside* quoted regions or it will regress on the same inputs. Worth stating
  explicitly in R2's low-level plan.
- **§2.1's ask is smaller than it looks.** Kepler proves the single-pass
  contract is testable field-by-field. R2's new schema pass should assert the
  same shape — type *and* the values coerced to it — not merely that a column
  ends up `Float64`.

### 6.4 What the baseline document omits that matters for Terrn

Being an accurate description, its gaps are ones of framing rather than fact.
Two are worth recording:

**Kepler does not actually have one ingestion pipeline — it has two.** The
document's flowchart shows a single funnel into `createDataContainer()`. In
practice there is a second, parallel processor for the DuckDB path:

```
src/processors/src/data-processor.ts          694 lines
src/duckdb/src/processors/data-processor.ts   283 lines
```

Both define `CSV_NULLS` (`:42` and `:18`), both define
`cleanUpFalsyCsvValue` (`:213` and `:163`), and both define `processGeojson`
(`:349` and `:243`). **This is the single most useful thing in kepler.gl for
Terrn's R10 — as a warning, not a pattern.**

The two `processGeojson` implementations have not merely diverged in size; they
have diverged in **what they produce**. The file-path version extracts fields,
flattens properties and pads missing ones with `null` (`:360-387`). The
DuckDB-path version returns rows and no schema at all:

```ts
// src/duckdb/src/processors/data-processor.ts:253-258
// @ts-expect-error Don't pass empty fields, as duck db outputs an empty dataset
return {
  rows: normalizedGeojson
  // TODO get fields to preserve field names?
  // fields: []
};
```

A `@ts-expect-error` and an unanswered `TODO` about preserving field names, on
the path that feeds the analytical engine. **The drift is precisely a
schema-inference drift** — the exact failure this review recommends Terrn guard
against in §2.1 and below.

Terrn is about to create the same split. `CLAUDE.md` requires that
`ingest.worker.ts` import nothing from `src/utils/` ("Keep pure copies of math
helpers inside the worker"), and R10's plan adds an import-boundary spec to
enforce it. That invariant is right — but it *structurally guarantees* a second
copy of the schema-inference logic the moment §2.1's unification lands on the
main thread in R2 and then moves into the worker in R10. Kepler shows where that
ends: two copies that have already drifted apart in size and scope.

Terrn already has the mitigation and does not seem to have noticed it applies
here. The existing geo-worker invariant pairs the import boundary with
`geo-worker.spec.ts`, which "checks that the worker's copies still compute what
`utils/geo` and `renderer` compute". R10's plan specifies the *import-boundary*
spec but **not** an equivalence spec.

**Suggested edit to R10 validation:**

> - **Schema-equivalence spec.** The worker's inference must produce byte-identical
>   `SchemaField[]` and Arrow types to the main-thread implementation R2 landed,
>   over the R2 fixture corpus. Follows the `geo-worker.spec.ts` pattern: the
>   import boundary prevents sharing, so a test must prevent drift.

Without it, R2's schema unification has a shelf life of exactly one release.

**The document describes without costing.** Kepler.gl is a 20-package monorepo
(`package.json` workspaces); Terrn's `src/` is ~10,250 lines. Several patterns
the document presents neutrally — `DataContainerInterface`, the visual-channel
system, GPU filter slot allocation — exist because kepler supports row arrays,
Arrow, and DuckDB *simultaneously* and must stay fast across all three. Terrn
has decided to support one store (review §0.4). Read without that context, the
document reads as an endorsement of abstractions Terrn would be adopting for a
problem it has chosen not to have. §3 above records which ones and why.

---

## 7. Consolidated recommendation

| # | Action | Release | Confidence |
|---|---|---|---|
| 1 | Unify the three type-inference paths into one schema pass; delete `getLayerSchema` | **R2** (moves to worker in R10) | High — divergence demonstrated |
| 2 | Fix CSV lat/lon detection with token-boundary matching | **R2** | High — defect reproduced |
| 3 | Correct DEF-14's table to `NaN`; specify `null` as the fixed sentinel; add a `quantile_cont` assertion | **R2 / review edit** | High — reproduced |
| 4 | Null-sanitize (`CSV_NULLS`) before inference | **R2** | High |
| 5 | Delimiter sniffing (`, \t ; \|`) — apply **outside** quoted regions | **R2 or R10** | **High** (was Medium) — 18 passing assertions incl. quoted-delimiter cases (§6.3) |
| 6 | BigInt/DECIMAL cast in SQL (`CAST … AS DOUBLE`), never a JS loop | **R5/R6** | High |
| 7 | Adopt kepler's `ColorBreak` record shape (range + formatted inputs + label + RGBA) | **R6** | High |
| 8 | Ingest progress in the worker message protocol | **R10** | Medium |
| 8b | Schema-**equivalence** spec between worker and main-thread inference (not just the import-boundary spec) | **R10** | High — kepler shows the drift this prevents |
| 9 | Declared-type cascade (`PRAGMA table_info` first, WKB BLOB sniff as fallback) for query results — **restored, see correction** | **R13** | High — code read at `data-processor.ts:505-560`; `GEOMETRY` branch blocked on spatial |
| 10 | Record the filter/time-animation question in G1's exit criteria | **G1** | Medium — strategic, not blocking |
| — | ~~`getQuantileDomain` / `getScaleFunction` → style compiler~~ | **drop** | Kepler compiles no expressions; GeoLibre is the better reference (§1.3) |
| — | `compactArrowTable` guard | note only | Not reachable today |

Items 1–4 are the ones that change work already scheduled. Everything else is
additive and can be taken or left without disturbing the plan's shape.

---

## Appendix — verification method, coverage and confidence

**Coverage, stated honestly.** kepler.gl at this checkout is **167,878 lines
across 841 TypeScript files**. This review read roughly 600–800 of them — about
**0.5%** — selected by grepping for the symbols each claim named and reading the
surrounding 30–60 lines. This is *targeted verification of specific claims*, not
a depth read of the codebase. The §1.2 error is the characteristic failure mode
of that method: a scoped grep missed code that a repo-wide grep finds at once.

**Confidence by claim class:**

| Class | Confidence | Why |
|---|---|---|
| Terrn defects (§2.1, §2.2, §4.1) | **Highest** | Executed against Terrn's installed dependencies; reproductions included. Independent of kepler depth. |
| deck.gl 255-layer cap (§1.5) | **Highest** | Read in Terrn's own `node_modules`. |
| Positive kepler claims (code exists and does X) | **High** | Code read, not just grepped. |
| Kepler ingestion-path behavior (§2.1, §2.3, §2.4, §2.5) | **High** (raised 2026-09-21) | Kepler's own suite installed and run — 833 assertions, all passing (§6.3). Behavior, not shape. |
| Former ◐ rows | **High** (raised 2026-09-21) | `vis-state-updaters.ts` read end to end; chain traced through `findDefaultLayer` (§6.2). |
| Absence claims ("kepler has no X") | **Treat with caution** | This is where I was wrong once. §1.3's absence claim has since been re-checked repo-wide *and* against git history; §1.2's was not, and was wrong. |

**Follow-up completed 2026-09-21.** The two gaps flagged in the first revision
have both been closed: kepler's processor suite was installed
(`yarn install --immutable`, 21 workspaces, no submodules — checkout unmutated)
and run (833/833 passing, §6.3), and `vis-state-updaters.ts` was read end to end
(§6.2). Both confirmed the claims and added one nuance (option-gated auto-steps)
and one upgrade (§2.4 Medium → High).

**Correction to my own earlier note:** I wrote that the tests to run were
`src/processors/**/*.spec.ts`. No such files exist — kepler's processor tests are
tape suites under `test/node/`, and the data-processor ones are in
`test/node/utils/`, not `test/node/processors/`. Same class of error as §1.2:
asserting a path's contents without checking.

**What would raise confidence further:** the browser-side suites
(`yarn test-browser`) exercise the layer and rendering claims in §6's table,
which the node suites do not. Not run.

---

### Method notes

- All kepler.gl line references were resolved with `grep -n` / `sed -n` against
  `/tern/kepler.gl` at the checkout above, not from the prior documents.
- Terrn behavior claims marked "verified" were produced by executing the
  function's logic verbatim against Terrn's installed dependencies
  (`apache-arrow` 21.1.0, `@deck.gl/core` 9.3.2) from within `/tern/tern_poc`.
- Reproductions: CSV column detection (§2.2), schema divergence (§2.1),
  DEF-14 coercion table and the `NaN`/`null` distinction (§4.1), deck.gl's
  255-layer cap (§1.5).
- Absence claims: **§1.2's was wrong and is retracted** — the grep was scoped to
  the directory the citation named, and a directory-scoped miss was reported as
  a repository-wide absence. §1.3's absence claim was subsequently re-checked
  repo-wide and against `git log -S`, and survives.
- The baseline document (§6) was audited claim by claim; every row of its
  verification table was resolved to a file and line before being marked
  correct. Claims I could not resolve are listed as slips, not omitted.
