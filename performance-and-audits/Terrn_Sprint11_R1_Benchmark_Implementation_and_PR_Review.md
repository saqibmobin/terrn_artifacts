# Terrn Sprint 11: R1 Benchmark Implementation, PR Review, and Benchmark Flow Analysis

**Date:** 2026-09-23  
**Subject:** Sprint 11 R1 Milestone (`v0.10.2.0`), Pull Requests #42 through #56, and Benchmark Harness Flow  
**Target Repository:** `/tern/tern_poc`  
**Reference Documents:**
- [`docs/plan/sprint-11/R1-benchmark-harness-and-baseline.md`](file:///tern/tern_poc/docs/plan/sprint-11/R1-benchmark-harness-and-baseline.md)
- [`docs/plan/sprint-11/Terrn_Performance_Implementation_Plan.md`](file:///tern/tern_poc/docs/plan/sprint-11/Terrn_Performance_Implementation_Plan.md)
- [`docs/benchmark/README.md`](file:///tern/tern_poc/docs/benchmark/README.md)
- [`docs/benchmark/2026-09-21-v0.10.1.0-baseline.md`](file:///tern/tern_poc/docs/benchmark/2026-09-21-v0.10.1.0-baseline.md)
- [`docs/benchmark/2026-09-22-ci-smoke-noise-baseline.md`](file:///tern/tern_poc/docs/benchmark/2026-09-22-ci-smoke-noise-baseline.md)
- [`docs/benchmark/2026-09-22-render-counters.md`](file:///tern/tern_poc/docs/benchmark/2026-09-22-render-counters.md)
- [`docs/architecture/hld/registers/fitness-functions.yaml`](file:///tern/tern_poc/docs/architecture/hld/registers/fitness-functions.yaml)
- [`docs/architecture/hld/registers/constraints.yaml`](file:///tern/tern_poc/docs/architecture/hld/registers/constraints.yaml)
- [`docs/architecture/hld/registers/open-questions.yaml`](file:///tern/tern_poc/docs/architecture/hld/registers/open-questions.yaml)

---

## 1. Executive Summary

Sprint 11 initiates a comprehensive overhaul of Terrn's data ingest and rendering pipeline to eliminate long-standing responsiveness and capacity defects identified during architecture reviews. Milestone **R1 (`v0.10.2.0`)** was governed by a strict architectural constraint:

> **R1 ships no application performance changes.** Everything introduced is measurement, dataset generation, telemetry, or test plumbing.

The engineering rationale was foundational: optimizing code before having a verified, reproducible benchmark invites confirmation bias and risks introducing regressions that go unnoticed. Across **fifteen pull requests (PR #42 through PR #56)**, the team constructed, tested, debugged, and hardened a dual-track performance harness.

Starting from clock-based timing metrics (TBT, task durations), the harness confronted the physical limits of running software WebGL rasterization (SwiftShader) on shared CI runners. Through empirical observation, the team executed three major architectural evolutions:
1. **Machine-Checked Governance:** Converting prose rules (re-baselining, selector integrity) into automated, zero-overhead unit-test gates ([`FF-11`](file:///tern/tern_poc/docs/architecture/hld/registers/fitness-functions.yaml#L193-L240)).
2. **"Counters, Not Clocks":** Introducing deterministic work-demand counters (`Attributes updated`, `Layer updates`, `Pick Count`), solving the DEF-08 clone-per-tick puzzle, and establishing zero-headroom thresholds for render regressions.
3. **Seam Encapsulation & Upstream Drift Immunity:** Replacing the raw overlay handle with a contracted diagnostics seam (`window.__terrnDiagnostics`), guarding against upstream Deck.gl 9.4 renames with fail-fast `missingCounters()` assertions, and defeating the "Silent Pass" trap.

With the merge of **PR #56**, Milestone R1 is fully concluded. Terrn possesses a bulletproof, jitter-immune measurement foundation ready for **R2 (Web Worker Ingest & Schema Verification)**.

---

## 2. Review of the Full Pull Request Sequence (PR #42 – PR #56)

The R1 lifecycle unfolded across fifteen pull requests spanning core infrastructure, statistical stabilization, metric corrections, invariant automation, counter instrumentation, operational governance, and architectural seam encapsulation:

```
PR #42 (Core R1 Harness: C1–C9, Datasets, Baseline v0.10.1.0, Release v0.10.2.0)
  │
  ├── PR #43 (Fix: Compositor Boundary Settling / renderIdle)
  │
  ├── PR #44 (Chore: Node 24 & GitHub Actions v7 Bumps)
  │
  ├── PR #45 (Fix: Drop Unsound LoAF Cross-Check Assertion)
  │
  ├── PR #46 (CI: On-Demand bench_only Workflow Dispatch)
  │
  ├── PR #47 (Docs: Bank 5 Fixed-Commit Noise Baseline Runs)
  │
  ├── PR #48 (Feat: Enforce C7 Smoke Thresholds; FF-02 Implemented)
  │
  ├── PR #49 (Docs: 3-Run Tail Addendum; 2.43x Spread Validates 3x Multiplier)
  │
  ├── PR #50 (Docs: Correct Root Causes for 3 Ineligible Metrics)
  │
  ├── PR #51 (Test: Enforce Re-Baselining Provenance & Selector Integrity in Commit Gate)
  │
  ├── PR #52 (Docs: Add FF-11 to HLD Register & Correct Stale L3.2 Status)
  │
  ├── PR #53 (Feat: Implement Work-Demand Render Counters; Resolve DEF-08)
  │
  ├── PR #54 (Feat: Enforce Zero-Headroom Counter Thresholds in CI)
  │
  ├── PR #55 (Feat: Split FF-02.on_failure by Metric Class; Document K-13 & Q-22)
  │
  └── PR #56 (Refactor: Diagnostics Seam window.__terrnDiagnostics & Renamed Stat Detection)
```

### Detailed Pull Request Catalog

| PR | Branch | Key Commits | Description & Architectural Significance |
|---|---|---|---|
| **#42** | `feat/r1-benchmark-harness` | [`d5a0c0c`](file:///tern/tern_poc/e2e/global-setup.ts#L8), [`53344b1`](file:///tern/tern_poc/src/services/duckdb.ts#L36), [`ca31c7a`](file:///tern/tern_poc/scripts/generate-benchmark-samples.js), [`308caae`](file:///tern/tern_poc/scripts/generate-benchmark-samples.js#L187), [`c4816df`](file:///tern/tern_poc/docs/benchmark/2026-09-21-v0.10.1.0-baseline.md), [`9b81690`](file:///tern/tern_poc/bench/long-task-smoke.spec.ts), [`05d4b58`](file:///tern/tern_poc/package.json#L4) | **The Core R1 Delivery:**<br>• De-raced e2e preview server URL via Vitest `provide`/`inject`.<br>• Added categorized DuckDB initialization failure telemetry (`duckdb_init_failed`).<br>• Implemented deterministic sample generator ([`scripts/generate-benchmark-samples.js`](file:///tern/tern_poc/scripts/generate-benchmark-samples.js)).<br>• Pre-registered [`QA-03`](file:///tern/tern_poc/docs/architecture/hld/registers/quality-scenarios.yaml#L63) target block in HLD and closed [`Q-08`](file:///tern/tern_poc/docs/architecture/hld/registers/open-questions.yaml#L125).<br>• Executed `v0.10.1.0` baseline; created [`scripts/run-benchmark.mjs`](file:///tern/tern_poc/scripts/run-benchmark.mjs).<br>• Added CI smoke test ([`bench/long-task-smoke.spec.ts`](file:///tern/tern_poc/bench/long-task-smoke.spec.ts)). Released `v0.10.2.0`. |
| **#43** | `fix/bench-scenario-boundary` | [`e2c7f87`](file:///tern/tern_poc/bench/lib/gestures.mjs#L42) | **Compositor Settling Boundary:**<br>Discovered that S2 (table ready) showed 12× variance in CI because software rasterization from S1 was still executing at ~2 FPS for 2–3s after S1's task queue cleared. Introduced [`renderIdle()`](file:///tern/tern_poc/bench/lib/gestures.mjs#L42-L70) to wait for rAF frame rate return (`<= 25ms` stable for 500ms) between scenarios. S2 wall-time variance dropped from 2.31× to 1.01×. |
| **#44** | `chore/node-24-and-action-bumps` | `71310fa`, `9fc7bde` | **Infrastructure Modernization:**<br>Upgraded runtime to Node 24 (following Node 20 EOL) and upgraded GitHub Actions (`checkout`, `setup-node`, `upload-artifact`) to `v7`. |
| **#45** | `fix/bench-instrument-crosscheck` | [`bbd616a`](file:///tern/tern_poc/bench/long-task-smoke.spec.ts#L173) | **Dropping Unsound Cross-Check:**<br>Removed `expect(r.loaf_count).toBeGreaterThan(0)` from S1. Established that LoAF reports animation frames while Long Tasks reports tasks. Ingest tasks running without intermediate rendering do not trigger LoAF, making `loaf_count = 0` legitimate rather than broken. |
| **#46** | `ci/dispatch-bench-samples` | [`7cb5b98`](file:///tern/tern_poc/.github/workflows/test.yml#L15) | **On-Demand Noise Sampling:**<br>Added `workflow_dispatch` with `inputs.bench_only: true` to [`.github/workflows/test.yml`](file:///tern/tern_poc/.github/workflows/test.yml) to collect fixed-commit CI samples without burning minutes on lint, unit, and e2e gate suites. |
| **#47** | `docs/c7-noise-baseline` | [`c24c6ea`](file:///tern/tern_poc/docs/benchmark/2026-09-22-ci-smoke-noise-baseline.md) | **Banking 5 Fixed-Commit Runs:**<br>Executed and banked 5 fixed-commit CI runs on `7f5e0aa`. Analyzed metric spreads to determine which metrics were eligible for thresholding under C7. |
| **#48** | `feat/c7-smoke-thresholds` | [`e8f06de`](file:///tern/tern_poc/bench/long-task-smoke.spec.ts#L87), [`aca7124`](file:///tern/tern_poc/docs/architecture/hld/registers/fitness-functions.yaml#L22) | **Enforcing Tripwire Thresholds (C7):**<br>Activated thresholds in [`bench/long-task-smoke.spec.ts`](file:///tern/tern_poc/bench/long-task-smoke.spec.ts) via `Math.ceil(max_observed * 3)`: S1 TBT <= 3114 ms, S1 Longest Task <= 2166 ms, S2 Wall <= 2248 ms (holding out the 600 ms quiesce floor). Updated HLD [`FF-02`](file:///tern/tern_poc/docs/architecture/hld/registers/fitness-functions.yaml#L22) from null to implemented. |
| **#49** | `docs/c7-threshold-addendum` | [`e47817b`](file:///tern/tern_poc/docs/benchmark/2026-09-22-ci-smoke-noise-baseline.md#L121) | **Post-Threshold Verification:**<br>Recorded 3 additional CI runs on `fff1230`. All passed, but all 3 exceeded the 5-sample maximum for S1 TBT (spread reached 2.43×). Proved that 5 samples under-sampled the tail, validating the necessity of the 3× safety multiplier. |
| **#50** | `docs/correct-metric-causes` | [`1c2d70f`](file:///tern/tern_poc/docs/benchmark/2026-09-22-ci-smoke-noise-baseline.md#L78) | **Correction of Ineligible Metric Causes:**<br>Refined and corrected the technical justifications for why LoAF, `frame_p95_ms`, and S2 blocking metrics cannot be thresholded in CI. Aligned HLD [`FF-02`](file:///tern/tern_poc/docs/architecture/hld/registers/fitness-functions.yaml#L46) with exact browser mechanics. |
| **#51** | `test/benchmark-harness-enforcement` | [`e25434b`](file:///tern/tern_poc/scripts/benchmark-selectors.spec.ts) | **Mechanical Invariant Enforcement:**<br>• Implemented [`scripts/benchmark-selectors.spec.ts`](file:///tern/tern_poc/scripts/benchmark-selectors.spec.ts): AST/regex sweep asserting all driver selectors resolve in markup (CSS excluded; hyphen-safe lookarounds).<br>• Implemented [`scripts/benchmark-thresholds.spec.ts`](file:///tern/tern_poc/scripts/benchmark-thresholds.spec.ts): Verifies complete machine-readable provenance chain from Markdown baseline tables $\rightarrow$ spec constants $\rightarrow$ `FF-02.threshold`. Includes decay tripwire on new dated docs.<br>• Checked off [`TODOS.md:185`](file:///tern/tern_poc/TODOS.md#L185). |
| **#52** | `docs/ff-11-and-l32-correction` | [`9dcaa1c`](file:///tern/tern_poc/docs/architecture/hld/registers/fitness-functions.yaml#L193) | **Architectural Formalization:**<br>• Added [`FF-11`](file:///tern/tern_poc/docs/architecture/hld/registers/fitness-functions.yaml#L193) (`measurement.harness-integrity`) to HLD register.<br>• Corrected stale L3.2 operations table where `QA-03` had remained marked "Blocked on Q-08".<br>• Opened [`Q-21`](file:///tern/tern_poc/docs/architecture/hld/registers/open-questions.yaml#L459) to track HLD rendered-table drift. |
| **#53** | `feat/render-counters` | [`cd7c656`](file:///tern/tern_poc/bench/lib/counters.mjs), [`6fc79cd`](file:///tern/tern_poc/docs/benchmark/2026-09-22-render-counters.md) | **The Counter Breakthrough (Demand, Not Frames):**<br>• Implemented [`bench/lib/counters.mjs`](file:///tern/tern_poc/bench/lib/counters.mjs): Reads `Attributes updated`, `Layer updates`, and `Pick Count` from `deck.stats`.<br>• Neutralized deck.gl's 60-frame stats wipe via prototype patching on `Stats.prototype.reset`.<br>• Unconditionally exposed `window.__terrnDeckOverlay` in all builds (later refined to `window.__terrnDiagnostics` in PR #56).<br>• Added S3 pan/zoom sweep to smoke test.<br>• Resolved DEF-08: Discovered 86 full attribute rebuilds occurring during a 3s drag (masked by TBT < 50ms). |
| **#54** | `feat/counter-thresholds` | [`9155940`](file:///tern/tern_poc/bench/long-task-smoke.spec.ts#L87) | **Enforcing Zero-Headroom Counter Thresholds:**<br>• Enforced invariant counter thresholds in CI: S1 `attrs <= 1`, S2 `attrs <= 0`, S3 `attrs <= 0`. Zero multiplier needed due to zero runner variance.<br>• Validated negative control: Injected regression caused 39 attribute rebuilds while appearing *faster* on wall-clock time.<br>• Banked 5 fixed-commit runs in baseline §8 (S1 TBT reached 3.03× spread). Updated `FF-02` and `benchmark-thresholds.spec.ts`. |
| **#55** | `feat/ff02-split-on-failure` | [`ccfa644`](file:///tern/tern_poc/bench/long-task-smoke.spec.ts#L18) | **Metric-Class Failure Split & Operational Reality:**<br>• Split `FF-02.on_failure`: `timing: warn` (routed through `budget()`, writes summary, leaves job green) vs. `counters: block-merge` (routed through `expect()`, fails job on attribute churn).<br>• Documented Constraint [`K-13`](file:///tern/tern_poc/docs/architecture/hld/registers/constraints.yaml#L159) and Open Question [`Q-22`](file:///tern/tern_poc/docs/architecture/hld/registers/open-questions.yaml#L487): GitHub free-tier private repo cannot enforce branch protection; `block-merge` turns check red but relies on human enforcement. |
| **#56** | `refactor/diagnostics-seam` | [`2705de5`](file:///tern/tern_poc/src/core/canvas/base-map.ts#L84), [`ac7d46d`](file:///tern/tern_poc/bench/lib/counters.mjs#L74) | **Diagnostics Seam & Renamed Stat Key Guard:**<br>• Replaced raw mutable overlay global `window.__terrnDeckOverlay` with contracted diagnostics seam `window.__terrnDiagnostics` (`getDeckStats()`, `getDeckLayers()`).<br>• Updated all e2e and benchmark consumers.<br>• Hardened against upcoming Deck.gl 9.4 upgrade (Sprint 11 R3): Isolated undocumented `_deck` and `_props` internals in `base-map.ts`.<br>• Defeated the "Silent Pass" vulnerability: Deck `stats.stats[key]` lookup returns `null` rather than `0` for unrecognised keys. Added `missingCounters()` running post-work (respecting Deck's lazy stat creation) to fail fast on renamed upstream counters instead of silently passing zero-rebuild invariants.<br>• Structured installation failure diagnostics: `installCounters` returns `{ ok, reason }`.<br>• Documented R3 upgrade note in [`TODOS.md`](file:///tern/tern_poc/TODOS.md). |

---

## 3. Deep-Dive: Key Breakthroughs & Structural Evolutions

### A. The "Counters, Not Clocks" Breakthrough (PR #53 & PR #54)
Running WebGL under software emulation (SwiftShader) on shared CI runners presented a fatal challenge: **clock jitter**. Wall-clock and TBT metrics fluctuated by up to 3.03×, forcing a $3\times$ threshold multiplier that blinded CI to regressions below +200%.

The breakthrough was distinguishing between **Work Demand** and **Frame Supply**:
- **Work Demand (Deterministic):** `Attributes updated`, `Layer updates`, `Pick Count`. These count what the **application asked the GPU to do**.
- **Frame Supply (Noisy):** `Redraw Count` (6 vs 0 across identical runs), `Layers rendered` (183 vs 180). These count how often the CPU rasterizer yielded a frame.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           THREE RUNS OF THE SAME GESTURE                    │
├─────────────────┬────────────────────┬─────────────────┬────────────────────┤
│    Scenario     │ Attributes updated │  Layer updates  │     Pick Count     │
├─────────────────┼────────────────────┼─────────────────┼────────────────────┤
│ Single set ×3   │ 1 / 1 / 1          │ 2 / 2 / 2       │ 0                  │
│ 3 s drag ×3     │ 87 / 86 / 86       │ 174 / 172 / 172 │ 0                  │
│ Pan/zoom ×2     │ 0 / 0 (exact)      │ 0 / 0 (exact)   │ 25 / 25 (exact)    │
└─────────────────┴────────────────────┴─────────────────┴────────────────────┘
```

#### Resolving DEF-08
In the `v0.10.1.0` baseline, S3 (opacity drag) appeared "nearly free" with only 17 ms TBT, leading reviewers to wonder whether DEF-08 (deep clone per tick) was a phantom defect. Counters proved that **a 3-second drag triggered 86 full attribute rebuilds for 53,316 features**. TBT was completely blind to it because individual updates took <50ms.

#### The Negative Control (PR #54)
To prove the counter tripwire works, the team injected two R2-shaped architectural mistakes (a `move` handler triggering `updateDeckOverlay` and `styleConfig` shallow-cloned per build):
```
Wall-clock time:  44,422 ms  (BELOW the clean minimum of 46,278 ms — looked faster!)
TBT:              13,062 ms  (Mid-range normal)
Attrs Updated:        39     (FAILED: expected <= 0)
```
A catastrophic regression that regenerated buffers 39 times sailed past every clock metric and was caught exclusively by the counter.

### B. Machine-Checked Governance: PR #51 & PR #52 (`FF-11`)
To prevent the benchmark harness from decaying into unmaintained debt, two automated unit specs were introduced in `scripts/`:

1. **Selector Guard ([`scripts/benchmark-selectors.spec.ts`](file:///tern/tern_poc/scripts/benchmark-selectors.spec.ts)):**
   - AST/regex sweep extracting all `#ids`, `.classes`, and `cardAction` parameters from drivers.
   - Asserts each exists in `index.html` or `src/` (CSS excluded to prevent false passes from dead stylesheets).
   - Uses negative lookarounds `(?<![\w-])${token}(?![\w-])` to avoid regex `\b` boundary false positives on hyphenated tokens.
2. **Provenance & Decay Guard ([`scripts/benchmark-thresholds.spec.ts`](file:///tern/tern_poc/scripts/benchmark-thresholds.spec.ts)):**
   - Parses Markdown tables from banked baseline docs.
   - Re-derives `Math.ceil(max * 3)` for timing and exact invariants for counters.
   - Asserts that `THRESHOLDS`, `COUNTERS`, and `FF-02.threshold` in YAML agree literally.
   - **Decay Tripwire:** Fails the commit gate if any newer dated file appears in `docs/benchmark/` without thresholds being updated.
3. **Formal HLD Registration ([`FF-11`](file:///tern/tern_poc/docs/architecture/hld/registers/fitness-functions.yaml#L193)):**
   Registered as `measurement.harness-integrity`, providing formal architectural standing alongside `FF-05` (telemetry schemas) and `FF-10` (HLD counts).

### C. Splitting `FF-02.on_failure` and Operational Reality (PR #55)
Because shared runner timing jitter reached 3.03× over thirteen samples, timing thresholds carrying a $3\times$ multiplier are on the verge of flaking. If the job failed on a timing flake, developers would mute the job, killing the counter tripwire.

PR #55 split the failure handling:
- **Timing (`warn`):** Evaluated via [`budget()`](file:///tern/tern_poc/bench/long-task-smoke.spec.ts#L78). Emits GitHub warning annotations and step-summary entries, leaving the CI job green.
- **Counters (`block-merge`):** Evaluated via `expect()`. Fails the job on attribute churn regressions.

#### Constraint `K-13` and Open Question `Q-22`
Investigating how `block-merge` was enforced revealed that GitHub gates branch protection and rulesets behind Pro or public visibility. Because Terrn is a private repository on a free tier, **there are no mechanical status check gates on `main`**.
- Documented in [`K-13`](file:///tern/tern_poc/docs/architecture/hld/registers/constraints.yaml#L159): `block-merge` represents an **intent** (turning the check red loudly) rather than an enforced API block.
- Documented in [`Q-22`](file:///tern/tern_poc/docs/architecture/hld/registers/open-questions.yaml#L487) to decide whether to upgrade, make the repo public, or adjust register vocabulary.

### D. The Diagnostics Seam & Defeating the "Silent Pass" Vulnerability (PR #56)

Review of the counter instrument in PR #53 highlighted that exposing `window.__terrnDeckOverlay` leaked the entire mutable `MapboxOverlay` instance into the global scope. Consumers reached through private properties (`_deck`, `_props`), creating a multi-file maintenance risk ahead of the **Deck.gl 9.4 upgrade in Sprint 11 R3**.

PR #56 executed a precise architectural refactoring:

#### 1. Narrowing to Contracted Accessors (`__terrnDiagnostics`)
In [`src/core/canvas/base-map.ts`](file:///tern/tern_poc/src/core/canvas/base-map.ts#L84-L126), the raw global was replaced with:
```typescript
if (typeof window !== 'undefined') {
  (window as any).__terrnDiagnostics = {
    /** Deck's live Stats collector, or null when there is no Deck. */
    getDeckStats: () => (deckOverlay as any)?._deck?.stats ?? null,
    /** The layer array last handed to Deck. Empty when there is none. */
    getDeckLayers: () => (deckOverlay as any)?._props?.layers ?? [],
  };
}
```
- **Architectural Scope (Not False Security):** The codebase explicitly clarifies that Terrn is a client-side application without a privilege boundary (MapLibre's control list already grants full access). The accessors exist for **maintainability** (localizing third-party private property access to two lines) and **contractual clarity** (a named observability seam rather than an accidental debug global).
- **Honest Mutability:** The seam is an observability hook, not a sandbox. `getDeckStats()` hands back the live `Stats` instance, which the harness must mutate by patching `Stats.prototype.reset` to neutralize Deck's 60-frame wipe.

#### 2. The Upstream Dependency Chain
The counter harness relies on four upstream internals:
1. `MapboxOverlay._deck` (isolated behind `getDeckStats()`).
2. `Deck.stats` (`protected` in TypeScript definitions).
3. The `@probe.gl` shape of `stats.stats[key].count`.
4. The exact string keys (`"Attributes updated"`, `"Layer updates"`, `"Pick Count"`).

While the seam isolates #1, items #2–#4 cannot be encapsulated without upstream support. They are defended via **mechanical test assertions** that fail the Benchmark Smoke job if upstream changes break the instrument.

#### 3. Defeating the "Silent Pass" Trap
The stat string keys (#4) represented the most dangerous vulnerability in the harness: **a failure that looks like success**.
- `stats.stats[key]` is an object property lookup. If Deck.gl 9.4 renames `"Attributes updated"` (e.g., to camelCase or a new label), an unhandled lookup returns `undefined`.
- If defaulted with `?? 0`, the harness would report a confident `0`.
- In Scenario S3 (pan/zoom), the threshold is `attrs_updated <= 0`. A quiet `0` is identical to "the app rebuilt 0 buffers" — **a passing test result on a completely blind instrument!**

PR #56 resolved this by:
1. Returning `null` rather than `0` for unrecognised stat keys in [`readCounters()`](file:///tern/tern_poc/bench/lib/counters.mjs#L125).
2. Implementing [`missingCounters(page)`](file:///tern/tern_poc/bench/lib/counters.mjs#L148) to assert that all required counter keys exist in Deck's dictionary.

#### 4. The Lazy Stat Allocation Trap
During implementation, the developer discovered that **Deck creates each stat lazily on first use**. When checked before real work had occurred, `"Layer updates"` did not yet exist in `stats.stats`, causing a false-positive failure on a healthy run.
- **Resolution:** `missingCounters(page)` is asserted immediately *after* S1 Ingest work in [`bench/long-task-smoke.spec.ts`](file:///tern/tern_poc/bench/long-task-smoke.spec.ts#L380), where all three counters are guaranteed to have been created.
- In `deltaCounters()`, missing stats are treated as `0` in arithmetic, correctly representing "zero occurrences prior to creation."

#### 5. Structured Installation Diagnostics
`installCounters()` now returns `{ ok: boolean, reason: string | null }`, clearly distinguishing between WebGL initialization failure (headless fallback) and a sealed prototype patch rejection.

---

## 4. Full Benchmark Architecture

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                   DATASET SYNTHESIS                                    │
│ scripts/generate-benchmark-samples.js (Seeded xorshift32 PRNG; deterministic, uncommitted) │
└──────────────────────────┬──────────────────────────────────────────┬──────────────────┘
                           │                                          │
            [bench_points_53k.geojson]                 [All 3 files: 53k, 2.3k, 50MB]
                           │                                          │
                           ▼                                          ▼
┌──────────────────────────────────────────────┐ ┌─────────────────────────────────────────┐
│              TRACK A: CI SMOKE               │ │            TRACK B: PROTOCOL            │
│       bench/long-task-smoke.spec.ts          │ │        scripts/run-benchmark.mjs        │
├──────────────────────────────────────────────┤ ├─────────────────────────────────────────┤
│ Target: e2e bundle (NODE_ENV=development)    │ │ Target: Production bundle (vite build)  │
│ Display: Headless Chrome (SwiftShader CPU)   │ │ Display: Windowed Chrome (Hardware GPU) │
│ Scenarios: S1 (Ingest), S2 (Table), S3 (Pan) │ │ Scenarios: S1-S10 + S11 (Full Suite)    │
│ Clocks: budget() -> Warn / Non-blocking      │ │ Clocks: Evaluated against QA-03 Target  │
│ Counters: expect() -> Fails job on churn     │ │ Counters: Banked across major releases  │
└──────────────────────────────────────────────┘ └─────────────────────────────────────────┘
                           │                                          │
                           └────────────────────┬─────────────────────┘
                                                ▼
                               ┌──────────────────────────────────┐
                               │       SHARED INSTRUMENTATION     │
                               │  bench/lib/probe.mjs (Clocks)    │
                               │  bench/lib/counters.mjs (Stats)  │
                               │  bench/lib/gestures.mjs (CDP)    │
                               │  window.__terrnDiagnostics       │
                               │  (getDeckStats, getDeckLayers)   │
                               └──────────────────────────────────┘
                                                │
                                                ▼
                               ┌──────────────────────────────────┐
                               │       ARCHITECTURAL GUARDS       │
                               │  FF-02: Timing warn / Count block│
                               │  FF-11: Selector drift & decay   │
                               │  K-13: Human-enforced merge gate │
                               │  missingCounters(): Upstream keys│
                               └──────────────────────────────────┘
```

---

## 5. Summary of Pre-Registered Targets & CI Thresholds

### A. Pre-Registered Target Verdict (`v0.10.1.0`)
Scored against [`QA-03`](file:///tern/tern_poc/docs/architecture/hld/registers/quality-scenarios.yaml#L63) (TBT < 200 ms and longest task < 100 ms per gesture at 53,316 features):
- **Verdict:** **MISSED (4 pass, 5 fail)**.
- **Passes:** S3 (Opacity drag, 17 ms), S4 (Graduated breaks, 47 ms), S5 (Category color, 0 ms unverified), S10 (Basemap switch, 0 ms).
- **Failures:** S6 (Polygon toggle, 341 ms), S7a/b (Hover sweep, 719 ms / 2,019 ms), S8 (Pan/zoom, 722 ms), S9 (Cluster sweep, 768 ms).
- Kept intact to serve as the benchmark against which Sprint 11 R2–R4 improvements are judged.

### B. Enforced CI Smoke Thresholds
Configured in [`bench/long-task-smoke.spec.ts`](file:///tern/tern_poc/bench/long-task-smoke.spec.ts#L87-L105):

| Scenario | Metric | Class | Threshold | Enforcement | Provenance |
|---|---|---|---|---|---|
| **S1 Ingest** | `tbt_ms` | Clock | `<= 3114 ms` | `budget()` (warn) | Banked 5-run max (1038 ms) × 3 |
| **S1 Ingest** | `longest_task_ms` | Clock | `<= 2166 ms` | `budget()` (warn) | Banked 5-run max (722 ms) × 3 |
| **S1 Ingest** | `attrs_updated` | Counter | `<= 1` | `expect()` (fail) | Initial buffer upload invariant |
| **S2 Table Ready** | `wall_ms` | Clock | `<= 2248 ms` | `budget()` (warn) | 600 ms floor + (max - 600 ms) × 3 |
| **S2 Table Ready** | `attrs_updated` | Counter | `<= 0` | `expect()` (fail) | Arrow unmarshalling invariant |
| **S3 Pan/Zoom** | `attrs_updated` | Counter | `<= 0` | `expect()` (fail) | Clean pan invariant (DEF-08 guard) |

---

## 6. Conclusion and Readiness for Sprint 11 R2

Milestone R1 (`v0.10.2.0`) is concluded with exceptional architectural rigor across 15 pull requests:
1. **Deterministic Fixtures:** 53k point, 2.3k polygon, and 50 MB point datasets generated from a fixed seed and protected against stub substitution by `MIN_BYTES` guards.
2. **Jitter-Immune Tripwires:** The counter harness provides $+0$ invariant assertions that catch rendering regressions in CI under SwiftShader without clock noise.
3. **Self-Policing Governance:** [`FF-11`](file:///tern/tern_poc/docs/architecture/hld/registers/fitness-functions.yaml#L193) guarantees that UI redesigns cannot break benchmark selectors and that performance improvements cannot silently rot thresholds.
4. **Honest Operational Alignment:** [`K-13`](file:///tern/tern_poc/docs/architecture/hld/registers/constraints.yaml#L159) and [`Q-22`](file:///tern/tern_poc/docs/architecture/hld/registers/open-questions.yaml#L487) ensure the HLD registers do not overstate their enforcement mechanisms on free-tier infrastructure.
5. **Contracted Observability & Upstream Armor (PR #56):** Encapsulated MapboxOverlay internals behind `window.__terrnDiagnostics` accessors, and defeated the "silent pass" vulnerability via `missingCounters()` so upstream Deck.gl 9.4 changes cannot quietly blind the render tripwires.

The codebase is fully armored and ready to begin **Sprint 11 R2 (Fixes for Silently Wrong Data / Web Worker Ingestion)**.

