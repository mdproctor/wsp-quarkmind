# Parallel Training Queues Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-building training queues (max 5) with supply reserved at queue time, and make EnemyBehavior use real building tags from `enemy.buildings`.

**Architecture:** Extend `PlayerState` with two maps (`buildingQueues`, `buildingTrainingUntil`). Rewrite `handleTrain` to validate the building, reserve supply immediately, and start or enqueue. Add `drainBuildingQueues()` to `tick()` to chain completions. Update `EnemyBehavior` to look up the real building tag via `SC2Data.trainedBy()`.

**Tech Stack:** Plain Java (no CDI, no Quarkus), JUnit 5 + AssertJ, IntelliJ refactor MCPs.

---

## File Map

| File | Change |
|------|--------|
| `src/main/java/io/quarkmind/sc2/intent/TrainIntent.java` | Rename field `unitTag → buildingTag` (IntelliJ) |
| `src/main/java/io/quarkmind/domain/SC2Data.java` | Add `trainedBy(UnitType) → BuildingType` |
| `src/main/java/io/quarkmind/sc2/emulated/PlayerState.java` | Add `buildingQueues`, `buildingTrainingUntil`, update `clear()` |
| `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` | Rewrite `handleTrain`, add `startTraining`, `drainBuildingQueues`, update `tick()`, add test helpers |
| `src/main/java/io/quarkmind/sc2/emulated/EnemyBehavior.java` | Replace fake tag with real building lookup |
| `src/test/java/io/quarkmind/domain/SC2DataTest.java` | Add `trainedByMapsCorrectly` test |
| `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` | 6 new tests; update `enemyBehavior_trainsUnitsWhenMineralsAccumulate` |
| `docs/roadmap-sc2-engine.md` | Remove "Parallel training queues" from Phase 5 gaps |

---

## Task 1: Rename `TrainIntent.unitTag` → `buildingTag`

The field holds a building tag, not a unit tag. Rename now so all subsequent tasks use the correct name.

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/intent/TrainIntent.java` (via IntelliJ refactor)

- [ ] **Step 1: Locate the symbol**

Open `src/main/java/io/quarkmind/sc2/intent/TrainIntent.java`. The record is:
```java
public record TrainIntent(String unitTag, UnitType unitType) implements Intent {}
```
The rename target is `unitTag` at line 5, column 35 (the field declaration in the record).

- [ ] **Step 2: Run ide_refactor_rename**

```
ide_refactor_rename(
  file: "src/main/java/io/quarkmind/sc2/intent/TrainIntent.java",
  line: 5,
  column: 35,
  newName: "buildingTag"
)
```

Expected: IntelliJ updates `TrainIntent.java` and all accessor call sites (`t.unitTag()` → `t.buildingTag()`) across the project. Constructor call sites (`new TrainIntent("nexus-0", ...)`) are positional — they do NOT change.

- [ ] **Step 3: Verify the record**

`TrainIntent.java` should now read:
```java
public record TrainIntent(String buildingTag, UnitType unitType) implements Intent {}
```

- [ ] **Step 4: Build to confirm no compile errors**

```bash
mvn compile -q
```
Expected: `BUILD SUCCESS`

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "refactor: rename TrainIntent.unitTag to buildingTag

The field holds the building doing the training, not a unit tag.
Accessor call sites updated by IntelliJ rename refactor.

Refs #128"
```

---

## Task 2: Add `SC2Data.trainedBy(UnitType)`

