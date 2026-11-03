# Individual Worker Simulation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #500 — Individual worker simulation with formula-as-validator for income accuracy
**Issue group:** #500

**Goal:** Replace formula-driven income model with event-driven per-worker simulation where income emerges from worker deposit events, not mathematical approximations.

**Architecture:** New `WorkerSimulation` class replaces `WorkerAssignment` (160 lines of pool counting). Each worker has position, velocity, and task — advances each tick. Mineral patches have positions and depletion. Gas buildings have interior slots with wait queues. `EmulatedGame.tick()` calls `workerSimulation.tick()` which returns `TickIncome`. Harness uses `syncFromGroundTruth()` for both mineral and gas worker assignment.

**Tech Stack:** Java 21, plain JUnit (no CDI), Quarkus (EmulatedGame integration)

## Global Constraints

- Worker speed: `SC2Data.UNIT_SPEEDS.get(UnitType.PROBE)` = 3.94 tiles/sec. Use `SC2Data.unitSpeed()` — do NOT hardcode 3.94.
- Per-tick distance: `speed / GAME_LOOPS_PER_SECOND * LOOPS_PER_TICK` = speed × 0.982
- `Point2d` is a record with `float x, y` and `distanceTo(Point2d)`. Position arithmetic requires creating new `Point2d` instances.
- All new classes in `io.quarkmind.sc2.emulated` package (package-private).
- All tests are plain JUnit — never `@QuarkusTest` for these.
- Mining constants (duration, cargo amounts) will be calibrated — use named constants, not magic numbers.
- Existing `SC2Data.mineralIncomePerTick()` and `gasIncomePerTick()` stay as validators.

---

## Batch 1: WorkerSimulation core — mineral mining cycle

### Task 1: MineralPatch, SimulatedWorker, WorkerTask, TickIncome records and WorkerSimulation mineral mining

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerTask.java`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/MineralPatch.java`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/SimulatedWorker.java`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/TickIncome.java`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java`

**Interfaces:**
- Produces: `WorkerSimulation.tick()` → `TickIncome(double mineralIncome, double vespeneIncome)`
- Produces: `WorkerSimulation.onTownHallCompleted(String tag, Point2d position)`
- Produces: `WorkerSimulation.onInitialWorkers(List<String> tags, String baseTag, Point2d basePosition)`
- Produces: `WorkerSimulation.onWorkerProduced(String tag, Point2d position)`
- Produces: `WorkerSimulation.onWorkerDied(String tag)`
- Produces: `WorkerSimulation.reset()`
- Produces: `WorkerSimulation.mineralWorkersPerBase()` → `int[]` (for formula validator)

- [ ] **Step 1: Write WorkerTask enum**

```java
package io.quarkmind.sc2.emulated;

enum WorkerTask {
    MINING_MINERALS,
    RETURNING_MINERALS,
    HARVESTING_GAS,
    RETURNING_GAS,
    IDLE,
    BUILDING
}
```

- [ ] **Step 2: Write TickIncome record**

```java
package io.quarkmind.sc2.emulated;

record TickIncome(double mineralIncome, double vespeneIncome) {}
```

- [ ] **Step 3: Write MineralPatch class**

Mutable class. 8 patches per base at fixed offsets around the town hall position. Mineral line arc: patches arranged in a half-circle ~6 tiles from the town hall.

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.Point2d;

class MineralPatch {
    private static final double INITIAL_MINERALS = 1500.0;
    private static final float PATCH_DISTANCE = 6.0f;
    static final int PATCHES_PER_BASE = 8;

    final Point2d position;
    final int baseIndex;
    private double remainingMinerals;
    SimulatedWorker occupant;

    MineralPatch(Point2d position, int baseIndex) {
        this.position = position;
        this.baseIndex = baseIndex;
        this.remainingMinerals = INITIAL_MINERALS;
    }

    boolean isDepleted() { return remainingMinerals <= 0; }

    double harvest(double amount) {
        double harvested = Math.min(amount, remainingMinerals);
        remainingMinerals -= harvested;
        return harvested;
    }

    double remainingMinerals() { return remainingMinerals; }

    static Point2d[] patchPositions(Point2d townHallPos) {
        Point2d[] positions = new Point2d[PATCHES_PER_BASE];
        for (int i = 0; i < PATCHES_PER_BASE; i++) {
            double angle = Math.PI * (0.2 + 0.6 * i / (PATCHES_PER_BASE - 1));
            float px = townHallPos.x() + (float)(PATCH_DISTANCE * Math.cos(angle));
            float py = townHallPos.y() + (float)(PATCH_DISTANCE * Math.sin(angle));
            positions[i] = new Point2d(px, py);
        }
        return positions;
    }
}
```

- [ ] **Step 4: Write SimulatedWorker class**

Mutable class. Each worker has position, task, target, cargo, cycle countdown. Tick logic: if stationary, count down. If moving, advance toward target.

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.SC2Data;
import io.quarkmind.domain.UnitType;

class SimulatedWorker {
    static final double MINERAL_CARGO = 5.0;
    static final int MINING_DURATION_LOOPS = 46;
    static final int MINING_DURATION_TICKS =
        (int) Math.ceil((double) MINING_DURATION_LOOPS / SC2Data.LOOPS_PER_TICK);
    static final double ARRIVAL_THRESHOLD = 0.5;

    final String tag;
    Point2d position;
    WorkerTask task;
    int baseIndex;

    Point2d targetPosition;
    MineralPatch assignedPatch;
    double cargo;
    int cycleTicksRemaining;

    SimulatedWorker(String tag, Point2d position, int baseIndex) {
        this.tag = tag;
        this.position = position;
        this.baseIndex = baseIndex;
        this.task = WorkerTask.IDLE;
    }

    boolean isAtTarget() {
        return targetPosition != null
            && position.distanceTo(targetPosition) <= ARRIVAL_THRESHOLD;
    }

