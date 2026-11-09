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

- **Dual-run SC2 comparisons added:**
  - `dualRunCalMineralBaselineTerranAgainstSC2` — diverges at tick 0 (UNKNOWN mapping)
  - `dualRunCalMineralBaselineZergAgainstSC2` — diverges at tick 0 (UNKNOWN mapping)

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
| **599** | Saturation contention model | L | High | — | Calibrated — docking=34 at 7.0u matches SC2 within 2% |
| **597** | SC2 adapter gas dispatch | S | Med | — | Fix committed, untested — needs SC2 running |
| **598** | SC2 adapter MULE/expansion | M | High | — | Morph/calldown implemented, untested |
| 592 | Overhead absorption limit | S | Med | — | May be resolved by calibration work |

### Closed this session
| # | Title | Resolution |
|---|---|---|
| 596 | EM implementation efficiency gap | Contention fix — assignment-based spreading + wake-up pass |

---

## What to do next

### Priority 1 — Validate against remaining SC2 baselines

The 16w saturation baseline (`cal-mineral-saturated-16w`) records 23 workers at SC2 patch distances. A dual-run comparison would validate the saturation model against real SC2 data. Currently only the 12w Protoss baseline is compared (Terran/Zerg blocked by UNKNOWN unit type mapping in ObservationTranslator).

### Priority 2 — Fix Terran/Zerg unit type mapping

`ObservationTranslator` maps SCVs and Drones as UNKNOWN. Fix this to enable Terran/Zerg SC2 baseline comparisons. Both baselines show 3415 minerals at tick 304 (Protoss: 3460).

### Priority 3 — #597: Gas calibration

1. Verify Assimilator dispatch fix works (run `recordCalGasBaseline3w` with SC2)
2. If vespene=0, add explicit worker-to-gas assignment
3. Calibrate gas-harvest-duration-loops against SC2 gas baseline

### Priority 4 — Record more SC2 baselines

- 24w saturation baseline (3/patch) — validate steep 3rd-worker penalty
- Re-record gas baselines with fixed adapter
- Record patch positions in baseline JSON for exact distance matching

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

- **Terran/Zerg UNKNOWN mapping** — `ObservationTranslator` doesn't map SCV/Drone unit types. Blocks Terran/Zerg SC2 baseline comparisons.
- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name.
