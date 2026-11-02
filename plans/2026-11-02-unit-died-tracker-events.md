# UnitDied Tracker Events Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #493 — Process UnitDied tracker events — remove dead units from PlayerState
**Issue group:** #493

**Goal:** Process UnitDied tracker events so dead units are removed from PlayerState, fixing foodUsed (56%→≥90%) and workersActiveCount (48%→≥85%) accuracy.

**Architecture:** TrackerIntentExtractor already processes ID_UNIT_BORN/ID_UNIT_INIT/ID_UPGRADE from replay tracker events and emits TimedIntents. We add ID_UNIT_DIED handling that emits a new `UnitDiedIntent`. EmulatedGame's `applyIntent()` switches handle the new intent by removing the unit from PlayerState, deducting supply, and cleaning physics state (targets, cooldowns, blink, movement) — mirroring the existing combat death cleanup at lines 753-758.

**Tech Stack:** Java 21, sealed interfaces, Scelight replay parser

## Global Constraints

- Intent is a sealed interface — any new subtype must be added to the permits clause AND every switch expression over Intent must be updated in the same commit
- Extraction logic belongs in dedicated extractor classes, not SimulatedGame (protocol PP-20260528-612dee)
- Unit identity uses tags, not record equality (protocol PP-20260606-506f33)
- No `@QuarkusTest` for tests that can be plain JUnit

---

## Batch 1: UnitDied intent pipeline

### Task 1: Add UnitDiedIntent and wire through EmulatedGame

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/UnitDiedIntent.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/Intent.java:3` — add to permits clause
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java:322-348` — add cases to both switch expressions
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java:337-350` — add case to second switch
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` (add test method)

**Interfaces:**
- Produces: `UnitDiedIntent(String unitTag, UnitType unitType)` — implements `Intent`
- Consumes: `PlayerState.removeUnit(String tag)`, `PlayerState.addSupplyUsed(int)`, `SC2Data.supplyCost(UnitType)`, `PhysicsState` cleanup (unitTargets, unitCooldowns, blinkCooldowns), `MovementStrategy.clearUnit(String)`

- [ ] **Step 1: Write the failing test**

Add to `EmulatedGameTest`:

```java
@Test
void unitDiedIntentRemovesUnitAndDeductsSupply() {
    EmulatedGame game = new EmulatedGame();
    game.reset();
    game.setPlayerRaceModel(RaceModelFactory.forRace(Race.PROTOSS));

    // Train a Zealot via intent
    game.applyIntent(new TimedIntent(0, new TrainIntent("r-1-0", UnitType.ZEALOT, true, false, 0)));
    game.tick(); // process the train

    GameState before = game.snapshot();
    long zealotsBefore = before.myUnits().stream()
        .filter(u -> u.type() == UnitType.ZEALOT).count();
    int supplyBefore = before.supplyUsed();

    // Kill the Zealot
    String zealotTag = before.myUnits().stream()
        .filter(u -> u.type() == UnitType.ZEALOT)
        .findFirst().orElseThrow().tag();
    game.applyIntent(new TimedIntent(100, new UnitDiedIntent(zealotTag, UnitType.ZEALOT)));

    GameState after = game.snapshot();
    long zealotsAfter = after.myUnits().stream()
        .filter(u -> u.type() == UnitType.ZEALOT).count();

    assertEquals(zealotsBefore - 1, zealotsAfter, "Dead Zealot should be removed");
    assertEquals(supplyBefore - SC2Data.supplyCost(UnitType.ZEALOT), after.supplyUsed(),
        "Supply should decrease by Zealot's cost");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameTest#unitDiedIntentRemovesUnitAndDeductsSupply -q`
Expected: Compilation error — `UnitDiedIntent` does not exist

- [ ] **Step 3: Create UnitDiedIntent record**

```java
package io.quarkmind.sc2.intent;

import io.quarkmind.domain.UnitType;

public record UnitDiedIntent(String unitTag, UnitType unitType) implements Intent {}
```