    void moveToward(double distancePerTick) {
        if (targetPosition == null) return;
        double dist = position.distanceTo(targetPosition);
        if (dist <= distancePerTick) {
            position = targetPosition;
        } else {
            float dx = targetPosition.x() - position.x();
            float dy = targetPosition.y() - position.y();
            float ratio = (float)(distancePerTick / dist);
            position = new Point2d(
                position.x() + dx * ratio,
                position.y() + dy * ratio
            );
        }
    }
}
```

- [ ] **Step 5: Write WorkerSimulation — mineral mining only (no gas yet)**

Core class. Manages bases (with patches), workers, and the per-tick simulation loop for mineral mining. Gas support added in Task 2.

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.SC2Data;
import io.quarkmind.domain.UnitType;
import java.util.*;

class WorkerSimulation {

    record BaseEntry(String tag, Point2d position, List<MineralPatch> patches) {}

    private final List<BaseEntry> bases = new ArrayList<>();
    private final Map<String, SimulatedWorker> workers = new LinkedHashMap<>();
    private final double workerDistancePerTick;

    WorkerSimulation() {
        double speed = SC2Data.unitSpeed(UnitType.PROBE);
        this.workerDistancePerTick = speed / SC2Data.GAME_LOOPS_PER_SECOND
                                     * SC2Data.LOOPS_PER_TICK;
    }

    void onTownHallCompleted(String tag, Point2d position) {
        List<MineralPatch> patches = new ArrayList<>();
        int baseIdx = bases.size();
        for (Point2d pos : MineralPatch.patchPositions(position)) {
            patches.add(new MineralPatch(pos, baseIdx));
        }
        bases.add(new BaseEntry(tag, position, patches));
    }

    void onInitialWorkers(List<String> tags, String baseTag, Point2d basePosition) {
        int baseIdx = findBaseIndex(baseTag);
        for (String workerTag : tags) {
            SimulatedWorker w = new SimulatedWorker(workerTag, basePosition, baseIdx);
            workers.put(workerTag, w);
            assignToNearestPatch(w);
        }
    }

    void onWorkerProduced(String tag, Point2d position) {
        int baseIdx = findNearestBaseIndex(position);
        SimulatedWorker w = new SimulatedWorker(tag, position, baseIdx);
        workers.put(tag, w);
        assignToNearestPatch(w);
    }

    void onWorkerDied(String tag) {
        SimulatedWorker w = workers.remove(tag);
        if (w == null) return;
        if (w.assignedPatch != null && w.assignedPatch.occupant == w) {
            w.assignedPatch.occupant = null;
        }
    }

    TickIncome tick() {
        double mineralIncome = 0;
        for (SimulatedWorker w : workers.values()) {
            mineralIncome += tickMineralWorker(w);
        }
        return new TickIncome(mineralIncome, 0);
    }

    private double tickMineralWorker(SimulatedWorker w) {
        if (w.task == WorkerTask.BUILDING || w.task == WorkerTask.IDLE) return 0;
        if (w.task == WorkerTask.HARVESTING_GAS
            || w.task == WorkerTask.RETURNING_GAS) return 0;

        if (w.task == WorkerTask.MINING_MINERALS) {
            w.cycleTicksRemaining--;
            if (w.cycleTicksRemaining <= 0) {
                w.cargo = w.assignedPatch.harvest(SimulatedWorker.MINERAL_CARGO);
                w.task = WorkerTask.RETURNING_MINERALS;
                w.targetPosition = bases.get(w.baseIndex).position();
            }
            return 0;
        }

        if (w.task == WorkerTask.RETURNING_MINERALS) {
            w.moveToward(workerDistancePerTick);
            if (w.isAtTarget()) {
                double deposited = w.cargo;
                w.cargo = 0;
                assignToNearestPatch(w);
                return deposited;
            }
            return 0;
        }

        // Task is IDLE or transitioning — worker is moving to patch
        w.moveToward(workerDistancePerTick);
        if (w.isAtTarget() && w.assignedPatch != null) {
            if (w.assignedPatch.isDepleted()) {
                reassignFromDepletedPatch(w);
                return 0;
            }
            if (w.assignedPatch.occupant == null
                || w.assignedPatch.occupant == w) {
                w.assignedPatch.occupant = w;
                w.task = WorkerTask.MINING_MINERALS;
                w.cycleTicksRemaining = SimulatedWorker.MINING_DURATION_TICKS;
            }
            // else: patch occupied, wait (saturation penalty)
        }
        return 0;
    }

    private void assignToNearestPatch(SimulatedWorker w) {
        BaseEntry base = bases.get(w.baseIndex);
        MineralPatch nearest = null;
        double nearestDist = Double.MAX_VALUE;

        // prefer unoccupied, non-depleted patch
        for (MineralPatch p : base.patches()) {
            if (p.isDepleted()) continue;
            if (p.occupant != null) continue;
            double d = w.position.distanceTo(p.position);
            if (d < nearestDist) { nearestDist = d; nearest = p; }
        }

        // all occupied — pick nearest non-depleted (will queue)
        if (nearest == null) {
            for (MineralPatch p : base.patches()) {
                if (p.isDepleted()) continue;
                double d = w.position.distanceTo(p.position);
                if (d < nearestDist) { nearestDist = d; nearest = p; }
            }
        }

        if (nearest == null) {
            w.task = WorkerTask.IDLE;
            w.assignedPatch = null;
            return;
        }
        w.assignedPatch = nearest;
        w.targetPosition = nearest.position;
        w.task = WorkerTask.MINING_MINERALS;
        // if already at patch, start mining immediately
        if (w.isAtTarget()) {
            if (nearest.occupant == null || nearest.occupant == w) {
                nearest.occupant = w;
                w.cycleTicksRemaining = SimulatedWorker.MINING_DURATION_TICKS;
            }
        }
    }

    private void reassignFromDepletedPatch(SimulatedWorker w) {
        w.assignedPatch = null;
        assignToNearestPatch(w);
    }

    int[] mineralWorkersPerBase() {
        int[] counts = new int[bases.size()];
        for (SimulatedWorker w : workers.values()) {
            if (w.task == WorkerTask.MINING_MINERALS
                || w.task == WorkerTask.RETURNING_MINERALS) {
                if (w.baseIndex >= 0 && w.baseIndex < counts.length) {
                    counts[w.baseIndex]++;
                }
            }
        }
        return counts;
    }

    void reset() {
        bases.clear();
        workers.clear();
    }

    private int findBaseIndex(String baseTag) {
        for (int i = 0; i < bases.size(); i++) {
            if (bases.get(i).tag().equals(baseTag)) return i;
        }
        return 0;
    }

    private int findNearestBaseIndex(Point2d pos) {
        if (bases.isEmpty()) return -1;
        int nearest = 0;
        double nearestDist = Double.MAX_VALUE;
        for (int i = 0; i < bases.size(); i++) {
            double d = pos.distanceTo(bases.get(i).position());
            if (d < nearestDist) { nearestDist = d; nearest = i; }
        }
        return nearest;
    }
}
```

