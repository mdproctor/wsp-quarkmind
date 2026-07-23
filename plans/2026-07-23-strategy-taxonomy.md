# Strategy Taxonomy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #243 — feat: expand tactic detection taxonomy — full game-phase strategy catalogue with counters and transitions
**Issue group:** #243

**Goal:** Build a hybrid strategy taxonomy infrastructure — structured YAML for archetype signatures/counters, hand-authored Drools for complex CEP — that scales from 10 early-game archetypes to 50-90 across all game phases, with coaching pipeline integration.

**Architecture:** Rename `EnemyArchetype` → `StrategyArchetype` (player-agnostic), add `GamePhase`/`ArchetypeCategory` enums with metadata on each archetype. Rename existing `summarisation.GamePhase` → `TacticalPosture` to resolve collision. YAML taxonomy loaded at startup provides signatures for data-driven Drools detection rules and counter data for coaching prompt enrichment. `PhaseResolver` interface with time-based default.

**Tech Stack:** Java 21, Quarkus, Drools Rule Units, SnakeYAML, LangChain4j (coaching), JUnit 5 + AssertJ

## Global Constraints

- Pre-release platform: breaking changes cost nothing
- `domain/` package: plain Java only — no CDI, no Quarkus imports
- `.drl` files require manual update — IDE refactor doesn't cover Drools rule text
- All CaseFile key constants in `QuarkMindCaseFile` — never raw strings
- IntelliJ MCP mandatory for all .java edits — use `ide_refactor_rename`, `ide_edit_member`, `ide_insert_member`, `ide_replace_member`
- Commit attribution: no `Co-Authored-By` trailers

---

### Task 1: Domain Model Foundation

**Files:**
- Create: `src/main/java/io/quarkmind/domain/GamePhase.java`
- Create: `src/main/java/io/quarkmind/domain/ArchetypeCategory.java`
- Create: `src/main/java/io/quarkmind/domain/PhaseResolver.java`
- Test: `src/test/java/io/quarkmind/domain/GamePhaseTest.java`
- Test: `src/test/java/io/quarkmind/domain/ArchetypeCategoryTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `GamePhase { EARLY, MID, LATE }`, `ArchetypeCategory { RUSH, TIMING, HARASS, MACRO, TECH, COMPOSITION }`, `PhaseResolver { GamePhase resolve(double gameTimeMinutes) }`

- [ ] **Step 1: Write tests for GamePhase and ArchetypeCategory**

```java
// GamePhaseTest.java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;
import static org.assertj.core.api.Assertions.assertThat;