- [ ] **Step 4: Add UnitDiedIntent to Intent permits clause**

Change `Intent.java` line 3:
```java
public sealed interface Intent permits BuildIntent, TrainIntent, AttackIntent, MoveIntent, BlinkIntent, MuleCalldownIntent, ResearchIntent, MorphIntent, AbilityIntent, UnitDiedIntent {
```

- [ ] **Step 5: Add UnitDiedIntent cases to both switch expressions in EmulatedGame**

In `applyIntent(TimedIntent ti)` (line 322), add after the AbilityIntent case:
```java
case UnitDiedIntent d -> () -> handleUnitDied(d, friendly, friendlyPhysics);
```

In `applyIntent(Intent intent, PlayerState state, PhysicsState physics)` (line 337), add after the AbilityIntent case:
```java
case UnitDiedIntent d -> () -> handleUnitDied(d, state, physics);
```

- [ ] **Step 6: Implement handleUnitDied**

Add to `EmulatedGame`:
```java
private void handleUnitDied(UnitDiedIntent d, PlayerState state, PhysicsState physics) {
    state.removeUnit(d.unitTag());
    state.addSupplyUsed(-SC2Data.supplyCost(d.unitType()));
    physics.unitTargets.remove(d.unitTag());
    physics.unitCooldowns.remove(d.unitTag());
    physics.blinkCooldowns.remove(d.unitTag());
    movementStrategy.clearUnit(d.unitTag());
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameTest#unitDiedIntentRemovesUnitAndDeductsSupply -q`
Expected: PASS

- [ ] **Step 8: Run full test suite to check for switch exhaustiveness**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: Any class with a switch over `Intent` that missed the new case will fail to compile. Fix any that surface.

- [ ] **Step 9: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/UnitDiedIntent.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/Intent.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
       quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "feat(#493): add UnitDiedIntent and wire through EmulatedGame"
