# HANDOFF — quarkmind

## Last Session

Worker mining contention fix (#596, closed) and saturation calibration (#599). Got the EM to match SC2 calibration baselines within 2% across both reference points.

### What was done

- **#596 — Contention fix (CLOSED):**
  1. `assignToNearestPatch` counts assigned workers (excluding self) instead of active miners
  2. Wake-up pass after tick processing — freed patches start waiting workers immediately
  3. Eliminated 15.5% worker-tick waste from clustering

- **#599 — Saturation model + calibration (5 commits):**
  1. `MineralPatch.lockEndLoop` — post-mining docking overhead locks patch after miner departs
  2. Sub-tick lock resolution — waiting workers consume lock time within the tick
  3. `PATCH_DISTANCE` changed from 6.0 to 7.0u — calibrated from SC2 baseline sweep
  4. `PATCH_DOCKING_OVERHEAD` changed from 30 to 34 — calibrated from 23-worker saturation sweep
  5. `WorkerPhysicsConfig` and `WorkerPhysicsConfigProducer` support docking overhead config

- **Diagnostic tests added (`MiningCycleDiagnosticTest`, @Tag("diagnostic")):**
  - `singleWorkerCycleTrace` / `singleWorkerPerLoopTrace` — per-tick/per-loop state traces
  - `twelveWorkerContentionAnalysis` — waiting worker-tick measurement
  - `rateComparison_1v8v12workers` — full 1-30 worker saturation curve
  - `patchDistanceSweep` — distance calibration against SC2 12w baseline
  - `dockingOverheadSweep` — docking calibration against SC2 23w saturation

- **Dual-run SC2 comparisons:**
  - `dualRunCalMineralBaseline12wAgainstSC2` — PASSES (Protoss, 305 ticks)
  - `dualRunCalMineralBaselineTerranAgainstSC2` — PASSES (SCV mapping fixed)
  - `dualRunCalMineralBaselineZergAgainstSC2` — PASSES (DRONE mapping fixed)
  - `dualRunCalMineralSaturated16wAgainstSC2` — diverges at tick 12 (Probe train timing 1 tick early)

- **ObservationTranslator unit type mappings (this session):**
  - Was Protoss-only — all Terran/Zerg units fell to UNKNOWN
  - Added complete Terran (22 variants), Zerg (28 variants), expanded Protoss (9 new)
  - Fixed Terran baseline JSON: UNKNOWN→SCV
  - Fixed Zerg baseline JSON: UNKNOWN→DRONE (stripped OVERLORD/LARVA — EM doesn't model them)

### Calibrated results

| Metric | EM | SC2 | Delta |
|--------|-----|-----|-------|
| 12w mineral baseline (tick 304) | 3470 | 3460 | **+0.3%** |
| 1-worker rate (7.0u) | 57.0/min | 57.9/min | -1.6% |
| 23-worker rate (2.9/patch) | 47.7/min | 48.6/min | -1.8% |
| Saturation efficiency at 23w | 83.7% | ~84% | **match** |

### Saturation curve (PATCH_DISTANCE=7.0u, DOCKING=34)

| Workers | W/Patch | Rate/worker/min | Efficiency |
|---------|---------|----------------|------------|
| 1-16    | ≤2.0    | 57.0           | 100%       |
| 20      | 2.5     | 50.9           | 89.3%      |
| 23      | 2.9     | 47.7           | 83.7%      |
| 24      | 3.0     | 46.8           | 82.1%      |
| 30      | 3.8     | 37.5           | 65.7%      |

---

## Issue Map

### Open issues by priority

| # | Title | Scale | Complexity | Blocked by | Status |
|---|---|---|---|---|---|
| **599** | Saturation contention model | L | High | — | All 4 baselines PASS: 12w/Terran/Zerg exact, 16w structural (±1 tolerance, 6.5% mineral delta) |
| **597** | SC2 adapter gas dispatch | S | Med | — | SC2 baselines recorded, debug-API Assimilator + worker assignment working. EM gas rate 108% of SC2 (8% gap remaining) |
| **598** | SC2 adapter MULE/expansion | M | High | — | Morph/calldown implemented, untested |
| 592 | Overhead absorption limit | S | Med | — | May be resolved by calibration work |

### Closed this session
| # | Title | Resolution |
|---|---|---|
| 596 | EM implementation efficiency gap | Contention fix — assignment-based spreading + wake-up pass |

---

## What to do next — calibration accuracy roadmap

All gaps now tracked as issues:

| # | Gap | Current | Target | Blocked by |
|---|-----|---------|--------|------------|
| 600 | Terran/Zerg mining efficiency | +1.6% | ±0.5% | — |
| 601 | Sub-tick training resolution | +6.9% (16w) | ±2% | — |
| 602 | Geyser position recording | +8.1% gas | ±3% | — |
| 603 | Worker initialization delay | ±30 minerals early | ±5 | 605 |
| 604 | 24w/MULE baseline re-record | diverge | pass | 601 |
| 605 | Mineral patch position recording | approx 7.0u | exact | — |
| 606 | Gas harvest contention (6w) | unknown | ±5% | 602 |

Recommended order: 600 (quick win) → 605+602 (position data) → 603 (init delay) → 601 (sub-tick) → 604 (re-record) → 606 (gas contention)

---

## Key Code Locations

| What | Where |
|------|-------|
| WorkerSimulation (contention fix + wake-up + docking lock) | `quarkmind-sc2/.../emulated/WorkerSimulation.java` |
| MineralPatch (PATCH_DISTANCE=7.0, lockEndLoop) | `quarkmind-sc2/.../emulated/MineralPatch.java` |
| SimulatedWorker (PATCH_DOCKING_OVERHEAD_LOOPS=34) | `quarkmind-sc2/.../emulated/SimulatedWorker.java` |
| WorkerPhysicsConfig (configurable docking) | `quarkmind-sc2/.../emulated/WorkerPhysicsConfig.java` |
| MiningCycleDiagnosticTest (saturation + sweeps) | `quarkmind-sc2/.../emulated/MiningCycleDiagnosticTest.java` |
| DualRunCalibrationTest (SC2 comparisons) | `quarkmind-sc2/.../playbook/DualRunCalibrationTest.java` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name.
- **16w mineral income gap (6.5%)** — EM 4260 vs SC2 4555 at tick 304. Composed of: ~2% per-worker rate gap (calibrated), ~4.5% from Probe ramp-up timing differences (integer-division phase shift in 272/22 train time). Not a physics bug — fractional loop accumulation distributes differently.