TDD: test first, then implement.

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/SC2Data.java`
- Modify: `src/test/java/io/quarkmind/domain/SC2DataTest.java`

- [ ] **Step 1: Write the failing test**

Add to `SC2DataTest.java`:
```java
@Test
void trainedByMapsKnownUnits() {
    assertThat(SC2Data.trainedBy(UnitType.PROBE))    .isEqualTo(BuildingType.NEXUS);
    assertThat(SC2Data.trainedBy(UnitType.ZEALOT))   .isEqualTo(BuildingType.GATEWAY);
    assertThat(SC2Data.trainedBy(UnitType.STALKER))  .isEqualTo(BuildingType.GATEWAY);
    assertThat(SC2Data.trainedBy(UnitType.IMMORTAL)) .isEqualTo(BuildingType.ROBOTICS_FACILITY);
    assertThat(SC2Data.trainedBy(UnitType.OBSERVER)) .isEqualTo(BuildingType.ROBOTICS_FACILITY);
    assertThat(SC2Data.trainedBy(UnitType.MARINE))   .isEqualTo(BuildingType.BARRACKS);
    assertThat(SC2Data.trainedBy(UnitType.MARAUDER)) .isEqualTo(BuildingType.BARRACKS);
    assertThat(SC2Data.trainedBy(UnitType.ZERGLING)) .isEqualTo(BuildingType.HATCHERY);
    assertThat(SC2Data.trainedBy(UnitType.ROACH))    .isEqualTo(BuildingType.HATCHERY);
    assertThat(SC2Data.trainedBy(UnitType.HYDRALISK)).isEqualTo(BuildingType.HATCHERY);
}
```

- [ ] **Step 2: Run to confirm failure**

```bash
mvn test -Dtest=SC2DataTest#trainedByMapsKnownUnits -q
```
Expected: FAIL — `cannot find symbol: method trainedBy(UnitType)`

- [ ] **Step 3: Implement `trainedBy` in `SC2Data.java`**

Add after `gasCost(UnitType)`:
```java
public static BuildingType trainedBy(UnitType type) {
    return switch (type) {
        case PROBE                                  -> BuildingType.NEXUS;
        case ZEALOT, STALKER, ADEPT, ARCHON        -> BuildingType.GATEWAY;
        case IMMORTAL, OBSERVER, COLOSSUS,
             DISRUPTOR                             -> BuildingType.ROBOTICS_FACILITY;
        case PHOENIX, ORACLE, VOID_RAY,
             CARRIER, TEMPEST, MOTHERSHIP          -> BuildingType.STARGATE;
        case SCV                                   -> BuildingType.COMMAND_CENTER;
        case MARINE, MARAUDER                      -> BuildingType.BARRACKS;
        case MEDIVAC, VIKING                       -> BuildingType.STARPORT;
        case HELLION                               -> BuildingType.FACTORY;
        case DRONE, ZERGLING, ROACH, HYDRALISK,
             MUTALISK, BANELING, ULTRALISK,
             OVERLORD, OVERSEER                    -> BuildingType.HATCHERY;
        default                                    -> BuildingType.NEXUS;
    };
}
```

- [ ] **Step 4: Run test to confirm pass**

```bash
mvn test -Dtest=SC2DataTest#trainedByMapsKnownUnits -q
```
Expected: `BUILD SUCCESS`, test passes.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/quarkmind/domain/SC2Data.java \
        src/test/java/io/quarkmind/domain/SC2DataTest.java
git commit -m "feat: add SC2Data.trainedBy(UnitType) mapping unit to trainer building

Required by EnemyBehavior to look up real building tags for TrainIntents.

Refs #128"
```

---

## Task 3: Extend `PlayerState` and add test helpers

Add the per-building queue fields, update `clear()`, and add three test helpers to `EmulatedGame`.

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/PlayerState.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`

- [ ] **Step 1: Add fields to `PlayerState`**

Add after the `pendingCompletions` field declaration and update `clear()`:

```java
// Add imports at top of file:
import java.util.ArrayDeque;
import java.util.Deque;

