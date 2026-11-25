# HANDOFF — quarkmind

## Orientation

**High-saturation contention investigation (#763) — comprehensive sweep complete, root cause identified, partial fix found.**

The pinned single-base test on TorchesAIE shows -6.2% at tick 304 (36w, 4.5 w/p) and +6.5% at tick 150 (24w, 3 w/p). The EM's saturation curve has the wrong shape: per-worker income is too high at moderate saturation and falls off too steeply at high saturation.

---

## Key findings (this session, 2026-11-25)

### 1. Wait-time absorption was a no-op — now fixed (dormant)

`tryStartMining()` line 686: `max(APPROACH_ANIMATION_LOOPS, approachDocking - waited)` where both variables are 20 — always returns 20. Workers that waited 100+ loops still paid full 20-loop approach animation.

**Fix implemented:** Step-function absorption with base-level saturation guard. Workers that waited > threshold AND base saturation > 4.0 get `MIN_APPROACH_AFTER_WAIT` instead of full approach. Defaults disabled (`WAIT_ABSORPTION_THRESHOLD = Double.MAX_VALUE`).

**Best result:** threshold=22, minApproach=5 on TorchesAIE: tick 304 at **-0.3%** (was -6.2%). Tick 150 at +7.6% (similar to current +6.5%).

### 2. No single parameter fixes both low-sat and high-sat

Three diagnostic sweeps explored 70+ parameter combinations:
- **Wait absorption threshold** (11 values): increases income uniformly
- **High-saturation dodge scale** (9 values): increases income uniformly
- **Approach animation × threshold** (15 combinations): shifts crossover point but can't fix shape
- **Step absorption threshold × minApproach** (20 combinations): step function + saturation guard works on real terrain

All mechanisms increase income at ALL saturation levels. None selectively helps high-sat without also worsening moderate-sat.

### 3. The fix is terrain-dependent

The step absorption works on real TorchesAIE terrain (pinned test: -0.3%) but causes +16-23% over-production on the flat grid (MULE test, SC2PlaybookCalibrationTest). The flat grid has uniform 7.0u patches with different A* paths — different contention patterns than real terrain with 3-6u variable-distance patches.

The dual-run baselines use the `calibrate()` method which DOES load real terrain when baselines have `basePosition`. But `SC2PlaybookCalibrationTest.muleIsolation()` creates a plain `EmulatedGame()` — no terrain, no mineral patches. This runs on the default flat grid.

### 4. Root cause: saturation curve shape problem

| Tick | Probes | SC2 cum inc | EM cum inc | Δ cum inc |
|------|--------|------------|-----------|-----------|
| 100 | 20 | 1160 | 1395 | +20% |
| 150 | 24 | 2125 | 2215 | +4.2% |
| 200 | 28 | 3060 | 3070 | +0.3% |
| 250 | 32 | 4115 | 3930 | -4.5% |
| 304 | 36 | 5145 | 4890 | -5.0% |

Crossover at tick 200 (28w, 3.5 w/p). Before: EM over-produces (straight-line walks shorter than SC2 curves). After: EM under-produces (collision dodge over-penalizes).

---

## Changes made (this session)

1. **WAIT_ABSORPTION_THRESHOLD** (`WorkerSimulation.java:20`): Configurable threshold for step-function absorption. Default `Double.MAX_VALUE` (disabled).

2. **MIN_APPROACH_AFTER_WAIT** (`WorkerSimulation.java:22`): Minimum approach loops for workers that passed the absorption threshold. Default 0.

3. **HIGH_SATURATION_DODGE_SCALE** (`WorkerSimulation.java:21`): Configurable dodge scale for >2 w/p. Default 1.0 (no change). Guard: `HIGH_SATURATION_DODGE_SCALE < 1.0f`.

4. **Step-function tryStartMining** (`WorkerSimulation.java:686-695`): Replaced linear absorption with step function. Guard: `waited > threshold AND baseWorkers/basePatches > 4.0`.

5. **workerStateCounts()** (`WorkerSimulation.java:877-901`): Diagnostic method counting workers in each state (mining/walking/waiting/returning/depositing/idle).

6. **Diagnostic sweeps** (`ContentionCalibrationDiagnosticTest.java`): Four sweep tests:
   - `sweepWaitAbsorptionThreshold` — threshold sweep (pinned test)
   - `sweepDodgeScaleAtHighSaturation` — dodge scale sweep
   - `sweepApproachAnimationWithAbsorption` — approach × threshold
   - `sweepStepAbsorptionThresholdAndMinApproach` — threshold × minApproach (step function)
   - `sweepCombinedAbsorptionAndDodge` — threshold × dodge 2D sweep

---

## Approaches tried and rejected

1. **Linear wait-time absorption** (threshold-based): `max(0, approach - (waited - threshold))`. Same terrain dependency as step function. Non-monotonic results suggest cycle alignment effects.

2. **Uniform dodge scale reduction** (0.10-1.0): Reduces income uniformly across all saturation levels. Can't differentially fix high-sat.

3. **Approach animation increase** (20→28): Reduces income uniformly. approach=22 + threshold=10 gives tick 304 at +1.2% but tick 150 at +7.6%.

4. **Contention guard on patch level** (`contention > 2`, `>3`, `>4`): Patch-level contention varies with rerouting. Even at 3 avg w/p, some patches have 5+ assigned workers, bypassing any per-patch guard.

---

## Next steps (priority order)

1. **Migrate MULE playbook test to real terrain** — `SC2PlaybookCalibrationTest.muleIsolation()` uses a plain `EmulatedGame()` without terrain. Loading the TorchesAIE pathing grid and mineral positions would make it consistent with the dual-run baselines and enable the absorption fix. This is the blocking issue for enabling absorption.

2. **Record SC2 per-worker cadence at 36w** — the absorption fix gets pinned to -0.3% but doesn't fix the shape problem. SC2 per-worker instrumentation at 24w and 36w would reveal the actual mechanism that makes high-sat workers more efficient than the EM models.

3. **Investigate the unexplained MULE regression** — setting `WAIT_ABSORPTION_THRESHOLD = 22.0` alone (guard blocks, minApproach=0) causes 23.6% over-production in the MULE test. The step function code path itself changes timing even when the guard should block. This may be a compiler optimization artifact or a subtle interaction.

---

## Notes

- **Branch:** `issue-761-intent-rejection-diagnostic` (stacked: 756→757→758→761→762→763→764→766)
- **All 222 tests pass** (with defaults disabled) — DualRunCalibrationTest (46), SC2PlaybookCalibrationTest (17), WorkerSimulationTest (47), EmulatedGameTest (128)
- **SC2 is NOT running** — killed after recordings in previous session
- **Issue #766 open** — tracks the A* tile-center snap fix and obstacle radius changes
- **Issue #763 open** — tracks this high-saturation contention investigation
- **IntelliJ MCP:** Crashed mid-session. Restarted but quarkmind project may need re-opening (`ide_open_project`).
