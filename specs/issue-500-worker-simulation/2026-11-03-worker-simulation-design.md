# Individual Worker Simulation — Design Spec

**Issue:** #500
**Date:** 2026-11-03
**Epic:** #481 (EmulatedGame playbook calibration)

## Problem

EmulatedGame computes resource income via aggregate formulas (`SC2Data.mineralIncomePerTick(N)`, `SC2Data.gasIncomePerTick(N)`) rather than from individual worker simulation. The formula encodes worker competition as a mathematical approximation. When the approximation's assumptions break — particularly for gas, where the harness can't track gas workers accurately — errors accumulate.

After #499, the CmdEvent path's vespeneCollectionRate error dropped from 98% to 34%. The remaining error is architectural: there is no ground-truth override for gas workers (only `setMiningProbesPerBase` for minerals), and the formula can't capture the actual worker rotation dynamics.

## Approach

Replace the formula-driven income model with event-driven per-worker simulation. Each worker has a position, velocity, and current task. Income is a side-effect of deposit events — no formulas compute income. The formula stays as a diagnostic validator, not the income source.

## Design

### Worker State Machine

Each worker is a `SimulatedWorker` — a mutable object inside `WorkerSimulation`:

- **tag** (`String`) — identity, matches the Unit tag
- **position** (`Point2d`) — current x,y
- **task** (enum `WorkerTask`: `MINING_MINERALS`, `RETURNING_MINERALS`, `HARVESTING_GAS`, `RETURNING_GAS`, `IDLE`, `BUILDING`)
- **target** — the resource (mineral patch or gas building) or town hall being moved toward
- **cargo** (`double`) — minerals or gas carried (set when mining/harvesting completes)
- **cycleTicksRemaining** (`int`) — countdown for stationary tasks (mining at patch, harvesting inside gas building)

**Tick logic per worker:**
1. If stationary (mining/harvesting): decrement `cycleTicksRemaining`. When zero, pick up cargo, switch task to `RETURNING_*`, set target to nearest town hall
2. If moving: advance position toward target at worker move speed. When within arrival threshold, transition:
   - Arrived at mineral patch → start mining (`cycleTicksRemaining` = mining duration constant)
   - Arrived at gas building → enter building interior (if empty) or wait in queue
   - Arrived at town hall with cargo → deposit (add cargo to income accumulator), switch task back to mining/harvesting, set target to assigned resource

**Worker speed:** SC2 worker speed 3.94 (normal speed) = ~0.173 distance per game loop = ~3.81 distance per tick (22 loops/tick). Constant in `SC2Data`.

### Mineral Patch Model

`MineralPatch` — a mutable object per mineral field:

