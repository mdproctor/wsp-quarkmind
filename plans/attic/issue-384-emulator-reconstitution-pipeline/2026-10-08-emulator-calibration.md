# Emulator Calibration Implementation Plan (Phases 1–2)

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #384 — Epic: Emulator-driven reconstitution pipeline
**Issue group:** Child issues to be created per batch

**Goal:** Raise EmulatedGame unit accuracy from 36.4% to ≥80% at 5-min checkpoint by extracting all replay command types and adding morph support, then calibrate economy physics.

**Architecture:** `ReplayCommandExtractor` already parses Build, Upgrade, and Morph commands via `AbilityMapping` but discards them. `EmulatedGame` already handles `BuildIntent` and `ResearchIntent`. The core work is: (1) stop discarding commands, (2) convert them to `TimedIntent`, (3) add `MorphIntent` to the sealed `Intent` hierarchy, (4) handle morphs in `EmulatedGame`.

**Tech Stack:** Java 21, Quarkus, Scelight replay parser, JUnit 5

## Global Constraints

- All timing constants must be calibrated from replay ground truth (PP-20260522-572156)
- Command extraction belongs in extractor classes, not SimulatedGame (PP-20260528-612dee)
- `Intent` is a sealed interface — updating `permits` requires updating every `switch` over `Intent`
- Tests that can be plain JUnit must not use `@QuarkusTest`
- Every commit references an issue

---

## Batch 1: Extract all command types from replays

