# Worker-to-Resource Assignment Tracking — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #495 — Worker-to-resource assignment tracking from replay events
**Issue group:** #495

**Goal:** Replace the transient per-tick `countWorkersPerBase()` computation and heuristic gas worker budget in EmulatedGame with an event-driven `WorkerAssignment` class that tracks per-base mineral and gas worker counts.

**Architecture:** New package-private `WorkerAssignment` class in `io.quarkmind.sc2.emulated`, composed into `EmulatedGame` alongside `EconomyTracker`. Tracks per-worker-tag base assignment via a `Map<String, String>` for O(1) death lookup. Events fired from existing intent handlers (`handleTrain` completion, `handleBuild` completion, `handleUnitDied`). Query API replaces `miningProbesPerBase` array and gas worker budget in `tick()`.

**Tech Stack:** Java 21, plain JUnit (no Quarkus), EmulatedGame physics layer

## Global Constraints

- WorkerAssignment is package-private — no public API beyond what EmulatedGame needs
- `setMiningProbesPerBase` override retained for ReplayValidationHarness
- No changes to `EconomyTracker`, `PlayerState`, `RaceModel` interfaces, or `SC2Data` constants
- All new tests are plain JUnit (not `@QuarkusTest`)
- Existing playbook calibration tests (`SC2PlaybookCalibrationTest`, `DualRunCalibrationTest`) must pass after integration

---

## Batch 1: WorkerAssignment class with tests

### Task 1: WorkerAssignment — core data model and event API

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerAssignment.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerAssignmentTest.java`

**Interfaces:**
- Consumes: `Point2d` (from `io.quarkmind.domain`), `BuildingType` (for town hall / gas building checks), `SC2Data.isGasBuilding()`, `SC2Data.MINERAL_PATCHES_PER_BASE`
- Produces:
  - `void onTownHallCompleted(String baseTag, Point2d position)` — adds base, triggers redistribution
  - `void onGasBuildingCompleted(String buildingTag, Point2d position)` — associates gas building with nearest base, transfers up to 3 mineral workers
  - `void onWorkerProduced(String workerTag, Point2d buildingPosition)` — assigns to nearest base
  - `void onWorkerDied(String workerTag)` — removes from assigned base
  - `void onInitialWorkers(List<String> workerTags, Point2d basePosition, String baseTag)` — bulk-assigns starting workers
  - `int[] mineralWorkersPerBase()` — mineral workers per base (indexed by base order)
  - `int gasWorkersForBuilding(String buildingTag)` — gas workers assigned to a specific gas building
  - `void reset()` — clears all state

- [ ] **Step 1: Write failing test — onInitialWorkers assigns all workers to a single base**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.Point2d;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class WorkerAssignmentTest {

    private WorkerAssignment assignment;

    @BeforeEach
    void setUp() {
        assignment = new WorkerAssignment();
    }

    @Test
    void initialWorkers_assignedToBase() {
        assignment.onTownHallCompleted("nexus-0", new Point2d(8, 8));
        assignment.onInitialWorkers(
            List.of("probe-0", "probe-1", "probe-2", "probe-3",
                    "probe-4", "probe-5", "probe-6", "probe-7",
                    "probe-8", "probe-9", "probe-10", "probe-11"),
            new Point2d(8, 8), "nexus-0");
        int[] miners = assignment.mineralWorkersPerBase();
        assertEquals(1, miners.length);
        assertEquals(12, miners[0]);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerAssignmentTest#initialWorkers_assignedToBase -q`
Expected: FAIL — `WorkerAssignment` class does not exist

- [ ] **Step 3: Implement WorkerAssignment with core data model**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.Point2d;
import java.util.*;

class WorkerAssignment {

    private record BaseEntry(String baseTag, Point2d position,
                             Set<String> mineralWorkerTags,
                             Map<String, Set<String>> gasWorkerTagsByBuilding) {
        BaseEntry(String baseTag, Point2d position) {
            this(baseTag, position, new LinkedHashSet<>(), new LinkedHashMap<>());
        }
    }

    private final List<BaseEntry> bases = new ArrayList<>();
    private final Map<String, String> workerToBase = new HashMap<>();
    private final Set<String> unassignedWorkers = new LinkedHashSet<>();