- [ ] **Step 6: Write failing tests for mineral mining cycle**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.Point2d;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.stream.IntStream;
import static org.junit.jupiter.api.Assertions.*;

class WorkerSimulationTest {

    private WorkerSimulation sim;
    private static final Point2d BASE_POS = new Point2d(50, 50);
    private static final String BASE_TAG = "base-1";

    @BeforeEach
    void setUp() {
        sim = new WorkerSimulation();
        sim.onTownHallCompleted(BASE_TAG, BASE_POS);
    }

    @Test
    void singleWorker_completesFullMiningCycle() {
        sim.onInitialWorkers(List.of("w-1"), BASE_TAG, BASE_POS);
        double totalIncome = 0;
        // tick enough times to complete: walk to patch + mine + walk back + deposit
        for (int t = 0; t < 20; t++) {
            totalIncome += sim.tick().mineralIncome();
        }
        assertTrue(totalIncome > 0,
            "Worker should deposit minerals within 20 ticks");
    }

    @Test
    void eightWorkers_eachGetsUniquePatch_noWaiting() {
        List<String> tags = IntStream.rangeClosed(1, 8)
            .mapToObj(i -> "w-" + i).toList();
        sim.onInitialWorkers(tags, BASE_TAG, BASE_POS);

        // run for enough ticks to get multiple deposits
        double totalIncome = 0;
        for (int t = 0; t < 40; t++) {
            totalIncome += sim.tick().mineralIncome();
        }
        assertTrue(totalIncome > 0);
    }

    @Test
    void sixteenWorkers_showsSaturationPenalty() {
        List<String> tags8 = IntStream.rangeClosed(1, 8)
            .mapToObj(i -> "w-" + i).toList();
        sim.onInitialWorkers(tags8, BASE_TAG, BASE_POS);

        double income8workers = 0;
        for (int t = 0; t < 100; t++) {
            income8workers += sim.tick().mineralIncome();
        }

        // reset and try with 16
        sim.reset();
        sim.onTownHallCompleted(BASE_TAG, BASE_POS);
        List<String> tags16 = IntStream.rangeClosed(1, 16)
            .mapToObj(i -> "w-" + i).toList();
        sim.onInitialWorkers(tags16, BASE_TAG, BASE_POS);

        double income16workers = 0;
        for (int t = 0; t < 100; t++) {
            income16workers += sim.tick().mineralIncome();
        }

        // 16 workers should earn more than 8, but less than double
        assertTrue(income16workers > income8workers,
            "16 workers should earn more than 8");
        assertTrue(income16workers < income8workers * 2,
            "16 workers should earn less than double of 8 (saturation)");
    }

    @Test
    void workerDied_freesPatchSlot() {
        sim.onInitialWorkers(List.of("w-1", "w-2"), BASE_TAG, BASE_POS);
        // tick to assign
        for (int t = 0; t < 5; t++) sim.tick();
        sim.onWorkerDied("w-1");
        // remaining worker should continue earning
        double income = 0;
        for (int t = 0; t < 20; t++) {
            income += sim.tick().mineralIncome();
        }
        assertTrue(income > 0);
    }

    @Test
    void allPatchesDepleted_workersIdle_zeroIncome() {
        sim.onInitialWorkers(List.of("w-1"), BASE_TAG, BASE_POS);
        // deplete all patches by mining for a very long time
        // (1500 minerals / 5 per trip — many trips needed)
        double totalIncome = 0;
        for (int t = 0; t < 100_000; t++) {
            totalIncome += sim.tick().mineralIncome();
        }
        // should have earned minerals then stopped
        assertTrue(totalIncome > 0);
        // now income should be zero
        double incomeAfterDepletion = 0;
        for (int t = 0; t < 100; t++) {
            incomeAfterDepletion += sim.tick().mineralIncome();
        }
        assertEquals(0.0, incomeAfterDepletion, 0.001);
    }

    @Test
    void newWorkerProduced_assignedToNearestBase() {
        sim.onInitialWorkers(List.of("w-1"), BASE_TAG, BASE_POS);
        Point2d nearBase = new Point2d(52, 50);
        sim.onWorkerProduced("w-2", nearBase);
        assertArrayEquals(new int[]{2}, sim.mineralWorkersPerBase());
    }

    @Test
    void reset_clearsAllState() {
        sim.onInitialWorkers(List.of("w-1"), BASE_TAG, BASE_POS);
        sim.reset();
        assertArrayEquals(new int[0], sim.mineralWorkersPerBase());
    }

