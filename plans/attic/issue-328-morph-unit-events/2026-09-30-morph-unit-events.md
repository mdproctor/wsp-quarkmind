# Morph Unit Events Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #328 — Synthesize morph-based unit events (Baneling, Ravager, BroodLord, Archon)
**Issue group:** #328 (under epic #318)

**Goal:** Discover morph abilLinks, wire AbilityMapping dispatch, add selection-based multiplication, morph spending tracking, and timed UnitInit+UnitDone for all morph unit types.

**Architecture:** Vertical slice through the replay extraction pipeline: discovery (AbilityDiscoveryCalibrationTest) → mapping (AbilityMapping) → emission (StrippedReplayFeatureExtractor.handleMorph) → calibration (SC2Data). The morph emission infrastructure already exists — the gap is upstream (no abilLinks known) and downstream (no spending, wrong timing, no multiplication).

**Tech Stack:** Java 21, Quarkus, scelight replay parser, AssertJ

## Global Constraints

- Protocol: morph logic stays in extractor classes, not SimulatedGame (PP-20260528-612dee)
- Protocol: calibrate from replay data, not formula (sc2data-train-times-require-calibration)
- Protocol: exhaustive switches on project enums (PP-20260913-4d73d1)
- Protocol: unit tag prefix per replay source (PP-20260528-d9f967)
- SC2Data already stores morph delta costs — use directly, never subtract source cost
- `trainTimeInLoops()` is the single source of truth for unit morph times
- `MORPH_TARGET_TO_SOURCE_LINKS` keyed by target name (not abilLink) because MorphCommand doesn't carry abilLink

---

## Batch 1: Timed morph emission + spending tracking

After this batch: `handleMorph()` emits timed UnitInit+UnitDone for all morph types (not just buildings/Archon), tracks morph spending, and handles Overseer supply correctly. All existing tests updated. No new abilLinks yet — that's Batch 2.

