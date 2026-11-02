# Worker-to-Resource Assignment Tracking — Design Spec

**Issue:** #495
**Parent:** #481 (Epic: EmulatedGame playbook calibration — Phase 3 Economy accuracy)
**Scale:** M | **Complexity:** Med

## Problem

EmulatedGame's economy model computes worker-to-base distribution transiently each tick via `countWorkersPerBase()` (nearest-distance assignment to completed town halls). Gas workers are a heuristic budget: `completedGasBuildings * 3`, deducted from the largest base first. Neither approach tracks the actual SC2 worker lifecycle — workers assigned to gas, workers transferring on expansion, or which base loses a worker on death.

The income model (calibrated 3-tier mineral saturation rates at 77.7/38.8/7.8 min/min from #496, gas rates at 54.8/54.8/28.8 from #496, MULE at 277.8 from #498) is accurate per-worker but imprecise in worker distribution, especially for:
- Gas assignment timing: workers don't instantly move to a completed Assimilator
- Expansion transfers: workers don't redistribute instantly when a new base completes
- Worker death attribution: which base's mineral/gas line actually loses the worker

## Solution

A new `WorkerAssignment` class that tracks per-base mineral and gas worker counts via an event-driven model, replacing the transient `countWorkersPerBase()` computation and heuristic gas budget.

## Architecture

### WorkerAssignment class

Package-private class in `io.quarkmind.sc2.emulated`, composed into `EmulatedGame` alongside `EconomyTracker`.

**Internal data model:**

```java
class WorkerAssignment {
    // Per-base tracking
    record BaseEntry(String baseTag, Point2d position,
                     Set<String> mineralWorkerTags,
                     Map<String, Set<String>> gasWorkerTagsByBuilding) {}

    private final List<BaseEntry> bases = new ArrayList<>();
    private final Map<String, String> workerToBase = new HashMap<>();  // workerTag -> baseTag
    private final Set<String> unassignedWorkers = new LinkedHashSet<>();
}
```

Key design points:
- `workerToBase` map enables O(1) lookup on worker death — no scanning
- `BaseEntry` tracks both mineral workers (by tag) and gas workers per gas building (by tag)
- `unassignedWorkers` holds workers not yet at any base (scouting, in transit)
- `bases` list order determines the `int[]` index for `mineralWorkersPerBase()`

### Event API

| Method | When fired | Effect |
|---|---|---|
| `onTownHallCompleted(tag, position)` | Town hall `PendingCompletion` callback, `markReplayBuildingComplete` | Adds new `BaseEntry`. Triggers redistribution: oversaturated bases (>16 mineral workers) transfer excess to the new base |
| `onGasBuildingCompleted(tag, position)` | Gas building `PendingCompletion` callback, `markReplayBuildingComplete` | Associates gas building with nearest base. Transfers up to 3 mineral workers from that base to gas. If base has <3 mineral workers, assigns as many as available |
| `onWorkerProduced(workerTag, position)` | `handleTrain` completion for worker types, `spawnUnit` for replay | Assigns to nearest base's mineral line. If no base exists, adds to unassigned |
| `onWorkerDied(workerTag)` | `handleUnitDied` for worker types | Removes from assigned base (mineral or gas). `workerToBase` lookup finds the base. If gas worker dies, that gas building slot opens |
| `onInitialWorkers(workerTags, position)` | `reset()` after `seedInitialState` | Bulk-assigns starting workers (12 Probes/SCVs/Drones) to the starting base |

### Query API

| Method | Returns | Used by |
|---|---|---|
| `mineralWorkersPerBase()` | `int[]` | `EmulatedGame.tick()` — replaces `miningProbesPerBase` |
| `gasWorkersForBuilding(String buildingTag)` | `int` | `EmulatedGame.tick()` — replaces heuristic gas budget |
| `totalGasWorkers()` | `int` | Optional diagnostics |
| `totalMineralWorkers()` | `int` | Optional diagnostics |

### Integration with EmulatedGame

**tick() changes:**

Before (current):
```java
miningProbesPerBase = countWorkersPerBase(playerRaceModel, ...);
// heuristic gas budget
int gasWorkerBudget = min(completedGasBuildings * 3, totalWorkers);
// deduct gas workers from miningProbesPerBase (largest base first)
for (int workersAtBase : miningProbesPerBase) {
    income += mineralIncomePerTick(workersAtBase);
}
for (Building b : gasBuildings) {
    int workers = min(3, gasWorkersRemaining);
    income += gasIncomePerTick(workers);
}
```

After:
```java
int[] mineralPerBase = workerAssignment.mineralWorkersPerBase();
for (int workersAtBase : mineralPerBase) {
    income += mineralIncomePerTick(workersAtBase);
}
for (Building b : completedGasBuildings) {
    int workers = workerAssignment.gasWorkersForBuilding(b.tag());
    income += gasIncomePerTick(workers);
}
```

The per-tick recomputation of `countWorkersPerBase()` is removed. Worker counts update only when events fire, which is correct — workers don't teleport between bases each tick.

**Event fire points in EmulatedGame:**

| Existing code | Event to fire |
|---|---|
| `handleBuild` completion callback (line 697) | `onTownHallCompleted` (if town hall) or `onGasBuildingCompleted` (if gas building) |
| `handleTrain` completion for workers (in `PhysicsState.PendingCompletion`) | `onWorkerProduced` |
| `handleUnitDied` (line 870) | `onWorkerDied` (if worker type) |
| `reset()` after `seedInitialState` (line 110) | `onInitialWorkers` |
| `markReplayBuildingComplete` (line 1507) | `onTownHallCompleted` or `onGasBuildingCompleted` |
| `spawnUnit` for replay worker injection (line 1530) | `onWorkerProduced` |

**setMiningProbesPerBase override:**

Retained for `ReplayValidationHarness`, which injects ground-truth worker counts before each tick. When the override is set, `tick()` uses the override array instead of querying `workerAssignment.mineralWorkersPerBase()`. The one-shot consumption flag (`miningProbesOverridden`) remains unchanged.

### Worker redistribution on expansion

When a new town hall completes:
1. Find all bases with >16 mineral workers (oversaturated)
2. Transfer excess workers to the new base, up to 16 total at the new base
3. Transfer from most-oversaturated bases first
4. Update `workerToBase` mappings for transferred workers

Threshold of 16 is the optimal saturation (2 workers per patch × 8 patches). Beyond 16, tier-2 rates (7.8 min/min) apply, so transferring to a fresh base is always economically better.

### Worker reassignment on gas building completion

When an Assimilator/Refinery/Extractor completes:
1. Find nearest base by position
2. Transfer up to 3 mineral workers from that base to the gas building
3. If base has fewer than 3 mineral workers, assign what's available
4. FIFO ordering: first workers assigned to the base transfer first (mirrors SC2 rally behavior)

### Reset

`WorkerAssignment.reset()` clears all bases, worker mappings, and unassigned workers. Called from `EmulatedGame.reset()` before `seedInitialState`. After seeding, `onInitialWorkers` populates the starting state.

### RaceModel interaction

No changes to `RaceModel` interface. `WorkerAssignment` uses `RaceModel.townHallTypes()` and `RaceModel.workerType()` to identify town halls and workers — passed at construction.

## Testing

**Unit tests (`WorkerAssignmentTest`):**
- `onWorkerProduced` assigns to nearest base
- `onGasBuildingCompleted` transfers 3 mineral workers to gas
- `onWorkerDied` removes from correct base (mineral and gas cases)
- `onTownHallCompleted` triggers redistribution from oversaturated bases
- `mineralWorkersPerBase()` returns correct array shape and values
- `gasWorkersForBuilding()` returns correct per-building count
- Edge cases: no bases (all unassigned), single base, gas building with <3 available workers, worker death when already unassigned
- Reset clears all state

**Integration verification:**
- Existing `SC2PlaybookCalibrationTest` (9 playbooks) must pass — no regression in economy accuracy
- Existing `DualRunCalibrationTest` must pass — no regression in dual-run comparison
- `EconomyTrackerTest` scope unchanged (EconomyTracker is not modified)

**Accuracy measurement:**
- Run `mvn test -pl quarkmind-sc2 -Preport` after implementation
- Compare `OracleAccuracyBaselineTest` economy MAPE before and after
- Expect improvement in mineral/gas collection rate accuracy at 5-min checkpoint

## Scope exclusions

- **Worker transfer commands from replay:** SC2 doesn't have explicit "transfer worker to gas" commands in tracker events. The assignment is heuristic based on building completion timing. This is the same level of accuracy as the current model, just tracked persistently rather than recomputed.
- **Idle workers:** Workers building structures or scouting are not explicitly tracked as idle. They remain assigned to their base but don't mine. This doesn't affect income since they aren't included in the production-based worker count.
- **Enemy worker tracking:** Only friendly worker assignment is tracked. Enemy workers use the existing `EnemyBehavior` model.

## References

- `EmulatedGame.java:122-203` — current tick economy flow
- `EmulatedGame.java:1282-1306` — `countWorkersPerBase()` being replaced
- `EmulatedGame.java:1258-1262` — `setMiningProbesPerBase` override (retained)
- `EmulatedGame.java:679-703` — `handleBuild` completion callback
- `EmulatedGame.java:870-877` — `handleUnitDied`
- `EmulatedGame.java:1507-1517` — `markReplayBuildingComplete`
- `EmulatedGame.java:1530-1538` — `spawnUnit` for replay
- `EconomyTracker.java` — accumulator/rate calculator (unchanged)
- `PlayerState.java` — flat container (unchanged)
- `SC2Data.java:39-89` — calibrated tier rates (unchanged)
- Issue #496 — mining income calibration (closed, shipped without per-base tracking)
- Issue #498 — MULE income calibration (closed)
- Issue #493 — UnitDied tracking (closed, prerequisite)
- `decisions.md` — D1-D4 design decisions