// Add after pendingCompletions field:
final Map<String, Deque<UnitType>> buildingQueues       = new HashMap<>();
final Map<String, Long>            buildingTrainingUntil = new HashMap<>();
```

In `clear()`, add after `pendingCompletions.clear()`:
```java
buildingQueues.clear();
buildingTrainingUntil.clear();
```

- [ ] **Step 2: Add three test helpers to `EmulatedGame`**

Add near the other test helpers (around line 564):

```java
/** Adds a complete friendly building — for tests that need a specific building to train from. */
Building spawnBuildingForTesting(BuildingType type, Point2d position) {
    String tag = "bldg-test-" + nextTag++;
    Building b = new Building(tag, type, position,
        SC2Data.maxBuildingHealth(type), SC2Data.maxBuildingHealth(type), true);
    friendly.buildings.add(b);
    return b;
}

/** Adds a complete enemy building — for tests that seed enemy production infrastructure. */
void spawnEnemyBuildingForTesting(BuildingType type, Point2d position) {
    String tag = "enemy-bldg-test-" + nextTag++;
    int hp = SC2Data.maxBuildingHealth(type);
    enemy.buildings.add(new Building(tag, type, position, hp, hp, true));
}

/** Sets friendly supply caps — for tests that need more than the 3 default free supply. */
void setSupplyForTesting(int supply, int supplyUsed) {
    friendly.supply     = supply;
    friendly.supplyUsed = supplyUsed;
}
```

- [ ] **Step 3: Build to confirm no compile errors**

```bash
mvn compile -q
```
Expected: `BUILD SUCCESS`

- [ ] **Step 4: Run existing tests to confirm nothing broke**

```bash
mvn test -Dtest=EmulatedGameTest -q
```
Expected: all existing tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/PlayerState.java \
        src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git commit -m "feat: add PlayerState training queue fields and EmulatedGame test helpers

buildingQueues and buildingTrainingUntil support per-building
queue tracking. spawnBuildingForTesting / spawnEnemyBuildingForTesting /
setSupplyForTesting helpers enable the new queue tests.

Refs #128"
```

---

## Task 4: Write all failing tests

Write the six new tests and update the existing enemy test. All should fail (or produce wrong assertions) with the current `handleTrain` implementation.

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

Add a `// ---- #128: parallel training queues ----` section. All imports needed:
```java
import io.quarkmind.domain.Building;
import io.quarkmind.domain.BuildingType;
// (these may already be imported — check and add only if missing)
```

- [ ] **Step 1: Write `parallelTrainingTwoGateways`**

```java
@Test
void parallelTrainingTwoGateways() {
    game.setMineralsForTesting(500);
    game.setSupplyForTesting(200, 0);
    Building gw1 = game.spawnBuildingForTesting(BuildingType.GATEWAY, new Point2d(20, 20));
    Building gw2 = game.spawnBuildingForTesting(BuildingType.GATEWAY, new Point2d(22, 20));
    int before = game.snapshot().myUnits().size();

    game.applyIntent(new TrainIntent(gw1.tag(), UnitType.ZEALOT));
    game.applyIntent(new TrainIntent(gw2.tag(), UnitType.ZEALOT));

    for (int i = 0; i < 28; i++) game.tick(); // Zealot = 28 ticks
    assertThat(game.snapshot().myUnits()).hasSize(before + 2);
}
```

- [ ] **Step 2: Write `supplyReservedAtQueueTime`**

```java
@Test
void supplyReservedAtQueueTime() {
    game.setMineralsForTesting(500);
    // Initial: supply=15, supplyUsed=12 → 3 free; PROBE costs 1 supply
    assertThat(game.snapshot().supplyUsed()).isEqualTo(12);

    game.applyIntent(new TrainIntent("nexus-0", UnitType.PROBE)); // starts training
    assertThat(game.snapshot().supplyUsed()).isEqualTo(13); // reserved immediately

    game.applyIntent(new TrainIntent("nexus-0", UnitType.PROBE)); // queued behind first
    assertThat(game.snapshot().supplyUsed()).isEqualTo(14); // queued unit also reserves supply

    // No ticks yet — both increments are at queue time, not completion time
}
```

- [ ] **Step 3: Write `queueFullRejected`**

