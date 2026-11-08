# Worker Movement Acceleration Model — Design Spec

**Issue:** #574 (child of Epic #534: EmulatedGame 100% accuracy)
**Branch:** `issue-572-worker-activity-modelling`
**Date:** 2026-11-08

## Problem

Worker movement uses constant-speed linear pathing. Real SC2 workers accelerate from rest to max speed and decelerate when approaching a target. At the 6-unit patch-to-base distance, workers may never reach full speed — the trip is dominated by acceleration and deceleration phases.

This is the structural floor for economy accuracy: 10% gas harvest deficit (#571), 15.6% mineralsCollectionRate MAPE (#567).

## Published SC2 Constants

| Constant | Value | Source |
|---|---|---|
| Worker max speed | 3.94 units/game-second | Liquipedia |
| Worker acceleration | 2.5 units/sec² | SC2Mapster data editor wiki |
| Worker deceleration | 2.5 units/sec² | SC2Mapster (defaults to acceleration) |
| Game loops per second | 22.4 | Liquipedia |
| Loops per tick | 22 (SC2Data.LOOPS_PER_TICK) | Empirical |

All three worker types (SCV, Probe, Drone) share the same acceleration.

## Design

### Per-Loop Movement Simulation

Replace constant-speed `moveToward(distancePerTick)` with a per-loop physics simulation inside `moveToward`. Each call runs 22 sub-steps (one per game loop within the tick):

```
for each loop in tick:
    distToTarget = position.distanceTo(targetPosition)
    decelDist = currentSpeed² / (2 * DECELERATION)
    
    if distToTarget <= decelDist:
        // Decelerate
        currentSpeed = max(0, currentSpeed - DECELERATION * LOOP_DURATION)
    else if currentSpeed < MAX_SPEED:
        // Accelerate
        currentSpeed = min(MAX_SPEED, currentSpeed + ACCELERATION * LOOP_DURATION)
    
    stepDist = currentSpeed * LOOP_DURATION
    move position toward targetPosition by stepDist
    
    if arrived:
        currentSpeed = 0
        break
```

Where `LOOP_DURATION = 1.0 / GAME_LOOPS_PER_SECOND` (the real-time duration of one game loop).

### SimulatedWorker Changes

Add one field:
- `double currentSpeed` — current movement speed in units/game-second. Starts at 0, updated each loop.

Add constants:
- `static final double ACCELERATION = 2.5` — units/sec²
- `static final double DECELERATION = 2.5` — units/sec²
- `static final double LOOP_DURATION = 1.0 / SC2Data.GAME_LOOPS_PER_SECOND` — seconds per game loop

### Method Signature Change

`moveToward(double distancePerTick)` → `moveToward()`

The `distancePerTick` parameter is no longer needed — the method computes distance per loop from the acceleration model. Callers that currently pass `workerDistancePerTick` just call `moveToward()` instead.

The `workerDistancePerTick` field in `WorkerSimulation` can be removed.

### Where currentSpeed resets to 0

Workers start from rest (speed = 0) when:
- Beginning a mining trip (after `assignToNearestPatch` sets target to patch position)
- Beginning a return trip (after mining completes, target set to base position)
- Starting a transfer between bases (task set to TRANSFERRING)
- Returning from scouting (via `onWorkerReturnFromScout`)

The `currentSpeed = 0` happens naturally: when the worker arrives at a target (`isAtTarget()` triggers), `currentSpeed` is set to 0. When the worker is then assigned a new target, it starts from 0 and accelerates.

### What doesn't change

- `WorkerTask` enum — no new states
- `WorkerSimulation.tick()` — still calls `moveToward` per worker; the sub-loop is internal to `moveToward`
- Mining duration — still `MINING_DURATION_LOOPS = 44.576`
- Gas harvest — still `GAS_HARVEST_DURATION_LOOPS = 31.696`
- Patch distribution — still `assignToNearestPatch`

## Testing Strategy

**Unit tests (plain JUnit):**

1. `workerAcceleration_reachesMaxSpeedOverDistance` — a worker moving over a long distance (20+ units) should reach max speed
2. `workerAcceleration_neverReachesMaxSpeed_shortDistance` — at 6 units, the worker may never reach 3.94 u/s (triangular profile)
3. `workerAcceleration_deceleratesNearTarget` — worker speed should approach 0 as it nears the target
4. `workerAcceleration_tripTakesLongerThanConstantSpeed` — a mining cycle should take longer with acceleration than with constant speed
5. `saturationCurve_thirdWorkerEarnsLessThanSecond` — re-run the saturation curve test; the acceleration model should produce steeper diminishing returns (longer trips = more time waiting at saturated patches)
6. `mineralIncomeRate_16workers_matchesSC2` — per-worker income should be closer to SC2's published ~40 min/min/worker

**Validation:**
- Run full WorkerSimulationTest suite
- Run playbook calibration tests (DualRunCalibrationTest, SC2PlaybookCalibrationTest)

## Implementation Order

1. Add `currentSpeed`, `ACCELERATION`, `DECELERATION`, `LOOP_DURATION` to SimulatedWorker
2. Rewrite `moveToward` with per-loop sub-stepping
3. Update callers to remove `distancePerTick` parameter
4. Remove `workerDistancePerTick` from WorkerSimulation
5. Write acceleration-specific unit tests
6. Run full regression suite

## Out of Scope

- Close vs far patch distances (patch geometry is separate from movement physics)
- Speed mining tricks (SC2 bot optimisation, not standard physics)
- Worker collision/stacking (priority pathing)
- Non-worker unit movement (only workers have acceleration < 1000)

## References

- SC2Mapster wiki (Data/Units) — worker acceleration 2.5, deceleration defaults to acceleration
- Liquipedia Speed page — worker speed 3.94, acceleration is defined for workers
- VersusAI speed mining guide — 10-12% income boost from speed mining, confirms deceleration matters
- SimulatedWorker.java:49-63 — current moveToward implementation
- WorkerSimulation.java:23-26 — workerDistancePerTick calculation
- Issue #574, #571, #567 — evidence for the structural floor
- Memory: feedback-simulation-over-formulas — formulas don't work, simulate instead