    void onTownHallCompleted(String baseTag, Point2d position) {
        if (bases.stream().anyMatch(b -> b.baseTag.equals(baseTag))) return;
        BaseEntry newBase = new BaseEntry(baseTag, position);
        bases.add(newBase);
        redistributeOnExpansion(newBase);
    }

    void onInitialWorkers(List<String> workerTags, Point2d basePosition, String baseTag) {
        BaseEntry base = findBaseByTag(baseTag);
        if (base == null) return;
        for (String tag : workerTags) {
            base.mineralWorkerTags.add(tag);
            workerToBase.put(tag, baseTag);
        }
    }

    void onWorkerProduced(String workerTag, Point2d buildingPosition) {
        BaseEntry nearest = findNearestBase(buildingPosition);
        if (nearest == null) {
            unassignedWorkers.add(workerTag);
            return;
        }
        nearest.mineralWorkerTags.add(workerTag);
        workerToBase.put(workerTag, nearest.baseTag);
    }

    void onGasBuildingCompleted(String buildingTag, Point2d position) {
        BaseEntry nearest = findNearestBase(position);
        if (nearest == null) return;
        Set<String> gasWorkers = new LinkedHashSet<>();
        nearest.gasWorkerTagsByBuilding.put(buildingTag, gasWorkers);
        Iterator<String> it = nearest.mineralWorkerTags.iterator();
        int transferred = 0;
        while (it.hasNext() && transferred < 3) {
            String workerTag = it.next();
            it.remove();
            gasWorkers.add(workerTag);
            transferred++;
        }
    }

    void onWorkerDied(String workerTag) {
        String baseTag = workerToBase.remove(workerTag);
        if (baseTag == null) {
            unassignedWorkers.remove(workerTag);
            return;
        }
        BaseEntry base = findBaseByTag(baseTag);
        if (base == null) return;
        if (base.mineralWorkerTags.remove(workerTag)) return;
        for (Set<String> gasWorkers : base.gasWorkerTagsByBuilding.values()) {
            if (gasWorkers.remove(workerTag)) return;
        }
    }

    int[] mineralWorkersPerBase() {
        int[] result = new int[bases.size()];
        for (int i = 0; i < bases.size(); i++) {
            result[i] = bases.get(i).mineralWorkerTags.size();
        }
        return result;
    }

    int gasWorkersForBuilding(String buildingTag) {
        for (BaseEntry base : bases) {
            Set<String> workers = base.gasWorkerTagsByBuilding.get(buildingTag);
            if (workers != null) return workers.size();
        }
        return 0;
    }

    void reset() {
        bases.clear();
        workerToBase.clear();
        unassignedWorkers.clear();
    }

    private void redistributeOnExpansion(BaseEntry newBase) {
        final int optimalSaturation = 16;
        for (BaseEntry existing : bases) {
            if (existing == newBase) continue;
            int excess = existing.mineralWorkerTags.size() - optimalSaturation;
            if (excess <= 0) continue;
            int toTransfer = Math.min(excess, optimalSaturation - newBase.mineralWorkerTags.size());
            if (toTransfer <= 0) continue;
            Iterator<String> it = existing.mineralWorkerTags.iterator();
            int moved = 0;
            while (it.hasNext() && moved < toTransfer) {
                String tag = it.next();
                it.remove();
                newBase.mineralWorkerTags.add(tag);
                workerToBase.put(tag, newBase.baseTag);
                moved++;
            }
        }
        Iterator<String> unassigned = unassignedWorkers.iterator();
        while (unassigned.hasNext() && newBase.mineralWorkerTags.size() < optimalSaturation) {
            String tag = unassigned.next();
            unassigned.remove();
            newBase.mineralWorkerTags.add(tag);
            workerToBase.put(tag, newBase.baseTag);
        }
    }

    private BaseEntry findNearestBase(Point2d position) {
        BaseEntry nearest = null;
        double minDistSq = Double.MAX_VALUE;
        for (BaseEntry base : bases) {
            double dx = position.x() - base.position.x();
            double dy = position.y() - base.position.y();
            double dSq = dx * dx + dy * dy;
            if (dSq < minDistSq) {
                minDistSq = dSq;
                nearest = base;
            }
        }
        return nearest;
    }