```java
@Test
void queueFullRejected() {
    game.setMineralsForTesting(5000);
    game.setSupplyForTesting(200, 0);

    // Fill queue to 5 total (1 training + 4 waiting)
    for (int i = 0; i < 5; i++) {
        game.applyIntent(new TrainIntent("nexus-0", UnitType.PROBE));
    }
    assertThat(game.snapshot().supplyUsed()).isEqualTo(5); // 5 probes at 1 supply each

    // 6th must be rejected — supply unchanged
    game.applyIntent(new TrainIntent("nexus-0", UnitType.PROBE));
    assertThat(game.snapshot().supplyUsed()).isEqualTo(5);
}
```

- [ ] **Step 4: Write `queueDrainsSequentially`**

```java
@Test
void queueDrainsSequentially() {
    game.setMineralsForTesting(500);
    // supply=15, used=12, free=3 — enough for 3 probes at 1 each
    int before = game.snapshot().myUnits().size();

    game.applyIntent(new TrainIntent("nexus-0", UnitType.PROBE)); // starts (12 ticks)
    game.applyIntent(new TrainIntent("nexus-0", UnitType.PROBE)); // queued

    for (int i = 0; i < 12; i++) game.tick(); // first completes
    assertThat(game.snapshot().myUnits()).hasSize(before + 1);

    for (int i = 0; i < 12; i++) game.tick(); // second completes (started right after first)
    assertThat(game.snapshot().myUnits()).hasSize(before + 2);
}
```

- [ ] **Step 5: Write `buildingValidationRejectsUnknownTag`**

```java
@Test
void buildingValidationRejectsUnknownTag() {
    game.setMineralsForTesting(500);
    int mineralsBefore = (int) game.snapshot().minerals();

    game.applyIntent(new TrainIntent("no-such-building", UnitType.ZEALOT));

    assertThat(game.snapshot().minerals()).isEqualTo(mineralsBefore); // no deduction
    assertThat(game.snapshot().supplyUsed()).isEqualTo(12);           // no supply change
}
```

- [ ] **Step 6: Write `enemyUsesRealBuildingTag`**

```java
@Test
void enemyUsesRealBuildingTagFromEnemyBuildings() {
    // setEnemyStrategy uses a permissive TechTree — we must also seed the
    // training building since EnemyBehavior now looks it up from enemy.buildings.
    game.setEnemyStrategy(new FixedBuildOrderStrategy("TEST", Race.PROTOSS,
        List.of(UnitType.ZEALOT), 10, new EnemyAttackConfig(3, 200, 0, 0)));
    game.reset();
    // Seed a Gateway so EnemyBehavior can find it via SC2Data.trainedBy(ZEALOT)=GATEWAY
    game.spawnEnemyBuildingForTesting(BuildingType.GATEWAY, new Point2d(52, 51));
    game.enemy.minerals = 200;

    int zealotTrainTime = SC2Data.trainTimeInTicks(UnitType.ZEALOT);
    for (int i = 0; i < zealotTrainTime + 5; i++) game.tick();

    boolean hasZealot = game.enemy.stagingArea.stream().anyMatch(u -> u.type() == UnitType.ZEALOT)
        || game.enemy.units.stream().anyMatch(u -> u.type() == UnitType.ZEALOT);
    assertThat(hasZealot).isTrue();
}
```

- [ ] **Step 7: Update the existing `enemyBehavior_trainsUnitsWhenMineralsAccumulate` test**

The existing test also uses `setEnemyStrategy` (permissive TechTree) and trains a Zealot, but doesn't seed a Gateway. After our change it will fail because EnemyBehavior needs a real Gateway. Update it to also seed the building:

Replace the existing test body:
```java
@Test
void enemyBehavior_trainsUnitsWhenMineralsAccumulate() {
    game.setEnemyStrategy(new FixedBuildOrderStrategy("TEST", Race.PROTOSS,
        List.of(UnitType.ZEALOT), 10, new EnemyAttackConfig(3, 200, 0, 0)));
    game.reset();
    // Seed Gateway: EnemyBehavior now looks up real building tags.
    // setEnemyStrategy uses a permissive TechTree so no BuildIntent fires,
    // but trainedBy(ZEALOT)=GATEWAY means we must have one in enemy.buildings.
    game.spawnEnemyBuildingForTesting(BuildingType.GATEWAY, new Point2d(52, 51));
    game.enemy.minerals = 200;

    int zealotTrainTime = SC2Data.trainTimeInTicks(UnitType.ZEALOT);
    for (int i = 0; i < zealotTrainTime + 5; i++) game.tick();

    boolean hasZealot = game.enemy.stagingArea.stream().anyMatch(u -> u.type() == UnitType.ZEALOT)
        || game.enemy.units.stream().anyMatch(u -> u.type() == UnitType.ZEALOT);
    assertThat(hasZealot).isTrue();
}
```

- [ ] **Step 8: Run all new tests to confirm they fail for the right reasons**

```bash
mvn test -Dtest=EmulatedGameTest#parallelTrainingTwoGateways+supplyReservedAtQueueTime+queueFullRejected+queueDrainsSequentially+buildingValidationRejectsUnknownTag+enemyUsesRealBuildingTagFromEnemyBuildings -q 2>&1 | tail -20
```
Expected: failures related to wrong supply counts or wrong unit counts — not compilation errors.

---

## Task 5: Implement `handleTrain` and `startTraining`

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`

- [ ] **Step 1: Replace `handleTrain` with the new implementation**

Replace the entire `handleTrain` method:
```java
private void handleTrain(TrainIntent t, PlayerState state) {
    String buildingTag = t.buildingTag();
    boolean buildingReady = state.buildings.stream()
        .anyMatch(b -> b.tag().equals(buildingTag) && b.isComplete());
    if (!buildingReady) {
        log.debugf("[EMULATED] Train rejected — building %s not ready", buildingTag);
        return;
    }
    int mCost = SC2Data.mineralCost(t.unitType());
    int gCost = SC2Data.gasCost(t.unitType());
    int sCost = SC2Data.supplyCost(t.unitType());
    if ((int) state.minerals < mCost || state.vespene < gCost
            || state.supplyUsed + sCost > state.supply) {
        log.debugf("[EMULATED] Cannot train %s — insufficient resources", t.unitType());
        return;
    }
    boolean isBusy = state.buildingTrainingUntil.containsKey(buildingTag);
    int total = (isBusy ? 1 : 0)
        + state.buildingQueues.getOrDefault(buildingTag, new ArrayDeque<>()).size();
    if (total >= 5) {
        log.debugf("[EMULATED] Train rejected — building %s queue full", buildingTag);
        return;
    }
    state.supplyUsed += sCost;
    state.minerals   -= mCost;
    state.vespene    -= gCost;
    if (!isBusy) {
        startTraining(buildingTag, t.unitType(), state);
    } else {
        state.buildingQueues.computeIfAbsent(buildingTag, k -> new ArrayDeque<>())
            .add(t.unitType());
    }
}
```

- [ ] **Step 2: Add `startTraining` method**

Add directly after `handleTrain`:
```java
private void startTraining(String buildingTag, UnitType unitType, PlayerState state) {
    boolean isEnemy  = (state == enemy);
    long completesAt = gameFrame + SC2Data.trainTimeInTicks(unitType);
    state.buildingTrainingUntil.put(buildingTag, completesAt);
    state.pendingCompletions.add(new PlayerState.PendingCompletion(completesAt, () -> {
        state.buildingTrainingUntil.remove(buildingTag);
        String tag = "unit-" + nextTag++;
        int hp = SC2Data.maxHealth(unitType);
        state.units.add(new Unit(tag, unitType, new Point2d(9, 9), hp, hp,
            SC2Data.maxShields(unitType), SC2Data.maxShields(unitType), 0, 0));
        log.debugf("[EMULATED] Trained %s (tag=%s)", unitType, tag);
        if (isEnemy && enemyBehavior != null) {
            enemyBehavior.notifyUnitTrained();
        }
    }));
}
```

Add `import java.util.ArrayDeque;` at the top of `EmulatedGame.java` if not already present.

- [ ] **Step 3: Build to confirm no compile errors**

```bash
mvn compile -q
```
Expected: `BUILD SUCCESS`

- [ ] **Step 4: Run the new tests (expect most to pass, sequential drain may still fail)**

```bash
mvn test -Dtest=EmulatedGameTest#parallelTrainingTwoGateways+supplyReservedAtQueueTime+queueFullRejected+buildingValidationRejectsUnknownTag -q
```
Expected: these four pass. `queueDrainsSequentially` still fails — drain not wired yet.

- [ ] **Step 5: Run existing training tests to confirm no regressions**

```bash
mvn test -Dtest=EmulatedGameTest#trainIntentDeductsMinerals+trainedUnitAppearsAfterBuildTime+trainBlockedIfInsufficientMinerals -q
```
Expected: all pass.

---

## Task 6: Add `drainBuildingQueues` and wire into `tick()`

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`

