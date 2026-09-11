# MACRO to ALL_IN Pivot Detection — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #305 — Detect MACRO to ALL_IN pivot (expansion sacrifice)
**Issue group:** #305

**Goal:** Enable MACRO → ALL_IN posture reversion when the enemy sacrifices their expansion, using a two-tier detection system (suspected from unit heuristic, confirmed from building observation).

**Architecture:** ScoutingSessionManager gains a `processBuildings()` method that tracks confirmed expansion buildings by tag. When `hasEverConfirmed` is true, DroolsScoutingTask bypasses DRL posture rules and resolves posture directly from confirmed expansion state. Fog-of-war is handled by sticky-until-contradicted semantics.

**Tech Stack:** Java 21, Quarkus, Drools Rule Units, JUnit 5, AssertJ

## Global Constraints

- Domain model (`domain/`) must remain plain Java — no CDI
- `SimulatedGame` is the living specification — test scenarios use it
- Spatial constants must be calibrated from replay data (protocol `sc2data-spatial-constants-require-calibration.md`)
- Package-private static methods on CDI beans tested from the same package without CDI

---

## Batch 1: Core — ScoutingSessionManager Tier 2 expansion detection

### Task 1: Add processBuildings() with confirmed expansion tracking

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/ScoutingSessionManager.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/ScoutingSessionManagerTest.java` (new or existing)

**Interfaces:**
- Consumes: `List<Building> enemyBuildings` from `GameState.enemyBuildings()`, `Point2d estimatedEnemyBase`, `List<Unit> friendlyUnits`, `List<Building> friendlyBuildings`
- Produces: `confirmedExpansionCount()`, `hasEverConfirmed()` — consumed by DroolsScoutingTask in Task 2

- [ ] **Step 1: Write the failing test — base-type building at non-starting location is confirmed**

```java
@Test
void processBuildings_confirmsExpansionAtNonStartingLocation() {
    ScoutingSessionManager mgr = new ScoutingSessionManager();
    Point2d enemyBase = new Point2d(224, 224);
    Building expansion = new Building("nexus-1", BuildingType.NEXUS,
        new Point2d(180, 180), 1000, 1000, true);
    Building mainBase = new Building("nexus-0", BuildingType.NEXUS,
        new Point2d(224, 224), 1000, 1000, true);

    mgr.processBuildings(List.of(mainBase, expansion), enemyBase,
        List.of(), List.of());

    assertThat(mgr.hasEverConfirmed()).isTrue();
    assertThat(mgr.confirmedExpansionCount()).isEqualTo(1);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ScoutingSessionManagerTest#processBuildings_confirmsExpansionAtNonStartingLocation -q`
Expected: FAIL — `processBuildings` method does not exist

- [ ] **Step 3: Write minimal implementation**

Add to `ScoutingSessionManager`:

```java
public static final float CONFIRMED_EXPANSION_DISTANCE = 25f;
private static final float VISION_RANGE = 9.0f;

private static final Set<BuildingType> BASE_TYPES = Set.of(
    BuildingType.NEXUS, BuildingType.HATCHERY, BuildingType.COMMAND_CENTER,
    BuildingType.LAIR, BuildingType.HIVE, BuildingType.ORBITAL_COMMAND,
    BuildingType.PLANETARY_FORTRESS);

private final Map<String, Point2d> confirmedExpansions = new LinkedHashMap<>();
private boolean hasEverConfirmed = false;
private Point2d confirmedMainBase = null;

public void processBuildings(List<Building> enemyBuildings, Point2d estimatedEnemyBase,
                             List<Unit> friendlyUnits, List<Building> friendlyBuildings) {
    List<Building> baseBldgs = enemyBuildings.stream()
        .filter(b -> BASE_TYPES.contains(b.type()))
        .toList();

    if (confirmedMainBase == null) {
        baseBldgs.stream()
            .filter(b -> b.position().distanceTo(estimatedEnemyBase) <= EXPANSION_DISTANCE_THRESHOLD)
            .min(Comparator.comparingDouble(b -> b.position().distanceTo(estimatedEnemyBase)))
            .ifPresent(b -> confirmedMainBase = b.position());
    }

    Point2d reference = confirmedMainBase != null ? confirmedMainBase : estimatedEnemyBase;
    float threshold = confirmedMainBase != null ? CONFIRMED_EXPANSION_DISTANCE : EXPANSION_DISTANCE_THRESHOLD;

    for (Building b : baseBldgs) {
        if (b.position().distanceTo(reference) > threshold) {
            confirmedExpansions.put(b.tag(), b.position());
            hasEverConfirmed = true;
        }
    }

    confirmedExpansions.entrySet().removeIf(entry -> {
        String tag = entry.getKey();
        Point2d location = entry.getValue();
        boolean buildingPresent = enemyBuildings.stream().anyMatch(b -> b.tag().equals(tag));
        if (!buildingPresent && hasVisionOf(location, friendlyUnits, friendlyBuildings)) {
            return true;
        }
        return false;
    });
}

private boolean hasVisionOf(Point2d location, List<Unit> friendlyUnits,
                            List<Building> friendlyBuildings) {
    for (Unit u : friendlyUnits) {
        if (u.position().distanceTo(location) < VISION_RANGE) return true;
    }
    for (Building b : friendlyBuildings) {
        if (b.position().distanceTo(location) < VISION_RANGE) return true;
    }
    return false;
}

public int confirmedExpansionCount() { return confirmedExpansions.size(); }
public boolean hasEverConfirmed() { return hasEverConfirmed; }
```

Add to `reset()`:
```java
confirmedExpansions.clear();
hasEverConfirmed = false;
confirmedMainBase = null;
```

Add imports: `java.util.Comparator`, `java.util.LinkedHashMap`, `java.util.Map`, `java.util.Set` (some may already exist), `io.quarkmind.domain.Building`, `io.quarkmind.domain.BuildingType`.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ScoutingSessionManagerTest#processBuildings_confirmsExpansionAtNonStartingLocation -q`
Expected: PASS

- [ ] **Step 5: Write remaining tests**

```java
@Test
void processBuildings_ignoresMainBaseBuilding() {
    ScoutingSessionManager mgr = new ScoutingSessionManager();
    Point2d enemyBase = new Point2d(224, 224);
    Building mainBase = new Building("nexus-0", BuildingType.NEXUS,
        new Point2d(224, 224), 1000, 1000, true);

    mgr.processBuildings(List.of(mainBase), enemyBase, List.of(), List.of());

    assertThat(mgr.hasEverConfirmed()).isFalse();
    assertThat(mgr.confirmedExpansionCount()).isEqualTo(0);
}

@Test
void processBuildings_retractsWhenBuildingAbsentAndVisionAvailable() {
    ScoutingSessionManager mgr = new ScoutingSessionManager();
    Point2d enemyBase = new Point2d(224, 224);
    Point2d expansionLoc = new Point2d(180, 180);
    Building mainBase = new Building("nexus-0", BuildingType.NEXUS,
        new Point2d(224, 224), 1000, 1000, true);
    Building expansion = new Building("nexus-1", BuildingType.NEXUS,
        expansionLoc, 1000, 1000, true);

    mgr.processBuildings(List.of(mainBase, expansion), enemyBase,
        List.of(), List.of());
    assertThat(mgr.confirmedExpansionCount()).isEqualTo(1);

    Unit friendlyNearExpansion = new Unit("stalker-1", UnitType.STALKER,
        new Point2d(182, 182), 160, 160, 80, 80, 0, 0);
    mgr.processBuildings(List.of(mainBase), enemyBase,
        List.of(friendlyNearExpansion), List.of());

    assertThat(mgr.confirmedExpansionCount())
        .as("Expansion retracted — building absent + vision available")
        .isEqualTo(0);
}

@Test
void processBuildings_retainsWhenBuildingAbsentButNoVision() {
    ScoutingSessionManager mgr = new ScoutingSessionManager();
    Point2d enemyBase = new Point2d(224, 224);
    Building mainBase = new Building("nexus-0", BuildingType.NEXUS,
        new Point2d(224, 224), 1000, 1000, true);
    Building expansion = new Building("nexus-1", BuildingType.NEXUS,
        new Point2d(180, 180), 1000, 1000, true);

    mgr.processBuildings(List.of(mainBase, expansion), enemyBase,
        List.of(), List.of());
    assertThat(mgr.confirmedExpansionCount()).isEqualTo(1);

    mgr.processBuildings(List.of(mainBase), enemyBase,
        List.of(), List.of());

    assertThat(mgr.confirmedExpansionCount())
        .as("Expansion retained — no vision of location (fog)")
        .isEqualTo(1);
}

@Test
void processBuildings_mainBaseIsClosestToEstimate() {
    ScoutingSessionManager mgr = new ScoutingSessionManager();
    Point2d enemyBase = new Point2d(224, 224);
    Building natural = new Building("nexus-nat", BuildingType.NEXUS,
        new Point2d(200, 200), 1000, 1000, true);
    Building mainBase = new Building("nexus-main", BuildingType.NEXUS,
        new Point2d(223, 223), 1000, 1000, true);

    mgr.processBuildings(List.of(natural, mainBase), enemyBase,
        List.of(), List.of());

    assertThat(mgr.hasEverConfirmed())
        .as("Natural at (200,200) is confirmed as expansion — beyond 25f from main at (223,223)")
        .isTrue();
    assertThat(mgr.confirmedExpansionCount()).isEqualTo(1);
}

@Test
void reset_clearsConfirmedExpansions() {
    ScoutingSessionManager mgr = new ScoutingSessionManager();
    Point2d enemyBase = new Point2d(224, 224);
    Building mainBase = new Building("nexus-0", BuildingType.NEXUS,
        new Point2d(224, 224), 1000, 1000, true);
    Building expansion = new Building("nexus-1", BuildingType.NEXUS,
        new Point2d(180, 180), 1000, 1000, true);

    mgr.processBuildings(List.of(mainBase, expansion), enemyBase,
        List.of(), List.of());
    assertThat(mgr.hasEverConfirmed()).isTrue();

    mgr.reset();

    assertThat(mgr.hasEverConfirmed()).isFalse();
    assertThat(mgr.confirmedExpansionCount()).isEqualTo(0);
}
```

- [ ] **Step 6: Run all tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ScoutingSessionManagerTest -q`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/ScoutingSessionManager.java
git add quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/ScoutingSessionManagerTest.java
git commit -m "feat: two-tier expansion detection — processBuildings with fog-of-war retraction

Refs #305"
```

---

## Batch 2: Wiring — DroolsScoutingTask post-confirmation posture bypass

### Task 2: Wire processBuildings and add posture bypass

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java:238-255`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/DroolsScoutingTaskIT.java`

**Interfaces:**
- Consumes: `ScoutingSessionManager.hasEverConfirmed()`, `ScoutingSessionManager.confirmedExpansionCount()` from Task 1
- Produces: `ENEMY_POSTURE` CaseFile key — now reflects confirmed expansion state when available

- [ ] **Step 1: Write the failing IT test — posture reverts MACRO→ALL_IN on expansion sacrifice**

```java
@Test
void postureRevertsMacroToAllInOnConfirmedExpansionSacrifice() {
    simulatedGame.reset();
    orchestrator.startGame();
    // Spawn enemy expansion building far from enemy base
    simulatedGame.spawnEnemyBuildingForTesting(BuildingType.NEXUS, new Point2d(180, 180));
    // Spawn enemy main base building
    simulatedGame.spawnEnemyBuildingForTesting(BuildingType.NEXUS, new Point2d(224, 224));
    // Spawn friendly unit near expansion for vision
    simulatedGame.spawnFriendlyUnitForTesting(UnitType.STALKER, new Point2d(182, 182));

    for (int i = 0; i < 30; i++) orchestrator.gameTick();

    var ctx = orchestrator.getLastTickResult().caseContext();
    assertThat(ctx.getAs(QuarkMindCaseFile.ENEMY_POSTURE, String.class))
        .as("MACRO while expansion building alive")
        .isEqualTo("MACRO");

    // Remove expansion building (sacrifice)
    simulatedGame.removeEnemyBuildingForTesting("enemy-bldg-test-200");
    // Note: exact tag depends on SimulatedGame.nextTag at spawn time;
    // determine the correct tag from the snapshot after spawn

    for (int i = 0; i < 10; i++) orchestrator.gameTick();

    ctx = orchestrator.getLastTickResult().caseContext();
    assertThat(ctx.getAs(QuarkMindCaseFile.ENEMY_POSTURE, String.class))
        .as("ALL_IN after expansion building destroyed with vision")
        .isEqualTo("ALL_IN");
}
```

Note: The exact building tag depends on `SimulatedGame.nextTag`. Use `simulatedGame.snapshot().enemyBuildings()` after spawn to capture the actual tags. Adjust the test accordingly.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DroolsScoutingTaskIT#postureRevertsMacroToAllInOnConfirmedExpansionSacrifice -q`
Expected: FAIL — posture stays MACRO (no processBuildings call, no bypass)

- [ ] **Step 3: Wire processBuildings and add posture bypass in DroolsScoutingTask.execute()**

In `DroolsScoutingTask.execute()`, after line ~239 (`sessionManager.evict(gameTimeMs)`), add:

```java
GameState gameState = ctx.getAs(QuarkMindCaseFile.GAME_STATE, GameState.class);
if (gameState != null) {
    sessionManager.processBuildings(gameState.enemyBuildings(), estimatedBase,
                                    gameState.myUnits(), buildings);
}
```

Replace the posture resolution block (lines ~251-254):

```java
// Before:
if (data != null && !data.getPostureDecisions().isEmpty()) {
    cachedPosture = data.getPostureDecisions().get(0);
}
String posture = cachedPosture;

// After:
if (sessionManager.hasEverConfirmed()) {
    cachedPosture = sessionManager.confirmedExpansionCount() > 0 ? "MACRO" : "ALL_IN";
} else if (data != null && !data.getPostureDecisions().isEmpty()) {
    cachedPosture = data.getPostureDecisions().get(0);
}
String posture = cachedPosture;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DroolsScoutingTaskIT#postureRevertsMacroToAllInOnConfirmedExpansionSacrifice -q`
Expected: PASS

- [ ] **Step 5: Write fog-of-war IT test — posture stays MACRO when no vision**

```java
@Test
void postureStaysMacroWhenExpansionRemovedWithoutVision() {
    simulatedGame.reset();
    orchestrator.startGame();
    simulatedGame.spawnEnemyBuildingForTesting(BuildingType.NEXUS, new Point2d(180, 180));
    simulatedGame.spawnEnemyBuildingForTesting(BuildingType.NEXUS, new Point2d(224, 224));
    // No friendly unit near expansion — no vision

    for (int i = 0; i < 30; i++) orchestrator.gameTick();

    var ctx = orchestrator.getLastTickResult().caseContext();
    assertThat(ctx.getAs(QuarkMindCaseFile.ENEMY_POSTURE, String.class))
        .isEqualTo("MACRO");

    // Remove expansion — but no vision of the location
    // (tag from snapshot)
    simulatedGame.removeEnemyBuildingForTesting("enemy-bldg-test-200");

    for (int i = 0; i < 10; i++) orchestrator.gameTick();

    ctx = orchestrator.getLastTickResult().caseContext();
    assertThat(ctx.getAs(QuarkMindCaseFile.ENEMY_POSTURE, String.class))
        .as("MACRO stays — expansion removed but no vision (fog)")
        .isEqualTo("MACRO");
}
```

- [ ] **Step 6: Run all scouting IT tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DroolsScoutingTaskIT -q`
Expected: ALL PASS

- [ ] **Step 7: Run existing unit test suite for regression**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DroolsScoutingTaskTest -q`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java
git add quarkmind-sc2/src/test/java/io/quarkmind/plugin/DroolsScoutingTaskIT.java
git commit -m "feat: wire processBuildings + post-confirmation posture bypass

Post-confirmation phase resolves posture from confirmed expansion state
in Java, bypassing DRL. MACRO reverts to ALL_IN when all confirmed
expansions are retracted (destroyed + vision available).

Refs #305"
```

---

## Batch 3: Verification — SpatialCalibrationTest scenarios

### Task 3: Add SimulatedGame ALL_IN pivot scenarios to calibration test

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/SpatialCalibrationTest.java`

**Interfaces:**
- Consumes: `ScoutingSessionManager.processBuildings()` from Task 1
- Produces: calibration report data confirming MACRO→ALL_IN pivot works

- [ ] **Step 1: Update measureReplay() to call processBuildings()**

After the `mgr.evict(gameTimeMs)` call, add:

```java
mgr.processBuildings(state.enemyBuildings(), estimatedEnemyBase,
    state.myUnits(), state.myBuildings());
```

Update the posture heuristic to use confirmed tier when available:

```java
// Replace existing posture resolution:
String drlPosture;
if (hasUnits && !hasExpansions) {
    drlPosture = "ALL_IN";
} else if (hasExpansions) {
    drlPosture = "MACRO";
} else {
    drlPosture = null;
}
if (mgr.hasEverConfirmed()) {
    cachedPosture = mgr.confirmedExpansionCount() > 0 ? "MACRO" : "ALL_IN";
} else if (drlPosture != null) {
    cachedPosture = drlPosture;
}
String posture = cachedPosture;
```

- [ ] **Step 2: Write macroRevertsToAllInOnExpansionSacrifice test**

```java
@Test
void macroRevertsToAllInOnExpansionSacrifice() {
    SimulatedGame game = new SimulatedGame();
    game.reset();
    game.spawnEnemyBuildingForTesting(BuildingType.NEXUS, new Point2d(224, 224));
    game.spawnEnemyBuildingForTesting(BuildingType.NEXUS, new Point2d(180, 180));
    game.spawnFriendlyUnitForTesting(UnitType.STALKER, new Point2d(182, 182));

    ScoutingSessionManager mgr = new ScoutingSessionManager();
    Point2d ourNexus = new Point2d(8, 8);
    Point2d estimatedEnemyBase = new Point2d(224, 224);
    String cachedPosture = "UNKNOWN";
    boolean sawMacro = false;
    boolean sawAllIn = false;
    int transitions = 0;
    String lastPosture = "UNKNOWN";

    for (int tick = 0; tick < TICKS_FULL_GAME; tick++) {
        if (tick == 300) {
            game.removeEnemyBuildingForTesting(
                game.snapshot().enemyBuildings().stream()
                    .filter(b -> b.position().distanceTo(new Point2d(180, 180)) < 1.0f)
                    .findFirst().map(Building::tag).orElse(""));
        }

        game.tick();
        GameState state = game.snapshot();
        long gameTimeMs = (long)(state.gameFrame() * (1000.0 / 22.4));

        mgr.processFrame(state.enemyUnits(), gameTimeMs, ourNexus, estimatedEnemyBase);
        mgr.evict(gameTimeMs);
        mgr.processBuildings(state.enemyBuildings(), estimatedEnemyBase,
            state.myUnits(), state.myBuildings());

        if (mgr.hasEverConfirmed()) {
            cachedPosture = mgr.confirmedExpansionCount() > 0 ? "MACRO" : "ALL_IN";
        }

        if ("MACRO".equals(cachedPosture)) sawMacro = true;
        if ("ALL_IN".equals(cachedPosture) && sawMacro) sawAllIn = true;

        if (!cachedPosture.equals(lastPosture)) {
            transitions++;
            lastPosture = cachedPosture;
        }
    }

    assertThat(sawMacro).as("MACRO detected while expansion alive").isTrue();
    assertThat(sawAllIn).as("ALL_IN detected after expansion sacrifice").isTrue();
    assertThat(transitions).as("UNKNOWN→MACRO→ALL_IN").isEqualTo(2);

    System.out.printf("MACRO→ALL_IN pivot: transitions=%d sawMacro=%b sawAllIn=%b%n",
        transitions, sawMacro, sawAllIn);
}
```

- [ ] **Step 3: Write macroStableWhenExpansionPersists test**

```java
@Test
void macroStableWhenExpansionPersists() {
    SimulatedGame game = new SimulatedGame();
    game.reset();
    game.spawnEnemyBuildingForTesting(BuildingType.NEXUS, new Point2d(224, 224));
    game.spawnEnemyBuildingForTesting(BuildingType.NEXUS, new Point2d(180, 180));

    ScoutingSessionManager mgr = new ScoutingSessionManager();
    Point2d estimatedEnemyBase = new Point2d(224, 224);
    String cachedPosture = "UNKNOWN";
    int transitions = 0;
    String lastPosture = "UNKNOWN";

    for (int tick = 0; tick < TICKS_FULL_GAME; tick++) {
        game.tick();
        GameState state = game.snapshot();
        long gameTimeMs = (long)(state.gameFrame() * (1000.0 / 22.4));

        mgr.processFrame(state.enemyUnits(), gameTimeMs,
            new Point2d(8, 8), estimatedEnemyBase);
        mgr.evict(gameTimeMs);
        mgr.processBuildings(state.enemyBuildings(), estimatedEnemyBase,
            state.myUnits(), state.myBuildings());

        if (mgr.hasEverConfirmed()) {
            cachedPosture = mgr.confirmedExpansionCount() > 0 ? "MACRO" : "ALL_IN";
        }

        if (!cachedPosture.equals(lastPosture)) {
            transitions++;
            lastPosture = cachedPosture;
        }
    }

    assertThat(transitions).as("One transition: UNKNOWN→MACRO, then stable").isEqualTo(1);
    assertThat(cachedPosture).isEqualTo("MACRO");
}
```

- [ ] **Step 4: Run all benchmark tests**

Run: `mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=SpatialCalibrationTest -q`
Expected: ALL PASS (existing replay tests + new SimulatedGame scenarios)

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/SpatialCalibrationTest.java
git commit -m "feat: calibration tests — MACRO→ALL_IN pivot + stable MACRO scenarios

Closes #305"
```

---

## References

- [2026-09-11-macro-allin-pivot-design.md] — design spec this plan implements
- [ScoutingSessionManager.java] — expansion buffer and unit heuristic
- [DroolsScoutingTask.java:238-255] — processFrame/evict/posture resolution
- [DroolsScoutingTask.drl] — expansion posture rules (unchanged)
- [Building.java] — domain record with tag, type, position, health
- [ObservationTranslator.java:122-137] — enemy building separation (deferred — %sc2 profile only)
- Protocol `sc2data-spatial-constants-require-calibration.md` — calibration for CONFIRMED_EXPANSION_DISTANCE and VISION_RANGE
- GitHub #305
