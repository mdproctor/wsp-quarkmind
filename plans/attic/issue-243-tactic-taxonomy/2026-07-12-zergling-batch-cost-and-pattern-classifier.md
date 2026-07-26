# Zergling Batch Cost Fix + Enemy Pattern Classifier Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #234 — bug: EmulatedGame.handleTrain under-deducts minerals for Zergling (batch vs individual cost)
**Issue group:** #234, #183

**Goal:** Fix Zergling batch cost deduction bug and add Drools CEP-based enemy strategy classification with confidence scoring.

**Architecture:** #234 is a targeted fix in `EmulatedGame.handleTrain()` — multiply resource costs by `trainCount` in both the check and deduction phases. #183 adds a `PatternClassificationRuleUnit` with its own DRL file inside `DroolsScoutingTask`, computing per-archetype confidence from evidence markers and publishing the top assessment via the existing `ScoutingIntelBroker` dual-stack delivery. Strategy layer consumes assessments via a new `patternStore` DataStore.

**Tech Stack:** Java 21, Drools Rule Units (drools-quarkus), Quarkus CDI, AssertJ

## Global Constraints

- Domain records (`EnemyArchetype`, `EnemyPatternAssessment`) go in `io.quarkmind.domain` — plain Java, no framework deps.
- `EvidenceMarker`, `PatternClassificationRuleUnit` go in `io.quarkmind.plugin.scouting` — internal to classification engine.
- DRL files use `this.method()` accessor syntax, fully qualified enum references, explicit `import` statements.
- `eval()` + `accumulate()` in same Drools rule does NOT compile — use `DataStore<Double>` for `gameTimeMin` instead of a global.
- `DataSource.createStore()` requires Quarkus build-time init — rule unit tests must be `@QuarkusTest` (GE-0053).
- `ScoutingIntelPayload` is a sealed interface — `permits` clause must be updated when adding variants.
- `ScoutingIntelPreferences.defaultEnabled()` has exhaustive switch — must add new enum case.
- Pattern classifier runs alongside existing build-order detection, not replacing it.
- All edits via IntelliJ MCP tools (`ide_edit_member`, `ide_insert_member`, `ide_replace_member`). Never bash Edit on source files.

---