- [ ] **Step 1: Add `drainBuildingQueues` method**

Add after `startTraining`:
```java
private void drainBuildingQueues(PlayerState state) {
    for (String buildingTag : new ArrayList<>(state.buildingQueues.keySet())) {
        if (state.buildingTrainingUntil.containsKey(buildingTag)) continue;
        Deque<UnitType> queue = state.buildingQueues.get(buildingTag);
        if (queue == null || queue.isEmpty()) {
            state.buildingQueues.remove(buildingTag);
            continue;
        }
        UnitType next = queue.poll();
        if (queue.isEmpty()) state.buildingQueues.remove(buildingTag);
        startTraining(buildingTag, next, state);
    }
}
```

Add `import java.util.Deque;` at the top if not already present.

- [ ] **Step 2: Wire into `tick()` after `fireCompletions`**

In `tick()`, after the two `fireCompletions` calls:
```java
friendly.fireCompletions(gameFrame);
enemy.fireCompletions(gameFrame);
drainBuildingQueues(friendly);   // ← add these two lines
drainBuildingQueues(enemy);      // ← add these two lines
```

- [ ] **Step 3: Run the sequential drain test**

```bash
mvn test -Dtest=EmulatedGameTest#queueDrainsSequentially -q
```
Expected: PASS.

- [ ] **Step 4: Run all six new tests together**

```bash
mvn test -Dtest=EmulatedGameTest#parallelTrainingTwoGateways+supplyReservedAtQueueTime+queueFullRejected+queueDrainsSequentially+buildingValidationRejectsUnknownTag+enemyUsesRealBuildingTagFromEnemyBuildings -q
```
Expected: first five pass; `enemyUsesRealBuildingTagFromEnemyBuildings` may still fail (EnemyBehavior not updated yet).

---

## Task 7: Update `EnemyBehavior` to use real building tags

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EnemyBehavior.java`

- [ ] **Step 1: Replace fake tag generation in `tickProduction`**

Find and replace this block (around line 143–149):
```java
// BEFORE
String buildingTag = "enemy-nexus-" + nextTag++;
queue.add(new TrainIntent(buildingTag, target));
trainingPending = true;
currentTarget = Optional.empty();
log.debugf("[ENEMY] Queued TrainIntent: %s (cost=%d, minerals_left=%.0f)",
    target, cost, enemy.minerals);
```

Replace with:
```java
// AFTER
BuildingType needed = SC2Data.trainedBy(target);
Optional<Building> trainer = enemy.buildings.stream()
    .filter(b -> b.isComplete() && b.type() == needed)
    .findFirst();