### Task 1: Convert standard unit morphs from instant UnitBorn to timed UnitInit+UnitDone

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java:395-444` (handleMorph)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java:173-184` (trainTimeInLoops entries)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayMorphTest.java:43-61`

**Interfaces:**
- Consumes: `SC2Data.trainTimeInLoops(UnitType)` — returns morph time in game loops
- Consumes: `SC2Data.buildTimeInLoops(BuildingType)` — returns building morph time (already exists)
- Produces: `getMorphTime(String targetName)` — private method dispatching to trainTimeInLoops for units, buildTimeInLoops for buildings

- [ ] **Step 1: Update existing test to expect UnitInit+UnitDone instead of UnitBorn**

In `StrippedReplayMorphTest.standardUnitMorphEmitsSingleDeath()`, change assertions:

```java
@Test
void standardUnitMorphEmitsSingleDeath() {
    var extractor = new StrippedReplayFeatureExtractor();
    var morph = new ReplayCommand.MorphCommand(2000, "Zergling", "Baneling");

    List<Map<String, Object>> events = extractor.processMorphForTest(morph, 1, 100);

    var deaths = events.stream()
        .filter(e -> "UnitDied".equals(e.get("evtTypeName")))
        .toList();
    var inits = events.stream()
        .filter(e -> "UnitInit".equals(e.get("evtTypeName")))
        .toList();
    var dones = events.stream()
        .filter(e -> "UnitDone".equals(e.get("evtTypeName")))
        .toList();

    assertThat(deaths).hasSize(1);
    assertThat(deaths.get(0).get("unitTypeName")).isEqualTo("Zergling");

    assertThat(inits).hasSize(1);
    assertThat(inits.get(0).get("unitTypeName")).isEqualTo("Baneling");
    assertThat(((Number) inits.get(0).get("loop")).longValue()).isEqualTo(2000);

    assertThat(dones).hasSize(1);
    assertThat(dones.get(0).get("unitTypeName")).isEqualTo("Baneling");
    long expectedDoneLoop = 2000 + SC2Data.trainTimeInLoops(UnitType.BANELING);
    assertThat(((Number) dones.get(0).get("loop")).longValue()).isEqualTo(expectedDoneLoop);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrippedReplayMorphTest#standardUnitMorphEmitsSingleDeath -q`
Expected: FAIL — still emits UnitBorn, not UnitInit+UnitDone

- [ ] **Step 3: Add getMorphTime() and modify handleMorph()**

In `StrippedReplayFeatureExtractor`, add a private method and simplify handleMorph:

```java
private int getMorphTime(String targetName) {
    if (BUILDING_MORPH_TARGETS.contains(targetName)) {
        return getBuildTime(targetName);
    }
    UnitType ut = UNIT_PYTHON_NAMES.entrySet().stream()
        .filter(e -> e.getValue().equals(targetName))
        .map(Map.Entry::getKey)
        .findFirst()
        .orElse(null);
    if (ut != null) {
        return SC2Data.trainTimeInLoops(ut);
    }
    return ARCHON_MORPH_TIME;
}
```

In `handleMorph()`, replace the else branch (lines 432-441) — remove the UnitBorn emission and use UnitInit+UnitDone for all morphs:

```java
int handleMorph(ReplayCommand.MorphCommand mc, int playerId,
                List<SyntheticEvent> events, int tagCounter) {
    String sourceName  = mc.sourceName();
    String targetName  = mc.targetName();
    long   commandLoop = mc.loop();

    int sourceDeathCount = "Archon".equals(targetName) ? 2 : 1;
    for (int i = 0; i < sourceDeathCount; i++) {
        int tag = tagCounter++;
        events.add(new SyntheticEvent(commandLoop, EventOrdinal.UNIT_DIED, playerId,
                                      Map.of("evtTypeName", "UnitDied",
                                             "loop", commandLoop,
                                             "controlPlayerId", playerId,
                                             "unitTypeName", sourceName,
                                             "unitTagIndex", tag,
                                             "unitTagRecycle", 0)));
    }

    int morphTime = getMorphTime(targetName);
    int tag = tagCounter++;
    events.add(new SyntheticEvent(commandLoop, EventOrdinal.UNIT_INIT, playerId,
                                  Map.of("evtTypeName", "UnitInit",
                                         "loop", commandLoop,
                                         "controlPlayerId", playerId,
                                         "unitTypeName", targetName,
                                         "unitTagIndex", tag,
                                         "unitTagRecycle", 0)));
    long doneLoop = commandLoop + morphTime;
    events.add(new SyntheticEvent(doneLoop, EventOrdinal.UNIT_DONE, playerId,
                                  Map.of("evtTypeName", "UnitDone",
                                         "loop", doneLoop,
                                         "controlPlayerId", playerId,
                                         "unitTypeName", targetName,
                                         "unitTagIndex", tag,
                                         "unitTagRecycle", 0)));

    return tagCounter;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrippedReplayMorphTest -q`
Expected: ALL PASS (including the Archon and building morph tests which should still work)

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayMorphTest.java
git commit -m "feat: convert standard unit morphs to timed UnitInit+UnitDone Refs #328"
```

### Task 2: Add morph spending tracking and fix Overseer supply

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java:395-444` (handleMorph signature), `582-623` (spending methods), `680-701` (applyEventToEconomy)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayMorphTest.java` (new tests)

**Interfaces:**
- Consumes: `SC2Data.mineralCost(UnitType)`, `SC2Data.gasCost(UnitType)`, `SC2Data.supplyCost(UnitType)` — delta costs for morph targets
- Consumes: `SC2Data.mineralCost(BuildingType)`, `gasCostForBuilding(BuildingType)` — delta costs for building morphs
- Produces: `trackMorphSpending(String targetName, PlayerState state)` — private method
- Produces: Updated `handleMorph(MorphCommand mc, int playerId, PlayerState state, List<SyntheticEvent> events, int tagCounter)` — adds PlayerState parameter

- [ ] **Step 1: Write failing tests for morph spending and Overseer supply**

Add two new tests to `StrippedReplayMorphTest`. These need a test helper that exposes PlayerState. Add a new package-private test helper method to `StrippedReplayFeatureExtractor`:

```java
// Test helper — returns [events, state] for morph spending validation
record MorphTestResult(List<Map<String, Object>> events, int mineralsUsedArmy, int gasUsedArmy,
                       int foodUsed, int foodMade) {}

MorphTestResult processMorphWithStateForTest(ReplayCommand.MorphCommand mc, int playerId, int startTag) {
    List<SyntheticEvent> events = new ArrayList<>();
    var state = new PlayerState();
    state.mineralsCurrent = 10000;
    state.vespeneCurrent = 10000;
    handleMorph(mc, playerId, state, events, startTag);
    return new MorphTestResult(
        events.stream().map(SyntheticEvent::data).toList(),
        state.mineralsUsedArmy, state.gasUsedArmy,
        state.foodUsed, state.foodMade);
}
```

Tests:

```java
@Test
void morphTracksBanelingSpending() {
    var extractor = new StrippedReplayFeatureExtractor();
    var morph = new ReplayCommand.MorphCommand(2000, "Zergling", "Baneling");

    var result = extractor.processMorphWithStateForTest(morph, 1, 100);

    assertThat(result.mineralsUsedArmy()).isEqualTo(25);
    assertThat(result.gasUsedArmy()).isEqualTo(25);
}

@Test
void overseerMorphPreservesSupply() {
    var extractor = new StrippedReplayFeatureExtractor();
    var morph = new ReplayCommand.MorphCommand(2000, "Overlord", "Overseer");

    var result = extractor.processMorphWithStateForTest(morph, 1, 100);

    // Overseer provides same supply as Overlord — net 0 supply change
    // UnitDied(Overlord) handled by applyEventToEconomy: foodMade -= 8*4096
    // UnitDone(Overseer) must restore: foodMade += 8*4096
    // But handleMorph itself doesn't run applyEventToEconomy, so check
    // that spending is tracked (50 min, 50 gas, 0 supply delta)
    assertThat(result.mineralsUsedArmy()).isEqualTo(50);
    assertThat(result.gasUsedArmy()).isEqualTo(50);
    assertThat(result.foodUsed()).isEqualTo(0);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrippedReplayMorphTest -q`
Expected: FAIL — `processMorphWithStateForTest` doesn't exist yet, `handleMorph` doesn't accept PlayerState

- [ ] **Step 3: Add PlayerState to handleMorph signature and implement trackMorphSpending**

Change `handleMorph` signature to include `PlayerState state`:

```java
int handleMorph(ReplayCommand.MorphCommand mc, int playerId,
                PlayerState state, List<SyntheticEvent> events, int tagCounter) {
```

Add at the end of handleMorph (before `return tagCounter`):

```java
    if (state != null) {
        trackMorphSpending(targetName, state);
    }
```

Add the spending method:

```java
private void trackMorphSpending(String targetName, PlayerState state) {
    // Try unit morph first
    UnitType ut = UNIT_PYTHON_NAMES.entrySet().stream()
        .filter(e -> e.getValue().equals(targetName))
        .map(Map.Entry::getKey)
        .findFirst()
        .orElse(null);
    if (ut != null) {
        int mineralCost = SC2Data.mineralCost(ut);
        int gasCost = SC2Data.gasCost(ut);
        state.mineralsUsedArmy += mineralCost;
        state.gasUsedArmy += gasCost;
        state.mineralsCurrent -= mineralCost;
        state.vespeneCurrent -= gasCost;
        state.foodUsed += SC2Data.supplyCost(ut) * 4096;
        return;
    }
    // Building morph
    BuildingType bt = BUILDING_NAME_TO_TYPE.get(targetName);
    if (bt != null) {
        int mineralCost = SC2Data.mineralCost(bt);
        int gasCost = gasCostForBuilding(bt);
        state.mineralsUsedTechnology += mineralCost;
        state.gasUsedTechnology += gasCost;
        state.mineralsCurrent -= mineralCost;
        state.vespeneCurrent -= gasCost;
    }
}
```

Update the call site in `extract()` (line 222-225):

```java
case ReplayCommand.MorphCommand mc -> {
    tagCounter = handleMorph(mc, playerId,
                             state, syntheticEvents, tagCounter);
}
```

Update `processMorphForTest` to pass null state (backwards compatible):

```java
List<Map<String, Object>> processMorphForTest(ReplayCommand.MorphCommand mc,
                                               int playerId, int startTag) {
    List<SyntheticEvent> events = new ArrayList<>();
    handleMorph(mc, playerId, null, events, startTag);
    return events.stream().map(SyntheticEvent::data).toList();
}
```

Add the `MorphTestResult` record and `processMorphWithStateForTest` helper.

- [ ] **Step 4: Fix Overseer supply in applyEventToEconomy**

In `applyEventToEconomy` (line 692-694), add Overseer handling to the UnitDone case:

```java
case "UnitDone" -> {
    if (GAS_BUILDINGS.contains(unitName)) state.gasBuildingCount++;
    if ("Overseer".equals(unitName)) state.foodMade += 8 * 4096;
}
```

- [ ] **Step 5: Run all tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrippedReplayMorphTest -q`
Expected: ALL PASS

- [ ] **Step 6: Run full test suite to check for regressions**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayMorphTest.java
git commit -m "feat: add morph spending tracking + fix Overseer supply in UnitDone path Refs #328"
```

---

## Batch 2: Morph abilLink discovery + AbilityMapping dispatch

After this batch: morph abilLinks are discovered from oracle replays, AbilityMapping dispatches MorphCommands for all morph types. No multiplication yet — each CmdEvent produces exactly 1 MorphCommand.

### Task 3: Add discoverMorphAbilLinks() to AbilityDiscoveryCalibrationTest

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AbilityDiscoveryCalibrationTest.java`

**Interfaces:**
- Consumes: `ITrackerEvents.ID_UNIT_TYPE_CHANGE` (value at line 34 of ITrackerEvents.java)
- Consumes: `discoverAll(String label, int trackerEventId)` — existing method
- Produces: stdout output with `MorphUnit:<unitName> → modal=(abilLink=X,idx=Y)` mappings

- [ ] **Step 1: Write the new discovery test method**

Add to `AbilityDiscoveryCalibrationTest`:

```java
@Test
@Tag("diagnostic")
@EnabledIf("oracleExists")
void discoverMorphAbilLinks() throws Exception {
    var mappings = discoverAll("MorphUnit",
        hu.scelightapi.sc2.rep.model.trackerevents.ITrackerEvents.ID_UNIT_TYPE_CHANGE);
    printMappings("=== Morph Unit AbilLinks ===", mappings);
    assertThat(mappings).isNotEmpty();
}
```

Note: the existing `discoverAll` method uses `IBaseUnitEvent` to cast tracker events. `UnitTypeChangeEvent` must also implement this interface for the cast to work. If the cast fails at runtime, the fallback is to write a dedicated `discoverMorphFromReplay` method that handles the different event interface.

- [ ] **Step 2: Run the diagnostic test**

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=AbilityDiscoveryCalibrationTest#discoverMorphAbilLinks -q`

If oracle replays exist: Expected output is a table of morph abilLinks. Record the values.
If oracle replays don't exist: test is skipped (EnabledIf guard). Use AbilityMappingCoverageDiagnosticTest fallback.

- [ ] **Step 3: If discoverAll cast fails, write dedicated discoverMorphFromReplay**

If `UnitTypeChangeEvent` doesn't implement `IBaseUnitEvent`, add a dedicated method:

```java
private void discoverMorphFromReplay(Path oraclePath, Path strippedPath,
                                      Map<String, Map<String, Integer>> mappings) {
    Replay rep = RepParserEngine.parseReplay(oraclePath,
        EnumSet.of(RepContent.TRACKER_EVENTS));
    if (rep == null || rep.trackerEvents == null) return;

    List<CmdRecord> commands = extractCommands(strippedPath);
    if (commands.isEmpty()) return;

    for (Event raw : rep.trackerEvents.getEvents()) {
        if (raw.getId() != ITrackerEvents.ID_UNIT_TYPE_CHANGE) continue;
        // Access fields via reflection or the specific event interface
        // UnitTypeChangeEvent has getUnitTypeName(), getControlPlayerId(), getLoop()
        Integer ctrlId = raw.getControlPlayerId();
        if (ctrlId == null || ctrlId == 0) continue;
        if (raw.getLoop() == 0) continue;

        String unitName = /* extract from event */;
        String key = "MorphUnit:" + unitName;

        findClosestCommand(commands, ctrlId, raw.getLoop(), key, mappings, 0, 500);
    }
}
```

Update `discoverMorphAbilLinks()` to use this method instead of `discoverAll()`.

- [ ] **Step 4: Record discovered abilLink values**

Save the output. The abilLink values will be used in Task 4.

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AbilityDiscoveryCalibrationTest.java
git commit -m "feat: add discoverMorphAbilLinks() diagnostic for morph abilLink discovery Refs #328"
```

### Task 4: Add morph abilLink constants and dispatch cases to AbilityMapping

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java:79` (constants), `368-394` (dispatchHuman)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AbilityMappingTest.java`

**Interfaces:**
- Consumes: abilLink values discovered in Task 3
- Produces: `AbilityMapping.dispatchHuman()` returns `MorphCommand` for all morph abilLinks

- [ ] **Step 1: Write failing tests for each morph dispatch**

Add constants and tests to `AbilityMappingTest`. Use the abilLink values from Task 3's discovery output. Example for Baneling (replace `???` with actual values):

```java
static final int ABIL_BANELING_MORPH = ???;

@Test
void humanMode_banelingMorph_producesMorphCommand() {
    humanMapping.setSelectionForTest(0, List.of("u-1"));
    var cmd = fakeCmdEvent(ABIL_BANELING_MORPH, 0, 1000, null, null, 0);
    var results = humanMapping.process(cmd);
    assertThat(results).hasSize(1);
    assertThat(results.get(0)).isInstanceOf(ReplayCommand.MorphCommand.class);
    var morph = (ReplayCommand.MorphCommand) results.get(0);
    assertThat(morph.sourceName()).isEqualTo("Zergling");
    assertThat(morph.targetName()).isEqualTo("Baneling");
}
```

Repeat for: Ravager (Roach→Ravager), Lurker (Hydralisk→Lurker), BroodLord (Corruptor→BroodLord), Overseer (Overlord→Overseer), OrbitalCommand (CommandCenter→OrbitalCommand), PlanetaryFortress (CommandCenter→PlanetaryFortress), Lair (Hatchery→Lair), Hive (Lair→Hive), GreaterSpire (Spire→GreaterSpire).

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=AbilityMappingTest -q`
Expected: FAIL — unknown abilLinks not dispatched

- [ ] **Step 3: Add constants and dispatch cases to AbilityMapping**

Add constants after `ABIL_ARCHON_MERGE` (line 79):

```java
private static final int ABIL_BANELING_MORPH = ???;
private static final int ABIL_RAVAGER_MORPH = ???;
private static final int ABIL_LURKER_MORPH = ???;
private static final int ABIL_BROODLORD_MORPH = ???;
private static final int ABIL_OVERSEER_MORPH = ???;
private static final int ABIL_ORBITAL_COMMAND_MORPH = ???;
private static final int ABIL_PLANETARY_FORTRESS_MORPH = ???;
private static final int ABIL_LAIR_MORPH = ???;
private static final int ABIL_HIVE_MORPH = ???;
private static final int ABIL_GREATER_SPIRE_MORPH = ???;
```

Add cases to `dispatchHuman()` (after the ABIL_ARCHON_MERGE case, line 391):

```java
case ABIL_BANELING_MORPH -> isRace(Race.ZERG)
    ? List.of(new ReplayCommand.MorphCommand(loop, "Zergling", "Baneling")) : null;
case ABIL_RAVAGER_MORPH -> isRace(Race.ZERG)
    ? List.of(new ReplayCommand.MorphCommand(loop, "Roach", "Ravager")) : null;
case ABIL_LURKER_MORPH -> isRace(Race.ZERG)
    ? List.of(new ReplayCommand.MorphCommand(loop, "Hydralisk", "Lurker")) : null;
case ABIL_BROODLORD_MORPH -> isRace(Race.ZERG)
    ? List.of(new ReplayCommand.MorphCommand(loop, "Corruptor", "BroodLord")) : null;
case ABIL_OVERSEER_MORPH -> isRace(Race.ZERG)
    ? List.of(new ReplayCommand.MorphCommand(loop, "Overlord", "Overseer")) : null;
case ABIL_ORBITAL_COMMAND_MORPH -> isRace(Race.TERRAN)
    ? List.of(new ReplayCommand.MorphCommand(loop, "CommandCenter", "OrbitalCommand")) : null;
case ABIL_PLANETARY_FORTRESS_MORPH -> isRace(Race.TERRAN)
    ? List.of(new ReplayCommand.MorphCommand(loop, "CommandCenter", "PlanetaryFortress")) : null;
case ABIL_LAIR_MORPH -> isRace(Race.ZERG)
    ? List.of(new ReplayCommand.MorphCommand(loop, "Hatchery", "Lair")) : null;
case ABIL_HIVE_MORPH -> isRace(Race.ZERG)
    ? List.of(new ReplayCommand.MorphCommand(loop, "Lair", "Hive")) : null;
case ABIL_GREATER_SPIRE_MORPH -> isRace(Race.ZERG)
    ? List.of(new ReplayCommand.MorphCommand(loop, "Spire", "GreaterSpire")) : null;
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=AbilityMappingTest -q`
Expected: ALL PASS

- [ ] **Step 5: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AbilityMappingTest.java
git commit -m "feat: add morph abilLink dispatch for 10 morph types Refs #328"
```

---

## Batch 3: Selection-based multiplication

After this batch: batch morphs (e.g., 5 Zerglings selected → 5 Banelings) produce the correct count of morph events via SelectionUnitLinkTracker.

### Task 5: Wire SelectionUnitLinkTracker into extract() and add morph multiplication

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java:161-252` (extract loop), `128-131` (add MORPH_TARGET_TO_SOURCE_LINKS)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayMorphTest.java` (new multiplication test)

**Interfaces:**
- Consumes: `SelectionUnitLinkTracker.countMatching(Set<Integer>)` — returns count of selected units matching given unitLinks
- Consumes: `SelectionUnitLinkTracker.onSelection(SelectionDeltaEvent)` — feeds selection events
- Produces: `MORPH_TARGET_TO_SOURCE_LINKS` — `Map<String, Set<Integer>>` mapping morph target name to source unit unitLink values

- [ ] **Step 1: Write failing test for morph multiplication**

Add a test helper that exercises the full extract loop with selection + morph. Since `extract()` takes a replay Path, we need a lower-level test. Add a package-private method:

```java
record MorphMultiplicationResult(List<Map<String, Object>> events, int morphCount) {}

MorphMultiplicationResult processMorphWithSelectionForTest(
        ReplayCommand.MorphCommand mc, int playerId, int startTag,
        int selectedSourceCount, int sourceUnitLink) {
    List<SyntheticEvent> events = new ArrayList<>();
    var state = new PlayerState();
    state.mineralsCurrent = 10000;
    state.vespeneCurrent = 10000;

    int count = 1;
    if (!BUILDING_MORPH_TARGETS.contains(mc.targetName())) {
        Set<Integer> sourceLinks = MORPH_TARGET_TO_SOURCE_LINKS.get(mc.targetName());
        if (sourceLinks != null && selectedSourceCount > 0) {
            count = Math.min(selectedSourceCount, MAX_MULTIPLICATION);
        }
    }

    int tagCounter = startTag;
    for (int r = 0; r < count; r++) {
        tagCounter = handleMorph(mc, playerId, state, events, tagCounter);
    }
    return new MorphMultiplicationResult(
        events.stream().map(SyntheticEvent::data).toList(), count);
}
```

Test:

```java
@Test
void morphMultiplicationEmitsMultipleEvents() {
    var extractor = new StrippedReplayFeatureExtractor();
    var morph = new ReplayCommand.MorphCommand(2000, "Zergling", "Baneling");

    var result = extractor.processMorphWithSelectionForTest(morph, 1, 100, 3, 105);

    assertThat(result.morphCount()).isEqualTo(3);
    var deaths = result.events().stream()
        .filter(e -> "UnitDied".equals(e.get("evtTypeName")))
        .toList();
    var inits = result.events().stream()
        .filter(e -> "UnitInit".equals(e.get("evtTypeName")))
        .toList();
    assertThat(deaths).hasSize(3);
    assertThat(inits).hasSize(3);
}

@Test
void buildingMorphIgnoresMultiplication() {
    var extractor = new StrippedReplayFeatureExtractor();
    var morph = new ReplayCommand.MorphCommand(3000, "Hatchery", "Lair");

    var result = extractor.processMorphWithSelectionForTest(morph, 1, 100, 3, 999);

    assertThat(result.morphCount()).isEqualTo(1);
}

@Test
void morphMultiplicationCappedAtMax() {
    var extractor = new StrippedReplayFeatureExtractor();
    var morph = new ReplayCommand.MorphCommand(2000, "Zergling", "Baneling");

    var result = extractor.processMorphWithSelectionForTest(morph, 1, 100, 10, 105);

    assertThat(result.morphCount()).isEqualTo(4); // MAX_MULTIPLICATION = 4
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrippedReplayMorphTest -q`
Expected: FAIL — `MORPH_TARGET_TO_SOURCE_LINKS` doesn't exist, `processMorphWithSelectionForTest` doesn't exist

- [ ] **Step 3: Add MORPH_TARGET_TO_SOURCE_LINKS map and test helper**

Add the map as a static field. UnitLink values for source units need to be discovered from replays alongside abilLinks (Task 3). Use placeholder values initially — calibrate from `UnitLinkDiscoveryTest` or `BarracksUnitLinkDiscoveryTest` pattern:

```java
private static final Map<String, Set<Integer>> MORPH_TARGET_TO_SOURCE_LINKS = Map.of(
    "Baneling", Set.of(/* Zergling unitLink — discovered value */),
    "Ravager", Set.of(/* Roach unitLink */),
    "Lurker", Set.of(/* Hydralisk unitLink */),
    "BroodLord", Set.of(/* Corruptor unitLink */),
    "Overseer", Set.of(/* Overlord unitLink */)
);
```

Add the test helper method and wire multiplication into `extract()`:

```java
case ReplayCommand.MorphCommand mc -> {
    int morphCount = 1;
    if (!BUILDING_MORPH_TARGETS.contains(mc.targetName())) {
        Set<Integer> sourceLinks = MORPH_TARGET_TO_SOURCE_LINKS.get(mc.targetName());
        if (sourceLinks != null) {
            int selected = tracker.countMatching(sourceLinks);
            if (selected > 0) {
                morphCount = Math.min(selected, MAX_MULTIPLICATION);
            }
        }
    }
    for (int r = 0; r < morphCount; r++) {
        tagCounter = handleMorph(mc, playerId, state, syntheticEvents, tagCounter);
    }
}
```

Also wire `SelectionUnitLinkTracker` creation and feeding in `extract()`:

```java
// After AbilityMapping creation (line 176):
var tracker = new SelectionUnitLinkTracker(playerId);

// In the SelectionDeltaEvent handling (line 183-184):
if (raw instanceof SelectionDeltaEvent sel) {
    mapping.onSelection(sel);
    tracker.onSelection(sel);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrippedReplayMorphTest -q`
Expected: ALL PASS

- [ ] **Step 5: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayMorphTest.java
git commit -m "feat: add selection-based morph multiplication via SelectionUnitLinkTracker Refs #328"
```

---

## Batch 4: Calibration + validation

After this batch: morph times are calibrated from replay data, coverage validation passes acceptance criteria.

### Task 6: Calibrate morph times in SC2Data and run coverage validation

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java:173-184` (trainTimeInLoops entries)
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/SC2MorphTimeCalibrationTest.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayValidationTest.java` (run for coverage)

**Interfaces:**
- Consumes: Oracle replay data — CmdEvent loop vs UnitTypeChangeEvent loop delta
- Produces: Calibrated values in `SC2Data.trainTimeInLoops()` for Baneling, Ravager, Lurker, BroodLord, Overseer

- [ ] **Step 1: Write calibration test**

```java
@Tag("diagnostic")
class SC2MorphTimeCalibrationTest {

    private static final Path ORACLE_RESTORED = Path.of("../quarkmind-classifier/data/replay_packs/oracle_restored");
    private static final Path ORACLE_INPUT = Path.of("../quarkmind-classifier/data/replay_packs/oracle_input");

    @Test
    @EnabledIf("oracleExists")
    void measureMorphTimesFromReplays() throws Exception {
        // For each UnitTypeChangeEvent, find the preceding CmdEvent
        // Measure the loop delta = UnitTypeChangeEvent.loop - CmdEvent.loop
        // Group by unit type, take the median
        // Print results for calibration
        Map<String, List<Long>> morphDeltas = new TreeMap<>();

        try (var stream = Files.list(ORACLE_RESTORED)) {
            for (Path oraclePath : stream.filter(p -> p.toString().endsWith(".SC2Replay")).sorted().toList()) {
                Path strippedPath = ORACLE_INPUT.resolve(oraclePath.getFileName());
                if (!Files.exists(strippedPath)) continue;
                measureMorphTimesFromReplay(oraclePath, strippedPath, morphDeltas);
            }
        }

        System.out.println("=== Morph Time Calibration ===");
        for (var entry : morphDeltas.entrySet()) {
            var deltas = entry.getValue().stream().sorted().toList();
            long median = deltas.get(deltas.size() / 2);
            System.out.printf("  %-20s  median=%d loops (%d samples)%n",
                entry.getKey(), median, deltas.size());
        }
    }

    static boolean oracleExists() {
        return Files.isDirectory(ORACLE_RESTORED);
    }
}
```

- [ ] **Step 2: Run calibration test and record values**

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=SC2MorphTimeCalibrationTest -q`
Record median morph times for each unit type.

- [ ] **Step 3: Update SC2Data.trainTimeInLoops() with calibrated values**

Replace the placeholder 672-loop entries:

```java
map.put(UnitType.BANELING, ???);     // calibrated from replays
map.put(UnitType.RAVAGER, ???);      // calibrated from replays
map.put(UnitType.LURKER, ???);       // calibrated from replays
map.put(UnitType.BROOD_LORD, ???);   // calibrated from replays
map.put(UnitType.OVERSEER, ???);     // calibrated from replays
```

If insufficient replay data for a unit, use Liquipedia values: Baneling=448, Ravager=269, Lurker=536, BroodLord=804, Overseer=269. Note as a known deviation in a code comment.

- [ ] **Step 4: Run StrippedReplayValidationTest for coverage check**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=StrippedReplayValidationTest -q`
Check output for Baneling coverage >= 70% and building morph presence.

- [ ] **Step 5: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/SC2MorphTimeCalibrationTest.java
git commit -m "feat: calibrate morph times from replay data Refs #328"
```

### Task 7: Update CLAUDE.md test listings and file issues for out-of-scope items

**Files:**
- Modify: `CLAUDE.md` — add new test classes to listings

**Interfaces:**
- None (documentation task)

- [ ] **Step 1: Add new test classes to CLAUDE.md**

Add to the unit tests listing: `SC2MorphTimeCalibrationTest`
Verify `StrippedReplayMorphTest` and `AbilityMappingTest` and `AbilityDiscoveryCalibrationTest` are already listed.

- [ ] **Step 2: File GitHub issues for out-of-scope items**

1. Archon DT source identification: `ABIL_ARCHON_MERGE` hardcodes "HighTemplar" — DarkTemplar merge not distinguished
2. Morph time re-calibration for rare units: if Liquipedia fallback was used, file issue to re-calibrate when more replay data is available

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md test listings for morph events Refs #328"
```

---

## References

- [2026-09-30-morph-unit-events-design.md] — design spec this plan implements
- [quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java] — morph dispatch (lines 368-394)
- [quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java] — handleMorph (lines 395-444), extract loop (lines 161-252), applyEventToEconomy (lines 680-701), SelectionUnitLinkTracker (lines 822-933)
- [quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayCommand.java] — MorphCommand record
- [quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java] — trainTimeInLoops (lines 173-184), UnitCosts (lines 370-381)
- [quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayMorphTest.java] — existing morph tests
- [quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AbilityDiscoveryCalibrationTest.java] — discovery infrastructure
- [quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AbilityMappingTest.java] — dispatch tests
- [PP-20260528-612dee] — extractor-separate-from-simulated-game
- [PP-20260913-4d73d1] — enum-switch-exhaustive-required
- [PP-20260528-d9f967] — replay-tag-prefix-per-source
- [sc2data-train-times-require-calibration] — calibrate from replay data
- [GitHub #328] — focal issue
- [GitHub #318] — parent epic
