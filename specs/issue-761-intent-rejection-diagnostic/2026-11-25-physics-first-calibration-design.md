# Physics-First Calibration Methodology

**Branch:** issue-763-physics-first-calibration
**Issue:** #763
**Date:** 2026-11-25

## Problem

The EmulatedGame's worker physics accuracy depends on several constants that were calibrated by fitting against aggregate SC2 baselines (dual-run comparisons, cadence matching). These constants are classified as **Calibrated** in `sc2-physics-constants.md` and produce accurate results (§16 shows ±0.3–2.8% across saturation levels). However, their values were derived by aggregate fitting — not by directly measuring the SC2 mechanisms they model.

**Why this matters:** Aggregate-fit calibration creates coupling. Each constant was swept to match baselines with all other constants held fixed. If one constant changes (because its mechanism is better understood), others may need re-sweeping. The project rule is "no approximations, replicate SC2's actual mechanisms." Direct measurement replaces the compensation treadmill with independent, verifiable values.

**Constants audited:**

| Constant | Value | Nature | Derivation | Existing classification | Measurement target |
|----------|-------|--------|------------|------------------------|--------------------|
| `APPROACH_ANIMATION_LOOPS` | 20.0 | Observable | Aggregate-fit: cadence fitting #679, simplified #707 | **Calibrated** — #679/#707 | Direct timing of SC2 worker approach animation |
| `GAS_APPROACH_DOCKING_LOOPS` | 11.5 | Observable | Aggregate-fit: dual-run sweep #689 | **Calibrated** — #689 | Direct timing of gas entry animation |
| `LOW_SATURATION_DODGE_SCALE` | 0.25 | Emergent | Aggregate-fit: dodge sweep #656 | **Calibrated** — #656 | Validate: should emerge from correct collision physics |
| `INITIAL_WORKER_STAGGER` | 6.1 | Emergent | Aggregate-fit: dual-run sweep #578 | **Calibrated** — #609/#578 | Direct observation of SC2 starting worker desync |
| `GAS_DEPOSIT_LOCK_LOOPS` | 8.0 | Unknown | Aggregate-fit: 6w sweep #682 | **Default** | Investigate: does SC2 have a deposit lock mechanism? |
| `MINERAL_PATCH_OBSTACLE_RADIUS` | 0.5 | Implementation artifact | Constrained: tile-geometry #766 | **Calibrated** — #766 | Not aggregate-fit — constrained by A* grid resolution on TorchesAIE integer-x positions |
| `MIN_APPROACH_AFTER_WAIT` | 0 (code) / 12 (doc) | Observable | Aggregate-fit: sweep #763 | **Calibrated** — #763 (doc only) | Direct timing of SC2 irreducible pickup animation |
| `WAIT_ABSORPTION_THRESHOLD` | ∞ (disabled) | Unknown | Swept in `ContentionCalibrationDiagnosticTest`, not activated | — | Investigate: does SC2 have a threshold-based absorption model? |
| `HIGH_SATURATION_DODGE_SCALE` | 1.0 (disabled) | Emergent | Swept in `ContentionCalibrationDiagnosticTest`, not activated | — | Validate alongside `LOW_SATURATION_DODGE_SCALE` |