class GamePhaseTest {
    @Test
    void threePhases() {
        assertThat(GamePhase.values()).containsExactly(
            GamePhase.EARLY, GamePhase.MID, GamePhase.LATE);
    }
}
```

```java
// ArchetypeCategoryTest.java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ArchetypeCategoryTest {
    @Test
    void sixCategories() {
        assertThat(ArchetypeCategory.values()).containsExactly(
            ArchetypeCategory.RUSH, ArchetypeCategory.TIMING,
            ArchetypeCategory.HARASS, ArchetypeCategory.MACRO,
            ArchetypeCategory.TECH, ArchetypeCategory.COMPOSITION);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=GamePhaseTest,ArchetypeCategoryTest -q`
Expected: compilation failure — classes don't exist

- [ ] **Step 3: Create GamePhase, ArchetypeCategory, and PhaseResolver**

Use `ide_create_file` for each:

```java
// GamePhase.java
package io.quarkmind.domain;

public enum GamePhase { EARLY, MID, LATE }
```

```java
// ArchetypeCategory.java
package io.quarkmind.domain;

public enum ArchetypeCategory { RUSH, TIMING, HARASS, MACRO, TECH, COMPOSITION }
```

```java
// PhaseResolver.java
package io.quarkmind.domain;

public interface PhaseResolver {
    GamePhase resolve(double gameTimeMinutes);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -Dtest=GamePhaseTest,ArchetypeCategoryTest -q`
Expected: PASS

- [ ] **Step 5: Verify with ide_diagnostics**

Run `ide_diagnostics` on all three new files. Expected: no errors.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/domain/GamePhase.java src/main/java/io/quarkmind/domain/ArchetypeCategory.java src/main/java/io/quarkmind/domain/PhaseResolver.java src/test/java/io/quarkmind/domain/GamePhaseTest.java src/test/java/io/quarkmind/domain/ArchetypeCategoryTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#243): add GamePhase, ArchetypeCategory enums and PhaseResolver interface"
```

---

### Task 2: Rename EnemyArchetype → StrategyArchetype

**Files:**
- Rename: `EnemyArchetype` → `StrategyArchetype` (use `ide_refactor_rename`)
- Modify: `src/main/java/io/quarkmind/domain/StrategyArchetype.java` — add `phase`, `category` constructor args
- Modify: `src/main/java/io/quarkmind/plugin/scouting/EvidenceMarker.java` — type ref (auto by rename)
- Modify: `src/main/java/io/quarkmind/plugin/scouting/ConfidenceRevision.java` — type ref (auto by rename)
- Modify: `src/main/java/io/quarkmind/plugin/scouting/PatternClassifier.java` — type refs (auto by rename)
- Modify: `src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java` — type refs (auto by rename)
- Modify: `src/main/java/io/quarkmind/agent/cbr/SC2StrategyRouterTask.java` — type refs (auto by rename)
- Modify: `src/main/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserver.java` — type refs (auto by rename)
- Modify: `src/main/resources/io/quarkmind/plugin/scouting/PatternClassification.drl` — manual DRL updates
- Modify: `src/main/resources/io/quarkmind/plugin/drools/DominanceWeightAdjustment.drl` — manual DRL updates + `.name().contains()` → `.category()`
- Modify: `src/main/resources/io/quarkmind/plugin/drools/StarCraftStrategy.drl` — manual DRL updates + `.name().contains()` → `.category()`
- Rename test: `EnemyArchetypeTest` → `StrategyArchetypeTest` (auto by rename)
- Modify: all test files referencing `EnemyArchetype` (auto by rename)

**Interfaces:**
- Consumes: `GamePhase` and `ArchetypeCategory` from Task 1
- Produces: `StrategyArchetype` enum with `race()`, `phase()`, `category()` accessors

- [ ] **Step 1: Run IDE refactor rename on EnemyArchetype**

Use `ide_refactor_rename` on `src/main/java/io/quarkmind/domain/EnemyArchetype.java` line 3 column 13, `newName: "StrategyArchetype"`. This renames the class and all Java references. `.drl` files are NOT covered.

- [ ] **Step 2: Add GamePhase and ArchetypeCategory fields to StrategyArchetype**

Use `ide_edit_member` to replace the `StrategyArchetype` enum declaration. Each existing enum constant gets `GamePhase` and `ArchetypeCategory` constructor args:

```java
public enum StrategyArchetype {
    TERRAN_MARINE_RUSH(Race.TERRAN, GamePhase.EARLY, ArchetypeCategory.RUSH),
    TERRAN_BIO_TIMING(Race.TERRAN, GamePhase.MID, ArchetypeCategory.TIMING),
    TERRAN_MECH_PUSH(Race.TERRAN, GamePhase.MID, ArchetypeCategory.TIMING),
    TERRAN_BANSHEE_HARASS(Race.TERRAN, GamePhase.EARLY, ArchetypeCategory.HARASS),
    ZERG_ZERGLING_RUSH(Race.ZERG, GamePhase.EARLY, ArchetypeCategory.RUSH),
    ZERG_ROACH_RUSH(Race.ZERG, GamePhase.EARLY, ArchetypeCategory.RUSH),
    ZERG_MACRO(Race.ZERG, GamePhase.EARLY, ArchetypeCategory.MACRO),
    PROTOSS_GATEWAY_RUSH(Race.PROTOSS, GamePhase.EARLY, ArchetypeCategory.RUSH),
    PROTOSS_CANNON_RUSH(Race.PROTOSS, GamePhase.EARLY, ArchetypeCategory.RUSH),
    PROTOSS_MACRO(Race.PROTOSS, GamePhase.EARLY, ArchetypeCategory.MACRO);

    private final Race race;
    private final GamePhase phase;
    private final ArchetypeCategory category;

    StrategyArchetype(Race race, GamePhase phase, ArchetypeCategory category) {
        this.race = race;
        this.phase = phase;
        this.category = category;
    }

    public Race race() { return race; }
    public GamePhase phase() { return phase; }
    public ArchetypeCategory category() { return category; }
}
```

- [ ] **Step 3: Update StrategyArchetypeTest for new fields**

Use `ide_edit_member` to replace the test class. Add `phase()` and `category()` assertions:

```java
@ParameterizedTest
@EnumSource(StrategyArchetype.class)
void everyArchetype_hasAllFields(StrategyArchetype arch) {
    assertThat(arch.race()).isNotNull();
    assertThat(arch.phase()).isNotNull();
    assertThat(arch.category()).isNotNull();
}
```

- [ ] **Step 4: Update PatternClassification.drl manually**

Replace all `EnemyArchetype` references with `StrategyArchetype` in `src/main/resources/io/quarkmind/plugin/scouting/PatternClassification.drl`. The import line and all `EnemyArchetype.CONSTANT` references change.

- [ ] **Step 5: Update DominanceWeightAdjustment.drl manually**

In `src/main/resources/io/quarkmind/plugin/drools/DominanceWeightAdjustment.drl`:
- Replace `import io.quarkmind.domain.EnemyArchetype` → `import io.quarkmind.domain.StrategyArchetype`
- Add: `import io.quarkmind.domain.ArchetypeCategory`
- Replace `EnemyArchetype.TERRAN_BIO_TIMING` → `StrategyArchetype.TERRAN_BIO_TIMING` (and all other specific refs)
- Replace `archetype().name().contains("RUSH")` → `archetype().category() == ArchetypeCategory.RUSH`
- Replace `archetype().name().contains("MACRO")` → `archetype().category() == ArchetypeCategory.MACRO`

- [ ] **Step 6: Update StarCraftStrategy.drl manually**

In `src/main/resources/io/quarkmind/plugin/drools/StarCraftStrategy.drl`:
- Replace `import io.quarkmind.domain.EnemyPatternAssessment` → `import io.quarkmind.domain.EnemyPatternAssessment` (stays until Task 3)
- Replace `.archetype().name().contains("RUSH")` → `.archetype().category() == ArchetypeCategory.RUSH`
- Add: `import io.quarkmind.domain.ArchetypeCategory`

- [ ] **Step 7: Run full test suite**

Run: `mvn test -q`
Expected: all tests pass. The IDE rename handles Java references; the DRL manual edits handle Drools references. If `ClassTooLargeException` occurs, run `mvn clean test -q`.

- [ ] **Step 8: Verify with ide_diagnostics**

Run `ide_diagnostics` on `StrategyArchetype.java`. Expected: no errors.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#243): rename EnemyArchetype → StrategyArchetype with phase and category fields"
```

---

### Task 3: Rename EnemyPatternAssessment → PatternAssessment

**Files:**
- Rename: `EnemyPatternAssessment` → `PatternAssessment` (use `ide_refactor_rename`)
- Modify: `src/main/java/io/quarkmind/agent/plugin/ScoutingIntelPayload.java` — rename inner record `PatternAssessment` → `PatternAssessmentPayload` (collision avoidance — do BEFORE the domain rename)
- Modify: `src/main/resources/io/quarkmind/plugin/drools/StarCraftStrategy.drl` — manual DRL update
- Modify: all test files (auto by rename)

**Interfaces:**
- Consumes: `StrategyArchetype` from Task 2
- Produces: `PatternAssessment` record with `archetype()`, `confidence()`, `detectedAtFrame()`, `rationale()`

- [ ] **Step 1: Rename ScoutingIntelPayload.PatternAssessment → PatternAssessmentPayload FIRST**

Use `ide_refactor_rename` on `src/main/java/io/quarkmind/agent/plugin/ScoutingIntelPayload.java` line 38, column 12, `newName: "PatternAssessmentPayload"`. This must happen BEFORE the domain rename to avoid a name collision.

- [ ] **Step 2: Rename domain EnemyPatternAssessment → PatternAssessment**

Use `ide_refactor_rename` on `src/main/java/io/quarkmind/domain/EnemyPatternAssessment.java` line 3, column 15, `newName: "PatternAssessment"`.

- [ ] **Step 3: Update StarCraftStrategy.drl manually**

Replace `import io.quarkmind.domain.EnemyPatternAssessment` → `import io.quarkmind.domain.PatternAssessment`. Replace all DRL references to the type.

- [ ] **Step 4: Run full test suite**

Run: `mvn test -q`
Expected: all pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#243): rename EnemyPatternAssessment → PatternAssessment, ScoutingIntelPayload inner type → PatternAssessmentPayload"
```

---

### Task 4: Rename GamePhase(summarisation) → TacticalPosture

**Files:**
- Rename: `io.quarkmind.plugin.summarisation.GamePhase` → `TacticalPosture` (use `ide_refactor_rename`)
- Modify: `src/main/java/io/quarkmind/plugin/summarisation/TacticalPosture.java` — rename record component `phase` → `posture`
- Modify: `src/main/java/io/quarkmind/plugin/summarisation/GamePhaseSummariser.java` — accessor calls
- Modify: `src/main/java/io/quarkmind/plugin/summarisation/GameArcSummariser.java` — type refs
- Modify: `src/main/java/io/quarkmind/plugin/summarisation/SummarisationLifecycle.java` — type refs
- Modify: `src/main/java/io/quarkmind/plugin/commentary/NarrativeContextHolder.java` — field + method renames
- Modify: `src/main/java/io/quarkmind/agent/GamePhaseTrigger.java` — type refs
- Modify: `src/main/java/io/quarkmind/agent/MultiFactorDominanceAssessor.java` — type refs
- Modify: `src/main/java/io/quarkmind/plugin/DroolsStrategyTask.java` — type refs
- Modify: `src/main/java/io/quarkmind/plugin/drools/StrategyRuleUnit.java` — field rename `phaseStore` → `tacticalPostureStore`, `postureStore` → `enemyPostureStore`
- Modify: `src/main/java/io/quarkmind/plugin/drools/DominanceWeightRuleUnit.java` — field rename `phaseStore` → `tacticalPostureStore`, type upgrade `DataStore<String>` → `DataStore<TacticalPosture>`
- Modify: `src/main/resources/io/quarkmind/plugin/drools/StarCraftStrategy.drl` — manual DRL field path updates
- Modify: `src/main/resources/io/quarkmind/plugin/drools/DominanceWeightAdjustment.drl` — manual DRL field path updates + pattern matching updates
- Modify: all test files

**Interfaces:**
- Consumes: nothing from prior tasks (independent rename)
- Produces: `TacticalPosture(String posture, long sinceFrame, String rationale)` record, renamed rule unit stores

- [ ] **Step 1: Rename summarisation.GamePhase → TacticalPosture**

Use `ide_refactor_rename` on `src/main/java/io/quarkmind/plugin/summarisation/GamePhase.java` line 3 column 15, `newName: "TacticalPosture"`.

- [ ] **Step 2: Rename record component phase → posture**

The `TacticalPosture` record has component `phase`. Use `ide_refactor_rename` on the `phase` component to `posture`. This renames the accessor and all call sites: `.phase()` → `.posture()`.

- [ ] **Step 3: Rename NarrativeContextHolder fields and methods**

Use `ide_refactor_rename` for each:
- Field `latestPhase` → `latestPosture`
- Method `latestPhase()` → `latestPosture()`
- Method `setLatestPhase(TacticalPosture)` → `setLatestPosture(TacticalPosture)`

- [ ] **Step 4: Rename StrategyRuleUnit fields**

Use `ide_refactor_rename` for each:
- Field `phaseStore` (line 77) → `tacticalPostureStore`
- Getter `getPhaseStore()` → `getTacticalPostureStore()` (may auto-rename with field)
- Field `postureStore` (line 60) → `enemyPostureStore`
- Getter `getPostureStore()` → `getEnemyPostureStore()` (may auto-rename with field)

- [ ] **Step 5: Rename and type-upgrade DominanceWeightRuleUnit.phaseStore**

Rename field `phaseStore` → `tacticalPostureStore` using `ide_refactor_rename`.

Then use `ide_edit_member` to change the field type from `DataStore<String>` to `DataStore<TacticalPosture>`:

```java
private final DataStore<TacticalPosture> tacticalPostureStore = DataSource.createStore();
```

Update the getter return type accordingly:

```java
public DataStore<TacticalPosture> getTacticalPostureStore() { return tacticalPostureStore; }
```

- [ ] **Step 6: Update the populator that feeds DominanceWeightRuleUnit**

Trace who calls `getPhaseStore().add(...)` on `DominanceWeightRuleUnit`. Use `ide_find_references` on the getter. Update to pass `TacticalPosture` records instead of raw strings.

- [ ] **Step 7: Update StarCraftStrategy.drl manually**

- `/phaseStore` → `/tacticalPostureStore`
- `/postureStore` → `/enemyPostureStore`
- `.phase().equals("MID_SKIRMISH")` → `.posture().equals("MID_SKIRMISH")`
- Import changes

- [ ] **Step 8: Update DominanceWeightAdjustment.drl manually**

- `/phaseStore[this == "DEFENSIVE_HOLD"]` → `/tacticalPostureStore[this.posture().equals("DEFENSIVE_HOLD")]`
- Same pattern for all phase modifier rules and combined signal rules
- Import `io.quarkmind.plugin.summarisation.TacticalPosture`

- [ ] **Step 9: Update test files**

All test files that construct `new GamePhase(...)` → `new TacticalPosture(...)`. The IDE rename handles the type name; record component rename handles accessor calls. Verify:
- `GamePhaseTriggerTest`
- `GamePhaseSummariserTest`
- `GameArcSummariserTest`
- `NarrativeContextHolderTest`
- `CommentaryAccumulatorTest`
- `SummarisationPipelineTest`
- `SummarisationPipelineIT`
- `DroolsStrategyL2L3Test`
- `DominanceWeightRuleUnitTest` (for the type upgrade)
- `DroolsDominanceWeightStrategyTest`

- [ ] **Step 10: Run full test suite**

Run: `mvn test -q`
Expected: all pass. If `ClassTooLargeException`, run `mvn clean test -q`.

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#243): rename summarisation.GamePhase → TacticalPosture, rename rule unit stores, upgrade DominanceWeightRuleUnit phaseStore type"
```

---

### Task 5: YAML Taxonomy + StrategyTaxonomy Loader

**Files:**
- Create: `src/main/resources/io/quarkmind/domain/strategy-taxonomy.yaml`
- Create: `src/main/java/io/quarkmind/domain/CounterInfo.java`
- Create: `src/main/java/io/quarkmind/domain/CounterEntry.java`
- Create: `src/main/java/io/quarkmind/agent/StrategyTaxonomy.java`
- Test: `src/test/java/io/quarkmind/agent/StrategyTaxonomyTest.java`
- Test: `src/test/java/io/quarkmind/agent/StrategyTaxonomyCoverageTest.java`

**Interfaces:**
- Consumes: `StrategyArchetype`, `GamePhase`, `ArchetypeCategory`, `UnitType` from domain
- Produces: `StrategyTaxonomy { lookup(StrategyArchetype) → ArchetypeEntry, countersFor(StrategyArchetype) → CounterInfo, signaturesForPhase(GamePhase) → List<SignatureSpec> }`, `CounterInfo(List<CounterEntry> strongCounters, List<CounterEntry> weakCounters)`, `CounterEntry(List<UnitType> units, String action)`

- [ ] **Step 1: Create CounterInfo and CounterEntry records**

```java
// CounterInfo.java
package io.quarkmind.domain;

import java.util.List;

public record CounterInfo(
    List<CounterEntry> strongCounters,
    List<CounterEntry> weakCounters
) {}
```

```java
// CounterEntry.java
package io.quarkmind.domain;

import java.util.List;

public record CounterEntry(
    List<UnitType> units,
    String action
) {}
```

- [ ] **Step 2: Write the YAML taxonomy file**

Create `src/main/resources/io/quarkmind/domain/strategy-taxonomy.yaml` with entries for all 10 existing archetypes. Each entry includes `displayName`, `race`, `phase`, `category`, `phaseWindow`, `handAuthored: true` (existing archetypes have hand-authored DRL rules), `signature`, `strongCounters`, `weakCounters`, `detectionSignals`.

Counters are Protoss-perspective. Populate signature fields based on the existing evidence rules in `PatternClassification.drl`.

- [ ] **Step 3: Write StrategyTaxonomy loader**

CDI `@ApplicationScoped` bean in `io.quarkmind.agent`:

```java
@ApplicationScoped
public class StrategyTaxonomy {
    // Inner types: ArchetypeEntry, SignatureSpec, UnitSignature
    // @PostConstruct loads strategy-taxonomy.yaml via SnakeYAML
    // Validates all entries against enum values
    // Provides: lookup(), countersFor(), forPhase(), signaturesForPhase()
}
```

Fail-fast at startup on missing fields, unknown enum values, invalid phaseWindow, unmapped entries.

- [ ] **Step 4: Write StrategyTaxonomyTest**

Unit test (no CDI) that loads the YAML directly and validates:
- Every `StrategyArchetype` enum value has a YAML entry
- Every YAML key maps to a valid enum value
- `race`, `phase`, `category` in YAML match the enum's constructor args
- `countersFor()` returns non-empty strong counters for each archetype
- `signaturesForPhase(EARLY)` returns entries
- `lookup()` returns valid `ArchetypeEntry` for every archetype

- [ ] **Step 5: Write StrategyTaxonomyCoverageTest**

Validates `handAuthored` safety and generic pipeline readiness:
- For each `handAuthored: true` archetype: verify `PatternClassification.drl` text contains the enum value name
- For each `handAuthored: false` archetype: verify YAML has non-empty `signature.units` with valid `UnitType` values and valid `phaseWindow`

- [ ] **Step 6: Run tests**

Run: `mvn test -Dtest=StrategyTaxonomyTest,StrategyTaxonomyCoverageTest -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/domain/CounterInfo.java src/main/java/io/quarkmind/domain/CounterEntry.java src/main/java/io/quarkmind/agent/StrategyTaxonomy.java src/main/resources/io/quarkmind/domain/strategy-taxonomy.yaml src/test/java/io/quarkmind/agent/StrategyTaxonomyTest.java src/test/java/io/quarkmind/agent/StrategyTaxonomyCoverageTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#243): YAML strategy taxonomy with StrategyTaxonomy loader, CounterInfo/CounterEntry records, coverage validation"
```

---

### Task 6: Detection Pipeline — SignatureFact + Generic Drools Rules

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/scouting/SignatureFact.java`
- Create: `src/main/java/io/quarkmind/plugin/scouting/SignatureFactBuilder.java`
- Modify: `src/main/java/io/quarkmind/plugin/scouting/PatternClassificationRuleUnit.java` — add `signatureStore`
- Modify: `src/main/resources/io/quarkmind/plugin/scouting/PatternClassification.drl` — add generic rules
- Modify: `src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java` — inject and wire `SignatureFactBuilder`
- Test: `src/test/java/io/quarkmind/plugin/scouting/SignatureFactBuilderTest.java`
- Test: `src/test/java/io/quarkmind/plugin/scouting/GenericPatternClassificationTest.java`

**Interfaces:**
- Consumes: `StrategyTaxonomy` from Task 5, `PatternClassificationRuleUnit` (existing)
- Produces: `SignatureFact(StrategyArchetype archetype, UnitType unitType, int minCount, double windowStart, double windowEnd, double weight, boolean noExpansion, Race race)`, `SignatureFactBuilder { List<SignatureFact> buildFacts(double gameTimeMinutes) }`

- [ ] **Step 1: Create SignatureFact record**

```java
package io.quarkmind.plugin.scouting;

import io.quarkmind.domain.Race;
import io.quarkmind.domain.StrategyArchetype;
import io.quarkmind.domain.UnitType;

public record SignatureFact(
    StrategyArchetype archetype,
    UnitType unitType,
    int minCount,
    double windowStart,
    double windowEnd,
    double weight,
    boolean noExpansion,
    Race race
) {}
```

- [ ] **Step 2: Write SignatureFactBuilderTest**

```java
package io.quarkmind.plugin.scouting;

import io.quarkmind.domain.GamePhase;
import io.quarkmind.domain.StrategyArchetype;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class SignatureFactBuilderTest {
    @Test
    void buildFacts_earlyGame_includesEarlyWindowArchetypes() {
        // Load taxonomy, build facts at gameTime=2.0
        // Assert facts include archetypes whose phaseWindow overlaps 2.0
        // Assert facts do NOT include archetypes with handAuthored=true
    }

    @Test
    void buildFacts_midGame_includesMidWindowArchetypes() {
        // Load taxonomy, build facts at gameTime=7.0
        // Assert facts include mid-game archetypes
    }

    @Test
    void buildFacts_handAuthored_excluded() {
        // Verify archetypes with handAuthored=true produce no SignatureFacts
    }
}
```

- [ ] **Step 3: Create SignatureFactBuilder**

```java
@ApplicationScoped
public class SignatureFactBuilder {
    @Inject StrategyTaxonomy taxonomy;

    public List<SignatureFact> buildFacts(double gameTimeMinutes) {
        // For each archetype where handAuthored != true:
        //   Check if gameTimeMinutes falls within [windowStart, windowEnd]
        //   For each unit in signature.units: emit a SignatureFact
    }
}
```

- [ ] **Step 4: Add signatureStore to PatternClassificationRuleUnit**

Use `ide_insert_member`:

```java
private final DataStore<SignatureFact> signatureStore = DataSource.createStore();
```

Add getter:

```java
public DataStore<SignatureFact> getSignatureStore() { return signatureStore; }
```

- [ ] **Step 5: Add generic Drools rules to PatternClassification.drl**

Append to the end of the file:

```drl
// --- GENERIC DATA-DRIVEN RULES ---

rule "Generic: unit count threshold"
when
    $sig: /signatureStore
    accumulate(/unitEvents[this.type() == $sig.unitType()];
               $count: count(); $count >= $sig.minCount())
    /gameTimeStore[this >= $sig.windowStart(), this <= $sig.windowEnd()]
then
    evidence.add(new EvidenceMarker($sig.archetype(), $sig.weight(),
        $count + " " + $sig.unitType() + " in window"));
end

rule "Generic: no expansion gate"
when
    $sig: /signatureStore[noExpansion == true]
    not /expansionEvents
    exists /unitEvents[this.type().race() == $sig.race()]
then
    evidence.add(new EvidenceMarker($sig.archetype(), 0.3,
        "No expansion with " + $sig.race() + " units"));
end
```

- [ ] **Step 6: Wire SignatureFactBuilder into DroolsScoutingTask**

In `DroolsScoutingTask`, inject `SignatureFactBuilder`. In the pattern classification block (around line 318), after `sessionManager.buildPatternRuleUnit(gameTimeMin)`, add:

```java
signatureFactBuilder.buildFacts(gameTimeMin)
    .forEach(patternData.getSignatureStore()::add);
```

- [ ] **Step 7: Write GenericPatternClassificationTest**

Test that fires the rule unit with a `SignatureFact` and verifies `EvidenceMarker` output:

```java
class GenericPatternClassificationTest {
    @Test
    void genericRule_unitCountThreshold_emitsEvidence() {
        // Create a SignatureFact for a hypothetical mid-game archetype
        // Add unit events matching the signature
        // Fire rules
        // Assert EvidenceMarker emitted for the archetype
    }

    @Test
    void genericRule_noExpansionGate_emitsEvidence() {
        // Create a SignatureFact with noExpansion=true
        // Add unit events but no expansion events
        // Fire rules
        // Assert EvidenceMarker emitted
    }

    @Test
    void genericRule_outsideWindow_noEvidence() {
        // Create a SignatureFact with window [4.0, 10.0]
        // Set gameTime to 2.0 (outside window)
        // Fire rules
        // Assert no EvidenceMarker emitted
    }
}
```

- [ ] **Step 8: Run tests**

Run: `mvn test -Dtest=SignatureFactBuilderTest,GenericPatternClassificationTest -q`
Expected: PASS

- [ ] **Step 9: Run full test suite**

Run: `mvn test -q`
Expected: all pass

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#243): data-driven detection pipeline — SignatureFact, SignatureFactBuilder, generic Drools rules"
```

---

### Task 7: Phase Resolution + Coaching Integration

**Files:**
- Create: `src/main/java/io/quarkmind/agent/TimeBasedPhaseResolver.java`
- Modify: `src/main/java/io/quarkmind/agent/QuarkMindCaseFile.java` — add `GAME_PHASE` constant
- Modify: `src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java` — inject `PhaseResolver`, set `GAME_PHASE`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingWorkerFactory.java` — inject `StrategyTaxonomy`, enrich prompt with counters and phase
- Test: `src/test/java/io/quarkmind/agent/TimeBasedPhaseResolverTest.java`
- Test: `src/test/java/io/quarkmind/plugin/coaching/CoachingWorkerFactoryTest.java` — extend existing tests for counter enrichment

**Interfaces:**
- Consumes: `PhaseResolver` from Task 1, `StrategyTaxonomy` from Task 5
- Produces: `TimeBasedPhaseResolver { resolve(double) → GamePhase }`, coaching prompt includes `ENEMY PATTERN`, `GAME PHASE`, `STRONG COUNTERS`, `WEAK COUNTERS`

- [ ] **Step 1: Write TimeBasedPhaseResolverTest**

```java
package io.quarkmind.agent;

import io.quarkmind.domain.GamePhase;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import static org.assertj.core.api.Assertions.assertThat;

class TimeBasedPhaseResolverTest {
    private final TimeBasedPhaseResolver resolver = new TimeBasedPhaseResolver();

    @ParameterizedTest
    @CsvSource({"0.0, EARLY", "2.5, EARLY", "4.99, EARLY", "5.0, MID", "8.0, MID", "11.99, MID", "12.0, LATE", "20.0, LATE"})
    void resolve_mapsTimeToPhase(double minutes, GamePhase expected) {
        assertThat(resolver.resolve(minutes)).isEqualTo(expected);
    }
}
```

- [ ] **Step 2: Create TimeBasedPhaseResolver**

```java
package io.quarkmind.agent;

import io.quarkmind.domain.GamePhase;
import io.quarkmind.domain.PhaseResolver;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class TimeBasedPhaseResolver implements PhaseResolver {
    static final double EARLY_END = 5.0;
    static final double MID_END = 12.0;

    @Override
    public GamePhase resolve(double gameTimeMinutes) {
        if (gameTimeMinutes < EARLY_END) return GamePhase.EARLY;
        if (gameTimeMinutes < MID_END) return GamePhase.MID;
        return GamePhase.LATE;
    }
}
```

- [ ] **Step 3: Add GAME_PHASE to QuarkMindCaseFile**

Use `ide_insert_member` to add:

```java
public static final String GAME_PHASE = "agent.intel.game.phase";
```

Add to `ALL_KEYS` set.

- [ ] **Step 4: Wire PhaseResolver into DroolsScoutingTask**

Inject `PhaseResolver`. In the `execute()` method, after computing `gameTimeMin`, add:

```java
GamePhase resolvedPhase = phaseResolver.resolve(gameTimeMin);
ctx.set(QuarkMindCaseFile.GAME_PHASE, resolvedPhase.name());
```

- [ ] **Step 5: Extend CoachingWorkerFactory.buildUserMessage() with counter enrichment**

Modify `buildUserMessage()` to look up counter data when pattern assessment is present:

```java
Object pattern = triggerMap.get("patternAssessment");
if (pattern instanceof Map<?, ?> patternMap) {
    Object archetype = patternMap.get("archetype");
    Object confidence = patternMap.get("confidence");
    if (archetype != null) {
        sb.append("ENEMY PATTERN: ").append(archetype);
        if (confidence != null) sb.append(" (confidence: ").append(confidence).append(")");
        sb.append("\n");

        // Counter enrichment from taxonomy
        try {
            StrategyArchetype arch = StrategyArchetype.valueOf(archetype.toString());
            CounterInfo counters = taxonomy.countersFor(arch);
            if (counters != null) {
                appendCounters(sb, "STRONG COUNTERS", counters.strongCounters());
                appendCounters(sb, "WEAK COUNTERS", counters.weakCounters());
            }
        } catch (IllegalArgumentException ignored) {}
    }
}
```

Add `GAME PHASE` from input context:

```java
Object phase = input.get(QuarkMindCaseFile.GAME_PHASE);
if (phase != null) sb.append("GAME PHASE: ").append(phase).append("\n");
```

- [ ] **Step 6: Add CoachingWorkerFactory tests for counter enrichment**

Extend existing `CoachingWorkerFactoryTest`:

```java
@Test
void buildUserMessage_withPatternAndTaxonomy_includesCounters() {
    // Build input map with pattern assessment and GAME_PHASE
    // Call buildUserMessage
    // Assert output contains "STRONG COUNTERS:" and "GAME PHASE:"
}
```

Note: `CoachingWorkerFactory` is currently a utility class with static methods. The `StrategyTaxonomy` injection requires making it a CDI bean or passing taxonomy as a parameter. Evaluate the cleanest approach at implementation time — passing as parameter to `buildUserMessage()` keeps the class testable without CDI.

- [ ] **Step 7: Run tests**

Run: `mvn test -Dtest=TimeBasedPhaseResolverTest,CoachingWorkerFactoryTest -q`
Expected: PASS

- [ ] **Step 8: Run full test suite**

Run: `mvn test -q`
Expected: all pass

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#243): TimeBasedPhaseResolver, GAME_PHASE CaseFile key, coaching prompt enrichment with counters"
```

---

### Task 8: New Archetypes + Multi-Window Calibration + Follow-Up Issues

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/StrategyArchetype.java` — add ~10-15 new enum values
- Modify: `src/main/resources/io/quarkmind/domain/strategy-taxonomy.yaml` — add entries for new archetypes
- Modify: `src/test/java/io/quarkmind/plugin/scouting/PatternClassificationCalibrationTest.java` — multi-window evaluation + expanded `deriveGroundTruth()`
- Modify: `src/test/java/io/quarkmind/domain/StrategyArchetypeTest.java` — validate new entries

**Interfaces:**
- Consumes: everything from Tasks 1-7
- Produces: ~20-25 total archetypes (10 existing + 10-15 new), multi-window calibration passing at 3/8/15 min

- [ ] **Step 1: Research SC2 mid and late-game strategies**

Before adding archetypes, research the most common mid/late-game strategies per race. Use Liquipedia, the existing replay datasets, and SC2 community knowledge. Target ~5 per race across mid and late phases. Examples:

Terran mid/late: `TERRAN_BATTLE_MECH`, `TERRAN_MARINE_TANK`, `TERRAN_MASS_BC`
Zerg mid/late: `ZERG_ROACH_HYDRA`, `ZERG_MUTA_LING`, `ZERG_BROOD_LORD_CORRUPTOR`
Protoss mid/late: `PROTOSS_STALKER_COLOSSUS`, `PROTOSS_CHARGELOT_ARCHON`, `PROTOSS_CARRIER_DEATHBALL`

Exact archetypes determined at implementation time based on replay data coverage.

- [ ] **Step 2: Add new enum values to StrategyArchetype**

Use `ide_edit_member` to add ~10-15 new values with appropriate `GamePhase` and `ArchetypeCategory`:

```java
TERRAN_BATTLE_MECH(Race.TERRAN, GamePhase.MID, ArchetypeCategory.COMPOSITION),
TERRAN_MARINE_TANK(Race.TERRAN, GamePhase.MID, ArchetypeCategory.COMPOSITION),
// ... etc
```

- [ ] **Step 3: Add YAML entries for new archetypes**

Each new archetype gets a full YAML entry with `handAuthored: false` (generic pipeline), `signature`, `strongCounters`, `weakCounters`, `detectionSignals`.

- [ ] **Step 4: Extend deriveGroundTruth() for mid/late game**

Add recognition of mid-game signatures (Marines + Medivacs after 5 min = bio timing, Siege Tanks + Thors = mech, Roach + Hydralisk = roach-hydra) and late-game compositions (Brood Lord + Corruptor, Carrier + Void Ray).

- [ ] **Step 5: Add multi-window evaluation to calibration test**

Extend `calibrateClassificationAccuracy()` to run replays to 8 min and 15 min checkpoints in addition to the existing 3 min:

```java
@Test @Tag("benchmark")
void calibrateClassificationAccuracy_multiWindow() {
    // Run at 3 min (existing) — threshold ≥ 70%
    // Run at 8 min (new) — threshold ≥ 60%
    // Run at 15 min (new) — threshold ≥ 60%
    // Report per-ArchetypeCategory accuracy
}
```

- [ ] **Step 6: Run calibration test**

Run: `mvn test -Pbenchmark -Dtest=PatternClassificationCalibrationTest -q`
Expected: accuracy thresholds met

- [ ] **Step 7: Run full test suite**

Run: `mvn test -q`
Expected: all pass

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#243): add mid/late-game archetypes, multi-window calibration at 3/8/15 min"
```

- [ ] **Step 9: Create follow-up issues under epic #252**

Verify that the design-review-created issues exist, and create any missing ones:

```bash
gh issue list --repo casehubio/quarkmind --label "epic:252" --state open
```

Required issues (check existence, create if missing):
- Fill out taxonomy to 50-90 archetypes (per race or per phase batches)
- Dynamic PhaseResolver (state-based phase detection)
- Visualizer counter overlay (consume CounterInfo API)
- Strategy transitions (temporal sequencing of archetype detections)
- Multi-race counters (matchup-keyed counter data)

- [ ] **Step 10: Final commit if issues were created**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind commit --allow-empty -m "chore(#243): verify follow-up issues under epic #252"
```