    private BaseEntry findBaseByTag(String baseTag) {
        for (BaseEntry base : bases) {
            if (base.baseTag.equals(baseTag)) return base;
        }
        return null;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerAssignmentTest#initialWorkers_assignedToBase -q`
Expected: PASS

- [ ] **Step 5: Write failing test — onWorkerProduced assigns to nearest base**

```java
@Test
void workerProduced_assignedToNearestBase() {
    assignment.onTownHallCompleted("nexus-0", new Point2d(8, 8));
    assignment.onTownHallCompleted("nexus-1", new Point2d(50, 50));
    assignment.onWorkerProduced("probe-new", new Point2d(9, 9));
    int[] miners = assignment.mineralWorkersPerBase();
    assertEquals(1, miners[0]);
    assertEquals(0, miners[1]);
}
```

- [ ] **Step 6: Run test — should PASS (implementation already handles this)**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerAssignmentTest#workerProduced_assignedToNearestBase -q`
Expected: PASS

- [ ] **Step 7: Write failing test — onGasBuildingCompleted transfers 3 mineral workers to gas**

```java
@Test
void gasBuildingCompleted_transfers3MineralWorkersToGas() {
    assignment.onTownHallCompleted("nexus-0", new Point2d(8, 8));
    for (int i = 0; i < 12; i++) {
        assignment.onWorkerProduced("probe-" + i, new Point2d(9, 9));
    }
    assignment.onGasBuildingCompleted("assim-0", new Point2d(5, 11));
    assertEquals(9, assignment.mineralWorkersPerBase()[0]);
    assertEquals(3, assignment.gasWorkersForBuilding("assim-0"));
}
```

- [ ] **Step 8: Run test — should PASS**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerAssignmentTest#gasBuildingCompleted_transfers3MineralWorkersToGas -q`
Expected: PASS

- [ ] **Step 9: Write failing test — onGasBuildingCompleted with fewer than 3 available workers**

```java
@Test
void gasBuildingCompleted_fewerThan3Available() {
    assignment.onTownHallCompleted("nexus-0", new Point2d(8, 8));
    assignment.onWorkerProduced("probe-0", new Point2d(9, 9));
    assignment.onWorkerProduced("probe-1", new Point2d(9, 9));
    assignment.onGasBuildingCompleted("assim-0", new Point2d(5, 11));
    assertEquals(0, assignment.mineralWorkersPerBase()[0]);
    assertEquals(2, assignment.gasWorkersForBuilding("assim-0"));
}
```

- [ ] **Step 10: Run test — should PASS**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerAssignmentTest#gasBuildingCompleted_fewerThan3Available -q`
Expected: PASS

- [ ] **Step 11: Write failing test — onWorkerDied removes mineral worker**

```java
@Test
void workerDied_removedFromMineralLine() {
    assignment.onTownHallCompleted("nexus-0", new Point2d(8, 8));
    assignment.onWorkerProduced("probe-0", new Point2d(9, 9));
    assignment.onWorkerProduced("probe-1", new Point2d(9, 9));
    assignment.onWorkerDied("probe-0");
    assertEquals(1, assignment.mineralWorkersPerBase()[0]);
}
```

- [ ] **Step 12: Run test — should PASS**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerAssignmentTest#workerDied_removedFromMineralLine -q`
Expected: PASS

- [ ] **Step 13: Write failing test — onWorkerDied removes gas worker**

```java
@Test
void workerDied_removedFromGasLine() {
    assignment.onTownHallCompleted("nexus-0", new Point2d(8, 8));
    for (int i = 0; i < 6; i++) {
        assignment.onWorkerProduced("probe-" + i, new Point2d(9, 9));
    }
    assignment.onGasBuildingCompleted("assim-0", new Point2d(5, 11));
    assertEquals(3, assignment.gasWorkersForBuilding("assim-0"));
    // Gas workers are the first 3 mineral workers transferred (FIFO)
    assignment.onWorkerDied("probe-0");
    assertEquals(2, assignment.gasWorkersForBuilding("assim-0"));
    assertEquals(3, assignment.mineralWorkersPerBase()[0]);
}
```

- [ ] **Step 14: Run test — should PASS**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerAssignmentTest#workerDied_removedFromGasLine -q`
Expected: PASS

- [ ] **Step 15: Write failing test — onTownHallCompleted triggers redistribution from oversaturated base**

```java
@Test
void townHallCompleted_redistributesFromOversaturatedBase() {
    assignment.onTownHallCompleted("nexus-0", new Point2d(8, 8));
    for (int i = 0; i < 20; i++) {
        assignment.onWorkerProduced("probe-" + i, new Point2d(9, 9));
    }
    assertEquals(20, assignment.mineralWorkersPerBase()[0]);
    assignment.onTownHallCompleted("nexus-1", new Point2d(50, 50));
    int[] miners = assignment.mineralWorkersPerBase();
    assertEquals(16, miners[0]);
    assertEquals(4, miners[1]);
}
```

- [ ] **Step 16: Run test — should PASS**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerAssignmentTest#townHallCompleted_redistributesFromOversaturatedBase -q`
Expected: PASS

- [ ] **Step 17: Write failing test — no bases means worker is unassigned**

```java
@Test
void workerProduced_noBases_unassigned() {
    assignment.onWorkerProduced("probe-0", new Point2d(9, 9));
    assertEquals(0, assignment.mineralWorkersPerBase().length);
}
```

- [ ] **Step 18: Run test — should PASS**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerAssignmentTest#workerProduced_noBases_unassigned -q`
Expected: PASS

- [ ] **Step 19: Write failing test — reset clears all state**

```java
@Test
void reset_clearsAllState() {
    assignment.onTownHallCompleted("nexus-0", new Point2d(8, 8));
    assignment.onWorkerProduced("probe-0", new Point2d(9, 9));
    assignment.reset();
    assertEquals(0, assignment.mineralWorkersPerBase().length);
}
```

- [ ] **Step 20: Run test — should PASS**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerAssignmentTest#reset_clearsAllState -q`
Expected: PASS

- [ ] **Step 21: Write failing test — duplicate town hall tag is idempotent**

```java
@Test
void townHallCompleted_duplicateTag_idempotent() {
    assignment.onTownHallCompleted("nexus-0", new Point2d(8, 8));
    assignment.onTownHallCompleted("nexus-0", new Point2d(8, 8));
    assertEquals(1, assignment.mineralWorkersPerBase().length);
}
```

- [ ] **Step 22: Run test — should PASS**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerAssignmentTest#townHallCompleted_duplicateTag_idempotent -q`
Expected: PASS

- [ ] **Step 23: Run full test class**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerAssignmentTest -q`
Expected: All 9 tests PASS

- [ ] **Step 24: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerAssignment.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerAssignmentTest.java
git commit -m "feat(#495): add WorkerAssignment class with event-driven per-base tracking

Refs #495"
```

## Batch 2: EmulatedGame integration

### Task 2: Wire WorkerAssignment into EmulatedGame and replace countWorkersPerBase

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
  - Add `workerAssignment` field
  - Fire events from: `reset()`, `handleBuild` completion callback, `startTraining` completion callback, `handleUnitDied`, `markReplayBuildingComplete`, `spawnUnit`
  - Replace `miningProbesPerBase` / gas budget in `tick()` with `workerAssignment` queries
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` (add new test)

**Interfaces:**
- Consumes: `WorkerAssignment` (from Task 1 — all event and query methods)
- Produces: No new public API — internal wiring only

- [ ] **Step 1: Write failing test — WorkerAssignment tracks workers through game lifecycle**

Add to `EmulatedGameTest.java`:

```java
@Test
void workerAssignment_tracksWorkersProducedAndKilled() {
    game.reset();
    // After reset: 12 Probes at nexus-0
    // Tick to get past mining delay
    game.tick();

    // Train a new Probe
    game.applyIntent(new TrainIntent(UnitType.PROBE, "nexus-0", false));
    // Tick until the probe completes (~17s = ~380 loops / 22 = ~17 ticks)
    for (int i = 0; i < 20; i++) game.tick();

    // Should have 13 mineral workers (12 initial + 1 trained)
    int[] miners = game.workerAssignment().mineralWorkersPerBase();
    assertEquals(1, miners.length);
    assertEquals(13, miners[0]);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameTest#workerAssignment_tracksWorkersProducedAndKilled -q`
Expected: FAIL — `workerAssignment()` method does not exist

- [ ] **Step 3: Add WorkerAssignment field and accessor to EmulatedGame**

In `EmulatedGame.java`, add field alongside `economyTracker`:

```java
private final WorkerAssignment workerAssignment = new WorkerAssignment();

WorkerAssignment workerAssignment() { return workerAssignment; }
```

Add `workerAssignment.reset()` call in `reset()` method, before `seedInitialState`:

```java
workerAssignment.reset();
```

- [ ] **Step 4: Fire onTownHallCompleted and onInitialWorkers after seedInitialState in reset()**

In `reset()`, after `playerRaceModel.seedInitialState(friendly, geysers)` (line 110) and after the existing `countWorkersPerBase` call (line 111), add:

```java
for (Building b : friendly.buildings()) {
    if (playerRaceModel.townHallTypes().contains(b.type()) && b.isComplete()) {
        workerAssignment.onTownHallCompleted(b.tag(), b.position());
    }
}
List<String> workerTags = friendly.units().stream()
    .filter(u -> u.type() == playerRaceModel.workerType())
    .map(Unit::tag).toList();
if (!friendly.buildings().isEmpty()) {
    Building mainBase = friendly.buildings().stream()
        .filter(b -> playerRaceModel.townHallTypes().contains(b.type()) && b.isComplete())
        .findFirst().orElse(null);
    if (mainBase != null) {
        workerAssignment.onInitialWorkers(workerTags, mainBase.position(), mainBase.tag());
    }
}
```

- [ ] **Step 5: Fire onWorkerProduced in startTraining completion callback**

In `startTraining()` at the unit spawn loop (line 638-647), after `state.addUnit(...)` and the `onUnitSpawned` call, add for friendly workers only:

```java
if (state == friendly && unitType == playerRaceModel.workerType()) {
    Building bldg = state.buildings().stream()
        .filter(bl -> bl.tag().equals(buildingTag)).findFirst().orElse(null);
    Point2d spawnPos = (bldg != null) ? bldg.position() : new Point2d(9, 9);
    workerAssignment.onWorkerProduced(tag, spawnPos);
}
```

Note: `state`, `buildingTag`, and `playerRaceModel` are effectively final in the lambda context. `friendly` is a field. The comparison `state == friendly` is safe because these are reference-identical.

- [ ] **Step 6: Fire onTownHallCompleted and onGasBuildingCompleted in handleBuild completion callback**

In `handleBuild()` completion callback (line 697-702), after `markBuildingComplete(tag, state)` and `state.addSupply(...)`, add for friendly buildings:

```java
if (state == friendly) {
    if (playerRaceModel.townHallTypes().contains(bt)) {
        workerAssignment.onTownHallCompleted(tag, b.location());
    } else if (SC2Data.isGasBuilding(bt)) {
        workerAssignment.onGasBuildingCompleted(tag, b.location());
    }
}
```

- [ ] **Step 7: Fire onWorkerDied in handleUnitDied**

In `handleUnitDied()` (line 870-877), after `state.removeUnit(d.unitTag())`, add:

```java
if (state == friendly && SC2Data.isWorker(d.unitType())) {
    workerAssignment.onWorkerDied(d.unitTag());
}
```

- [ ] **Step 8: Fire events in markReplayBuildingComplete**

In `markReplayBuildingComplete()` (line 1507-1517), after the existing `replaceAllBuildings` and `onBuildingComplete` calls, add:

```java
if (completedType != null && playerRaceModel != null) {
    Building completed = friendly.buildings().stream()
        .filter(b -> b.tag().equals(tag)).findFirst().orElse(null);
    if (completed != null) {
        if (playerRaceModel.townHallTypes().contains(completedType)) {
            workerAssignment.onTownHallCompleted(tag, completed.position());
        } else if (SC2Data.isGasBuilding(completedType)) {
            workerAssignment.onGasBuildingCompleted(tag, completed.position());
        }
    }
}
```

- [ ] **Step 9: Fire onWorkerProduced in spawnUnit for friendly workers**

In `spawnUnit()` (line 1530-1538), after `state.addUnit(...)`, add:

```java
if (owner == 1 && playerRaceModel != null && type == playerRaceModel.workerType()) {
    workerAssignment.onWorkerProduced(tag, position);
}
```

- [ ] **Step 10: Replace miningProbesPerBase and gas budget in tick() with WorkerAssignment queries**

In `tick()`, replace the block from line 123-175 (worker count computation through gas income) with:

```java
// Worker assignment is event-driven — no per-tick recomputation needed
if (miningProbesOverridden) {
    // ReplayValidationHarness ground-truth override — bypass WorkerAssignment
    miningProbesOverridden = false;
} else {
    miningProbesPerBase = workerAssignment.mineralWorkersPerBase();
}

// Mineral and gas income — delayed on game start to model worker travel time
double tickMineralIncome = 0;
double tickGasIncome = 0;
if (gameFrame > SC2Data.MINING_START_DELAY_TICKS) {
    for (final int workersAtBase : miningProbesPerBase) {
        double income = SC2Data.mineralIncomePerTick(workersAtBase);
        friendly.addMinerals(income);
        tickMineralIncome += income;
    }

    for (final Building b : friendly.buildings()) {
        if (!SC2Data.isGasBuilding(b.type()) || !b.isComplete()) continue;
        final int workersOnThisGeyser = workerAssignment.gasWorkersForBuilding(b.tag());
        if (workersOnThisGeyser <= 0) continue;
        double gasIncome = SC2Data.gasIncomePerTick(workersOnThisGeyser);
        friendly.addVespene(gasIncome);
        tickGasIncome += gasIncome;
    }
}
```

This keeps the `miningProbesPerBase` field for the harness override path but sources it from `workerAssignment` by default. The gas income loop now queries per-building gas worker counts instead of a global budget.

- [ ] **Step 11: Remove countWorkersPerBase calls from tick() and reset()**

Remove the line `miningProbesPerBase = countWorkersPerBase(playerRaceModel, friendly.buildings(), friendly.units());` from both:
- `reset()` (line 111)
- `tick()` (line 124)

Keep `countWorkersPerBase()` as a static method — it's used by `ReplayValidationHarness` and external callers.

- [ ] **Step 12: Run the new test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameTest#workerAssignment_tracksWorkersProducedAndKilled -q`
Expected: PASS

- [ ] **Step 13: Run full EmulatedGameTest suite**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameTest -q`
Expected: All existing tests PASS (no regressions)

- [ ] **Step 14: Run playbook calibration tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2PlaybookCalibrationTest -q`
Expected: All 9 playbook tests PASS

- [ ] **Step 15: Run DualRunCalibrationTest**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DualRunCalibrationTest -q`
Expected: All tests PASS

- [ ] **Step 16: Run full quarkmind-sc2 test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: No regressions. If any economy-related tests fail, inspect and fix — likely due to gas worker count differences between the old heuristic budget and the new event-driven tracking.

- [ ] **Step 17: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "feat(#495): wire WorkerAssignment into EmulatedGame — replace countWorkersPerBase

Replace transient per-tick countWorkersPerBase() computation and heuristic
gas worker budget with event-driven WorkerAssignment queries. Events fired
from handleBuild completion, startTraining completion, handleUnitDied,
markReplayBuildingComplete, and spawnUnit. setMiningProbesPerBase override
retained for ReplayValidationHarness.

Refs #495"
```

### Task 3: Register WorkerAssignment in CLAUDE.md test lists

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: None
- Produces: None (documentation only)

- [ ] **Step 1: Add WorkerAssignmentTest to the unit test list in CLAUDE.md**

Find the unit test list section containing `EconomyTrackerTest` (or nearby economy tests) and add `WorkerAssignmentTest` alphabetically.

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#495): register WorkerAssignmentTest in CLAUDE.md test list

Refs #495"
```

## References

- [2026-11-02-worker-resource-assignment-design.md] — design spec this plan implements
- [EmulatedGame.java:122-203] — tick() economy flow (replaced)
- [EmulatedGame.java:366-524] — handleTrain (worker production event source)
- [EmulatedGame.java:679-703] — handleBuild (building completion event source)
- [EmulatedGame.java:870-877] — handleUnitDied (worker death event source)
- [EmulatedGame.java:1258-1306] — setMiningProbesPerBase + countWorkersPerBase
- [EmulatedGame.java:1507-1538] — markReplayBuildingComplete + spawnUnit
- [EconomyTracker.java] — accumulator (unchanged)
- [RaceModel.java:117-121] — workerType() and townHallTypes() used by WorkerAssignment
- [ProtossRaceModel.java:23-47] — seedInitialState reference for initial tag/position format
- [SC2Data.java:39-89] — tier rate constants (unchanged)
- [GitHub #495] — focal issue
- [GitHub #481] — parent epic (Phase 3 Economy accuracy)