**Excluded from audit:** `PATCH_DOCKING_OVERHEAD_LOOPS` (22.0) — directly measured from SC2 per-worker cadence data as the difference between contested and uncontested cycle times (22 loops, #631). This has genuine direct-measurement evidence, unlike the aggregate-fit constants above.

**Known code/doc inconsistency — `MIN_APPROACH_AFTER_WAIT`:** The value is 0 in `WorkerSimulation.java:22` but documented as 12 in `sc2-physics-constants.md` §3 (**Calibrated** — #763). The formula was fixed from `max(20, 20-waited)` (always 20, a no-op) to `max(MIN_APPROACH_AFTER_WAIT, 20-waited)` at `WorkerSimulation.java:689`, but the constant value was not updated from 0 to the swept optimal of 12. The pinned calibration results in §16 (+0.8% single-base, −0.9% two-base) reference the value 12, suggesting the doc was updated aspirationally. Issue #763 is the same GitHub issue referenced in both this spec and `sc2-physics-constants.md` — the doc records intended results from the sweep in `ContentionCalibrationDiagnosticTest.sweepMinApproachAfterWait()`, but the production code was not updated. This spec's Phase 2 will measure the irreducible approach animation directly from SC2 to resolve this inconsistency with an independently measured value.

**Known config default mismatch — `MINERAL_PATCH_OBSTACLE_RADIUS`:** `WorkerPhysicsConfig.DEFAULT_MINERAL_PATCH_OBSTACLE_RADIUS = 0.5f` but `WorkerPhysicsConfigProducer` has `defaultValue = "0.3"`. Code using `WorkerPhysicsConfig.defaults()` gets 0.5; code using the Quarkus-injected config gets 0.3. This is a pre-existing bug to fix independently of the calibration work.

## Solution

A reusable methodology for validating aggregate-fit constants with direct per-mechanism measurements, executed in phases from lowest-noise to highest.

### Constant Classification

Every EM constant has two orthogonal properties:

**Nature** (what kind of mechanism it represents):

| Category | Meaning | Fix path |
|----------|---------|----------|
| **Data editor** | Value from SC2 game files | Look it up |
| **Observable** | Measurable from SC2 API per-loop observation | Record from SC2, measure, replace |
| **Emergent** | Should not be a constant — effect should emerge from correct simulation | Fix the physics, delete the constant |
| **Unknown** | We don't know if SC2 has this mechanism | Investigate first |

**Derivation method** (how the value was obtained):

| Method | Meaning |
|--------|---------|
| **Measured** | Directly observed from SC2 at per-mechanism level |
| **Aggregate-fit** | Swept/curve-fit against aggregate baselines |
| **Constrained** | Value forced by implementation artifacts (grid resolution, etc.) |
| **Default** | No calibration evidence |

The existing `sc2-physics-constants.md` **confidence levels** (Calibrated, Published, Estimated, Placeholder) remain unchanged — they describe validation status, which is orthogonal to nature and derivation method.

### SC2 Worker Profiler

New `@Tag("sc2")` test: `WorkerPhysicsProfilerTest` in `io.quarkmind.sc2.real`.

**Relationship to existing infrastructure:** `GasCycleProfilerTest.profileGasCycleDecomposition()` (in `io.quarkmind.sc2.real`) already profiles gas workers at per-game-loop resolution — per-worker position, cargo, action classification, phase decomposition via `LoopWorkerObs` records. `WorkerPhysicsProfilerTest` follows the same pattern and infrastructure (`QuarkusSC2Transport`, `PlaybookSC2Adapter`, per-loop `onStep()` callback) but targets mineral workers across saturation levels.

**Mechanism:** Drives real SC2 via the bot API at per-loop resolution (`RequestStep` with `step_size=1` — the existing `gameLoop()` method already uses this). Runs short controlled windows on TorchesAIE.

**Scenarios:**
- 8w economy-only, 50 ticks — zero contention, foundation constants
- 16w economy-only, 50 ticks — moderate contention (primary comparison tier)
- 24w economy-only, 50 ticks — high contention
- 36w economy-only, 50 ticks — extreme contention
- 3w gas single-geyser, 50 ticks — gas cycle (extends existing `GasCycleProfilerTest` data)
- Game start, 10 ticks — initial worker stagger

Each scenario reaches the target worker count via playbook, then switches to per-loop stepping for the measurement window. At 50 ticks with ~122 loops/cycle, 8 workers produce ~56 full cycles (8w) — sufficient for statistical significance given SC2 cycle σ=0.58 ticks (§1b). All scenarios use a uniform 50-tick window for consistency.

**Per-loop recording per worker:**
- Tag (worker identity — tag-based tracking, not index-based, per §1b observation-reordering fix #634)
- Position (x, y)
- Cargo (mineral/gas count)
- Orders (ability ID, target position)

**Output:** JSON file per scenario in `src/test/resources/profiler/`. One entry per game loop per worker.

**Multi-run protocol:** Each scenario is run 3 times minimum (SC2's random update ordering produces per-game variance, §1b). Report 3-run mean and σ per constant. For constants with high per-game variance, run 5 times (matching the existing 5-run protocol for gas, §4/§6 of sc2-physics-constants.md).

### Worker Physics Analyser

Plain Java class `WorkerPhysicsAnalyser` in `io.quarkmind.sc2.real` (test source). Reads profiler JSON and computes constants from state transitions.

**State inference (cargo-primary):**

Primary signals (from cargo transitions — deterministic, unambiguous):
- Cargo 0→5 → mining complete (cycle boundary)
- Cargo 5→0 → deposit complete (cycle boundary)

Secondary signals (from position — requires tolerance for high-saturation jitter):
- Position within tolerance of patch center → at patch
- Position within tolerance of TH center → at TH
- Position moving between patch and TH → walking

**Position tolerance:** At ≤16w, position stability threshold is tight (±0.3u). At 24w+, increase to ±0.75u to accommodate collision-driven position jitter. The 0.75u tolerance matches the collision radius documented in §3 of sc2-physics-constants.md. At high saturation (36w), SC2 workers jostle around mineral patches due to collision physics — a worker's position fluctuates within the collision radius even while logically "at" the patch.

**Mining vs waiting disambiguation:** A worker at a patch with stable position could be mining OR waiting. Cargo flip timing (0→5) is the definitive signal for mining completion. The interval between arrival at patch (position stable) and cargo flip (0→5) decomposes into wait time + approach animation + mining animation. At 8w with no contention, this directly measures approach + mining.

**Derived measurements per worker per cycle:**
- `walkToPatch` — loops from cargo flip (5→0, deposit complete) to position within tolerance of patch
- `approachAndMine` — loops from patch arrival to cargo flip (0→5)
- `walkToBase` — loops from patch departure (position starts moving from patch) to position within tolerance of TH
- `deposit` — loops from TH arrival to cargo flip (5→0)
- `waitAtPatch` — at contention tiers: loops from arrival at patch to start of mining (inferred as `approachAndMine - approachAndMine_8w`, since 8w has zero waits)
- `totalCycle` — sum, must equal measured cycle length

**Key comparison — 16w primary, 36w supplementary:** At 8w, `approachAndMine` = approach + mining (no waits) — directly decomposable. At **16w** (2 w/p, moderate contention), compare waiting vs non-waiting workers' `approachAndMine`. The delta measures approach absorption with modest confounding. At 36w, collision effects (lateral displacement extending approach paths), queue depth, variable approach angles, and SC2's random update ordering all degrade signal-to-noise. 36w data provides supplementary validation for extreme contention behavior but is not the primary measurement source.

**Same-worker cross-cycle analysis (supplementary):** Where the same worker has both waited and non-waited cycles, compare its own `approachAndMine` across those cycles. This controls for worker-specific confounders (approach angle, assigned patch distance) that cross-worker comparison cannot. Requires sufficient per-worker cycle count — achievable at 50 ticks for workers with mixed wait/non-wait cycles.

**Output:** Report with measured constants, mean ± σ, sample size per constant. Minimum N ≥ 30 cycles per constant for statistical significance.

### Phased Execution

**Phase 1 — Zero contention (8w), foundation constants:**
- Approach animation (from `approachAndMine` - known mining duration of 22 loops)
- Mining duration (direct measurement, validates existing 22-loop value)
- Deposit duration (direct measurement, validates existing 12-loop value)
- Full cycle decomposition (every loop accounted for)

**Phase 2 — Contention tiers (16w, 24w, 36w):**
- Wait-to-mine transition time for waiting workers
- Approach duration for waiting vs non-waiting workers (16w primary)
- Walk time growth per saturation level (reveals collision physics)
- `MIN_APPROACH_AFTER_WAIT` measured directly — resolve the code(0)/doc(12) inconsistency with an independently measured value

**Phase 3 — Startup and gas:**
- Initial worker stagger from first-10-tick observation
- Gas cycle decomposition at per-loop resolution (extend existing `GasCycleProfilerTest` data)
- Gas deposit lock investigation (does SC2 have this mechanism?)

**Phase 4 — Emergent constant validation:**

Two separate tests per emergent constant:

1. **Sensitivity test:** Run EM with all measured constants from Phases 1-3. Remove the emergent constant → does accuracy change? This measures whether the constant has any effect with the new physics.

2. **Mechanism test:** Can the effect be explained by measured physics? For `LOW_SATURATION_DODGE_SCALE`: SC2 workers at low saturation don't physically collide (§3 of sc2-physics-constants.md documents this as observed SC2 behavior) — does the collision model, with measured approach values, naturally produce reduced displacement at ≤2 w/p? For `INITIAL_WORKER_STAGGER`: can the SC2 starting worker desynchronisation be replayed from observed spawn positions and initial pathing rather than modeled as a fixed per-worker delay?

If the sensitivity test shows no effect AND the mechanism test confirms the effect emerges from measured physics, remove the constant. If the sensitivity test shows an effect, the mechanism test determines whether to keep the constant (models a real SC2 mechanism not captured elsewhere) or investigate further (compensation for a remaining error in another constant).

For `HIGH_SATURATION_DODGE_SCALE` (currently disabled at 1.0) and `WAIT_ABSORPTION_THRESHOLD` (currently disabled at ∞): run sensitivity tests with candidate values from `ContentionCalibrationDiagnosticTest` sweeps. If a value improves accuracy with the new measured constants, investigate the mechanism before activating.

### Success Criteria

**Measurement precision:** A measured value replaces an aggregate-fit value when:
1. Per-mechanism measurement has N ≥ 30 independent observations
2. Standard deviation σ < 2.0 loops (consistent with SC2 cycle σ=0.58, §1b)
3. The measured value agrees with the aggregate-fit value to within ±2 loops, OR if it disagrees, the EM accuracy improves or holds when the measured value replaces the aggregate-fit value

**Multi-run requirement:** Each scenario must be run at minimum 3 times. Report 3-run mean and σ. For constants with high per-game variance, run 5 times (matching the existing gas protocol, §4/§6).

**Replacement threshold:** A new measured value replaces an existing value when:
- It has stronger derivation evidence (measured > aggregate-fit > default)
- Aggregate EM accuracy does not worsen by more than 1.0% at any calibration checkpoint (8w through 36w, §16 pinned results)
- If accuracy worsens beyond 1.0%, the investigation protocol (§Validation Rule) applies before reverting

### Validation Rule

**Never revert a measured constant to a swept value.** A measured constant is what SC2 does. If aggregate accuracy worsens after replacing a swept value with a measured one, another constant was swept to compensate for the wrong value. Investigate what else is wrong — don't re-sweep.

**Escape clauses:**

1. **Methodology revision:** If a measured value disagrees with the aggregate-fit value and accuracy worsens, the first step is to review the measurement methodology. Did the profiler's heuristic misclassify a phase? Was the measurement window too short? Was the sample size insufficient? Revise the measurement and re-measure — do NOT revert to the swept value.

2. **Mechanism discovery:** If the measured value is correct but accuracy worsens, the gap reveals a missing mechanism. Example: if measured approach is 18 loops but the aggregate-fit was 20, the 2-loop difference may be a real SC2 animation (position settling, path correction) that the profiler classified as "walk" rather than "approach." File a new issue for the mechanism investigation.

3. **Coupled constant identification:** If replacing constant A worsens accuracy, sweep the remaining aggregate-fit constants with A set to its measured value. The constant that changes most is the one that was compensating — it is the next measurement target.

### Constant Registry

Extend `sc2-physics-constants.md`:
- Add **Derivation method** column to the constant tables (Measured / Aggregate-fit / Constrained / Default)
- Do NOT add a new confidence level — derivation method is orthogonal to validation status
- Reclassify constants identified in this audit with their nature and derivation method
- Add new §20 "Measurement Backlog" tracking each aggregate-fit constant's nature, phase assignment, and measurement status

## Scope

**In scope:** Mineral worker physics constants. The 9 constants identified in the audit (7 original + 2 disabled experimental), plus validation of existing "Calibrated" constants against direct mechanism measurements.

**Out of scope:** Combat stats, unit HP, train times, upgrade costs — these use different measurement methods (replay analysis, data editor) and are not part of the worker physics calibration.

## Deferred Issues

The following will be filed as GitHub issues before implementation begins:

1. **Fix `MINERAL_PATCH_OBSTACLE_RADIUS` config default mismatch** — `WorkerPhysicsConfig` defaults to 0.5, `WorkerPhysicsConfigProducer` defaults to 0.3. File independently of calibration work.
2. **Broader constant audit** — Apply the nature/derivation classification to all EM constants (§17 of sc2-physics-constants.md lists 527 total: 176 calibrated, 182 published, 25 estimated, 144 placeholder). Out of scope for this spec but tracked for future work.
3. **Phase deliverables** — One issue per phase (Phases 1-4) for tracking measurement progress and results.
4. **Resolve `MIN_APPROACH_AFTER_WAIT` code/doc inconsistency** — Either update `WorkerSimulation.java:22` from 0 to 12 (matching the doc), or update the doc to reflect the actual code state. This spec's Phase 2 measurement supersedes both values with a directly measured one.

## References

- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfig.java` — all configurable constants
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java` — tryStartMining absorption (line 689), collision model, MIN_APPROACH_AFTER_WAIT (line 22)
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/SimulatedWorker.java` — APPROACH_ANIMATION_LOOPS (line 50), compile-time constants
- `quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/GasCycleProfilerTest.java` — existing per-game-loop gas profiler (pattern for mineral profiler)
- `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ContentionCalibrationDiagnosticTest.java` — existing sweep tests for absorption, dodge, approach
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfigProducer.java` — Quarkus config defaults (MINERAL_PATCH_OBSTACLE_RADIUS mismatch: 0.3 vs 0.5)
- `docs/sc2-physics-constants.md` — existing constant documentation (authoritative reference)
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhaseListener.java` — EM-side phase tracking
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/real/ObservationTranslator.java` — SC2 API data extraction
- CLAUDE.md design rules: "No approximations. Replicate SC2's actual mechanisms."