### Task 1: Expand ReplayCommandStream and ReplayCommandExtractor to emit Build, Upgrade, and Morph commands

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayCommandStream.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayCommandExtractor.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ReplayCommandExtractorTest.java`

**Interfaces:**
- Consumes: `AbilityMapping.process(CmdEvent)` → `List<ReplayCommand>` (existing)
- Produces: `ReplayCommandStream` gains `buildCommands()`, `upgradeCommands()`, `morphCommands()` accessors

- [ ] **Step 1: Write the failing test**

Add a test that asserts `ReplayCommandExtractor.extract()` returns non-empty build commands for an oracle replay. Use an AI Arena Protoss replay (known to have Nexus/Pylon builds).

```java
@Test
void extractShouldReturnBuildCommands() {
    Path replay = Path.of("../quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored")
        .resolve(findFirstReplay());
    ReplayCommandStream stream = ReplayCommandExtractor.extract(replay, 1);
    assertThat(stream.buildCommands()).isNotEmpty();
    assertThat(stream.buildCommands().get(0).buildingName()).isNotNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ReplayCommandExtractorTest#extractShouldReturnBuildCommands -q`
Expected: FAIL — `buildCommands()` method does not exist on `ReplayCommandStream`

- [ ] **Step 3: Add fields to ReplayCommandStream**

Use `ide_replace_member` to update the record:

```java
public record ReplayCommandStream(
    List<UnitOrder>          movementOrders,
    List<TimedIntent>        intents,
    List<ReplayCommand.BuildCommand>   buildCommands,
    List<ReplayCommand.UpgradeCommand> upgradeCommands,
    List<ReplayCommand.MorphCommand>   morphCommands) {}
```

- [ ] **Step 4: Update ReplayCommandExtractor to collect all command types**

Replace the switch block in `extract()` that discards Build/Upgrade/Morph:

```java
List<ReplayCommand.BuildCommand>   builds   = new ArrayList<>();
List<ReplayCommand.UpgradeCommand> upgrades = new ArrayList<>();
List<ReplayCommand.MorphCommand>   morphs   = new ArrayList<>();

// In the switch:
case ReplayCommand.BuildCommand   b -> builds.add(b);
case ReplayCommand.UpgradeCommand u -> upgrades.add(u);
case ReplayCommand.MorphCommand   m -> morphs.add(m);
case ReplayCommand.CancelCommand  ignored -> {}

// Return:
return new ReplayCommandStream(
    Collections.unmodifiableList(orders),
    Collections.unmodifiableList(intents),
    Collections.unmodifiableList(builds),
    Collections.unmodifiableList(upgrades),
    Collections.unmodifiableList(morphs));
```

- [ ] **Step 5: Fix existing callers of ReplayCommandStream**

`ReplayCommandStream` is a record — adding fields changes the canonical constructor. Find all callers and update them. The `ReplayValidationHarness.run(Path, int, int)` at line 150 calls `ReplayCommandExtractor.extract()` and uses `.intents()` — this still works. Check for any test code that constructs `ReplayCommandStream` directly.

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ReplayCommandExtractorTest#extractShouldReturnBuildCommands -q`
Expected: PASS

- [ ] **Step 7: Add tests for upgrade and morph commands**

```java
@Test
void extractShouldReturnUpgradeCommands() {
    // Use an oracle replay long enough to have upgrades (5+ min)
    ReplayCommandStream stream = ReplayCommandExtractor.extract(replay, 1);
    // At least some replays will have upgrades
    // This is a smoke test — specific upgrade coverage tested in calibration tests
    assertThat(stream.upgradeCommands()).isNotNull();
}

@Test
void extractShouldReturnMorphCommandsForZerg() {
    // Find a Zerg player in the oracle replays
    ReplayCommandStream stream = ReplayCommandExtractor.extract(zergReplay, zergPlayerId);
    assertThat(stream.morphCommands()).isNotNull();
}
```

- [ ] **Step 8: Run full test suite and verify no regression**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all existing tests pass

- [ ] **Step 9: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayCommandStream.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayCommandExtractor.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ReplayCommandExtractorTest.java
git commit -m "feat: extract Build/Upgrade/Morph commands from replays

ReplayCommandExtractor previously discarded BuildCommand, UpgradeCommand,
and MorphCommand from AbilityMapping output. Now collects all command types
in ReplayCommandStream for downstream consumption.

Refs #384"
```

---

### Task 2: Add MorphIntent to the sealed Intent hierarchy

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/MorphIntent.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/Intent.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` (two `applyIntent` switch expressions)
- Modify: any other files with exhaustive `switch` over `Intent`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/intent/MorphIntentTest.java`

**Interfaces:**
- Consumes: `UnitType` (source and target types for the morph)
- Produces: `MorphIntent(String unitTag, UnitType sourceType, UnitType targetType)` — sealed permit on `Intent`

- [ ] **Step 1: Find all exhaustive switches over Intent**

Use `ide_find_references` on the `Intent` interface to find every switch expression. Known locations:
- `EmulatedGame.applyIntent(TimedIntent)` line 268
- `EmulatedGame.applyIntent(Intent, PlayerState, PhysicsState)` line 281
- Any other files

- [ ] **Step 2: Create MorphIntent record**

```java
package io.quarkmind.sc2.intent;

import io.quarkmind.domain.UnitType;

public record MorphIntent(String unitTag, UnitType sourceType, UnitType targetType) implements Intent {}
```

- [ ] **Step 3: Update Intent sealed permits**

```java
public sealed interface Intent permits BuildIntent, TrainIntent, AttackIntent, MoveIntent, BlinkIntent, MuleCalldownIntent, ResearchIntent, MorphIntent {
}
```

- [ ] **Step 4: Add MorphIntent case to EmulatedGame.applyIntent(TimedIntent)**

Add after the `ResearchIntent` case at line 275:

```java
case MorphIntent        m -> () -> handleMorph(m, friendly, friendlyPhysics, ti.loop());
```

- [ ] **Step 5: Add MorphIntent case to EmulatedGame.applyIntent(Intent, PlayerState, PhysicsState)**

Add after the `ResearchIntent` case at line 288:

```java
case MorphIntent        m -> () -> handleMorph(m, state, physics, gameFrame * SC2Data.LOOPS_PER_TICK);
```

- [ ] **Step 6: Implement handleMorph in EmulatedGame**

Add a private method. Morph removes the source unit and starts training the target unit. For unit morphs (Zergling→Baneling), the source is consumed. For building morphs (Hatchery→Lair), the building is replaced in-place.

```java
private void handleMorph(MorphIntent m, PlayerState state, PhysicsState physics, long absLoop) {
    if (SC2Data.isBuildingType(m.targetType())) {
        // Building morph: replace building in-place
        Building source = state.buildings().stream()
            .filter(b -> b.tag().equals(m.unitTag()) && b.isComplete())
            .findFirst().orElse(null);
        if (source == null) {
            log.debugf("[EMULATED] Morph rejected — building %s not found", m.unitTag());
            return;
        }
        BuildingType targetBt = SC2Data.toBuildingType(m.targetType());
        state.replaceAllBuildings(b -> b.tag().equals(m.unitTag())
            ? new Building(b.tag(), targetBt, b.position(), b.health(), b.maxHealth(), false)
            : b);
        long completesAt = gameFrame + SC2Data.buildTimeInLoops(targetBt) / SC2Data.LOOPS_PER_TICK;
        physics.pendingCompletions.add(new PhysicsState.PendingCompletion(completesAt, () -> {
            markBuildingComplete(m.unitTag(), state);
            log.debugf("[EMULATED] Morph complete — %s → %s", m.sourceType(), m.targetType());
        }));
    } else {
        // Unit morph: remove source unit, train target
        state.removeUnit(m.unitTag());
        String tag = nextTagString();
        Unit source = state.units().stream()
            .filter(u -> u.tag().equals(m.unitTag())).findFirst().orElse(null);
        Point2d pos = source != null ? source.position() : new Point2d(8, 8);
        int hp = SC2Data.maxHealth(m.targetType());
        state.addUnit(new Unit(tag, m.targetType(), pos, hp, hp, 0, 0, 0, 0));
        log.debugf("[EMULATED] Morph %s → %s (tag=%s)", m.sourceType(), m.targetType(), tag);
    }
}
```

Note: this is a simplified morph. Real SC2 morphs have build times and costs. Refine in Phase 2 calibration.

- [ ] **Step 7: Fix any other exhaustive switches**

Search for compilation errors from the `switch` exhaustiveness check. Any file with `case TrainIntent` in a switch over `Intent` needs a `case MorphIntent` branch.

- [ ] **Step 8: Write test for MorphIntent**

```java
@Test
void morphIntentShouldBeAnIntent() {
    MorphIntent morph = new MorphIntent("ling-1", UnitType.ZERGLING, UnitType.BANELING);
    assertThat(morph).isInstanceOf(Intent.class);
    assertThat(morph.sourceType()).isEqualTo(UnitType.ZERGLING);
    assertThat(morph.targetType()).isEqualTo(UnitType.BANELING);
}
```

- [ ] **Step 9: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: PASS (may need `mvn clean` if `ClassTooLargeException` — see CLAUDE.md)

- [ ] **Step 10: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/MorphIntent.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/Intent.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/intent/MorphIntentTest.java
git commit -m "feat: add MorphIntent to sealed Intent hierarchy

Supports unit morphs (Zergling→Baneling, Roach→Ravager, etc.) and building
morphs (Hatchery→Lair, CC→OrbitalCommand). EmulatedGame handles both:
unit morphs remove source and spawn target; building morphs replace in-place.

Refs #384"
```

---

### Task 3: Convert extracted commands to TimedIntents for the validation harness

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayCommandExtractor.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayCommandStream.java` (may need `allIntents()` combining train + build + upgrade + morph)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java` (use expanded intents)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ReplayCommandExtractorTest.java`

**Interfaces:**
- Consumes: `ReplayCommand.BuildCommand`, `ReplayCommand.UpgradeCommand`, `ReplayCommand.MorphCommand`
- Produces: `ReplayCommandStream.allIntents()` → sorted `List<TimedIntent>` combining all command types

- [ ] **Step 1: Write failing test**

```java
@Test
void allIntentsShouldIncludeBuildAndUpgradeIntents() {
    ReplayCommandStream stream = ReplayCommandExtractor.extract(oracleReplay, 1);
    List<TimedIntent> all = stream.allIntents();
    boolean hasBuild = all.stream().anyMatch(ti -> ti.intent() instanceof BuildIntent);
    boolean hasTrain = all.stream().anyMatch(ti -> ti.intent() instanceof TrainIntent);
    assertThat(hasTrain).isTrue();
    assertThat(hasBuild).isTrue();
}
```

- [ ] **Step 2: Run test to verify failure**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ReplayCommandExtractorTest#allIntentsShouldIncludeBuildAndUpgradeIntents -q`
Expected: FAIL — `allIntents()` method doesn't exist

- [ ] **Step 3: Convert commands to TimedIntents in ReplayCommandExtractor**

In the `extract()` method, convert each command type:

```java
// After the existing switch block, convert commands to intents:
for (ReplayCommand.BuildCommand b : builds) {
    BuildingType bt = Sc2ReplayShared.toBuildingType(b.buildingName());
    if (bt != null) {
        Point2d pos = b.position() != null ? b.position() : new Point2d(8, 8);
        intents.add(new TimedIntent(b.loop(),
            new BuildIntent(selection.first(), bt, pos)));
    }
}
for (ReplayCommand.UpgradeCommand u : upgrades) {
    UpgradeType ut = UpgradeType.fromName(u.upgradeName());
    if (ut != null) {
        intents.add(new TimedIntent(u.loop(),
            new ResearchIntent(selection.first(), ut)));
    }
}
for (ReplayCommand.MorphCommand m : morphs) {
    UnitType source = UnitType.fromName(m.sourceName());
    UnitType target = UnitType.fromName(m.targetName());
    if (source != null && target != null) {
        intents.add(new TimedIntent(m.loop(),
            new MorphIntent(selection.first(), source, target)));
    }
}
```

Note: The exact conversion depends on how `selection` state maps to building tags. `selection.first()` returns the tag of the currently selected unit/building. For Build commands from `dispatchHuman`, the selected unit is the worker. For Upgrade commands, it's the research building. This may need per-command-type tag resolution — diagnose from test output.

- [ ] **Step 4: Add allIntents() to ReplayCommandStream**

```java
public List<TimedIntent> allIntents() {
    // intents already contains TrainIntents; builds/upgrades/morphs
    // were converted to TimedIntents in the extractor. Just return intents.
    return intents;
}
```

Or, if the conversion happens in `allIntents()` instead of in the extractor, merge and sort:

```java
public List<TimedIntent> allIntents() {
    var all = new ArrayList<>(intents);
    // ... convert raw commands ...
    all.sort(Comparator.comparingLong(TimedIntent::loop));
    return Collections.unmodifiableList(all);
}
```

The exact design depends on where the conversion belongs. The extractor is more appropriate (per PP-20260528-612dee: extraction logic in extractor classes).

- [ ] **Step 5: Update ReplayValidationHarness to use expanded intents**

In `ReplayValidationHarness.run(Path, int, int)` at line 150, `intents` already comes from `ReplayCommandExtractor.extract().intents()`. If the conversion happens in the extractor, `intents` already includes Build/Upgrade/Morph — no harness change needed. If `allIntents()` is separate, change line 150:

```java
List<TimedIntent> intents = ReplayCommandExtractor.extract(replayPath, playerId).allIntents();
```

- [ ] **Step 6: Run test to verify pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ReplayCommandExtractorTest#allIntentsShouldIncludeBuildAndUpgradeIntents -q`
Expected: PASS

- [ ] **Step 7: Run accuracy baseline to measure improvement**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=EmulatedGameAccuracyBaselineTest -q`

This will show whether extracted Build/Upgrade/Morph intents are being applied. Capture the new accuracy numbers.

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayCommandExtractor.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayCommandStream.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ReplayCommandExtractorTest.java
git commit -m "feat: convert Build/Upgrade/Morph commands to TimedIntents

ReplayCommandExtractor now converts BuildCommand→BuildIntent,
UpgradeCommand→ResearchIntent, MorphCommand→MorphIntent and includes
them in the intents list. ReplayValidationHarness applies all command
types through EmulatedGame.

Refs #384"
```

---

## Batch 2: Diagnose and fix Terran/Zerg 0% unit accuracy

### Task 4: Diagnose root cause of 0% Terran/Zerg unit production

**Files:**
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/EmulatedGameAccuracyBaselineTest.java` (or new diagnostic test)

**Interfaces:**
- Consumes: Output from Task 3 (expanded extraction)
- Produces: Documented root cause and fix direction

- [ ] **Step 1: Run accuracy baseline with logging**

After Task 3, re-run the accuracy baseline and look at per-race breakdowns. The existing baseline already breaks down by unit type. Focus on:
- Are `TrainIntent`s being generated for Terran/Zerg buildings? (Check `stream.intents()` counts per race)
- Are `TrainIntent` building tags matching harness-injected building tags?
- Are intents being rejected by `handleTrain` (resource checks, building tag mismatch)?

- [ ] **Step 2: Write a diagnostic test**

```java
@Test
@EnabledIf("oracleExists")
void diagnosePerRaceExtractionCoverage() {
    // For each oracle replay, count intents by type per race
    // Print: race, total TrainIntents, total BuildIntents, total UpgradeIntents, total MorphIntents
    // This reveals whether extraction produces commands for all races
}
```

- [ ] **Step 3: Analyse results and identify fix**

The most likely root causes (in order of probability):
1. `TrainIntent` building tags from `selection.first()` don't match harness-injected building tags (harness uses replay tracker tags like `building-42`, but selection state may return a different format)
2. `AbilityMapping.dispatch()` race gates filter out valid commands (the `isRace()` check requires knowing the player's race, which depends on constructor args)
3. Non-Protoss ability IDs are not mapped in `dispatch()`

Document the root cause in the test output.

- [ ] **Step 4: Commit diagnostic findings**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/
git commit -m "diag: per-race extraction coverage analysis

Documents root cause of 0% Terran/Zerg unit accuracy.

Refs #384"
```

---

### Task 5: Fix the identified root cause for Terran/Zerg production

**Files:**
- Depends on Task 4 findings. Most likely candidates:
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java` (if ability ID mapping)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayCommandExtractor.java` (if tag resolution)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java` (if tag remapping)
- Test: relevant existing test + new regression test

**Interfaces:**
- Consumes: Task 4 diagnosis
- Produces: Non-zero Terran/Zerg unit counts in accuracy baseline

- [ ] **Step 1: Write a failing test targeting the specific root cause**

The test depends on what Task 4 finds. Example for building tag mismatch:

```java
@Test
void terranTrainIntentShouldMatchHarnessBuildingTags() {
    // Extract intents for a known Terran player
    // Verify TrainIntent building tags are in the format the harness injects
}
```

- [ ] **Step 2: Implement the fix**

Apply the fix identified in Task 4.

- [ ] **Step 3: Run accuracy baseline**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=EmulatedGameAccuracyBaselineTest -q`
Expected: Terran and Zerg unit counts are now non-zero. Overall unit accuracy should jump significantly from 36.4%.

- [ ] **Step 4: Update baseline document**

Write new accuracy numbers to `docs/benchmarks/emulated-game-accuracy-baseline.md`.

- [ ] **Step 5: Run regression tests**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all pass, no regression

- [ ] **Step 6: Commit**

```bash
git add <changed files>
git commit -m "fix: enable Terran/Zerg unit production in EmulatedGame

<describe the specific fix based on Task 4 findings>

Refs #384"
```

---

## Batch 3: Calibrate economy physics

### Task 6: Implement worker saturation curves

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` (mining income calculation)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

**Interfaces:**
- Consumes: `miningProbesPerBase` (already set by harness from GT)
- Produces: `SC2Data.mineralIncomePerTick(int workersPerBase)` with saturation-aware returns

- [ ] **Step 1: Write failing test for saturation**

SC2's mining model: 2 workers per mineral patch at full efficiency, then diminishing returns. With 8 mineral patches per base:
- 0-16 workers: full rate (~55 minerals/min per worker)
- 16-24 workers: ~40% efficiency per extra worker
- 24+ workers: near-zero additional income

```java
@Test
void mineralIncomeShouldDiminishWithSaturation() {
    int income16 = SC2Data.mineralIncomePerTickForBase(16);
    int income24 = SC2Data.mineralIncomePerTickForBase(24);
    int income32 = SC2Data.mineralIncomePerTickForBase(32);
    // 24 workers should produce less than 1.5x of 16 (not 1.5x linear)
    assertThat(income24).isLessThan(income16 * 3 / 2);
    // 32 workers should produce barely more than 24
    assertThat(income32 - income24).isLessThan(income24 - income16);
}
```

- [ ] **Step 2: Implement saturation curve in SC2Data**

Calibrate from replay data — run `SC2TrainTimeCalibrationTest` pattern but for economy. Use `PlayerStats` tracker events (mineralCollectionRate) correlated with worker counts at the same game loop.

```java
public static int mineralIncomePerTickForBase(int workers) {
    // SC2 saturation model: 8 mineral patches, 2 optimal workers each
    int optimal = Math.min(workers, 16);
    int oversaturated = Math.max(0, Math.min(workers, 24) - 16);
    int excess = Math.max(0, workers - 24);
    return optimal * MINERAL_PER_WORKER_OPTIMAL
         + oversaturated * MINERAL_PER_WORKER_OVERSATURATED
         + excess * MINERAL_PER_WORKER_EXCESS;
}
```

Constants TBD from calibration — do not guess. Write a calibration test first.

- [ ] **Step 3: Update EmulatedGame to use saturation income**

Replace the flat `mineralIncomePerTick` call in `EmulatedGame.tick()` with the saturation-aware version.

- [ ] **Step 4: Run accuracy baseline**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=EmulatedGameAccuracyBaselineTest -q`
Focus on: `mineralsCollectionRate` error (currently 98% error) and `mineralsCurrent` error (55%).

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "feat: implement worker saturation curves for mineral income

Replaces flat mineralIncomePerTick with saturation-aware model:
16 optimal workers at full rate, 8 oversaturated at ~40%, excess near-zero.
Constants calibrated from oracle replay PlayerStats.

Refs #384"
```

---

### Task 7: Track technology and building spending in EconomyTracker

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EconomyTracker.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` (call tracker from handleResearch, handleBuild)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

**Interfaces:**
- Consumes: `handleResearch()` and `handleBuild()` (both already deduct resources)
- Produces: `EconomyTracker.mineralsUsedCurrentTechnology()`, `vespeneUsedCurrentTechnology()`

- [ ] **Step 1: Write failing test**

```java
@Test
void economyTrackerShouldTrackTechnologySpending() {
    EmulatedGame game = new EmulatedGame();
    game.reset();
    // Setup: spawn a complete CyberneticsCore, give enough resources
    game.spawnBuildingForTesting(BuildingType.CYBERNETICS_CORE, new Point2d(10, 10));
    game.setMineralsForTesting(500);
    // Apply research
    game.applyIntent(new TimedIntent(100, new ResearchIntent("bldg-tag", UpgradeType.WARP_GATE_RESEARCH)));
    GameState snap = game.snapshot();
    assertThat(snap.playerEconomy().mineralsUsedCurrentTechnology()).isGreaterThan(0);
}
```

- [ ] **Step 2: Implement technology spending tracking**

In `handleResearch()`, after verifying the building exists, record spending:

```java
int mCost = SC2Data.upgradeMineralCost(r.upgradeType());
int vCost = SC2Data.upgradeVespeneCost(r.upgradeType());
economyTracker.recordTechnologySpending(mCost, vCost);
```

Add the cost lookup to `SC2Data` (calibrate from replay data) and the tracking to `EconomyTracker`.

- [ ] **Step 3: Run accuracy baseline**

Focus on: `mineralsUsedCurrentTechnology` (currently 100% error) and `vespeneUsedCurrentTechnology` (100% error).

- [ ] **Step 4: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EconomyTracker.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git add quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "feat: track technology and building spending in EconomyTracker

Adds upgrade mineral/vespene cost lookup to SC2Data and records spending
in EconomyTracker from handleResearch and handleBuild.

Refs #384"
```

---

## Batch 4: Accuracy gate and tournament baseline

### Task 8: Run final accuracy baseline and update documentation

**Files:**
- Modify: `docs/benchmarks/emulated-game-accuracy-baseline.md`
- Potentially modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceRegressionTest.java` (update thresholds)

**Interfaces:**
- Consumes: All prior fixes
- Produces: Updated accuracy baseline, updated regression thresholds

- [ ] **Step 1: Run full accuracy baseline**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=EmulatedGameAccuracyBaselineTest -q`

Capture output and write to `docs/benchmarks/emulated-game-accuracy-baseline.md`.

- [ ] **Step 2: Assess against success criteria**

| Metric | Target | Actual |
|--------|--------|--------|
| Unit accuracy at 5-min | ≥ 80% | ? |
| Upgrade accuracy | > 0% | ? |
| Economy MAPE | ≤ 50% | ? |

If unit accuracy < 80%, return to Task 4/5 pattern: diagnose the next largest gap and fix it.

- [ ] **Step 3: Update DivergenceRegressionTest thresholds**

Update the regression test's per-category accuracy thresholds to reflect the new baseline. Do not tighten beyond the measured values — leave 10% margin as the test already does.

- [ ] **Step 4: Run regression tests**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all pass including updated DivergenceRegressionTest

- [ ] **Step 5: Run tournament accuracy baseline**

Extend the baseline test to run against all 631 tournament replays (not just 118 oracle). Write results to `docs/benchmarks/tournament-accuracy-baseline.md`.

- [ ] **Step 6: Commit**

```bash
git add docs/benchmarks/
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceRegressionTest.java
git commit -m "docs: update accuracy baseline after Phase 2 calibration

Unit accuracy: X% (was 36.4%)
Upgrade accuracy: Y% (was 0%)
Economy MAPE: Z% (was ~200%)

Refs #384"
```

---

## References

- `specs/issue-384-emulator-reconstitution-pipeline/2026-10-08-emulator-reconstitution-pipeline-design.md` — design spec
- `quarkmind-sc2/.../ReplayCommandExtractor.java:21-60` — current extraction (discards Build/Upgrade/Morph)
- `quarkmind-sc2/.../ReplayCommand.java:6-21` — command types (BuildCommand, UpgradeCommand, MorphCommand already exist)
- `quarkmind-sc2/.../AbilityMapping.java:506-670` — dispatch already produces all command types
- `quarkmind-sc2/.../EmulatedGame.java:263-291` — applyIntent dispatches (already handles BuildIntent, ResearchIntent)
- `quarkmind-sc2/.../EmulatedGame.java:423-445` — handleBuild (fully implemented)
- `quarkmind-sc2/.../EmulatedGame.java:584-615` — handleResearch (fully implemented)
- `quarkmind-sc2/.../Intent.java:3` — sealed interface (needs MorphIntent added)
- `quarkmind-sc2/.../ReplayValidationHarness.java:55-213` — harness architecture
- `docs/benchmarks/emulated-game-accuracy-baseline.md` — current baseline (36.4% units at 5-min)
- PP-20260522-572156 — timing calibration protocol
- PP-20260528-612dee — extractor separation protocol
- GitHub #384 — epic issue