- **position** (`Point2d`) — location on the map
- **remainingMinerals** (`double`) — starts at 1500 (uniform; map-specific amounts deferred to #501)
- **baseIndex** (`int`) — which base this patch belongs to
- **occupant** — the worker currently mining this patch (null if free)

**8 patches per base**, positioned in a standard SC2 mineral line arc around the town hall. For standalone/emulated mode, patches are placed at fixed offsets from the town hall position. For replay validation, patch positions can come from GT observation data.

**Worker-to-patch assignment:** When a worker needs a patch (newly produced, or returning from deposit), it picks the nearest unoccupied patch. If all patches at its base have an occupant, it picks the nearest occupied patch and queues behind the occupant. This is where mineral saturation naturally emerges — queuing workers wait for the occupant to finish, reducing per-worker throughput without any formula.

**Mining duration:** Fixed constant calibrated from replays — approximately 46 game loops (~2.1 ticks). The time a worker spends at the patch before picking up 5 minerals of cargo.

**Depletion:** When `remainingMinerals` hits zero, the patch is marked depleted. Workers assigned to a depleted patch reassign to the nearest non-depleted patch at the same base. If all patches at a base are depleted, workers become `IDLE`.

### Gas Building Model

`GasSlot` — state tracked per gas building (Assimilator/Refinery/Extractor):

- **position** (`Point2d`) — building location
- **baseIndex** (`int`) — which base this belongs to
- **assignedWorkers** — list of up to 3 workers assigned to this geyser
- **insideWorker** — the worker currently inside harvesting (null if empty)
- **waitQueue** — workers waiting to enter

**Gas cycle per worker:**
1. Walk to gas building → arrive → if building interior is empty, enter (worker position locked to building). If another worker is inside, wait in queue outside
2. Inside building: harvest for fixed duration (~29 game loops / ~1.3 ticks, calibrated from replays). Pick up 4 gas cargo (standard SC2 amount)
3. Exit building → walk to nearest town hall → deposit → walk back to gas building
4. The 3rd-worker penalty emerges naturally: with 3 workers rotating through 1 interior slot, the third worker spends more time waiting outside, reducing effective throughput

**Worker assignment to gas:** When a gas building completes, `WorkerSimulation` transfers up to 3 mineral workers from the nearest base (same logic as current `WorkerAssignment.onGasBuildingCompleted`). Their task switches from mineral mining to gas harvesting.

### WorkerSimulation — Integration with EmulatedGame

`WorkerSimulation` is a package-private class in `io.quarkmind.sc2.emulated`, replacing `WorkerAssignment` entirely.

**API:**
- `tick()` → `TickIncome` (record: `mineralIncome`, `vespeneIncome`) — advances all workers one tick, returns total income from deposit events
- `onTownHallCompleted(tag, position)` — registers base, creates 8 mineral patches at offsets around position
- `onWorkerProduced(tag, position)` — creates SimulatedWorker, assigns to nearest available patch
- `onInitialWorkers(tags, baseTag, basePosition)` — bulk-assigns starting workers
- `onGasBuildingCompleted(tag, position)` — registers gas slot, transfers up to 3 mineral workers
- `onWorkerDied(tag)` — removes worker, frees its patch/gas slot
- `onWorkerTaskChanged(tag, task)` — for BUILDING state (worker pulled off mining to construct)
- `syncFromGroundTruth(List<Unit> gtWorkers, List<Building> gtBuildings)` — counts GT workers near each mineral patch and gas building, reassigns simulated workers to match resource-level counts
- `reset()` — clears all state

**EmulatedGame.tick() — before/after:**

```java
// Before (formula-driven):
for (int workersAtBase : miningProbesPerBase) {
    friendly.addMinerals(SC2Data.mineralIncomePerTick(workersAtBase));
}
for (Building b : friendly.buildings()) {
    if (isGasBuilding(b) && b.isComplete()) {
        friendly.addVespene(SC2Data.gasIncomePerTick(
            workerAssignment.gasWorkersForBuilding(b.tag())));
    }
}

// After (simulation-driven):
TickIncome income = workerSimulation.tick();
friendly.addMinerals(income.mineralIncome());
friendly.addVespene(income.vespeneIncome());
```

**Formula-as-validator:** After computing simulation income, also compute the formula result using derived worker counts. Log a warning if they diverge beyond a calibrated threshold. `SC2Data.mineralIncomePerTick()` and `gasIncomePerTick()` stay as diagnostic tools — no longer the income source.

**Harness integration — replacing setMiningProbesPerBase:**

`ReplayValidationHarness` calls `syncFromGroundTruth(gtWorkers, gtBuildings)` each tick instead of `setMiningProbesPerBase(countWorkersPerBase(...))`. This:
- Counts GT workers near each mineral patch → reassigns simulated mineral workers to match
- Counts GT workers near each gas building → reassigns simulated gas workers to match
- Fills the architectural gap: gas workers now get GT assignment, same as minerals

**Removed:**
- `setMiningProbesPerBase()`, `miningProbesOverridden` flag, `miningProbesPerBase` array — all replaced by `syncFromGroundTruth()`
- `WorkerAssignment` class — replaced entirely by `WorkerSimulation`

### Testing

**New unit tests (plain JUnit, no CDI) — `WorkerSimulationTest`:**

| Test | What it verifies |
|------|-----------------|
| Single worker mineral cycle | walk → mine → return → deposit → walk back produces expected income |
| 8 workers, 1 base | Each gets unique patch, no waiting — income matches tier-1 rate |
| 16 workers, 1 base | 2 per patch, second waits — saturation drop visible |
| 24 workers, 1 base | 3 per patch — deep saturation |
| Depleted patch reassignment | Worker moves to nearest non-depleted patch |
| All patches depleted | Workers become IDLE, zero income |
| Gas 3-worker rotation | 3 workers rotating through 1 building — expected gas rate |
| Gas 3rd-worker penalty | Measurably lower per-worker throughput than 2 workers |
| onGasBuildingCompleted | Transfers 3 mineral workers to gas |
| onWorkerDied | Frees patch/gas slot, survivors unaffected |
| onTownHallCompleted | Creates 8 patches, redistributes excess workers |
| syncFromGroundTruth | Correctly reassigns workers to match GT resource counts |
| reset | Clears all state |

**Formula validator test:** Run simulation for N ticks, compare income against formula prediction, assert within calibrated tolerance.

**Updated existing tests:**
- `EmulatedGameTest` — update assertions referencing `setMiningProbesPerBase` or hardcoded mineral values
- `ReplayValidationHarness` tests — use `syncFromGroundTruth` instead of `setMiningProbesPerBase`
- `WorkerAssignmentTest` — replaced by `WorkerSimulationTest`
- `EconomyTrackerTest` — unchanged (downstream, still receives `recordIncome`)

**Calibration validation:**
- `mvn test -pl quarkmind-sc2 -Preport` — vespeneCollectionRate error < 15% at 5-min
- `mvn test -pl quarkmind-sc2 -Pbenchmark` — no game loop performance regression

### Constants (calibrated from replays)

| Constant | Value | Source |
|----------|-------|--------|
| Worker move speed | 3.94 (normal speed) | SC2 game data |
| Mining duration | ~46 game loops | Calibrate from replay observation |
| Gas harvest duration | ~29 game loops | Calibrate from replay observation |
| Mineral cargo per trip | 5 | SC2 standard |
| Gas cargo per trip | 4 | SC2 standard |
| Patches per base | 8 | SC2 standard |
| Initial patch minerals | 1500 | SC2 standard (map-specific in #501) |
| Max gas workers per building | 3 | SC2 standard |

## Out of Scope

- Map-specific mineral patch initialisation (close 1500, far 900, gold bases) — #501
- Worker pathfinding around obstacles (workers move in straight lines)
- Long-distance worker transfers between bases (workers stay at assigned base)
- Race-specific worker differences (all races mine identically in SC2)
- MULE income — handled separately by `playerRaceModel.tickPassive()`, unchanged

## References

- `WorkerAssignment.java` — class being replaced (pool-based counting, 160 lines)
- `EmulatedGame.java:141-194` — current tick() income path
- `SC2Data.java:57-95` — existing tiered mineral/gas formulas (become validators)
- `ReplayValidationHarness.java:93` — current setMiningProbesPerBase GT override
- `docs/specs/2026-05-17-saturation-mining-design.md` — prior spec: flat → tiered formula
- `docs/specs/2026-05-22-multi-base-mining-design.md` — prior spec: single base → per-base arrays
- Issue #499 — predecessor: CmdEvent gas building extraction (34% vespene error)
- Issue #501 — follow-up: map-specific mineral patch amounts
