# EM Base Cycle Speed — Measure-Then-Fix Design

**Issue:** #661 — EM base cycle speed: edge-to-edge walks 25% faster than SC2 paths
**Branch:** issue-661-em-base-cycle-speed
**Parent:** #617 (Foundational tick-alignment audit)

## Problem

EM workers complete mining cycles ~16% faster than SC2 workers at 8w (uncontested). The edge-to-edge walk model gives ~30 loops per leg, while SC2 workers take ~33 loops per leg. This 3-loop/leg gap compounds across both walk legs to produce +16% per-worker income.

| Metric | EM (current) | SC2 (target) | Gap |
|--------|-------------|-------------|-----|
| 8w per-worker rate | 70.8 min/worker | ~61 min/worker | +16% |
| Walk per leg | ~30 loops | ~33 loops | −3 loops |
| Full cycle | ~100.5 loops | ~106 loops | −5.5 loops |

The existing unconditional approach docking (11.5 loops, #655) partially compensates but doesn't model the actual cause.

## Root Cause Analysis

Five independent physical differences contribute to the walk time gap. All are measurable from SC2 per-tick worker position data.

| # | Physical cause | Expected contribution | Measurable from |
|---|---------------|----------------------|-----------------|
| 1 | **Mining target position** — MINERAL_PATCH_RADIUS=1.0 assumes circular patches; SC2 patches are rectangular (~2×1 tiles), docking point may be closer to center | +0-3 loops/leg | Worker position during mining phase |
| 2 | **Deposit target position** — TOWN_HALL_RADIUS=2.75 is the collision radius; actual drop-off point may differ | +0-1.5 loops/leg | Worker position at cargo flip |
| 3 | **Path deviation** — EM walks straight lines; SC2 pathfinds around mineral patch obstacles, adding distance for edge patches | +0-2 loops/leg (per-patch) | Sum of position deltas vs straight-line |
| 4 | **Acceleration profile** — EM uses constant 3.5/3.5 u/s²; SC2 engine may have different curve shape | +0-1 loops/leg | Position delta patterns over time |
| 5 | **Arrival/departure mechanics** — EM snaps at 0.5u threshold; SC2 has turn-to-face and animation transitions | Absorbed into calibration | Indirectly |

## Approach: Measure-Then-Fix

**Phase 1 — Diagnostic:** Record an 8w SC2 baseline with per-tick worker cadence data. 8 workers on 8 patches gives zero-contention cycles — every walk and every mining stop is clean.

**Phase 2 — Measurement:** From the cadence data, extract per-worker per-cycle:
- Mining position (stable position at patch) → effective mining radius
- Deposit position (position at cargo flip) → effective deposit distance
- Walk distance (sum of per-tick position deltas) → actual path length
- Path deviation factor (actual distance / straight-line distance) → per-patch deviation

**Phase 3 — Fix:** Apply fixes proportional to measured contributions:
- If mining radius ≠ 1.0 → adjust MINERAL_PATCH_RADIUS
- If deposit distance ≠ 2.75 → adjust deposit target or TOWN_HALL_RADIUS
- If path > straight line → add per-patch walk distance adjustment based on mineral line geometry
- Likely: combination of smaller MINERAL_PATCH_RADIUS + modest path deviation

## Phase 1: 8w SC2 Baseline Recording

### Playbook: `cal-mineral-baseline-8w.yaml`

```yaml
playbook: cal-mineral-baseline-8w
schema: sc2
meta:
  race: PROTOSS
  duration: 305 ticks
  purpose: Uncontested per-worker mineral rate — 8 workers (1 per patch), no gas, no training

steps:
  - kill: 4 workers
    at: 0 ticks

  - assert: tick-61
    at: 61 ticks
    expect:
      units:
        PROBE: { min: 8, max: 8 }
```

**Kill action:** The PlaybookSC2Adapter uses `DebugKillUnit` to remove 4 workers at game start. This is already used in `MiningAnimationMeasurementTest` and `PlaybookSC2Adapter.translateMorphDebug`. The kill selects 4 workers by tag from the first observation.

The EM delivery handler needs a `doKill` action that calls `EmulatedGame.debugKillUnit(tag)` for the EM self-baseline test.

### Recorder Test

Add `recordCalMineralBaseline8w()` to `PlaybookSC2RecorderTest` following the `recordCalWorkerCadence12w` pattern — records both `CalibrationBaseline` and `WorkerCadenceBaseline` from the same game run.

### Dual-Run Test

Add `dualRunCalMineralBaseline8wAgainstSC2()` to `DualRunCalibrationTest`. The EM side runs the same playbook (killing 4 workers at tick 0).

## Phase 2: Walk Decomposition Diagnostic

### CycleDecompositionTest (diagnostic tag)

A new diagnostic test that loads the 8w worker cadence baseline and decomposes each worker's cycles:

1. **Identify cycle phases** per worker from per-tick position + action data:
   - MINING: position stable near a patch (action=GATHERING_MINERALS, not carrying)
   - RETURNING: moving toward base (action=RETURNING_MINERALS, carrying)
   - DEPOSITING: position stable near base (carrying → not carrying transition)
   - WALKING_TO_PATCH: moving toward a patch (action=GATHERING_MINERALS, not carrying, not at patch)

2. **Measure per phase:**
   - Mining position: average position during MINING phase → distance to patch center = effective mining radius
   - Deposit position: position at cargo flip tick → distance to base center = effective deposit distance
   - Walk distance to patch: sum of `distance(pos[t], pos[t+1])` during WALKING_TO_PATCH
   - Walk distance to base: sum of `distance(pos[t], pos[t+1])` during RETURNING
   - Straight-line distance: `distance(depositTarget, miningTarget)` using current radii

3. **Report per patch:**
   ```
   Patch | c2c dist | e2e dist | SC2 walk | EM walk | SC2 mine pos | SC2 deposit pos | deviation
   ```

4. **Aggregate:**
   - Mean effective mining radius across all workers/cycles
   - Mean effective deposit distance
   - Mean path deviation factor (SC2 walk / straight-line)
   - Breakdown: how many loops come from each cause

### Expected output

The diagnostic tells us exactly what to fix:
- If effective mining radius ≈ 0.5-0.7 → MINERAL_PATCH_RADIUS is too large
- If effective deposit distance ≈ 2.5 → TOWN_HALL_RADIUS is too large for deposits
- If deviation factor > 1.0 → path deviation contributes, need per-patch walk adjustment
- Quantifies each contribution so we fix proportionally, not by guessing

## Phase 3: Fix Application

Design deferred — depends on Phase 2 measurements. Candidate fixes:

**If MINERAL_PATCH_RADIUS is wrong:** Change the constant from 1.0 to measured value. Protocol PP-20261116-5f16ed requires matching CDI default. Verify 12w/16w calibration still holds (the longer walks affect all saturation levels).

**If path deviation is significant:** Add `effectiveWalkDistance` to `MineralPatch`, computed in `computeApproachDocking()` from mineral line geometry. Workers walk the effective distance rather than the straight-line distance. This is per-patch, not a uniform constant.

**If deposit distance is wrong:** Adjust deposit target calculation. Currently `edgePoint(basePos, patchPos, TOWN_HALL_RADIUS)` — may need a `TOWN_HALL_DEPOSIT_RADIUS` that differs from the collision radius.

**Constraint:** No compensation constants (CLAUDE.md §Current Priority). Every fix must correspond to a measured physical value from SC2 observation data.

## Acceptance Criteria

- [ ] 8w SC2 baseline recorded (CalibrationBaseline + WorkerCadenceBaseline)
- [ ] Walk decomposition diagnostic identifies root cause contributions
- [ ] Fix applied — 8w per-worker rate within 5% of SC2 baseline
- [ ] 12w dual-run gap stays within current tolerance (+2.7% or better)
- [ ] 16w dual-run gap stays within current tolerance (−1.2% or better)
- [ ] sc2-physics-constants.md §3 and §14 updated with new measurements
- [ ] sc2-physics-constants.md §15 "Worker pathing" impact assessment updated

## Test Plan

- `PlaybookSC2RecorderTest#recordCalMineralBaseline8w` — records 8w baseline from SC2
- `DualRunCalibrationTest#dualRunCalMineralBaseline8wAgainstSC2` — validates 8w EM vs SC2
- `DualRunCalibrationTest#selfBaselineCalMineralBaseline8w` — self-baseline consistency
- `CycleDecompositionTest` (diagnostic) — walk decomposition from cadence data
- Existing 12w, 16w, 24w, 3w gas, 6w gas dual-run tests — regression guard
- `WorkerSimulationTest#mineralIncomeRate_8workers_uncontested` — unit test rate check

## References

- WorkerSimulation.java:16-55 — edge-to-edge model, TOWN_HALL_RADIUS, MINERAL_PATCH_RADIUS
- MineralPatch.java:59-64 — computeApproachDocking
- SimulatedWorker.java:44 — MAX_SPEED, ARRIVAL_THRESHOLD
- sc2-physics-constants.md §3 — mineral mining constants and SC2 observed cycle
- sc2-physics-constants.md §14 — mining cycle breakdown
- sc2-physics-constants.md §15 — "Worker pathing around mineral line" (impact underestimated)
- PlaybookSC2Adapter.java:366 — DebugKillUnit usage
- MiningAnimationMeasurementTest.java:275 — DebugKillUnit usage
- PP-20261116-5f16ed — CDI defaults must match constants
- Memory: feedback-simulation-over-formulas — simulate per-loop, never formulas