if (trainer.isEmpty()) {
    log.debugf("[ENEMY] No ready %s to train %s — waiting", needed, target);
    return;
}
queue.add(new TrainIntent(trainer.get().tag(), target));
trainingPending = true;
currentTarget = Optional.empty();
log.debugf("[ENEMY] Queued TrainIntent: %s from %s (cost=%d, minerals_left=%.0f)",
    target, trainer.get().tag(), cost, enemy.minerals);
```

Ensure `import io.quarkmind.domain.SC2Data;` is at the top (it should already be present via `SC2Data.mineralCost` usages — check and add only if missing).

- [ ] **Step 2: Build**

```bash
mvn compile -q
```
Expected: `BUILD SUCCESS`

- [ ] **Step 3: Run all six new tests**

```bash
mvn test -Dtest=EmulatedGameTest#parallelTrainingTwoGateways+supplyReservedAtQueueTime+queueFullRejected+queueDrainsSequentially+buildingValidationRejectsUnknownTag+enemyUsesRealBuildingTagFromEnemyBuildings -q
```
Expected: all six pass.

- [ ] **Step 4: Run the updated existing enemy test**

```bash
mvn test -Dtest=EmulatedGameTest#enemyBehavior_trainsUnitsWhenMineralsAccumulate -q
```
Expected: PASS.

---

## Task 8: Full suite, doc update, and commit

- [ ] **Step 1: Run the complete test suite**

```bash
mvn test -q 2>&1 | tail -20
```
Expected: `BUILD SUCCESS`, 0 failures.

If there are failures, investigate and fix before proceeding. Common causes:
- A test that seeds TrainIntents using building tags that don't exist in `buildings` (need `spawnBuildingForTesting`)
- Enemy training tests that don't seed the required building type (need `spawnEnemyBuildingForTesting`)

- [ ] **Step 2: Update `docs/roadmap-sc2-engine.md`**

In the Phase 5 "Not yet implemented" section, remove:
```
- Parallel training queues (one per building)
```

The section should now read (if mineral saturation and auto-engage are still open):
```
**Not yet implemented:**
- Friendly auto-engage (units stop and fight without explicit `AttackIntent`)
- Realistic worker mineral collection model (currently flat +5/tick)
```

- [ ] **Step 3: Commit everything**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/main/java/io/quarkmind/sc2/emulated/EnemyBehavior.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java \
        docs/roadmap-sc2-engine.md
git commit -m "feat: parallel training queues with supply reservation at queue time

Each building now maintains its own training queue (max 5 total including
active unit). Supply is reserved when the unit enters the queue, not when
it completes training. EnemyBehavior now looks up real building tags from
enemy.buildings via SC2Data.trainedBy() instead of generating fake tags.

Building validation enforced: TrainIntents for unknown or incomplete
buildings are silently dropped.

Closes #128"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|-----------------|------|
| Per-building training queue (max 5) | Tasks 5–6 |
| Supply reserved at queue time | Task 5 (`handleTrain` step 4) |
| Full building validation | Task 5 (step 1 of handleTrain) |
| Enemy uses real building tags | Task 7 |
| `trainedBy(UnitType)` added to domain | Task 2 |
| `TrainIntent.unitTag → buildingTag` rename | Task 1 |
| New tests (6) | Task 4 |
| Updated enemy test | Task 4 step 7 |
| `docs/roadmap-sc2-engine.md` updated | Task 8 step 2 |

All spec requirements covered. ✅

**Placeholder scan:** No TBDs, no "similar to above", all code blocks complete. ✅

**Type consistency:**
- `startTraining(String buildingTag, UnitType unitType, PlayerState state)` — used consistently in Tasks 5, 6
- `drainBuildingQueues(PlayerState state)` — used in Task 6
- `buildingQueues: Map<String, Deque<UnitType>>` — consistent across Tasks 3, 5, 6
- `buildingTrainingUntil: Map<String, Long>` — consistent across Tasks 3, 5, 6
- `t.buildingTag()` — used after rename in Task 1 ✅
