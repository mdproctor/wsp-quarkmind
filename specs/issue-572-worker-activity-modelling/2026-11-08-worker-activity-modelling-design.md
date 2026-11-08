# Worker Activity Modelling — Design Spec

**Issue:** #572 (child of Epic #534: EmulatedGame 100% accuracy)
**Branch:** `issue-572-worker-activity-modelling`
**Date:** 2026-11-08

## Problem

In standalone mode (no GT syncs), all workers mine continuously. Real SC2 players have ~3 workers per player doing non-mining activities (scouting, transit between bases, idle between tasks). Per-worker mineral income is 5-10% above GT across all races. The simulation's mining physics are correct per-worker-per-trip; the gap is in worker ALLOCATION — the simulation doesn't model activities that take workers off minerals, nor does it model the diminishing returns of oversaturating patches beyond 2 workers.

Previous attempt (#566) applied per-race efficiency scalars (Protoss 0.92, Zerg 0.95, Terran 1.0). Reverted — that was a formula, not physics. This design models actual worker activities and validates the saturation curve.

## Scope

Two orthogonal changes to WorkerSimulation:

1. **Saturation curve validation** — verify the existing `MAX_MINERS=2` contention model produces the correct income curve at different saturation levels (12/16/20/24 workers per base), using published SC2 community data as the reference.

2. **Worker activity lifecycle** — add `SCOUTING` and `TRANSFERRING` states to `WorkerTask`, with event hooks that work in both replay and real-game modes. Replace the current instant base-transfer with physical walk time.

## Out of scope

- Mining duration calibration (#575)
- Worker acceleration/deceleration model (#574)
- Gas movement model (#577)
- Spending timing model (#576)
- Strategy-layer scout decision logic (when to scout — #572 provides the hooks, not the decisions)

## Reference Data — SC2 Mining Saturation

Published community data (GameReplays, TL.net, Liquipedia):

| Workers/patch | Income/worker/min | Efficiency | Notes |
|---|---|---|---|
| 1 | ~40 | 100% | Baseline |
| 2 | ~40 | ~100% | Workers alternate trips — near-zero wait time |
| 3 | ~20-22 | ~50-55% | 3rd worker waits while 2 alternate |
| 4+ | negligible | ~0% | Complete waste |

**Per-base totals (8 patches, standard minerals):**

| Workers | Total income/min | Per-worker avg |
|---|---|---|
| 16 (2/patch) | ~660 | ~41 |
| 20 | ~690-770 | ~36 |
| 24 (3/patch) | ~816 | ~34 |

**Key insights:**
- 2 workers per patch is optimal per-worker efficiency — they perfectly alternate mining trips
- 3rd worker drops to ~50% efficiency due to wait contention at the patch
- Close patches (nearer town hall) have shorter round trips, so 2 workers may not perfectly alternate — the 3rd worker adds even less value on close patches
- Good players fill 2/patch across all patches before adding 3rds

**Gas saturation (for reference, not primary scope):**
- Gas workers spend 31.696 loops inside the refinery — long harvest time
- 3 workers per geyser scales near-linearly (less contention than minerals)
- Current `GasSlot.MAX_WORKERS=3` with `canEnter()` blocking is likely correct

## Design

### Part 1: Saturation Curve Validation

**What:** A diagnostic test (`WorkerSaturationCurveTest`) that measures the simulation's income at different worker counts per base and compares against published SC2 curves.

**Test setup:**
- Single base, 8 mineral patches, no gas
- Run for 5 game-minutes (6720 loops at Faster speed)
- Measure total mineral income at worker counts: 8, 12, 16, 20, 24
- Compute per-worker income rate
- Compare against published reference: 2/patch workers should earn ~2x what 3/patch workers earn

**Expected outcome:** The current `MAX_MINERS=2` + `tryStartMining()` wait model should already produce diminishing returns. The test verifies the SHAPE of the curve matches SC2. If it doesn't, the fix is in the mining loop physics — adjusting `MAX_MINERS`, mining slot timing, or patch queue behavior.

**If the curve is wrong:** Create a playbook that adds workers one at a time to a single base in real SC2, recording income after each addition. Use `PlaybookSC2RecorderTest` to generate baseline JSON, then `DualRunCalibrationTest` to compare.

### Part 2: Worker Activity States

**New `WorkerTask` values:**

```java
enum WorkerTask {
    MINING_MINERALS,
    RETURNING_MINERALS,
    HARVESTING_GAS,
    RETURNING_GAS,
    IDLE,
    BUILDING,
    SCOUTING,       // new — worker is away from base scouting
    TRANSFERRING    // new — worker is walking between bases
}
```

### Part 3: Scouting Lifecycle

**Event hooks on WorkerSimulation:**

```java
void onWorkerScouting(String tag, double returnAtLoop)
void onWorkerReturnFromScout(String tag)
```

**`onWorkerScouting(tag, returnAtLoop)`:**
1. Find the worker by tag
2. Remove from assigned patch (`assignedPatch.miners.remove`, clear `assignedPatch`)
3. Set task to `SCOUTING`
4. Record `returnAtLoop` on the worker (new field on `SimulatedWorker`)
5. Worker produces no income while scouting

**`onWorkerReturnFromScout(tag)`:**
1. Find the worker by tag
2. Set task to `MINING_MINERALS`
3. Call `assignToNearestPatch(w)` to resume mining

**Tick integration:**
In `tick()`, check for scouts whose `returnAtLoop` has passed — automatically call `onWorkerReturnFromScout`. This handles the case where the caller doesn't explicitly return the worker.

**Replay mode integration:**
The replay validation harness detects workers far from any base in GT unit positions. When a worker's GT position is > X distance from all known base positions, call `onWorkerScouting`. When it returns to base proximity, call `onWorkerReturnFromScout`. Distance threshold TBD — calibrate from replay data.

**Real-game/playbook mode integration:**
The caller (strategy layer, playbook runner) invokes `onWorkerScouting` at appropriate game moments. #572 provides the hooks; the caller logic is out of scope.

### Part 4: Transit Modelling

**Change `transferMineralWorkers` to use physical walk time:**

Current behavior: instant reassignment (`w.baseIndex = toBase; assignToNearestPatch(w)`).

New behavior:
1. Remove worker from current patch
2. Set task to `TRANSFERRING`
3. Set `targetPosition` to new base position
4. Set `w.baseIndex` to new base index
5. Worker walks to new base using existing `moveToward(workerDistancePerTick)`

**New tick handler — `tickTransferringWorker`:**

```java
private void tickTransferringWorker(SimulatedWorker w) {
    w.moveToward(workerDistancePerTick);
    if (w.isAtTarget()) {
        assignToNearestPatch(w);
    }
}
```

Called from `tick()` for workers with `task == TRANSFERRING`.

**Impact:** Workers transferring between bases produce no income during transit. Transit time is distance / worker speed — typically 5-15 seconds (112-336 loops) depending on map geometry. With 4 workers transferred per expansion (`MAX_TRANSFER_PER_EXPANSION`), this creates a temporary income dip matching real-game behavior.

### Part 5: SimulatedWorker Changes

Add one field:

```java
double scoutReturnLoop;  // game loop at which a scouting worker auto-returns
```

No other changes to SimulatedWorker — the existing `task`, `targetPosition`, `assignedPatch`, and `moveToward` infrastructure handles TRANSFERRING naturally.

### Part 6: Tick Loop Changes

Update `WorkerSimulation.tick()` to handle new states:

```java
TickIncome tick(double tickStartLoop) {
    this.currentTickStartLoop = tickStartLoop;
    double tickEndLoop = tickStartLoop + SC2Data.LOOPS_PER_TICK;

    // Auto-return scouts
    for (SimulatedWorker w : workers.values()) {
        if (w.task == WorkerTask.SCOUTING && tickEndLoop >= w.scoutReturnLoop) {
            onWorkerReturnFromScout(w.tag);
        }
    }

    double mineralIncome = 0;
    for (SimulatedWorker w : workers.values()) {
        if (w.task == WorkerTask.TRANSFERRING) {
            tickTransferringWorker(w);
        } else if (w.assignedGasSlot == null) {
            mineralIncome += tickMineralWorker(w, tickStartLoop, tickEndLoop);
        }
    }

    double gasIncome = 0;
    for (SimulatedWorker w : workers.values()) {
        if (w.assignedGasSlot != null) {
            gasIncome += tickGasWorker(w, tickStartLoop, tickEndLoop);
        }
    }

    return new TickIncome(mineralIncome, gasIncome);
}
```

SCOUTING and TRANSFERRING workers must be excluded from income. Update the guard in `tickMineralWorker` (line 233):

```java
if (w.task == WorkerTask.BUILDING || w.task == WorkerTask.IDLE
        || w.task == WorkerTask.SCOUTING || w.task == WorkerTask.TRANSFERRING) return 0;
```

TRANSFERRING workers are handled separately in the tick loop (before the mineral income loop) so they don't need to enter `tickMineralWorker` at all — the guard is defence-in-depth.

### Testing Strategy

**Unit tests (plain JUnit, no Quarkus):**

1. `WorkerSaturationCurveTest` — measures income at 8/12/16/20/24 workers per base, asserts the 3rd-worker efficiency drop matches published data (~50% of 2-worker rate)
2. `WorkerScoutingTest` — verifies `onWorkerScouting` pulls worker off minerals, scout produces no income, `onWorkerReturnFromScout` re-assigns to patch, auto-return at `returnAtLoop`
3. `WorkerTransferTest` — verifies `transferMineralWorkers` produces a TRANSFERRING worker that walks to new base, produces no income during transit, and starts mining on arrival
4. Update existing `WorkerSimulationTest` — add cases for new states in the tick loop

**Diagnostic test (report profile):**
5. `WorkerActivityDiagnosticTest` — runs oracle replays, counts workers far from any base at each checkpoint, reports per-race scout frequency and duration. Calibrates the distance threshold for replay-mode scout detection.

**Standalone baseline impact:**
After implementation, re-run `StandaloneEmulatedGameBaselineTest` and compare per-worker income ratio against the pre-change baseline. The transit modelling should reduce per-worker income; scouting hooks won't fire in standalone mode until a caller is wired up.

## Implementation Order

1. `WorkerSaturationCurveTest` — validate current contention model (may discover it's already correct)
2. Add `SCOUTING` and `TRANSFERRING` to `WorkerTask`
3. Implement scouting lifecycle (`onWorkerScouting`, `onWorkerReturnFromScout`, auto-return in tick)
4. Implement transit modelling (modify `transferMineralWorkers`, add `tickTransferringWorker`)
5. Update tick loop to handle new states
6. Write unit tests for scouting and transit
7. Write `WorkerActivityDiagnosticTest` for replay-mode calibration
8. Wire replay validation harness to detect scouts from GT positions
9. Re-run standalone baseline, measure impact

## References

- `WorkerSimulation.java` — mining loop, saturation model, transfer logic
- `SimulatedWorker.java` — worker state, movement, phase timing
- `MineralPatch.java` — MAX_MINERS=2, patch distribution
- `GasSlot.java` — gas worker model (reference only)
- `WorkerSimulationTest.java` — existing test patterns
- `StandaloneEmulatedGameBaselineTest.java` — standalone accuracy measurement
- `ReplayValidationHarness.java` — GT sync integration point
- GameReplays.org — SC2 mineral mining mechanics (saturation curves)
- TL.net — optimal worker saturation thread
- Issue #572 — problem statement, original approaches
- Issue #566 — reverted per-race efficiency scalars (anti-pattern)
- Protocol: calibration-milestones-are-waypoints — reach, assess, replan
- Protocol: sc2data-train-times-require-calibration — constants from data, not formulas
- Memory: feedback-simulation-over-formulas — always simulate, never formula-fit
