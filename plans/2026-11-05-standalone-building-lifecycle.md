# Standalone Building Lifecycle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #523 — Standalone building lifecycle (eliminate syncBuildings)
**Issue group:** #523

**Goal:** Emit building lifecycle intents from TrackerIntentExtractor and process them through EmulatedGame's applyIntent() path, eliminating the syncBuildings GT sync.

**Architecture:** Two new intent types (BuildingBornIntent, BuildingDiedIntent) + extended BuildIntent/MorphIntent with replayTag/completionLoop. TrackerIntentExtractor emits building intents from UnitBorn/UnitInit/UnitDone/UnitTypeChange/UnitDied tracker events. EmulatedGame handles them through existing machinery (handleBuild, handleMorph, removeDestroyedBuilding). Harness syncBuildings removed.

**Tech Stack:** Java 21, Quarkus, sealed interfaces, Scelight replay parser

## Global Constraints

- Sealed Intent hierarchy: ALL switch expressions (3 total) must be updated atomically when permits change
- MorphIntent naming: sourceName = BuildingType.name() format; targetName = raw tracker event name
- Tracker-driven builds bypass affordability checks, always deduct costs
- Loop-0 ordering: BuildingBornIntent sorts before TrainIntent at same loop
- Building morphs detected via MORPH_BUILDING_SOURCES map, NOT NON_DRONE_BUILDINGS

---

## Batch 1: Intent type foundation