```

### Task 2: Extract UnitDied events in TrackerIntentExtractor

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/TrackerIntentExtractor.java:70-134` — add ID_UNIT_DIED case
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerIntentExtractorIntentTest.java` (add test method)

**Interfaces:**
- Consumes: `UnitDiedIntent(String unitTag, UnitType unitType)` from Task 1
- Consumes: `ITrackerEvents.ID_UNIT_DIED`, `IBaseUnitEvent.getControlPlayerId()`, `IBaseUnitEvent.getUnitTypeName()`, `IBaseUnitEvent.getUnitTagIndex()`, `IBaseUnitEvent.getUnitTagRecycle()`, `IBaseUnitEvent.getLoop()`
- Produces: `TimedIntent(loop, UnitDiedIntent)` entries in the extracted intent list

- [ ] **Step 1: Write the failing test**

Add to `TrackerIntentExtractorIntentTest`:

```java
@Test
@EnabledIf("oracleExists")
void extractsUnitDiedIntentsFromTrackerEvents() throws Exception {
    Path replay = Files.list(ORACLE_DIR)
        .filter(p -> p.toString().endsWith(".SC2Replay"))
        .sorted().findFirst().orElseThrow();

    List<TimedIntent> intents = TrackerIntentExtractor.extract(replay, 1);

    long deathCount = intents.stream()
        .filter(ti -> ti.intent() instanceof UnitDiedIntent).count();
    assertTrue(deathCount > 0,
        "Should have UnitDiedIntents from tracker UnitDied events");

    intents.stream()
        .filter(ti -> ti.intent() instanceof UnitDiedIntent)
        .forEach(ti -> {
            UnitDiedIntent d = (UnitDiedIntent) ti.intent();
            assertTrue(ti.loop() >= 0, "Death loop must be non-negative: " + ti);
            assertNotNull(d.unitTag(), "Unit tag must not be null");
            assertNotNull(d.unitType(), "Unit type must not be null");
        });
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TrackerIntentExtractorIntentTest#extractsUnitDiedIntentsFromTrackerEvents -q`
Expected: FAIL — deathCount is 0

- [ ] **Step 3: Add ID_UNIT_DIED case to TrackerIntentExtractor**

In the switch statement (line 70), add before `default -> {}`:

```java
case ITrackerEvents.ID_UNIT_DIED -> {
    Integer tagIndex   = raw.get("unitTagIndex");
    Integer tagRecycle = raw.get("unitTagRecycle");
    if (tagIndex == null || tagRecycle == null) {continue;}
    Integer pid = raw.get("controlPlayerId");
    if (pid == null || pid != playerId) {continue;}
    String typeName = raw.get("unitTypeName");
    if (typeName == null) {continue;}
    typeName = typeName.trim();
    UnitType unitType = Sc2ReplayShared.toUnitType(typeName);
    if (unitType == UnitType.UNKNOWN) {continue;}
    String tag = Sc2ReplayShared.makeTag(tagIndex, tagRecycle);
    intents.add(new TimedIntent(raw.getLoop(),
                                new UnitDiedIntent(tag, unitType)));
}
```

Note: UnitDied events use raw struct access (no `IBaseUnitEvent` cast) — same pattern as `ReplaySimulatedGame.applyUnitDied()`. The `controlPlayerId` field identifies whose unit died.

- [ ] **Step 4: Add import for UnitDiedIntent**

Add to TrackerIntentExtractor imports:
```java
import io.quarkmind.sc2.intent.UnitDiedIntent;
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TrackerIntentExtractorIntentTest#extractsUnitDiedIntentsFromTrackerEvents -q`
Expected: PASS

- [ ] **Step 6: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All tests pass

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/TrackerIntentExtractor.java \
       quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerIntentExtractorIntentTest.java
git commit -m "feat(#493): extract UnitDied tracker events in TrackerIntentExtractor"
```

### Task 3: Verify accuracy improvement against oracle baselines

**Files:**
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerDrivenAccuracyBaselineTest.java` (existing — run and check)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceRegressionTest.java` (existing — run and check)

**Interfaces:**
- Consumes: Full pipeline — TrackerIntentExtractor → ReplayValidationHarness → EmulatedGame

- [ ] **Step 1: Run tracker-driven accuracy baseline**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=TrackerDrivenAccuracyBaselineTest -q`
Expected: foodUsed accuracy ≥90% at 5min (was 56%), workersActiveCount ≥85% at 5min (was 48%). Unit production accuracy must stay at 100%.

- [ ] **Step 2: Run divergence regression test**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=DivergenceRegressionTest -q`
Expected: No regressions in unit/building accuracy thresholds.

- [ ] **Step 3: If accuracy thresholds need updating, update the baseline doc**

If `TrackerDrivenAccuracyBaselineTest` writes updated numbers to `docs/benchmarks/tracker-driven-accuracy-baseline.md`, commit the updated doc.

- [ ] **Step 4: Commit any baseline updates**

```bash
git add docs/benchmarks/
git commit -m "docs(#493): update tracker-driven accuracy baseline after UnitDied processing"
```

## References

- [TrackerIntentExtractor.java:34-157] — existing extractor, switch over tracker event IDs
- [Intent.java:3] — sealed interface with permits clause
- [EmulatedGame.java:322-350] — two switch expressions over Intent
- [EmulatedGame.java:753-758] — combat death cleanup pattern (targets, cooldowns, blink, movement)
- [PlayerState.java:58-62] — addSupplyUsed, removeUnit
- [SC2Data.java:670] — supplyCost(UnitType)
- [ReplaySimulatedGame.java:248-263] — ground-truth UnitDied handling (raw struct access pattern)
- [TrackerEventFeatureExtractor.java:71-86] — UnitDied event field access pattern
- [PP-20260528-612dee] — extraction logic in dedicated classes, not SimulatedGame
- [PP-20260606-506f33] — unit identity via tags
- [GitHub #493] — focal issue
- [GitHub #481] — parent epic (EmulatedGame playbook calibration)