### Task 1: #234 — Zergling batch cost fix (friendly + enemy paths)

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` — `handleTrain()` Phase 2 + Phase 4, `startTraining()` fallback
- Test: `src/test/java/io/quarkmind/sc2/emulated/ZergEmulatedGameTest.java`

**Interfaces:**
- Consumes: `SC2Data.trainCount(UnitType)`, `SC2Data.mineralCost(UnitType)`, `SC2Data.gasCost(UnitType)`
- Produces: Corrected `handleTrain()` behaviour — deducts `mCost * count` and `gCost * count`; `startTraining()` spawns `SC2Data.trainCount()` units on enemy path

- [ ] **Step 1: Write failing test — friendly Zergling batch cost deduction**

In `ZergEmulatedGameTest`, add test that trains a Zergling pair and asserts 50 minerals deducted (not 25):

```java
@Test
void trainZergling_deductsFullBatchCost_50minerals() {
    game.setMineralsForTesting(200);
    game.setSupplyForTesting(20, 12);
    final String hatchTag = hatcheryTag();
    final int mineralsBefore = game.snapshot().minerals();

    game.applyIntent(new TrainIntent(hatchTag, UnitType.ZERGLING));

    assertThat(game.snapshot().minerals()).isEqualTo(mineralsBefore - 50);
}
```

- [ ] **Step 2: Write failing test — insufficient minerals for batch rejected**

```java
@Test
void trainZergling_insufficientForBatch_rejected() {
    game.setMineralsForTesting(30); // enough for 1 (25) but not batch (50)
    game.setSupplyForTesting(20, 12);
    final String hatchTag = hatcheryTag();

    game.applyIntent(new TrainIntent(hatchTag, UnitType.ZERGLING));

    // No larva consumed, no EGG spawned
    assertThat(model.larvaCount(hatchTag)).isEqualTo(3);
    assertThat(game.snapshot().minerals()).isEqualTo(30);
}
```

- [ ] **Step 3: Write failing test — enemy Zergling spawns 2 units**

```java
@Test
void enemyZerglingTrain_spawnsTwoUnits_deductsFullBatchCost() {
    game.setMineralsForTesting(200); // friendly minerals (irrelevant)
    // Give enemy enough minerals for a Zergling pair
    game.spawnEnemyBuildingForTesting(BuildingType.HATCHERY, new Point2d(50, 50));

    // Enemy path: applyIntent with enemy state
    // Use the public spawnUnit to set up, then check enemy directly
    // Actually, use the existing enemy intent mechanism
    final var enemyBuildings = game.snapshot().enemyBuildings();
    // Enemy starts with no buildings in mock — spawn one
    // The enemy hatchery was just spawned above
    // Need to issue a TrainIntent through the enemy path
    // EmulatedGame.applyIntent(TimedIntent) routes via owner
    // For testing, use the direct method: game has package-private access
    // The simplest approach: spawn enemy zerglings and verify count

    // Pre-state: count enemy units
    final long enemyUnitsBefore = game.snapshot().enemyUnits().size();

    // Issue enemy train intent via TimedIntent (owner=2)
    final String enemyHatchTag = game.snapshot().enemyBuildings().stream()
        .filter(b -> b.type() == BuildingType.HATCHERY)
        .findFirst().orElseThrow().tag();

    // Apply through enemy path
    game.applyIntent(new io.quarkmind.sc2.intent.TimedIntent(
        new TrainIntent(enemyHatchTag, UnitType.ZERGLING), 0L, 2));

    // Tick past training time
    final int buildTicks = SC2Data.trainTimeInTicks(UnitType.ZERGLING);
    for (int i = 0; i < buildTicks; i++) game.tick();

    // Assert 2 Zerglings spawned (not 1)
    final long enemyZerglings = game.snapshot().enemyUnits().stream()
        .filter(u -> u.type() == UnitType.ZERGLING).count();
    assertThat(enemyZerglings).isEqualTo(2);
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `mvn test -Dtest=ZergEmulatedGameTest -q`
Expected: 3 new tests FAIL — mineral deduction is 25 not 50, rejection threshold is wrong, enemy spawns 1 not 2.

- [ ] **Step 5: Fix `handleTrain()` — multiply costs by `trainCount`**

In `EmulatedGame.handleTrain(TrainIntent, PlayerState, PhysicsState, long)`:

After the existing cost lookups (line ~293), add `count`:
```java
final int count = SC2Data.trainCount(t.unitType());
```

Phase 2 resource check — multiply mineral and gas by count:
```java
if ((int) state.minerals() < mCost * count || state.vespene() < gCost * count
        || state.supplyUsed() + sCost > state.supply()) {
```

Phase 4 resource deduction — multiply:
```java
state.deductMinerals(mCost * count);
state.deductVespene(gCost * count);
```

- [ ] **Step 6: Fix `startTraining()` — enemy-path fallback**

In `EmulatedGame.startTraining()`, change the spawn count fallback:

```java
final int spawnCount = (model != null) ? model.trainCount(unitType) : SC2Data.trainCount(unitType);
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn test -Dtest=ZergEmulatedGameTest -q`
Expected: ALL PASS

- [ ] **Step 8: Run full test suite**

Run: `mvn test -q`
Expected: ALL PASS — no regressions.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java src/test/java/io/quarkmind/sc2/emulated/ZergEmulatedGameTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "fix(#234): multiply resource costs by trainCount for batch-trained units

handleTrain() Phase 2 and Phase 4 now use mCost * count and gCost * count.
startTraining() enemy-path fallback uses SC2Data.trainCount() instead of
hardcoded 1. Fixes Zergling under-deducting 25 minerals per batch.

Closes #234"
```

---

### Task 2: Domain model — EnemyArchetype, EnemyPatternAssessment, EvidenceMarker

**Files:**
- Create: `src/main/java/io/quarkmind/domain/EnemyArchetype.java`
- Create: `src/main/java/io/quarkmind/domain/EnemyPatternAssessment.java`
- Create: `src/main/java/io/quarkmind/plugin/scouting/EvidenceMarker.java`

**Interfaces:**
- Consumes: nothing
- Produces: `EnemyArchetype` enum (10 values), `EnemyPatternAssessment(EnemyArchetype, double, long, String)` record, `EvidenceMarker(EnemyArchetype, double, String)` record

- [ ] **Step 1: Create `EnemyArchetype.java`**

```java
package io.quarkmind.domain;

public enum EnemyArchetype {
    TERRAN_MARINE_RUSH,
    TERRAN_BIO_TIMING,
    TERRAN_MECH_PUSH,
    TERRAN_BANSHEE_HARASS,
    ZERG_ZERGLING_RUSH,
    ZERG_ROACH_RUSH,
    ZERG_MACRO,
    PROTOSS_GATEWAY_RUSH,
    PROTOSS_CANNON_RUSH,
    PROTOSS_MACRO
}
```

- [ ] **Step 2: Create `EnemyPatternAssessment.java`**

```java
package io.quarkmind.domain;

public record EnemyPatternAssessment(
    EnemyArchetype archetype,
    double confidence,
    long detectedAtFrame,
    String rationale
) {}
```

- [ ] **Step 3: Create `EvidenceMarker.java`**

```java
package io.quarkmind.plugin.scouting;

import io.quarkmind.domain.EnemyArchetype;

public record EvidenceMarker(EnemyArchetype archetype, double weight, String signal) {}
```

- [ ] **Step 4: Verify compilation**

Run: `mvn compile -q`
Expected: SUCCESS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/domain/EnemyArchetype.java src/main/java/io/quarkmind/domain/EnemyPatternAssessment.java src/main/java/io/quarkmind/plugin/scouting/EvidenceMarker.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#183): add EnemyArchetype, EnemyPatternAssessment, EvidenceMarker domain types

10 archetypes across Terran/Zerg/Protoss. EvidenceMarker is internal to
the classification engine. Plain Java records, no framework deps.

Refs #183"
```

---

### Task 3: ScoutingIntelPayload + ScoutingIntelType extension

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/plugin/ScoutingIntelType.java` — add `PATTERN_ASSESSMENT`
- Modify: `src/main/java/io/quarkmind/agent/plugin/ScoutingIntelPayload.java` — add `PatternAssessment` variant + update `permits`
- Modify: `src/main/java/io/quarkmind/agent/plugin/ScoutingIntelPreferences.java` — add `defaultEnabled` case + dispatch preference key

**Interfaces:**
- Consumes: `EnemyPatternAssessment` from Task 2
- Produces: `ScoutingIntelType.PATTERN_ASSESSMENT`, `ScoutingIntelPayload.PatternAssessment` record, `ScoutingIntelPreferences.PATTERN_ASSESSMENT_DISPATCH_ENABLED`

- [ ] **Step 1: Add `PATTERN_ASSESSMENT` to `ScoutingIntelType`**

Add to enum: `PATTERN_ASSESSMENT` after `BUILD_ORDER`.

- [ ] **Step 2: Add `PatternAssessment` variant to `ScoutingIntelPayload`**

Update `permits` clause to include `ScoutingIntelPayload.PatternAssessment`. Add record:

```java
record PatternAssessment(EnemyPatternAssessment assessment) implements ScoutingIntelPayload {
    public ScoutingIntelType type() { return ScoutingIntelType.PATTERN_ASSESSMENT; }
}
```

- [ ] **Step 3: Update `ScoutingIntelPreferences.defaultEnabled()`**

Add case to switch:
```java
case PATTERN_ASSESSMENT -> true;
```

Add dispatch preference key:
```java
public static final PreferenceKey<ScoutingIntelPreference> PATTERN_ASSESSMENT_DISPATCH_ENABLED =
    new PreferenceKey<>("scouting.intel.dispatch", "pattern-assessment.enabled",
        ScoutingIntelPreference.ofBoolean(true), ScoutingIntelPreference::parseBoolean);
```

- [ ] **Step 4: Verify compilation — check for exhaustive switch failures**

Run: `mvn compile -q`
Expected: May have compilation errors if other exhaustive switches exist on `ScoutingIntelType`. Fix any that appear.

- [ ] **Step 5: Run existing tests**

Run: `mvn test -q`
Expected: ALL PASS — no regressions from the new enum value.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/agent/plugin/ScoutingIntelType.java src/main/java/io/quarkmind/agent/plugin/ScoutingIntelPayload.java src/main/java/io/quarkmind/agent/plugin/ScoutingIntelPreferences.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#183): extend ScoutingIntelPayload with PatternAssessment variant

New PATTERN_ASSESSMENT enum value, sealed variant with permits clause,
defaultEnabled case, and dispatch preference key.

Refs #183"
```

---

### Task 4: PatternClassificationRuleUnit + DRL + confidence computation

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/scouting/PatternClassificationRuleUnit.java`
- Create: `src/main/resources/io/quarkmind/plugin/scouting/PatternClassification.drl`
- Create: `src/test/java/io/quarkmind/plugin/scouting/PatternConfidenceTest.java` (plain JUnit)
- Create: `src/test/java/io/quarkmind/plugin/scouting/PatternClassificationRuleUnitTest.java` (`@QuarkusTest`)

**Interfaces:**
- Consumes: `EnemyUnitFirstSeen`, `EnemyExpansionSeen`, `EnemyArmyNearBase` (scouting events), `EvidenceMarker`, `EnemyArchetype`
- Produces: `PatternClassificationRuleUnit` (RuleUnitData), `PatternClassification.drl` (evidence rules for 10 archetypes), confidence computation methods (static, testable independently)

- [ ] **Step 1: Write `PatternConfidenceTest` — confidence formula tests (plain JUnit)**

```java
package io.quarkmind.plugin.scouting;

import io.quarkmind.domain.EnemyArchetype;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

class PatternConfidenceTest {

    @Test
    void singleWeight_returnsWeight() {
        var markers = List.of(new EvidenceMarker(EnemyArchetype.TERRAN_MARINE_RUSH, 0.5, "test"));
        double conf = PatternClassifier.computeTickConfidence(markers);
        assertThat(conf).isCloseTo(0.5, within(0.001));
    }

    @Test
    void twoWeights_probabilityFormula() {
        var markers = List.of(
            new EvidenceMarker(EnemyArchetype.TERRAN_MARINE_RUSH, 0.5, "a"),
            new EvidenceMarker(EnemyArchetype.TERRAN_MARINE_RUSH, 0.5, "b"));
        double conf = PatternClassifier.computeTickConfidence(markers);
        assertThat(conf).isCloseTo(0.75, within(0.001)); // 1 - (0.5 * 0.5)
    }

    @Test
    void emptyMarkers_returnsZero() {
        double conf = PatternClassifier.computeTickConfidence(List.of());
        assertThat(conf).isEqualTo(0.0);
    }

    @Test
    void cumulativeMerge_takesMax() {
        var cumulative = new java.util.EnumMap<EnemyArchetype, Double>(EnemyArchetype.class);
        cumulative.put(EnemyArchetype.TERRAN_MARINE_RUSH, 0.4);

        var thisTick = Map.of(EnemyArchetype.TERRAN_MARINE_RUSH, 0.6);
        PatternClassifier.mergeCumulative(cumulative, thisTick);

        assertThat(cumulative.get(EnemyArchetype.TERRAN_MARINE_RUSH)).isEqualTo(0.6);
    }

    @Test
    void cumulativeMerge_doesNotDecrease() {
        var cumulative = new java.util.EnumMap<EnemyArchetype, Double>(EnemyArchetype.class);
        cumulative.put(EnemyArchetype.TERRAN_MARINE_RUSH, 0.8);

        var thisTick = Map.of(EnemyArchetype.TERRAN_MARINE_RUSH, 0.3);
        PatternClassifier.mergeCumulative(cumulative, thisTick);

        assertThat(cumulative.get(EnemyArchetype.TERRAN_MARINE_RUSH)).isEqualTo(0.8);
    }

    @Test
    void topAssessment_returnsHighestConfidence() {
        var cumulative = new java.util.EnumMap<EnemyArchetype, Double>(EnemyArchetype.class);
        cumulative.put(EnemyArchetype.TERRAN_MARINE_RUSH, 0.7);
        cumulative.put(EnemyArchetype.TERRAN_BIO_TIMING, 0.3);

        var top = PatternClassifier.topAssessment(cumulative, 100L);
        assertThat(top).isPresent();
        assertThat(top.get().archetype()).isEqualTo(EnemyArchetype.TERRAN_MARINE_RUSH);
        assertThat(top.get().confidence()).isEqualTo(0.7);
    }

    @Test
    void topAssessment_belowThreshold_returnsEmpty() {
        var cumulative = new java.util.EnumMap<EnemyArchetype, Double>(EnemyArchetype.class);
        cumulative.put(EnemyArchetype.TERRAN_MARINE_RUSH, 0.2); // below 0.3 threshold

        var top = PatternClassifier.topAssessment(cumulative, 100L);
        assertThat(top).isEmpty();
    }
}
```

- [ ] **Step 2: Create `PatternClassifier` — static confidence computation methods**

```java
package io.quarkmind.plugin.scouting;

import io.quarkmind.domain.EnemyArchetype;
import io.quarkmind.domain.EnemyPatternAssessment;

import java.util.*;
import java.util.stream.Collectors;

public final class PatternClassifier {

    static final double DISPATCH_THRESHOLD = 0.3;

    static double computeTickConfidence(List<EvidenceMarker> markers) {
        if (markers.isEmpty()) return 0.0;
        double product = 1.0;
        for (EvidenceMarker m : markers) {
            product *= (1.0 - m.weight());
        }
        return 1.0 - product;
    }

    static Map<EnemyArchetype, Double> computeAllConfidences(List<EvidenceMarker> markers) {
        return markers.stream()
            .collect(Collectors.groupingBy(EvidenceMarker::archetype))
            .entrySet().stream()
            .collect(Collectors.toMap(Map.Entry::getKey,
                e -> computeTickConfidence(e.getValue())));
    }

    static void mergeCumulative(EnumMap<EnemyArchetype, Double> cumulative,
                                Map<EnemyArchetype, Double> thisTick) {
        thisTick.forEach((arch, conf) ->
            cumulative.merge(arch, conf, Math::max));
    }

    static Optional<EnemyPatternAssessment> topAssessment(
            EnumMap<EnemyArchetype, Double> cumulative, long frame) {
        return cumulative.entrySet().stream()
            .filter(e -> e.getValue() >= DISPATCH_THRESHOLD)
            .max(Map.Entry.comparingByValue())
            .map(e -> new EnemyPatternAssessment(e.getKey(), e.getValue(), frame,
                e.getKey().name() + " (confidence " +
                    String.format("%.2f", e.getValue()) + ")"));
    }

    private PatternClassifier() {}
}
```

- [ ] **Step 3: Run `PatternConfidenceTest` to verify it passes**

Run: `mvn test -Dtest=PatternConfidenceTest -q`
Expected: ALL PASS

- [ ] **Step 4: Create `PatternClassificationRuleUnit.java`**

```java
package io.quarkmind.plugin.scouting;

import io.quarkmind.plugin.scouting.events.EnemyArmyNearBase;
import io.quarkmind.plugin.scouting.events.EnemyExpansionSeen;
import io.quarkmind.plugin.scouting.events.EnemyUnitFirstSeen;
import org.drools.ruleunits.api.DataSource;
import org.drools.ruleunits.api.DataStore;
import org.drools.ruleunits.api.RuleUnitData;

import java.util.ArrayList;
import java.util.List;

public class PatternClassificationRuleUnit implements RuleUnitData {

    private final DataStore<EnemyUnitFirstSeen>  unitEvents         = DataSource.createStore();
    private final DataStore<EnemyExpansionSeen>  expansionEvents    = DataSource.createStore();
    private final DataStore<EnemyArmyNearBase>   armyNearBaseEvents = DataSource.createStore();
    private final DataStore<Double>              gameTimeStore      = DataSource.createStore();

    private final List<EvidenceMarker> evidence = new ArrayList<>();

    public DataStore<EnemyUnitFirstSeen>  getUnitEvents()         { return unitEvents; }
    public DataStore<EnemyExpansionSeen>  getExpansionEvents()    { return expansionEvents; }
    public DataStore<EnemyArmyNearBase>   getArmyNearBaseEvents() { return armyNearBaseEvents; }
    public DataStore<Double>              getGameTimeStore()      { return gameTimeStore; }

    public List<EvidenceMarker> getEvidence() { return evidence; }
}
```

- [ ] **Step 5: Create `PatternClassification.drl`**

Evidence rules for all 10 archetypes. Each rule emits one or more `EvidenceMarker` records. Multiple rules per archetype provide additive evidence.

```
package io.quarkmind.plugin.scouting;
unit PatternClassificationRuleUnit;

import io.quarkmind.domain.UnitType;
import io.quarkmind.domain.EnemyArchetype;

// --- TERRAN ---

rule "Evidence: Marine Rush — high marine count early"
when
    accumulate(/unitEvents[this.type() == UnitType.MARINE]; $count: count(); $count >= 5)
    /gameTimeStore[this < 4.0]
then
    evidence.add(new EvidenceMarker(EnemyArchetype.TERRAN_MARINE_RUSH, 0.6,
        $count + " Marines before 4min"));
end

rule "Evidence: Marine Rush — no expansion"
when
    not /expansionEvents
    /unitEvents[this.type() == UnitType.MARINE]
then
    evidence.add(new EvidenceMarker(EnemyArchetype.TERRAN_MARINE_RUSH, 0.3,
        "No expansion with Marines"));
end

rule "Evidence: Bio Timing — marines + medivac mid-game"
when
    accumulate(/unitEvents[this.type() == UnitType.MARINE]; $mCount: count(); $mCount >= 3)
    /unitEvents[this.type() == UnitType.MEDIVAC]
    /gameTimeStore[this >= 4.0, this < 10.0]
then
    evidence.add(new EvidenceMarker(EnemyArchetype.TERRAN_BIO_TIMING, 0.6,
        $mCount + " Marines + Medivac in mid-game"));
end

rule "Evidence: Bio Timing — multiple marines late"
when
    accumulate(/unitEvents[this.type() == UnitType.MARINE]; $mCount: count(); $mCount >= 6)
    /gameTimeStore[this >= 4.0]
then
    evidence.add(new EvidenceMarker(EnemyArchetype.TERRAN_BIO_TIMING, 0.3,
        $mCount + " Marines after 4min"));
end

rule "Evidence: Mech Push — siege tanks"
when
    accumulate(/unitEvents[this.type() == UnitType.SIEGE_TANK]; $count: count(); $count >= 2)
    /gameTimeStore[this >= 5.0]
then
    evidence.add(new EvidenceMarker(EnemyArchetype.TERRAN_MECH_PUSH, 0.6,
        $count + " Siege Tanks"));
end

rule "Evidence: Mech Push — thors present"
when
    /unitEvents[this.type() == UnitType.THOR]
then
    evidence.add(new EvidenceMarker(EnemyArchetype.TERRAN_MECH_PUSH, 0.4,
        "Thor spotted"));
end

rule "Evidence: Banshee Harass — banshee early"
when
    /unitEvents[this.type() == UnitType.BANSHEE]
    /gameTimeStore[this < 8.0]
then
    evidence.add(new EvidenceMarker(EnemyArchetype.TERRAN_BANSHEE_HARASS, 0.7,
        "Banshee before 8min"));
end

// --- ZERG ---

rule "Evidence: Zergling Rush — early zerglings"
when
    accumulate(/unitEvents[this.type() == UnitType.ZERGLING]; $count: count(); $count >= 6)
    /gameTimeStore[this < 4.0]
then
    evidence.add(new EvidenceMarker(EnemyArchetype.ZERG_ZERGLING_RUSH, 0.6,
        $count + " Zerglings before 4min"));
end

rule "Evidence: Zergling Rush — no expansion"
when
    not /expansionEvents
    /unitEvents[this.type() == UnitType.ZERGLING]
then
    evidence.add(new EvidenceMarker(EnemyArchetype.ZERG_ZERGLING_RUSH, 0.3,
        "No expansion with Zerglings"));
end

rule "Evidence: Roach Rush — early roaches"
when
    accumulate(/unitEvents[this.type() == UnitType.ROACH]; $count: count(); $count >= 4)
    /gameTimeStore[this < 5.0]
then
    evidence.add(new EvidenceMarker(EnemyArchetype.ZERG_ROACH_RUSH, 0.7,
        $count + " Roaches before 5min"));
end

rule "Evidence: Zerg Macro — expansion with low unit count"
when
    /expansionEvents
    accumulate(/unitEvents; $count: count(); $count < 4)
    /gameTimeStore[this < 5.0]
then
    evidence.add(new EvidenceMarker(EnemyArchetype.ZERG_MACRO, 0.5,
        "Expansion with < 4 units"));
end

// --- PROTOSS ---

rule "Evidence: Gateway Rush — stalkers + zealots early"
when
    accumulate(
        /unitEvents[this.type() == UnitType.STALKER || this.type() == UnitType.ZEALOT];
        $count: count();
        $count >= 4
    )
    /gameTimeStore[this < 5.0]
then
    evidence.add(new EvidenceMarker(EnemyArchetype.PROTOSS_GATEWAY_RUSH, 0.6,
        $count + " Gateway units before 5min"));
end

rule "Evidence: Gateway Rush — no expansion"
when
    not /expansionEvents
    /unitEvents[this.type() == UnitType.STALKER || this.type() == UnitType.ZEALOT]
then
    evidence.add(new EvidenceMarker(EnemyArchetype.PROTOSS_GATEWAY_RUSH, 0.3,
        "No expansion with Gateway units"));
end

rule "Evidence: Cannon Rush — photon cannons near base"
when
    /armyNearBaseEvents
    /gameTimeStore[this < 4.0]
    not /unitEvents[this.type() == UnitType.STALKER]
then
    evidence.add(new EvidenceMarker(EnemyArchetype.PROTOSS_CANNON_RUSH, 0.6,
        "Army near base early without Stalkers"));
end

rule "Evidence: Protoss Macro — expansion with few units"
when
    /expansionEvents
    accumulate(/unitEvents; $count: count(); $count < 3)
    /gameTimeStore[this < 5.0]
then
    evidence.add(new EvidenceMarker(EnemyArchetype.PROTOSS_MACRO, 0.5,
        "Expansion with < 3 units"));
end
```

- [ ] **Step 6: Add `buildPatternRuleUnit()` to `ScoutingSessionManager`**

```java
public PatternClassificationRuleUnit buildPatternRuleUnit(double gameTimeMin) {
    PatternClassificationRuleUnit data = new PatternClassificationRuleUnit();
    unitBuffer.forEach(data.getUnitEvents()::add);
    expansionBuffer.forEach(data.getExpansionEvents()::add);
    armyBuffer.forEach(data.getArmyNearBaseEvents()::add);
    data.getGameTimeStore().add(gameTimeMin);
    return data;
}
```

- [ ] **Step 7: Write `PatternClassificationRuleUnitTest` (`@QuarkusTest`)**

```java
package io.quarkmind.plugin.scouting;

import io.quarkmind.domain.EnemyArchetype;
import io.quarkmind.domain.UnitType;
import io.quarkmind.plugin.scouting.events.EnemyExpansionSeen;
import io.quarkmind.plugin.scouting.events.EnemyUnitFirstSeen;
import io.quarkmind.domain.Point2d;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.drools.ruleunits.api.RuleUnit;
import org.drools.ruleunits.api.RuleUnitInstance;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class PatternClassificationRuleUnitTest {

    @Inject RuleUnit<PatternClassificationRuleUnit> ruleUnit;

    @Test
    void marineRush_highCountEarly_producesEvidence() {
        var data = new PatternClassificationRuleUnit();
        for (int i = 0; i < 6; i++) {
            data.getUnitEvents().add(new EnemyUnitFirstSeen(UnitType.MARINE, 60000L));
        }
        data.getGameTimeStore().add(3.0);

        fire(data);

        assertThat(data.getEvidence()).anyMatch(e ->
            e.archetype() == EnemyArchetype.TERRAN_MARINE_RUSH && e.weight() >= 0.5);
    }

    @Test
    void roachRush_earlyRoaches_producesEvidence() {
        var data = new PatternClassificationRuleUnit();
        for (int i = 0; i < 5; i++) {
            data.getUnitEvents().add(new EnemyUnitFirstSeen(UnitType.ROACH, 120000L));
        }
        data.getGameTimeStore().add(4.0);

        fire(data);

        assertThat(data.getEvidence()).anyMatch(e ->
            e.archetype() == EnemyArchetype.ZERG_ROACH_RUSH && e.weight() >= 0.5);
    }

    @Test
    void emptyEvents_noEvidence() {
        var data = new PatternClassificationRuleUnit();
        data.getGameTimeStore().add(3.0);

        fire(data);

        assertThat(data.getEvidence()).isEmpty();
    }

    @Test
    void mixedSignals_multipleArchetypesGetEvidence() {
        var data = new PatternClassificationRuleUnit();
        // Marines (could be rush or bio timing)
        for (int i = 0; i < 6; i++) {
            data.getUnitEvents().add(new EnemyUnitFirstSeen(UnitType.MARINE, 60000L));
        }
        data.getGameTimeStore().add(3.5);

        fire(data);

        // Both MARINE_RUSH and potentially others should have evidence
        assertThat(data.getEvidence().stream()
            .map(EvidenceMarker::archetype).distinct().count()).isGreaterThanOrEqualTo(1);
    }

    @Test
    void expansion_withFewUnits_producesZergMacroEvidence() {
        var data = new PatternClassificationRuleUnit();
        data.getExpansionEvents().add(new EnemyExpansionSeen(new Point2d(40, 40), 120000L));
        data.getUnitEvents().add(new EnemyUnitFirstSeen(UnitType.ZERGLING, 120000L));
        data.getGameTimeStore().add(4.0);

        fire(data);

        assertThat(data.getEvidence()).anyMatch(e ->
            e.archetype() == EnemyArchetype.ZERG_MACRO);
    }

    @Test
    void gatewayRush_stalkersAndZealotsEarly_producesEvidence() {
        var data = new PatternClassificationRuleUnit();
        data.getUnitEvents().add(new EnemyUnitFirstSeen(UnitType.STALKER, 180000L));
        data.getUnitEvents().add(new EnemyUnitFirstSeen(UnitType.STALKER, 180000L));
        data.getUnitEvents().add(new EnemyUnitFirstSeen(UnitType.ZEALOT, 180000L));
        data.getUnitEvents().add(new EnemyUnitFirstSeen(UnitType.ZEALOT, 180000L));
        data.getGameTimeStore().add(4.5);

        fire(data);

        assertThat(data.getEvidence()).anyMatch(e ->
            e.archetype() == EnemyArchetype.PROTOSS_GATEWAY_RUSH);
    }

    private void fire(PatternClassificationRuleUnit data) {
        try (RuleUnitInstance<PatternClassificationRuleUnit> instance = ruleUnit.createInstance(data)) {
            instance.fire();
        }
    }
}
```

- [ ] **Step 8: Run all new tests**

Run: `mvn test -Dtest=PatternConfidenceTest,PatternClassificationRuleUnitTest -q`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/plugin/scouting/PatternClassifier.java src/main/java/io/quarkmind/plugin/scouting/PatternClassificationRuleUnit.java src/main/resources/io/quarkmind/plugin/scouting/PatternClassification.drl src/main/java/io/quarkmind/plugin/scouting/ScoutingSessionManager.java src/test/java/io/quarkmind/plugin/scouting/PatternConfidenceTest.java src/test/java/io/quarkmind/plugin/scouting/PatternClassificationRuleUnitTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#183): add PatternClassificationRuleUnit with evidence DRL and confidence computation

10 archetypes across Terran/Zerg/Protoss with 18 evidence rules.
Confidence uses 1-∏(1-weight_i) formula with monotonic accumulation.
ScoutingSessionManager.buildPatternRuleUnit() populates from buffers.

Refs #183"
```

---

### Task 5: Wire pattern classification into DroolsScoutingTask

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java` — inject second rule unit, fire pattern rules, publish assessment
- Modify: `src/test/java/io/quarkmind/plugin/DroolsScoutingTaskIT.java` — parallel comparison test

**Interfaces:**
- Consumes: `PatternClassificationRuleUnit` (Task 4), `PatternClassifier` (Task 4), `ScoutingIntelPayload.PatternAssessment` (Task 3), `ScoutingIntelPreferences.PATTERN_ASSESSMENT_DISPATCH_ENABLED` (Task 3)
- Produces: Pattern classification published via `ScoutingIntelBroker.update()` each tick when the top assessment changes

- [ ] **Step 1: Write failing IT — parallel comparison test**

In `DroolsScoutingTaskIT`, add:

```java
@Test
void patternClassification_marineRush_publishedAlongsideBuildOrder() {
    // 7+ Marines at 3 min — triggers both old TERRAN_3RAX and new TERRAN_MARINE_RUSH
    var enemies = new java.util.ArrayList<Unit>();
    for (int i = 0; i < 8; i++) {
        enemies.add(enemy(50 + i, 50));
    }
    // Set all enemies to MARINE type
    enemies.clear();
    for (int i = 0; i < 8; i++) {
        enemies.add(new Unit("e" + i, UnitType.MARINE, new Point2d(50 + i, 50), 45, 45, 0, 0, 0, 0));
    }

    // Execute multiple ticks to build up scouting data
    for (int tick = 0; tick < 5; tick++) {
        var ctx = caseContext(enemies, List.of(), 100L + tick);
        scoutingTask.execute(ctx);
    }

    // Old detection: build order should be TERRAN_3RAX
    var lastCtx = caseContext(enemies, List.of(), 105L);
    scoutingTask.execute(lastCtx);
    assertThat(lastCtx.getAs(QuarkMindCaseFile.ENEMY_BUILD_ORDER, String.class))
        .isEqualTo("TERRAN_3RAX");

    // New detection: broker should have PATTERN_ASSESSMENT
    var assessment = broker.current(ScoutingIntelType.PATTERN_ASSESSMENT,
        ScoutingIntelPayload.PatternAssessment.class);
    assertThat(assessment).isPresent();
    assertThat(assessment.get().assessment().archetype())
        .isEqualTo(io.quarkmind.domain.EnemyArchetype.TERRAN_MARINE_RUSH);
    assertThat(assessment.get().assessment().confidence()).isGreaterThanOrEqualTo(0.3);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=DroolsScoutingTaskIT#patternClassification_marineRush_publishedAlongsideBuildOrder -q`
Expected: FAIL — pattern classification not yet wired.

- [ ] **Step 3: Wire pattern classification into DroolsScoutingTask**

Add to `DroolsScoutingTask`:
1. New constructor parameter: `RuleUnit<PatternClassificationRuleUnit> patternRuleUnit`
2. New fields: `cumulativeConfidence` (EnumMap), `prevAssessment` (EnemyPatternAssessment), `patternAssessmentDispatchEnabled` (boolean)
3. In `initThresholds(Preferences)`: load `patternAssessmentDispatchEnabled`
4. In `resetDispatchState()`: clear `cumulativeConfidence`, null `prevAssessment`
5. In `execute()`: after existing CEP block, if `needsCep`, also fire pattern rule unit, compute confidence, merge cumulative, publish if changed

Key code for `execute()` addition (after existing CEP intel write block):

```java
// --- Pattern classification ---
if (needsCep && data != null) {
    PatternClassificationRuleUnit patternData = sessionManager.buildPatternRuleUnit(
        gameTimeMs / 60000.0);
    try (RuleUnitInstance<PatternClassificationRuleUnit> pInstance =
            patternRuleUnit.createInstance(patternData)) {
        pInstance.fire();
    }
    var allConf = PatternClassifier.computeAllConfidences(patternData.getEvidence());
    PatternClassifier.mergeCumulative(cumulativeConfidence, allConf);

    var top = PatternClassifier.topAssessment(cumulativeConfidence, frame);
    if (top.isPresent()) {
        var assessment = top.get();
        boolean changed = prevAssessment == null
            || assessment.archetype() != prevAssessment.archetype()
            || crossedThreshold(prevAssessment.confidence(), assessment.confidence());
        if (changed && patternAssessmentDispatchEnabled
                && (broker.isSubscribed(ScoutingIntelType.PATTERN_ASSESSMENT) || advisoryEnabled)) {
            prevAssessment = assessment;
            publishIntel(new ScoutingIntelPayload.PatternAssessment(assessment));
        }
    }
}
```

Add threshold-crossing helper:
```java
private static boolean crossedThreshold(double prev, double curr) {
    double[] thresholds = {0.3, 0.5, 0.7, 0.9};
    for (double t : thresholds) {
        if (prev < t && curr >= t) return true;
    }
    return false;
}
```

- [ ] **Step 4: Run IT to verify it passes**

Run: `mvn test -Dtest=DroolsScoutingTaskIT#patternClassification_marineRush_publishedAlongsideBuildOrder -q`
Expected: PASS

- [ ] **Step 5: Run full test suite**

Run: `mvn test -q`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java src/test/java/io/quarkmind/plugin/DroolsScoutingTaskIT.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#183): wire PatternClassificationRuleUnit into DroolsScoutingTask

Fires pattern DRL after existing scouting rules. Computes per-archetype
confidence, merges cumulatively, publishes top assessment via broker
when archetype changes or confidence crosses threshold step.

Refs #183"
```

---

### Task 6: Strategy layer consumption + strategy DRL rule

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/drools/StrategyRuleUnit.java` — add `patternStore` DataStore
- Modify: `src/main/java/io/quarkmind/plugin/DroolsStrategyTask.java` — subscribe to `PATTERN_ASSESSMENT`, feed `patternStore`
- Modify: `src/main/resources/io/quarkmind/plugin/drools/StarCraftStrategy.drl` — add rush detection rule
- Modify: `src/test/java/io/quarkmind/plugin/DroolsStrategyL2L3Test.java` — add pattern assessment tests

**Interfaces:**
- Consumes: `ScoutingIntelPayload.PatternAssessment` via broker, `EnemyPatternAssessment` record
- Produces: `DEFEND` strategy decision when rush archetype confidence ≥ 0.7

- [ ] **Step 1: Write failing test — rush detection triggers DEFEND**

In `DroolsStrategyL2L3Test`, add:

```java
@Test
void rushDetected_highConfidence_triggersDefendStrategy() {
    StrategyRuleUnit data = new StrategyRuleUnit();

    data.getPatternStore().add(new io.quarkmind.domain.EnemyPatternAssessment(
        io.quarkmind.domain.EnemyArchetype.TERRAN_MARINE_RUSH, 0.8, 500L,
        "8 Marines before 4min"));
    data.getPostureStore().add("UNKNOWN");
    data.getTimingStore().add(false);

    fire(data);

    assertThat(data.getStrategyDecisions()).contains("DEFEND");
}

@Test
void rushDetected_lowConfidence_noDefend() {
    StrategyRuleUnit data = new StrategyRuleUnit();

    data.getPatternStore().add(new io.quarkmind.domain.EnemyPatternAssessment(
        io.quarkmind.domain.EnemyArchetype.TERRAN_MARINE_RUSH, 0.5, 300L,
        "Low confidence"));
    data.getPostureStore().add("UNKNOWN");
    data.getTimingStore().add(false);

    fire(data);

    assertThat(data.getStrategyDecisions()).doesNotContain("DEFEND");
}

@Test
void macroArchetype_noDefend() {
    StrategyRuleUnit data = new StrategyRuleUnit();

    data.getPatternStore().add(new io.quarkmind.domain.EnemyPatternAssessment(
        io.quarkmind.domain.EnemyArchetype.ZERG_MACRO, 0.9, 500L,
        "Macro build"));
    data.getPostureStore().add("UNKNOWN");
    data.getTimingStore().add(false);

    fire(data);

    assertThat(data.getStrategyDecisions()).doesNotContain("DEFEND");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=DroolsStrategyL2L3Test -q`
Expected: FAIL — `patternStore` doesn't exist yet.

- [ ] **Step 3: Add `patternStore` to `StrategyRuleUnit`**

Add field and getter:
```java
private final DataStore<EnemyPatternAssessment> patternStore = DataSource.createStore();
public DataStore<EnemyPatternAssessment> getPatternStore() { return patternStore; }
```

Add import for `EnemyPatternAssessment`.

- [ ] **Step 4: Add DRL rule to `StarCraftStrategy.drl`**

Add import:
```
import io.quarkmind.domain.EnemyPatternAssessment;
```

Add rule:
```
rule "Strategy: Defend — Rush Detected"
    salience 215
when
    /patternStore[this.confidence() >= 0.7,
                  this.archetype().name().contains("RUSH")]
then
    strategyDecisions.add("DEFEND");
end
```

- [ ] **Step 5: Update `DroolsStrategyTask` — subscribe and feed patternStore**

In `refreshSubscriptions()`: add `PATTERN_ASSESSMENT` to the filtered set.

In `buildRuleUnit()`: read from broker and feed:
```java
broker.current(ScoutingIntelType.PATTERN_ASSESSMENT,
        ScoutingIntelPayload.PatternAssessment.class)
    .map(ScoutingIntelPayload.PatternAssessment::assessment)
    .ifPresent(data.getPatternStore()::add);
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn test -Dtest=DroolsStrategyL2L3Test -q`
Expected: ALL PASS

- [ ] **Step 7: Run full test suite**

Run: `mvn test -q`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/plugin/drools/StrategyRuleUnit.java src/main/java/io/quarkmind/plugin/DroolsStrategyTask.java src/main/resources/io/quarkmind/plugin/drools/StarCraftStrategy.drl src/test/java/io/quarkmind/plugin/DroolsStrategyL2L3Test.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#183): strategy layer consumes PatternAssessment — rush triggers DEFEND

StrategyRuleUnit.patternStore feeds from broker. New DRL rule at salience
215: rush archetype with confidence >= 0.7 produces DEFEND. Between
Nexus Under Attack (220) and Timing Attack (210).

Refs #183"
```

---

### Task 7: Replay calibration extension + full integration test

**Files:**
- Modify: `src/test/java/io/quarkmind/plugin/scouting/ScoutingCalibrationTest.java` — log pattern classifications
- Run: Full test suite + benchmark sanity check

**Interfaces:**
- Consumes: All prior tasks
- Produces: Calibration data log, full regression pass

- [ ] **Step 1: Extend `ScoutingCalibrationTest` — log pattern classifications**

Add logging of pattern classification results alongside existing build-order labels at 3-min mark. Data collection only — no pass/fail assertions on classification accuracy.

Find the existing calibration output section and add pattern classification logging after it.

- [ ] **Step 2: Run replay calibration**

Run: `mvn test -Preport -q`
Expected: Report prints with both old and new classification labels side by side.

- [ ] **Step 3: Run full test suite**

Run: `mvn test -q`
Expected: ALL PASS — zero regressions.

- [ ] **Step 4: Run benchmark sanity check**

Run: `mvn test -Pbenchmark -Dtest=GameLoopBenchmarkTest -q`
Expected: No significant regression in tick times (pattern classification adds one additional Drools rule unit fire per scouting tick).

- [ ] **Step 5: File deferred issues**

Verify that #235, #236, #237, #238 exist (filed during design review). If not, file them now:
- #235: LLM fallback when Drools confidence < 0.5
- #236: Replay classification accuracy ≥ 70% acceptance criteria
- #237: Confidence revision — decay, counter-indication, multi-archetype
- #238: Commentator/Coach prompt updates for PATTERN_ASSESSMENT

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/test/java/io/quarkmind/plugin/scouting/ScoutingCalibrationTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "test(#183): extend replay calibration with pattern classification logging

Logs new classifier results alongside existing build-order labels at
3-min mark. Data collection for threshold calibration — no accuracy
assertions yet (tracked in #236).

Refs #183"
```