### Task 1: Extend BuildIntent and MorphIntent records

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/BuildIntent.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/MorphIntent.java`

**Interfaces:**
- Produces: `BuildIntent(String unitTag, BuildingType buildingType, Point2d location, String replayTag, long completionLoop)` with `BuildIntent.of(unitTag, type, location)` factory
- Produces: `MorphIntent(String unitTag, String sourceName, String targetName, long completionLoop)` with `MorphIntent.of(unitTag, sourceName, targetName)` factory

- [ ] **Step 1: Write failing test for BuildIntent.of factory**

```java
// In EmulatedGameTest — verify factory sets defaults
var bi = BuildIntent.of("probe-0", BuildingType.PYLON, new Point2d(15, 15));
assertThat(bi.replayTag()).isNull();
assertThat(bi.completionLoop()).isEqualTo(-1L);
```

- [ ] **Step 2: Run test to verify it fails** — `BuildIntent.of` doesn't exist

- [ ] **Step 3: Extend BuildIntent record**

```java
public record BuildIntent(String unitTag, BuildingType buildingType, Point2d location,
                           String replayTag, long completionLoop) implements Intent {
    public static BuildIntent of(String unitTag, BuildingType buildingType, Point2d location) {
        return new BuildIntent(unitTag, buildingType, location, null, -1);
    }
}
```

- [ ] **Step 4: Update all 11 production call sites to use BuildIntent.of()**

ProtobufToIntent:123, ReplayCommandExtractor:91, EnemyBehavior:134, SC2DeliveryHandler:87, EconomicsDecisionService:68/112/149, BasicEconomicsTask:81, DroolsStrategyTask:288/291/296

- [ ] **Step 5: Update all test call sites** — EmulatedGameTest (5), ReplayEngineTest (1), ActionTranslatorTest (4), GameStateRoundTripTest (1) — use `new BuildIntent(...)` with 5 args or `BuildIntent.of()`

- [ ] **Step 6: Extend MorphIntent record**

```java
public record MorphIntent(String unitTag, String sourceName, String targetName,
                            long completionLoop) implements Intent {
    public static MorphIntent of(String unitTag, String sourceName, String targetName) {
        return new MorphIntent(unitTag, sourceName, targetName, -1);
    }
}
```

- [ ] **Step 7: Update all 3 production call sites** — TrackerIntentExtractor:148, ReplayCommandExtractor:108, SC2DeliveryHandler:123

- [ ] **Step 8: Update all 13 test call sites** — ZergEmulatedGameTest (5), TerranEmulatedGameTest (8)

- [ ] **Step 9: Run `mvn test -pl quarkmind-sc2 -q`** — all tests pass

- [ ] **Step 10: Commit**

```
feat(#523): extend BuildIntent/MorphIntent with replayTag and completionLoop
```

### Task 2: Create BuildingBornIntent, BuildingDiedIntent, update sealed hierarchy

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/BuildingBornIntent.java`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/BuildingDiedIntent.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/Intent.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` (both switch expressions)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/real/ActionTranslator.java`

**Interfaces:**
- Produces: `BuildingBornIntent(String tag, BuildingType type, Point2d position) implements Intent`
- Produces: `BuildingDiedIntent(String tag, BuildingType type) implements Intent`

- [ ] **Step 1: Write failing test for BuildingBornIntent handler**

```java
// In EmulatedGameTest
void buildingBornIntent_addsCompleteBuildingAndSupply() {
    EmulatedGame game = createProtossGame();
    game.applyIntent(new TimedIntent(0L,
        new BuildingBornIntent("r-1-1", BuildingType.PYLON, new Point2d(20, 20))));
    GameState s = game.snapshot();
    assertThat(s.myBuildings()).hasSize(2); // initial Nexus + new Pylon
    assertThat(s.supply()).isEqualTo(23); // 15 from Nexus + 8 from Pylon
}
```

- [ ] **Step 2: Create BuildingBornIntent.java**

```java
package io.quarkmind.sc2.intent;
import io.quarkmind.domain.BuildingType;
import io.quarkmind.domain.Point2d;
public record BuildingBornIntent(String tag, BuildingType type, Point2d position) implements Intent {}
```

- [ ] **Step 3: Create BuildingDiedIntent.java**

```java
package io.quarkmind.sc2.intent;
import io.quarkmind.domain.BuildingType;
public record BuildingDiedIntent(String tag, BuildingType type) implements Intent {}
```

- [ ] **Step 4: Update Intent.java permits clause** — add BuildingBornIntent, BuildingDiedIntent

- [ ] **Step 5: Add cases to EmulatedGame.applyIntent(TimedIntent) at line 333**

```java
case BuildingBornIntent bb -> () -> handleBuildingBorn(bb, friendly, friendlyPhysics);
case BuildingDiedIntent bd -> () -> handleBuildingDied(bd, friendly, friendlyPhysics);
```

- [ ] **Step 6: Add cases to EmulatedGame.applyIntent(Intent, PlayerState, PhysicsState) at line 349**

```java
case BuildingBornIntent bb -> () -> handleBuildingBorn(bb, state, physics);
case BuildingDiedIntent bd -> () -> handleBuildingDied(bd, state, physics);
```

- [ ] **Step 7: Add cases to ActionTranslator.translate() at line 44**

```java
case BuildingBornIntent bb -> null;
case BuildingDiedIntent bd -> null;
```

- [ ] **Step 8: Implement handleBuildingBorn in EmulatedGame**

```java
private void handleBuildingBorn(BuildingBornIntent bb, PlayerState state, PhysicsState physics) {
    state.addBuilding(new Building(bb.tag(), bb.type(), bb.position(),
        SC2Data.maxBuildingHealth(bb.type()), SC2Data.maxBuildingHealth(bb.type()), true));
    state.addSupply(SC2Data.supplyBonus(bb.type()));
    if (state == friendly && playerRaceModel != null) {
        if (playerRaceModel.townHallTypes().contains(bb.type())) {
            registerTownHallWithPatches(bb.tag(), bb.position());
        } else if (SC2Data.isGasBuilding(bb.type())) {
            workerSimulation.onGasBuildingCompleted(bb.tag(), bb.position());
        }
    }
}
```

- [ ] **Step 9: Implement handleBuildingDied in EmulatedGame** — delegate to existing `removeDestroyedBuilding`

```java
private void handleBuildingDied(BuildingDiedIntent bd, PlayerState state, PhysicsState physics) {
    state.buildings().stream().filter(b -> b.tag().equals(bd.tag())).findFirst().ifPresent(b -> {
        state.addSupply(-SC2Data.supplyBonus(bd.type()));
        if (state == friendly) economyTracker.recordBuildingDestroyed(bd.type());
    });
    state.removeBuilding(bd.tag());
}
```

- [ ] **Step 10: Write test for BuildingDiedIntent**

```java
void buildingDiedIntent_removesBuildingAndSubtractsSupply() {
    EmulatedGame game = createProtossGame();
    game.applyIntent(new TimedIntent(0L,
        new BuildingBornIntent("r-pylon", BuildingType.PYLON, new Point2d(20, 20))));
    assertThat(game.snapshot().supply()).isEqualTo(23);
    game.applyIntent(new TimedIntent(100L,
        new BuildingDiedIntent("r-pylon", BuildingType.PYLON)));
    assertThat(game.snapshot().supply()).isEqualTo(15);
    assertThat(game.snapshot().myBuildings()).hasSize(1); // only Nexus remains
}
```

- [ ] **Step 11: Run `mvn test -pl quarkmind-sc2 -q`** — all tests pass

- [ ] **Step 12: Commit**

```
feat(#523): BuildingBornIntent + BuildingDiedIntent — sealed hierarchy + handlers
```

## Batch 2: EmulatedGame handleBuild improvements + SC2Data gas costs

### Task 3: Fix SC2Data.gasCost for Protoss buildings + handleBuild gas/affordability

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`

**Interfaces:**
- Consumes: `BuildIntent(unitTag, buildingType, location, replayTag, completionLoop)` from Task 1
- Produces: `handleBuild` now deducts gas, uses replayTag for building tag, pins completion, bypasses affordability for tracker-driven

- [ ] **Step 1: Write failing test — gas cost for Twilight Council**

```java
// In SC2DataTest or EmulatedGameTest
assertThat(SC2Data.gasCost(BuildingType.TWILIGHT_COUNCIL)).isEqualTo(100);
```

- [ ] **Step 2: Fix SC2Data.gasCost** — move ROBOTICS_FACILITY, STARGATE, TWILIGHT_COUNCIL, FLEET_BEACON, ROBOTICS_BAY out of the 0-return group:

```java
case TWILIGHT_COUNCIL -> 100;
case ROBOTICS_FACILITY -> 100;
case STARGATE -> 150;
case FLEET_BEACON -> 200;
case ROBOTICS_BAY -> 150;
case DARK_SHRINE -> 150;   // was 100
case TEMPLAR_ARCHIVES -> 200; // was 100
```

- [ ] **Step 3: Write test — handleBuild with replayTag uses tracker tag**

```java
void handleBuild_withReplayTag_usesTrackerTag() {
    EmulatedGame game = createProtossGame();
    game.applyIntent(new TimedIntent(100L,
        new BuildIntent("worker", BuildingType.GATEWAY, new Point2d(20, 20), "r-5-1", 500L)));
    assertThat(game.snapshot().myBuildings().stream()
        .anyMatch(b -> b.tag().equals("r-5-1"))).isTrue();
}
```

- [ ] **Step 4: Write test — tracker-driven build bypasses affordability**

```java
void handleBuild_trackerDriven_bypassesAffordabilityCheck() {
    EmulatedGame game = createProtossGame();
    game.snapshot(); // minerals = 50
    // Gateway costs 150m — would fail affordability, but tracker-driven should succeed
    game.applyIntent(new TimedIntent(100L,
        new BuildIntent("worker", BuildingType.GATEWAY, new Point2d(20, 20), "r-5-1", 500L)));
    assertThat(game.snapshot().myBuildings().stream()
        .anyMatch(b -> b.tag().equals("r-5-1") && b.type() == BuildingType.GATEWAY)).isTrue();
}
```

- [ ] **Step 5: Write test — handleBuild deducts gas cost**

```java
void handleBuild_deductsGasCost() {
    EmulatedGame game = createProtossGame();
    // seed gas
    game.snapshot(); // set up some vespene via direct state manipulation
    // ... build a STARGATE (150m/150g)
    // assert vespene decreased by 150
}
```

- [ ] **Step 6: Modify handleBuild in EmulatedGame**

Key changes to `handleBuild(BuildIntent b, PlayerState state, PhysicsState physics, long absLoop)`:
1. Extract `replayTag` and `completionLoop` from BuildIntent
2. When `replayTag != null`: use it as tag, skip affordability, add to trackerDrivenBuildings
3. Add gas affordability check and gas deduction
4. Use 3-arg `recordBuildSpending(type, mCost, gCost)`
5. When `completionLoop >= 0`: pin completion to `completionLoop / SC2Data.LOOPS_PER_TICK`

- [ ] **Step 7: Write test — pinned completion**

```java
void handleBuild_withPinnedCompletion_completesAtCorrectTick() {
    EmulatedGame game = createProtossGame();
    long pinnedLoop = 1000L;
    game.applyIntent(new TimedIntent(100L,
        new BuildIntent("worker", BuildingType.PYLON, new Point2d(20, 20), "r-pylon", pinnedLoop)));
    // tick forward to the pinned completion tick
    long targetTick = pinnedLoop / SC2Data.LOOPS_PER_TICK;
    for (int i = 0; i < targetTick + 1; i++) game.tick();
    assertThat(game.snapshot().myBuildings().stream()
        .anyMatch(b -> b.tag().equals("r-pylon") && b.isComplete())).isTrue();
}
```

- [ ] **Step 8: Modify handleMorph for pinned completion** — when `m.completionLoop() >= 0`, use `m.completionLoop() / SC2Data.LOOPS_PER_TICK` instead of computed `gameFrame + buildTimeInLoops(targetBt) / LOOPS_PER_TICK`

- [ ] **Step 9: Add Reactor self-match prevention** — in the nearest-production-building search (or new findNearestProductionBuilding method), exclude the Reactor's own tag from candidates

- [ ] **Step 10: Run `mvn test -pl quarkmind-sc2 -q`** — all tests pass

- [ ] **Step 11: Commit**

```
feat(#523): handleBuild gas costs, tracker-driven bypass, pinned completion, reactor self-match
```

## Batch 3: TrackerIntentExtractor building intent emission

### Task 4: Emit building lifecycle intents from TrackerIntentExtractor

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/TrackerIntentExtractor.java`

**Interfaces:**
- Consumes: `BuildingBornIntent`, `BuildingDiedIntent`, `BuildIntent.of()`, `MorphIntent.of()` from Tasks 1-2
- Produces: Building lifecycle intents in the TimedIntent list

- [ ] **Step 1: Write failing test — building intents emitted from tracker events**

```java
// In TrackerIntentExtractorIntentTest
void extractsBuildingIntentsFromUnitInitDone() {
    // Use an existing replay that has buildings
    List<TimedIntent> intents = TrackerIntentExtractor.extract(replayPath, playerId);
    long buildingBornCount = intents.stream()
        .filter(ti -> ti.intent() instanceof BuildingBornIntent).count();
    assertThat(buildingBornCount).isGreaterThan(0); // at least initial town hall
    long buildIntentCount = intents.stream()
        .filter(ti -> ti.intent() instanceof BuildIntent bi && bi.replayTag() != null).count();
    assertThat(buildIntentCount).isGreaterThan(0); // at least one building constructed
}
```

- [ ] **Step 2: Add UnitDone pre-pass** — before the main loop, iterate tracker events and build `buildingDoneLoops` map (composite tag → UnitDone loop) for building tags

- [ ] **Step 3: Add buildingTagToType map** — `Map<Long, BuildingType>` alongside `unitTagToType`

- [ ] **Step 4: Add MORPH_BUILDING_SOURCES constant**

```java
private static final Map<String, BuildingType> MORPH_BUILDING_SOURCES = Map.of(
    "Lair", BuildingType.HATCHERY, "Hive", BuildingType.LAIR,
    "GreaterSpire", BuildingType.SPIRE,
    "OrbitalCommand", BuildingType.COMMAND_CENTER,
    "PlanetaryFortress", BuildingType.COMMAND_CENTER);
```

- [ ] **Step 5: Modify UnitBorn handler** — when building name, add to `buildingTagToType`, emit `BuildingBornIntent(tag, type, position)`

- [ ] **Step 6: Modify UnitInit handler** — for morph buildings (MORPH_BUILDING_SOURCES keys), emit `MorphIntent`; for fresh construction, emit `BuildIntent` with replayTag and completionLoop from pre-pass

- [ ] **Step 7: Modify UnitTypeChange handler** — check `buildingTagToType`, emit `MorphIntent.of(tag, sourceType.name(), rawTypeName)` with correct naming convention

- [ ] **Step 8: Modify UnitDied handler** — check `buildingTagToType`, emit `BuildingDiedIntent(tag, type)`

- [ ] **Step 9: Add intent type priority sort** — change `intents.sort(Comparator.comparingLong(TimedIntent::loop))` to include `.thenComparingInt(ti -> intentPriority(ti.intent()))` with priority table from spec

- [ ] **Step 10: Run `mvn test -pl quarkmind-sc2 -q`** — all tests pass

- [ ] **Step 11: Commit**

```
feat(#523): TrackerIntentExtractor emits building lifecycle intents
```

## Batch 4: Harness integration + validation

### Task 5: Remove syncBuildings from harness and validate

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StandaloneEmulatedGameBaselineTest.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TechSpendingDiagnosticTest.java`

**Interfaces:**
- Consumes: All building lifecycle intents from Tasks 1-4

- [ ] **Step 1: Remove syncBuildings from ReplayValidationHarness.runInternal()**

Remove lines 88 (injectedTags), 84-85 (reactoredTags), 101 (syncBuildings call), and the syncBuildings method itself (lines 202-214).

- [ ] **Step 2: Remove syncBuildings from StandaloneEmulatedGameBaselineTest.runStandalone()**

Remove injectedBuildings map, reactoredTags, syncBuildings call (line 276), and the syncBuildings method (lines 326-347).

- [ ] **Step 3: Remove syncBuildings from TechSpendingDiagnosticTest**

Same pattern — remove the local syncBuildings call and method.

- [ ] **Step 4: Run `mvn test -pl quarkmind-sc2 -q`** — verify all tests pass

- [ ] **Step 5: Run report profile for building accuracy**

```
mvn test -pl quarkmind-sc2 -Preport -Dtest=TrackerDrivenAccuracyBaselineTest
```

Verify building accuracy meets 100% target.

- [ ] **Step 6: Run standalone baseline**

```
mvn test -pl quarkmind-sc2 -Preport -Dtest=StandaloneEmulatedGameBaselineTest
```

Verify building accuracy without GT sync.

- [ ] **Step 7: Commit**

```
feat(#523): eliminate syncBuildings — standalone building lifecycle complete
```

## References

- [2026-11-05-standalone-building-lifecycle-design.md] — design spec
- [EmulatedGame.java:693-724] — handleBuild
- [EmulatedGame.java:1060-1134] — handleMorph + MORPH_BUILDING_TARGETS
- [EmulatedGame.java:1597-1698] — injectReplayBuilding, syncReplayBuilding, removeDestroyedBuilding
- [TrackerIntentExtractor.java:74-204] — main extraction loop
- [ReplayValidationHarness.java:202-214] — syncBuildings (to remove)
- [SC2Data.java:1033-1070] — gasCost(BuildingType)
- [ActionTranslator.java:44] — Intent switch expression
- [ReplaySimulatedGame.java:265-295] — GT applyUnitInit reference
- [GitHub #523] — issue
- [GitHub #516] — parent epic