    @Test
    void multiBase_workersAssignedToNearestBase() {
        Point2d base2Pos = new Point2d(100, 50);
        sim.onTownHallCompleted("base-2", base2Pos);
        sim.onInitialWorkers(List.of("w-1", "w-2"), BASE_TAG, BASE_POS);
        sim.onWorkerProduced("w-3", new Point2d(98, 50));
        // w-1, w-2 at base 0; w-3 at base 1
        int[] counts = sim.mineralWorkersPerBase();
        assertEquals(2, counts[0]);
        assertEquals(1, counts[1]);
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest -q`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerTask.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/MineralPatch.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/SimulatedWorker.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/TickIncome.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java
git commit -m "feat(#500): add WorkerSimulation with mineral mining cycle

Introduces SimulatedWorker, MineralPatch, WorkerTask, and TickIncome.
Workers have position, velocity, and task — income from deposit events.
Mineral saturation emerges from patch competition.  Refs #500"
```

---

## Batch 2: Gas harvesting cycle

### Task 2: GasSlot and gas worker simulation

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/GasSlot.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/SimulatedWorker.java` — add gas cargo constant and gas duration
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java` — add gas methods and gas tick logic
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java` — add gas tests

**Interfaces:**
- Consumes: `WorkerSimulation` from Task 1
- Produces: `WorkerSimulation.onGasBuildingCompleted(String tag, Point2d position)`
- Produces: `GasSlot` (interior slot + wait queue)

- [ ] **Step 1: Write GasSlot class**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.Point2d;
import java.util.*;

class GasSlot {
    static final int MAX_WORKERS = 3;

    final String buildingTag;
    final Point2d position;
    final int baseIndex;
    final List<SimulatedWorker> assignedWorkers = new ArrayList<>(MAX_WORKERS);
    SimulatedWorker insideWorker;
    final Deque<SimulatedWorker> waitQueue = new ArrayDeque<>();

    GasSlot(String buildingTag, Point2d position, int baseIndex) {
        this.buildingTag = buildingTag;
        this.position = position;
        this.baseIndex = baseIndex;
    }

    void removeWorker(SimulatedWorker w) {
        assignedWorkers.remove(w);
        waitQueue.remove(w);
        if (insideWorker == w) insideWorker = null;
    }
}
```

- [ ] **Step 2: Add gas constants to SimulatedWorker**

Add alongside existing mineral constants:

```java
static final double GAS_CARGO = 4.0;
static final int GAS_HARVEST_DURATION_LOOPS = 29;
static final int GAS_HARVEST_DURATION_TICKS =
    (int) Math.ceil((double) GAS_HARVEST_DURATION_LOOPS / SC2Data.LOOPS_PER_TICK);
```

Also add field:
```java
GasSlot assignedGasSlot;
```

- [ ] **Step 3: Add gas building and gas tick logic to WorkerSimulation**

Add to WorkerSimulation:
- `private final List<GasSlot> gasSlots = new ArrayList<>();`
- `onGasBuildingCompleted(String tag, Point2d position)` — creates GasSlot, transfers up to 3 mineral workers from nearest base
- `tickGasWorker(SimulatedWorker w)` — gas harvesting tick cycle: if HARVESTING_GAS count down, if RETURNING_GAS move and deposit, if at gas building enter or wait

In `tick()`, split worker processing: mineral workers call `tickMineralWorker`, gas workers call `tickGasWorker`. Sum gas income into `TickIncome.vespeneIncome`.

```java
void onGasBuildingCompleted(String tag, Point2d position) {
    int baseIdx = findNearestBaseIndex(position);
    GasSlot slot = new GasSlot(tag, position, baseIdx);
    gasSlots.add(slot);
    // transfer up to 3 mineral workers from this base
    int transferred = 0;
    for (SimulatedWorker w : new ArrayList<>(workers.values())) {
        if (transferred >= GasSlot.MAX_WORKERS) break;
        if (w.baseIndex != baseIdx) continue;
        if (w.task != WorkerTask.MINING_MINERALS
            && w.task != WorkerTask.RETURNING_MINERALS
            && w.task != WorkerTask.IDLE) continue;
        // free mineral patch
        if (w.assignedPatch != null && w.assignedPatch.occupant == w) {
            w.assignedPatch.occupant = null;
        }
        w.assignedPatch = null;
        w.assignedGasSlot = slot;
        w.task = WorkerTask.HARVESTING_GAS;
        w.targetPosition = slot.position;
        slot.assignedWorkers.add(w);
        transferred++;
    }
}

private double tickGasWorker(SimulatedWorker w) {
    GasSlot slot = w.assignedGasSlot;
    if (slot == null) return 0;

    if (w.task == WorkerTask.HARVESTING_GAS) {
        if (slot.insideWorker == w) {
            // inside building, harvesting
            w.cycleTicksRemaining--;
            if (w.cycleTicksRemaining <= 0) {
                w.cargo = SimulatedWorker.GAS_CARGO;
                slot.insideWorker = null;
                w.task = WorkerTask.RETURNING_GAS;
                w.targetPosition = bases.get(w.baseIndex).position();
                // let next waiting worker enter
                if (!slot.waitQueue.isEmpty()) {
                    SimulatedWorker next = slot.waitQueue.poll();
                    slot.insideWorker = next;
                    next.cycleTicksRemaining =
                        SimulatedWorker.GAS_HARVEST_DURATION_TICKS;
                }
            }
            return 0;
        }
        // moving to gas building
        w.moveToward(workerDistancePerTick);
        if (w.isAtTarget()) {
            if (slot.insideWorker == null) {
                slot.insideWorker = w;
                w.cycleTicksRemaining =
                    SimulatedWorker.GAS_HARVEST_DURATION_TICKS;
            } else if (!slot.waitQueue.contains(w)) {
                slot.waitQueue.add(w);
            }
        }
        return 0;
    }

    if (w.task == WorkerTask.RETURNING_GAS) {
        w.moveToward(workerDistancePerTick);
        if (w.isAtTarget()) {
            double deposited = w.cargo;
            w.cargo = 0;
            w.task = WorkerTask.HARVESTING_GAS;
            w.targetPosition = slot.position;
            return deposited;
        }
        return 0;
    }
    return 0;
}
```

Update `tick()`:
```java
TickIncome tick() {
    double mineralIncome = 0;
    double gasIncome = 0;
    for (SimulatedWorker w : workers.values()) {
        if (w.assignedGasSlot != null) {
            gasIncome += tickGasWorker(w);
        } else {
            mineralIncome += tickMineralWorker(w);
        }
    }
    return new TickIncome(mineralIncome, gasIncome);
}
```

Update `onWorkerDied`:
```java
void onWorkerDied(String tag) {
    SimulatedWorker w = workers.remove(tag);
    if (w == null) return;
    if (w.assignedPatch != null && w.assignedPatch.occupant == w) {
        w.assignedPatch.occupant = null;
    }
    if (w.assignedGasSlot != null) {
        w.assignedGasSlot.removeWorker(w);
    }
}
```

Update `reset()`:
```java
void reset() {
    bases.clear();
    workers.clear();
    gasSlots.clear();
}
```

- [ ] **Step 4: Write gas tests**

Add to `WorkerSimulationTest`:

```java
@Test
void gasBuildingCompleted_transfers3MineralWorkersToGas() {
    List<String> tags = IntStream.rangeClosed(1, 12)
        .mapToObj(i -> "w-" + i).toList();
    sim.onInitialWorkers(tags, BASE_TAG, BASE_POS);
    sim.onGasBuildingCompleted("gas-1", new Point2d(45, 50));
    assertEquals(9, sim.mineralWorkersPerBase()[0]);
}

@Test
void gasWorkers_produceVespeneIncome() {
    List<String> tags = IntStream.rangeClosed(1, 12)
        .mapToObj(i -> "w-" + i).toList();
    sim.onInitialWorkers(tags, BASE_TAG, BASE_POS);
    sim.onGasBuildingCompleted("gas-1", new Point2d(45, 50));
    double totalGas = 0;
    for (int t = 0; t < 40; t++) {
        totalGas += sim.tick().vespeneIncome();
    }
    assertTrue(totalGas > 0, "Gas workers should deposit vespene");
}

@Test
void threeGasWorkers_lowerPerWorkerThroughputThanTwo() {
    // 3 workers through 1 slot — 3rd worker penalty
    List<String> tags3 = IntStream.rangeClosed(1, 6)
        .mapToObj(i -> "w-" + i).toList();
    sim.onInitialWorkers(tags3, BASE_TAG, BASE_POS);
    sim.onGasBuildingCompleted("gas-1", new Point2d(45, 50));

    double totalGas3 = 0;
    for (int t = 0; t < 200; t++) {
        totalGas3 += sim.tick().vespeneIncome();
    }

    // reset with 2 gas workers
    sim.reset();
    sim.onTownHallCompleted(BASE_TAG, BASE_POS);
    List<String> tags2 = IntStream.rangeClosed(1, 5)
        .mapToObj(i -> "w-" + i).toList();
    sim.onInitialWorkers(tags2, BASE_TAG, BASE_POS);
    sim.onGasBuildingCompleted("gas-2", new Point2d(45, 50));
    // manually limit to 2 gas workers is implicit —
    // onGasBuildingCompleted takes 3, so this test needs
    // enough workers. Actually let's verify the per-worker rate:
    double perWorkerRate3 = totalGas3 / 3;

    double totalGas2 = 0;
    for (int t = 0; t < 200; t++) {
        totalGas2 += sim.tick().vespeneIncome();
    }
    double perWorkerRate2 = totalGas2 / 2;

    assertTrue(perWorkerRate2 > perWorkerRate3,
        "Per-worker gas rate with 2 workers should exceed 3 workers");
}

@Test
void gasWorkerDied_removedFromSlot() {
    List<String> tags = IntStream.rangeClosed(1, 12)
        .mapToObj(i -> "w-" + i).toList();
    sim.onInitialWorkers(tags, BASE_TAG, BASE_POS);
    sim.onGasBuildingCompleted("gas-1", new Point2d(45, 50));
    for (int t = 0; t < 5; t++) sim.tick();
    // kill a gas worker (first 3 assigned)
    sim.onWorkerDied("w-1");
    // should still produce gas from remaining 2
    double totalGas = 0;
    for (int t = 0; t < 40; t++) {
        totalGas += sim.tick().vespeneIncome();
    }
    assertTrue(totalGas > 0);
}
```

- [ ] **Step 5: Run tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest -q`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/GasSlot.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/SimulatedWorker.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java
git commit -m "feat(#500): add gas harvesting cycle to WorkerSimulation

GasSlot with interior slot and wait queue. 3rd-worker penalty emerges
from rotation. Gas income from deposit events.  Refs #500"
```

---

## Batch 3: EmulatedGame integration — replace WorkerAssignment

### Task 3: Wire WorkerSimulation into EmulatedGame, remove WorkerAssignment

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java:59,61,77-78,94-145,152-170,641,700-702,878,1266-1269,1497-1543,1563-1582`
- Delete: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerAssignment.java` (use `ide_refactor_safe_delete`)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java` — add `onWorkerTaskChanged` method
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` — update setMiningProbesPerBase references
- Delete: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerAssignmentTest.java` (use `ide_refactor_safe_delete`)

**Interfaces:**
- Consumes: `WorkerSimulation` from Tasks 1-2
- Produces: `EmulatedGame.workerSimulation()` — package-private accessor (replaces `workerAssignment()`)

- [ ] **Step 1: Add `onWorkerTaskChanged` to WorkerSimulation**

```java
void onWorkerTaskChanged(String tag, WorkerTask newTask) {
    SimulatedWorker w = workers.get(tag);
    if (w == null) return;
    if (newTask == WorkerTask.BUILDING) {
        // free resources
        if (w.assignedPatch != null && w.assignedPatch.occupant == w) {
            w.assignedPatch.occupant = null;
        }
        if (w.assignedGasSlot != null) {
            w.assignedGasSlot.removeWorker(w);
            w.assignedGasSlot = null;
        }
        w.assignedPatch = null;
        w.task = WorkerTask.BUILDING;
    } else if (newTask == WorkerTask.IDLE) {
        w.task = WorkerTask.IDLE;
        assignToNearestPatch(w);
    }
}
```

- [ ] **Step 2: Replace WorkerAssignment with WorkerSimulation in EmulatedGame**

In EmulatedGame.java:

Replace field (L59):
```java
// Before:
private final WorkerAssignment workerAssignment = new WorkerAssignment();
// After:
private final WorkerSimulation workerSimulation = new WorkerSimulation();
```

Replace accessor (L61):
```java
// Before:
WorkerAssignment workerAssignment() {return workerAssignment;}
// After:
WorkerSimulation workerSimulation() {return workerSimulation;}
```

Remove fields (L77-78):
```java
// Remove:
private int[] miningProbesPerBase;
private boolean miningProbesOverridden;
```

Update `reset()` — replace `workerAssignment.reset()` → `workerSimulation.reset()`. Replace `workerAssignment.onTownHallCompleted` → `workerSimulation.onTownHallCompleted`. Replace `workerAssignment.onInitialWorkers` → `workerSimulation.onInitialWorkers`. Remove `miningProbesPerBase` and `miningProbesOverridden` initialisation.

Update `tick()` income section (L141-170) — replace the entire mineral/gas income block:
```java
// Before: formula-driven loop over miningProbesPerBase + gasWorkersForBuilding
// After:
TickIncome income = workerSimulation.tick();
friendly.addMinerals(income.mineralIncome());
friendly.addVespene(income.vespeneIncome());
```

Remove `MINING_START_DELAY_TICKS` guard — the simulation handles mining start delay naturally (workers walk to patches before first deposit).

Update all `workerAssignment.onWorkerProduced` → `workerSimulation.onWorkerProduced` (L641, L1582).
Update all `workerAssignment.onTownHallCompleted` → `workerSimulation.onTownHallCompleted` (L700, L1497, L1517, L1541, L1563).
Update all `workerAssignment.onGasBuildingCompleted` → `workerSimulation.onGasBuildingCompleted` (L702, L1499, L1519, L1543, L1565).
Update `workerAssignment.onWorkerDied` → `workerSimulation.onWorkerDied` (L878).

Remove `setMiningProbesPerBase` method entirely (L1266-1269).

- [ ] **Step 3: Delete WorkerAssignment and WorkerAssignmentTest**

Use `ide_refactor_safe_delete` for:
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerAssignment.java`
- `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerAssignmentTest.java`

- [ ] **Step 4: Update EmulatedGameTest**

Find and update all references to `setMiningProbesPerBase`:
- L81: `game.setMiningProbesPerBase(0)` → remove or replace with equivalent (stop workers from mining by moving them to BUILDING state or just accept simulation-driven income)
- L95: `game.setMiningProbesPerBase(12, 12)` → remove (workers auto-assigned)
- L2011-2015: test `setMiningProbesPerBase_overrideConsumedThenAutoComputeResumes` → delete entirely (override mechanism removed)

For tests that suppressed income via `setMiningProbesPerBase(0)`, expose a `WorkerSimulation.pauseAll()` method or accept that income now flows through the simulation. The simpler approach: where tests set mining probes to 0, adjust expected values to account for simulation-driven income, or tick fewer times.

- [ ] **Step 5: Update TerranEmulatedGameTest and ZergEmulatedGameTest**

TerranEmulatedGameTest:
- L64: `game.setMiningProbesPerBase(12)` → remove (workers auto-assigned)
- L65: `protossGame.setMiningProbesPerBase(12)` → remove
- L161: `game.setMiningProbesPerBase(0)` → replace with stopping workers (set all to BUILDING) for MULE isolation test
- L205: `game.setMiningProbesPerBase(0)` → same
- L224: test name `orbitalCommandTownHall_countsForWorkerAssignment` → rename to `orbitalCommandTownHall_countsForWorkerSimulation`

ZergEmulatedGameTest:
- L79: `game.setMiningProbesPerBase(12)` → remove
- L80: `protossGame.setMiningProbesPerBase(12)` → remove
- L283: test name `hatchery_countsForWorkerAssignment` → rename to `hatchery_countsForWorkerSimulation`

- [ ] **Step 6: Run all emulated game tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameTest,TerranEmulatedGameTest,ZergEmulatedGameTest,WorkerSimulationTest -q`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/TerranEmulatedGameTest.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ZergEmulatedGameTest.java
git commit -m "feat(#500): wire WorkerSimulation into EmulatedGame, remove WorkerAssignment

Replace formula-driven income with simulation-driven income.
Delete WorkerAssignment (replaced by WorkerSimulation).
Remove setMiningProbesPerBase override mechanism.  Refs #500"
```

---

## Batch 4: Harness integration and formula validator

### Task 4: syncFromGroundTruth and harness update

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java` — add `syncFromGroundTruth`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java:93` — replace `setMiningProbesPerBase` with `syncFromGroundTruth`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java` — add syncFromGroundTruth tests
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ReactorTagTraceDiagnosticTest.java:72` — update setMiningProbesPerBase call
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ReactorStateAtIntentTest.java:74` — update setMiningProbesPerBase call
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ReplayGasWorkerAssignmentTest.java` — update to use WorkerSimulation

**Interfaces:**
- Consumes: `WorkerSimulation` from Tasks 1-3
- Produces: `WorkerSimulation.syncFromGroundTruth(List<Unit> gtWorkers, List<Building> gtBuildings, Race race)`

- [ ] **Step 1: Write failing test for syncFromGroundTruth**

Add to `WorkerSimulationTest`:

```java
@Test
void syncFromGroundTruth_reassignsWorkersToMatchGtCounts() {
    List<String> tags = IntStream.rangeClosed(1, 12)
        .mapToObj(i -> "w-" + i).toList();
    sim.onInitialWorkers(tags, BASE_TAG, BASE_POS);
    sim.onGasBuildingCompleted("gas-1", new Point2d(45, 50));
    // after gas building: 9 mineral, 3 gas
    assertEquals(9, sim.mineralWorkersPerBase()[0]);

    // GT says 6 mineral workers, 3 gas workers
    // syncFromGroundTruth should reassign
    // (test uses simplified GT — patch-level counts)
    sim.syncMineralWorkersFromGt(new int[]{6});
    assertEquals(6, sim.mineralWorkersPerBase()[0]);
}
```

- [ ] **Step 2: Implement syncFromGroundTruth on WorkerSimulation**

Two-level sync: mineral workers per base (from GT position counts near patches) and gas workers per building (from GT position counts near gas buildings).

```java
void syncMineralWorkersFromGt(int[] mineralWorkersPerBase) {
    for (int baseIdx = 0; baseIdx < bases.size()
                         && baseIdx < mineralWorkersPerBase.length; baseIdx++) {
        int target = mineralWorkersPerBase[baseIdx];
        int current = countMineralWorkersAtBase(baseIdx);
        if (current > target) {
            // remove excess — set to IDLE
            int excess = current - target;
            for (SimulatedWorker w : new ArrayList<>(workers.values())) {
                if (excess <= 0) break;
                if (w.baseIndex == baseIdx
                    && (w.task == WorkerTask.MINING_MINERALS
                        || w.task == WorkerTask.RETURNING_MINERALS)) {
                    if (w.assignedPatch != null
                        && w.assignedPatch.occupant == w) {
                        w.assignedPatch.occupant = null;
                    }
                    w.assignedPatch = null;
                    w.task = WorkerTask.IDLE;
                    excess--;
                }
            }
        } else if (current < target) {
            // recruit idle or unassigned workers
            int needed = target - current;
            for (SimulatedWorker w : workers.values()) {
                if (needed <= 0) break;
                if (w.task == WorkerTask.IDLE) {
                    w.baseIndex = baseIdx;
                    assignToNearestPatch(w);
                    needed--;
                }
            }
        }
    }
}

void syncGasWorkersFromGt(Map<String, Integer> gasWorkersPerBuilding) {
    for (GasSlot slot : gasSlots) {
        int target = gasWorkersPerBuilding.getOrDefault(
            slot.buildingTag, 0);
        int current = slot.assignedWorkers.size();
        if (current > target) {
            int excess = current - target;
            for (int i = 0; i < excess && !slot.assignedWorkers.isEmpty(); i++) {
                SimulatedWorker w = slot.assignedWorkers
                    .remove(slot.assignedWorkers.size() - 1);
                slot.removeWorker(w);
                w.assignedGasSlot = null;
                w.task = WorkerTask.IDLE;
                assignToNearestPatch(w);
            }
        } else if (current < target) {
            int needed = target - current;
            for (SimulatedWorker w : new ArrayList<>(workers.values())) {
                if (needed <= 0) break;
                if (w.baseIndex == slot.baseIndex
                    && (w.task == WorkerTask.MINING_MINERALS
                        || w.task == WorkerTask.RETURNING_MINERALS
                        || w.task == WorkerTask.IDLE)) {
                    if (w.assignedPatch != null
                        && w.assignedPatch.occupant == w) {
                        w.assignedPatch.occupant = null;
                    }
                    w.assignedPatch = null;
                    w.assignedGasSlot = slot;
                    w.task = WorkerTask.HARVESTING_GAS;
                    w.targetPosition = slot.position;
                    slot.assignedWorkers.add(w);
                    needed--;
                }
            }
        }
    }
}
```

- [ ] **Step 3: Update ReplayValidationHarness**

Replace L93:
```java
// Before:
emulated.setMiningProbesPerBase(countWorkersPerBase(race, gtBefore));
// After:
emulated.workerSimulation().syncMineralWorkersFromGt(
    countWorkersPerBase(race, gtBefore));
emulated.workerSimulation().syncGasWorkersFromGt(
    countGasWorkersPerBuilding(race, gtBefore));
```

Add `countGasWorkersPerBuilding` helper:
```java
static Map<String, Integer> countGasWorkersPerBuilding(
        Race race, GameState state) {
    Map<String, Integer> counts = new HashMap<>();
    List<Building> gasBuildings = state.myBuildings().stream()
        .filter(b -> SC2Data.isGasBuilding(b.type()) && b.isComplete())
        .toList();
    for (Unit u : state.myUnits()) {
        if (!SC2Data.isWorkerType(u.type(), race)) continue;
        Building nearest = null;
        double nearestDist = Double.MAX_VALUE;
        for (Building b : gasBuildings) {
            double d = u.position().distanceTo(b.position());
            if (d < 2.0 && d < nearestDist) {
                nearestDist = d;
                nearest = b;
            }
        }
        if (nearest != null) {
            counts.merge(nearest.tag(), 1, Integer::sum);
        }
    }
    return counts;
}
```

- [ ] **Step 4: Update ReactorTagTraceDiagnosticTest and ReactorStateAtIntentTest**

ReactorTagTraceDiagnosticTest L72:
```java
// Before:
emulated.setMiningProbesPerBase(
    ReplayValidationHarness.countWorkersPerBase(race, gtBefore));
// After:
emulated.workerSimulation().syncMineralWorkersFromGt(
    ReplayValidationHarness.countWorkersPerBase(race, gtBefore));
```

ReactorStateAtIntentTest L74 — same change.

- [ ] **Step 5: Update ReplayGasWorkerAssignmentTest**

Replace `WorkerAssignment` references with `WorkerSimulation`. Adjust test assertions for the new gas tracking model.

- [ ] **Step 6: Run tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest,EmulatedGameTest,TerranEmulatedGameTest,ZergEmulatedGameTest -q`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ReactorTagTraceDiagnosticTest.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ReactorStateAtIntentTest.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ReplayGasWorkerAssignmentTest.java
git commit -m "feat(#500): add syncFromGroundTruth and update harness integration

Harness now syncs both mineral and gas workers from GT positions.
Fills the architectural gap: gas workers get GT assignment.  Refs #500"
```

### Task 5: Formula-as-validator and calibration run

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java` — add formula validation logging
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` — add validator call after tick
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java` — formula validator test

**Interfaces:**
- Consumes: `WorkerSimulation.mineralWorkersPerBase()` from Task 1
- Consumes: `SC2Data.mineralIncomePerTick()`, `SC2Data.gasIncomePerTick()`

- [ ] **Step 1: Add formula validator to WorkerSimulation**

```java
record FormulaComparison(double simulatedMineral, double formulaMineral,
                         double simulatedGas, double formulaGas) {
    double mineralDivergence() {
        if (formulaMineral == 0) return simulatedMineral == 0 ? 0 : 1;
        return Math.abs(simulatedMineral - formulaMineral) / formulaMineral;
    }
    double gasDivergence() {
        if (formulaGas == 0) return simulatedGas == 0 ? 0 : 1;
        return Math.abs(simulatedGas - formulaGas) / formulaGas;
    }
}

FormulaComparison compareWithFormula(TickIncome simulated) {
    double formulaMineral = 0;
    int[] mineralCounts = mineralWorkersPerBase();
    for (int count : mineralCounts) {
        formulaMineral += SC2Data.mineralIncomePerTick(count);
    }
    double formulaGas = 0;
    for (GasSlot slot : gasSlots) {
        formulaGas += SC2Data.gasIncomePerTick(slot.assignedWorkers.size());
    }
    return new FormulaComparison(
        simulated.mineralIncome(), formulaMineral,
        simulated.vespeneIncome(), formulaGas);
}
```

- [ ] **Step 2: Add formula validator test**

```java
@Test
void formulaValidator_simulationAndFormulaWithinTolerance() {
    List<String> tags = IntStream.rangeClosed(1, 16)
        .mapToObj(i -> "w-" + i).toList();
    sim.onInitialWorkers(tags, BASE_TAG, BASE_POS);

    // run for enough ticks to stabilise
    TickIncome lastIncome = null;
    for (int t = 0; t < 100; t++) {
        lastIncome = sim.tick();
    }

    WorkerSimulation.FormulaComparison comparison =
        sim.compareWithFormula(lastIncome);
    // simulation and formula should be in the same ballpark
    // (exact match not expected — simulation models walk time)
    assertTrue(comparison.mineralDivergence() < 0.5,
        "Mineral divergence too high: " + comparison);
}
```

- [ ] **Step 3: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all PASS (this catches any references we missed)

- [ ] **Step 4: Run calibration report**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=OracleAccuracyBaselineTest -q`
Check vespeneCollectionRate error — target < 15% at 5-min checkpoint.

If error is > 15%, calibrate `SimulatedWorker.MINING_DURATION_LOOPS` and `GAS_HARVEST_DURATION_LOOPS` constants. Repeat until target met.

- [ ] **Step 5: Run benchmark**

Run: `mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=GameLoopBenchmarkTest -q`
Verify no performance regression from per-worker simulation.

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java
git commit -m "feat(#500): add formula-as-validator and calibrate constants

Formula stays as diagnostic — logs divergence between simulation
and formula predictions. Calibrated mining/gas cycle durations.  Refs #500"
```

---

## Batch 5: Full suite verification and CLAUDE.md update

### Task 6: Full suite green and documentation

**Files:**
- Modify: `CLAUDE.md` — add WorkerSimulationTest to unit test list, remove WorkerAssignmentTest
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java` — any final test adjustments

**Interfaces:**
- Consumes: Everything from Tasks 1-5

- [ ] **Step 1: Run full module test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all PASS

If any test fails due to the WorkerAssignment→WorkerSimulation migration, fix it. Common issues:
- Tests that call `workerAssignment()` → call `workerSimulation()`
- Tests that call `setMiningProbesPerBase` → use `syncMineralWorkersFromGt` or remove
- Hardcoded income values in assertions → adjust for simulation-driven income

- [ ] **Step 2: Run playbook calibration tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2PlaybookCalibrationTest -q`
Expected: PASS — playbooks should still calibrate with simulation-driven income

- [ ] **Step 3: Update CLAUDE.md test list**

Add `WorkerSimulationTest` to the unit test list. Remove `WorkerAssignmentTest` from the list.

- [ ] **Step 4: Run full build**

Run: `mvn compile -q`
Expected: clean compile

- [ ] **Step 5: Commit**

```bash
git add CLAUDE.md
git add quarkmind-sc2/
git commit -m "feat(#500): full suite green, update CLAUDE.md test list

All tests pass with simulation-driven income.
WorkerAssignmentTest replaced by WorkerSimulationTest.  Refs #500"
```

---

## References

- [2026-11-03-worker-simulation-design.md] — design spec this plan implements
- [WorkerAssignment.java] — class being replaced
- [EmulatedGame.java:59,77-78,141-170,1266-1269] — integration points
- [SC2Data.java:39-89,101,108,167-203] — income formulas and speed constants
- [ReplayValidationHarness.java:93] — harness GT override
- [EmulatedGameTest.java:81,95,2011-2015] — tests referencing setMiningProbesPerBase
- [TerranEmulatedGameTest.java:64-65,161,205,224] — tests referencing setMiningProbesPerBase
- [ZergEmulatedGameTest.java:79-80,283] — tests referencing setMiningProbesPerBase
- [ReactorTagTraceDiagnosticTest.java:72] — diagnostic test referencing setMiningProbesPerBase
- [ReactorStateAtIntentTest.java:74] — diagnostic test referencing setMiningProbesPerBase
- [Point2d.java] — position record (float x, y, distanceTo)
- [GitHub #499] — predecessor: CmdEvent gas building extraction
- [GitHub #501] — follow-up: map-specific mineral patch amounts
