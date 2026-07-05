# Multi-Factor DominanceAssessor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #223 — feat: multi-factor DominanceAssessor — economy, tech, army value, base count
**Issue group:** #223

**Goal:** Replace the single-factor `SupplyDominanceAssessor` with a weighted multi-factor
`MultiFactorDominanceAssessor` that computes economy, army value, tech tier, and base count
factors, each returning a per-factor score plus a weighted overall dominance score.

**Architecture:** The `DominanceAssessor` interface changes return type from `double` to a
`DominanceScore` record. `MultiFactorDominanceAssessor` computes four factors from `GameState`,
each normalised to [-1, 1], combined via fixed configurable weights. `SC2Data` gains a
`techTier(BuildingType)` method. A two-layer fog-of-war guard (combined threshold + per-factor
guards) prevents bias from sparse enemy visibility.

**Tech Stack:** Java 21, Quarkus (CDI, SmallRye Config), JUnit 5, AssertJ

## Global Constraints

- `DominanceScore` is a domain record (`io.quarkmind.domain`) — no framework dependencies.
- `MultiFactorDominanceAssessor` is a CDI `@ApplicationScoped` bean (`io.quarkmind.agent`).
- `SC2Data.techTier()` follows the existing static per-type lookup pattern.
- Worker types are: `PROBE`, `SCV`, `DRONE`.
- Base building types are: `NEXUS`, `COMMAND_CENTER`, `ORBITAL_COMMAND`, `PLANETARY_FORTRESS`, `HATCHERY`, `LAIR`, `HIVE`.
- All factor scores and overall score are clamped to [-1.0, 1.0].
- Config defaults: economy=0.30, army=0.35, tech=0.20, bases=0.15.
- No `@QuarkusTest` — all tests are plain JUnit.

---

### Task 1: DominanceScore record + DominanceAssessor interface change

**Files:**
- Create: `src/main/java/io/quarkmind/domain/DominanceScore.java`
- Modify: `src/main/java/io/quarkmind/agent/DominanceAssessor.java`

**Interfaces:**
- Consumes: nothing
- Produces: `DominanceScore(double overall, Map<String, Double> factors)`,
  `DominanceAssessor.assess(GameState) → DominanceScore`

- [ ] **Step 1: Write the failing test**

Create `src/test/java/io/quarkmind/domain/DominanceScoreTest.java`:

```java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class DominanceScoreTest {

    @Test
    void factorsAreImmutable() {
        var mutable = new java.util.HashMap<>(Map.of("economy", 0.5));
        var score = new DominanceScore(0.5, mutable);
        assertThatThrownBy(() -> score.factors().put("army", 0.3))
            .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void overallAndFactorsPreserved() {
        var score = new DominanceScore(0.7, Map.of("economy", 0.3, "army", 0.4));
        assertThat(score.overall()).isEqualTo(0.7);
        assertThat(score.factors()).containsEntry("economy", 0.3);
        assertThat(score.factors()).containsEntry("army", 0.4);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=DominanceScoreTest -q`
Expected: FAIL — `DominanceScore` class not found

- [ ] **Step 3: Create DominanceScore record**

Create `src/main/java/io/quarkmind/domain/DominanceScore.java`:

```java
package io.quarkmind.domain;

import java.util.Map;

public record DominanceScore(double overall, Map<String, Double> factors) {
    public DominanceScore {
        factors = Map.copyOf(factors);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -Dtest=DominanceScoreTest -q`
Expected: PASS

- [ ] **Step 5: Update DominanceAssessor interface**

Change `src/main/java/io/quarkmind/agent/DominanceAssessor.java`:

```java
package io.quarkmind.agent;

import io.quarkmind.domain.DominanceScore;
import io.quarkmind.domain.GameState;

public interface DominanceAssessor {
    DominanceScore assess(GameState state);
}
```

- [ ] **Step 6: Fix compilation — update MilestoneOutcomeRecorder**

`MilestoneOutcomeRecorder.java` references `DominanceAssessor`. The return type change
will cause compilation errors. Do not change the milestone logic yet — just make it compile:

In `MilestoneOutcomeRecorder.evaluateMilestones()`, if `dominanceAssessor.assess(state)`
is called, wrap with `.overall()`. Currently `evaluateMilestones` is a no-op (SPI gate),
so no functional change is needed — just ensure the field declaration compiles.

- [ ] **Step 7: Fix compilation — update MilestoneOutcomeRecorderTest**

The test uses `state -> 0.5` as a lambda for `DominanceAssessor`. Change all
lambdas to return `DominanceScore`:

```java
// Before:
state -> 0.5
// After:
state -> new DominanceScore(0.5, Map.of())
```

Update imports to include `io.quarkmind.domain.DominanceScore` and `java.util.Map`.

- [ ] **Step 8: Verify all existing tests pass**

Run: `mvn test -q`
Expected: All 1214+ tests pass

- [ ] **Step 9: Commit**

```
feat(#223): DominanceScore record and interface return type change

DominanceAssessor.assess() returns DominanceScore instead of double.
DominanceScore is a domain record with overall score and per-factor
breakdown map. All callers updated.

Refs #223
```

---

### Task 2: SC2Data.techTier() and SC2Data.isWorker()

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/SC2Data.java`
- Modify: `src/test/java/io/quarkmind/domain/SC2DataTest.java` (or create if absent)

**Interfaces:**
- Consumes: `BuildingType` enum, `UnitType` enum
- Produces: `SC2Data.techTier(BuildingType) → OptionalInt`,
  `SC2Data.isWorker(UnitType) → boolean`,
  `SC2Data.isBase(BuildingType) → boolean`

- [ ] **Step 1: Write the failing tests**

Create or extend `src/test/java/io/quarkmind/domain/SC2DataTest.java`:

```java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;

import static org.assertj.core.api.Assertions.assertThat;

class SC2DataTest {

    // --- techTier ---

    @Test
    void techTier_protossT1() {
        assertThat(SC2Data.techTier(BuildingType.GATEWAY)).hasValue(1);
    }

    @Test
    void techTier_protossT2() {
        assertThat(SC2Data.techTier(BuildingType.ROBOTICS_FACILITY)).hasValue(2);
        assertThat(SC2Data.techTier(BuildingType.STARGATE)).hasValue(2);
    }

    @Test
    void techTier_protossT3() {
        assertThat(SC2Data.techTier(BuildingType.TWILIGHT_COUNCIL)).hasValue(3);
        assertThat(SC2Data.techTier(BuildingType.TEMPLAR_ARCHIVES)).hasValue(3);
        assertThat(SC2Data.techTier(BuildingType.DARK_SHRINE)).hasValue(3);
    }

    @Test
    void techTier_protossT4() {
        assertThat(SC2Data.techTier(BuildingType.FLEET_BEACON)).hasValue(4);
        assertThat(SC2Data.techTier(BuildingType.ROBOTICS_BAY)).hasValue(4);
    }

    @Test
    void techTier_terranT1() {
        assertThat(SC2Data.techTier(BuildingType.BARRACKS)).hasValue(1);
    }

    @Test
    void techTier_terranT2() {
        assertThat(SC2Data.techTier(BuildingType.FACTORY)).hasValue(2);
        assertThat(SC2Data.techTier(BuildingType.STARPORT)).hasValue(2);
    }

    @Test
    void techTier_terranT3() {
        assertThat(SC2Data.techTier(BuildingType.GHOST_ACADEMY)).hasValue(3);
        assertThat(SC2Data.techTier(BuildingType.ARMORY)).hasValue(3);
    }

    @Test
    void techTier_terranT4() {
        assertThat(SC2Data.techTier(BuildingType.FUSION_CORE)).hasValue(4);
    }

    @Test
    void techTier_zergT1() {
        assertThat(SC2Data.techTier(BuildingType.SPAWNING_POOL)).hasValue(1);
    }

    @Test
    void techTier_zergT2() {
        assertThat(SC2Data.techTier(BuildingType.ROACH_WARREN)).hasValue(2);
        assertThat(SC2Data.techTier(BuildingType.HYDRALISK_DEN)).hasValue(2);
        assertThat(SC2Data.techTier(BuildingType.BANELING_NEST)).hasValue(2);
    }

    @Test
    void techTier_zergT3() {
        assertThat(SC2Data.techTier(BuildingType.INFESTATION_PIT)).hasValue(3);
        assertThat(SC2Data.techTier(BuildingType.LURKER_DEN)).hasValue(3);
    }

    @Test
    void techTier_zergT4() {
        assertThat(SC2Data.techTier(BuildingType.GREATER_SPIRE)).hasValue(4);
        assertThat(SC2Data.techTier(BuildingType.ULTRALISK_CAVERN)).hasValue(4);
    }

    @Test
    void techTier_nonTechBuildingsReturnEmpty() {
        assertThat(SC2Data.techTier(BuildingType.NEXUS)).isEmpty();
        assertThat(SC2Data.techTier(BuildingType.PYLON)).isEmpty();
        assertThat(SC2Data.techTier(BuildingType.ASSIMILATOR)).isEmpty();
        assertThat(SC2Data.techTier(BuildingType.PHOTON_CANNON)).isEmpty();
        assertThat(SC2Data.techTier(BuildingType.SUPPLY_DEPOT)).isEmpty();
        assertThat(SC2Data.techTier(BuildingType.HATCHERY)).isEmpty();
        assertThat(SC2Data.techTier(BuildingType.EXTRACTOR)).isEmpty();
        assertThat(SC2Data.techTier(BuildingType.UNKNOWN)).isEmpty();
    }

    // --- isWorker ---

    @Test
    void isWorker_workersReturnTrue() {
        assertThat(SC2Data.isWorker(UnitType.PROBE)).isTrue();
        assertThat(SC2Data.isWorker(UnitType.SCV)).isTrue();
        assertThat(SC2Data.isWorker(UnitType.DRONE)).isTrue();
    }

    @Test
    void isWorker_combatUnitsReturnFalse() {
        assertThat(SC2Data.isWorker(UnitType.ZEALOT)).isFalse();
        assertThat(SC2Data.isWorker(UnitType.MARINE)).isFalse();
        assertThat(SC2Data.isWorker(UnitType.ZERGLING)).isFalse();
    }

    // --- isBase ---

    @Test
    void isBase_baseBuildingsReturnTrue() {
        assertThat(SC2Data.isBase(BuildingType.NEXUS)).isTrue();
        assertThat(SC2Data.isBase(BuildingType.COMMAND_CENTER)).isTrue();
        assertThat(SC2Data.isBase(BuildingType.ORBITAL_COMMAND)).isTrue();
        assertThat(SC2Data.isBase(BuildingType.PLANETARY_FORTRESS)).isTrue();
        assertThat(SC2Data.isBase(BuildingType.HATCHERY)).isTrue();
        assertThat(SC2Data.isBase(BuildingType.LAIR)).isTrue();
        assertThat(SC2Data.isBase(BuildingType.HIVE)).isTrue();
    }

    @Test
    void isBase_nonBaseBuildingsReturnFalse() {
        assertThat(SC2Data.isBase(BuildingType.GATEWAY)).isFalse();
        assertThat(SC2Data.isBase(BuildingType.BARRACKS)).isFalse();
        assertThat(SC2Data.isBase(BuildingType.SPAWNING_POOL)).isFalse();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=SC2DataTest -q`
Expected: FAIL — `techTier`, `isWorker`, `isBase` methods not found

- [ ] **Step 3: Implement SC2Data.techTier(), isWorker(), isBase()**

Add to `src/main/java/io/quarkmind/domain/SC2Data.java`:

```java
public static OptionalInt techTier(BuildingType type) {
    return switch (type) {
        // Protoss T1
        case GATEWAY -> OptionalInt.of(1);
        // Protoss T2
        case ROBOTICS_FACILITY, STARGATE -> OptionalInt.of(2);
        // Protoss T3
        case TWILIGHT_COUNCIL, TEMPLAR_ARCHIVES, DARK_SHRINE, FORGE -> OptionalInt.of(3);
        // Protoss T4
        case FLEET_BEACON, ROBOTICS_BAY -> OptionalInt.of(4);
        // Terran T1
        case BARRACKS -> OptionalInt.of(1);
        case ENGINEERING_BAY -> OptionalInt.of(1);
        // Terran T2
        case FACTORY, STARPORT -> OptionalInt.of(2);
        // Terran T3
        case GHOST_ACADEMY, ARMORY -> OptionalInt.of(3);
        // Terran T4
        case FUSION_CORE -> OptionalInt.of(4);
        // Zerg T1
        case SPAWNING_POOL, EVOLUTION_CHAMBER -> OptionalInt.of(1);
        // Zerg T2
        case ROACH_WARREN, BANELING_NEST, HYDRALISK_DEN, SPIRE -> OptionalInt.of(2);
        // Zerg T3
        case INFESTATION_PIT, LURKER_DEN, NYDUS_NETWORK -> OptionalInt.of(3);
        // Zerg T4
        case GREATER_SPIRE, ULTRALISK_CAVERN -> OptionalInt.of(4);
        // Non-tech: bases, supply, defence, gas, unknown
        default -> OptionalInt.empty();
    };
}

public static boolean isWorker(UnitType type) {
    return type == UnitType.PROBE || type == UnitType.SCV || type == UnitType.DRONE;
}

public static boolean isBase(BuildingType type) {
    return switch (type) {
        case NEXUS, COMMAND_CENTER, ORBITAL_COMMAND, PLANETARY_FORTRESS,
             HATCHERY, LAIR, HIVE -> true;
        default -> false;
    };
}
```

Add `import java.util.OptionalInt;` to SC2Data imports.

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -Dtest=SC2DataTest -q`
Expected: PASS

- [ ] **Step 5: Run full suite to verify no regressions**

Run: `mvn test -q`
Expected: All tests pass

- [ ] **Step 6: Commit**

```
feat(#223): SC2Data.techTier(), isWorker(), isBase() lookups

Per-BuildingType tech tier mapping (1-4, empty for non-tech).
Worker and base building predicates for dominance factor calculations.

Refs #223
```

---

### Task 3: MilestoneConfig.Dominance expansion

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/MilestoneConfig.java`
- Modify: `src/main/resources/application.properties`

**Interfaces:**
- Consumes: nothing
- Produces: `MilestoneConfig.Dominance.economyWeight() → double`,
  `armyWeight()`, `techWeight()`, `basesWeight()`,
  `maxExpectedEconomyDelta()`, `maxExpectedArmyDelta()`,
  `maxExpectedTechDelta()`, `maxExpectedBaseDelta()`,
  `minEnemyVisibility() → int`

- [ ] **Step 1: Update MilestoneConfig.Dominance interface**

Replace the existing `Dominance` interface in `MilestoneConfig.java`:

```java
interface Dominance {
    @WithDefault("0.30")
    double economyWeight();
    @WithDefault("0.35")
    double armyWeight();
    @WithDefault("0.20")
    double techWeight();
    @WithDefault("0.15")
    double basesWeight();

    @WithName("max-expected-economy-delta")
    @WithDefault("25.0")
    double maxExpectedEconomyDelta();
    @WithName("max-expected-army-delta")
    @WithDefault("3000")
    int maxExpectedArmyDelta();
    @WithName("max-expected-tech-delta")
    @WithDefault("2.0")
    double maxExpectedTechDelta();
    @WithName("max-expected-base-delta")
    @WithDefault("3")
    int maxExpectedBaseDelta();

    @WithName("min-enemy-visibility")
    @WithDefault("3")
    int minEnemyVisibility();
}
```

- [ ] **Step 2: Update application.properties**

Replace `quarkmind.milestones.dominance.max-expected-delta=40` with:

```properties
quarkmind.milestones.dominance.economy-weight=0.30
quarkmind.milestones.dominance.army-weight=0.35
quarkmind.milestones.dominance.tech-weight=0.20
quarkmind.milestones.dominance.bases-weight=0.15
quarkmind.milestones.dominance.max-expected-economy-delta=25.0
quarkmind.milestones.dominance.max-expected-army-delta=3000
quarkmind.milestones.dominance.max-expected-tech-delta=2.0
quarkmind.milestones.dominance.max-expected-base-delta=3
quarkmind.milestones.dominance.min-enemy-visibility=3
```

- [ ] **Step 3: Fix SupplyDominanceAssessor compilation**

`SupplyDominanceAssessor` calls `config.dominance().maxExpectedDelta()` which no longer
exists. Since Task 4 will delete this class, for now change the CDI constructor to use
`config.dominance().maxExpectedArmyDelta()` as a temporary bridge. This is throwaway —
Task 4 deletes `SupplyDominanceAssessor` entirely.

- [ ] **Step 4: Verify compilation and tests pass**

Run: `mvn test -q`
Expected: All tests pass

- [ ] **Step 5: Commit**

```
chore(#223): expand MilestoneConfig.Dominance with multi-factor config

Weights (economy/army/tech/bases), per-factor max-expected deltas,
and fog-of-war min-enemy-visibility threshold. Replaces single
max-expected-delta.

Refs #223
```

---

### Task 4: MultiFactorDominanceAssessor implementation

**Files:**
- Create: `src/main/java/io/quarkmind/agent/MultiFactorDominanceAssessor.java`
- Create: `src/test/java/io/quarkmind/agent/MultiFactorDominanceAssessorTest.java`
- Delete: `src/main/java/io/quarkmind/agent/SupplyDominanceAssessor.java`
- Delete: `src/test/java/io/quarkmind/agent/SupplyDominanceAssessorTest.java`

**Interfaces:**
- Consumes: `DominanceAssessor` (Task 1), `DominanceScore` (Task 1),
  `SC2Data.techTier()`, `SC2Data.isWorker()`, `SC2Data.isBase()` (Task 2),
  `MilestoneConfig.Dominance` (Task 3), `GameState`, `Unit`, `Building`, `UnitType`, `BuildingType`
- Produces: `MultiFactorDominanceAssessor` CDI bean implementing `DominanceAssessor`

- [ ] **Step 1: Write the failing tests**

Create `src/test/java/io/quarkmind/agent/MultiFactorDominanceAssessorTest.java`:

```java
package io.quarkmind.agent;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.data.Offset.offset;

class MultiFactorDominanceAssessorTest {

    private final MultiFactorDominanceAssessor assessor = new MultiFactorDominanceAssessor(
        0.30, 0.35, 0.20, 0.15,  // weights
        25.0, 3000, 2.0, 3,       // max expected deltas
        3                          // min enemy visibility
    );

    // --- fog-of-war combined threshold ---

    @Test
    void assess_belowVisibilityThreshold_returnsNeutral() {
        GameState state = gameState(200, 100, 15, 10,
            List.of(zealot(), zealot()), List.of(nexus()),
            List.of(zealot()), List.of());  // 1 enemy unit + 0 buildings = 1 < 3
        DominanceScore score = assessor.assess(state);
        assertThat(score.overall()).isEqualTo(0.0);
        assertThat(score.factors().values()).allMatch(v -> v == 0.0);
    }

    @Test
    void assess_atVisibilityThreshold_calculates() {
        GameState state = gameState(200, 100, 15, 10,
            List.of(probe(), probe(), zealot()), List.of(nexus()),
            List.of(probe(), probe(), zealot()), List.of(nexus()));  // 3 + 1 = 4 >= 3
        DominanceScore score = assessor.assess(state);
        assertThat(score.overall()).isNotEqualTo(0.0).satisfies(v ->
            assertThat(Math.abs(v)).isLessThanOrEqualTo(1.0));
    }

    // --- per-factor fog guards ---

    @Test
    void assess_noEnemyWorkers_economyZero() {
        // 3 enemy combat units + 1 building → passes combined threshold
        // but no enemy workers → economy guard returns 0.0
        GameState state = gameState(200, 100, 15, 10,
            List.of(probe(), probe(), zealot()), List.of(nexus()),
            List.of(zealot(), zealot(), zealot()), List.of(nexus()));
        DominanceScore score = assessor.assess(state);
        assertThat(score.factors().get("economy")).isEqualTo(0.0);
        assertThat(score.factors().get("army")).isNotEqualTo(0.0);
    }

    @Test
    void assess_noEnemyUnits_armyAndEconomyZero() {
        // 0 enemy units + 3 buildings → passes combined threshold
        GameState state = gameState(200, 100, 15, 10,
            List.of(probe(), zealot()), List.of(nexus()),
            List.of(), List.of(nexus(), gateway(), gateway()));
        DominanceScore score = assessor.assess(state);
        assertThat(score.factors().get("economy")).isEqualTo(0.0);
        assertThat(score.factors().get("army")).isEqualTo(0.0);
        assertThat(score.factors().get("tech")).isNotEqualTo(0.0);
    }

    @Test
    void assess_noEnemyBuildings_techAndBasesZero() {
        // 3 enemy units + 0 buildings → passes combined threshold
        GameState state = gameState(200, 100, 15, 10,
            List.of(probe(), probe(), zealot()), List.of(nexus(), gateway()),
            List.of(probe(), probe(), zealot()), List.of());
        DominanceScore score = assessor.assess(state);
        assertThat(score.factors().get("tech")).isEqualTo(0.0);
        assertThat(score.factors().get("bases")).isEqualTo(0.0);
        assertThat(score.factors().get("economy")).isNotEqualTo(0.0);
    }

    // --- economy factor ---

    @Test
    void assess_economyAdvantage_positive() {
        // Own: 5 probes. Enemy: 2 probes. Income delta positive.
        GameState state = gameState(200, 100, 15, 10,
            List.of(probe(), probe(), probe(), probe(), probe()), List.of(nexus()),
            List.of(probe(), probe(), zealot()), List.of(nexus()));
        DominanceScore score = assessor.assess(state);
        assertThat(score.factors().get("economy")).isGreaterThan(0.0);
    }

    @Test
    void assess_equalEconomy_zero() {
        // Own: 3 probes. Enemy: 3 probes. Equal income.
        GameState state = gameState(200, 100, 15, 6,
            List.of(probe(), probe(), probe()), List.of(nexus()),
            List.of(probe(), probe(), probe()), List.of(nexus()));
        DominanceScore score = assessor.assess(state);
        assertThat(score.factors().get("economy")).isCloseTo(0.0, offset(0.001));
    }

    // --- army value factor ---

    @Test
    void assess_armyAdvantage_positive() {
        // Own: 4 zealots (400 minerals). Enemy: 1 zealot (100 minerals).
        GameState state = gameState(200, 100, 15, 10,
            List.of(probe(), zealot(), zealot(), zealot(), zealot()), List.of(nexus()),
            List.of(probe(), zealot()), List.of(nexus()));
        DominanceScore score = assessor.assess(state);
        assertThat(score.factors().get("army")).isGreaterThan(0.0);
    }

    @Test
    void assess_armyExcludesWorkers() {
        // Own: 3 probes only (no army). Enemy: 3 probes only.
        // Army value should be 0 on both sides → delta 0.
        GameState state = gameState(200, 100, 15, 6,
            List.of(probe(), probe(), probe()), List.of(nexus()),
            List.of(probe(), probe(), probe()), List.of(nexus()));
        DominanceScore score = assessor.assess(state);
        assertThat(score.factors().get("army")).isCloseTo(0.0, offset(0.001));
    }

    // --- tech tier factor ---

    @Test
    void assess_techAdvantage_positive() {
        // Own: NEXUS + GATEWAY + ROBOTICS_FACILITY (T1 + T2 → maxTier=2, breadth=2)
        // Enemy: NEXUS + GATEWAY only (T1 → maxTier=1, breadth=1)
        GameState state = gameState(200, 100, 15, 10,
            List.of(probe(), probe(), probe()), List.of(nexus(), gateway(), roboFacility()),
            List.of(probe(), probe(), probe()), List.of(nexus(), gateway()));
        DominanceScore score = assessor.assess(state);
        assertThat(score.factors().get("tech")).isGreaterThan(0.0);
    }

    @Test
    void assess_incompleteBuilding_notCounted() {
        // Own: NEXUS + incomplete ROBOTICS_FACILITY
        // Enemy: NEXUS + complete GATEWAY
        GameState state = gameState(200, 100, 15, 10,
            List.of(probe(), probe(), probe()), List.of(nexus(), incompleteBuilding(BuildingType.ROBOTICS_FACILITY)),
            List.of(probe(), probe(), probe()), List.of(nexus(), gateway()));
        DominanceScore score = assessor.assess(state);
        // Own has no complete tech buildings, enemy has GATEWAY (T1)
        assertThat(score.factors().get("tech")).isLessThan(0.0);
    }

    // --- base count factor ---

    @Test
    void assess_baseAdvantage_positive() {
        // Own: 2 nexuses. Enemy: 1 nexus.
        GameState state = gameState(200, 100, 15, 10,
            List.of(probe(), probe(), probe()), List.of(nexus(), nexus()),
            List.of(probe(), probe(), probe()), List.of(nexus()));
        DominanceScore score = assessor.assess(state);
        assertThat(score.factors().get("bases")).isGreaterThan(0.0);
    }

    // --- overall weighting ---

    @Test
    void assess_overallIsWeightedSum() {
        // Equal everything → all factors ~0 → overall ~0
        GameState state = gameState(200, 100, 15, 6,
            List.of(probe(), probe(), probe()), List.of(nexus(), gateway()),
            List.of(probe(), probe(), probe()), List.of(nexus(), gateway()));
        DominanceScore score = assessor.assess(state);
        double expectedOverall = score.factors().get("economy") * 0.30
            + score.factors().get("army") * 0.35
            + score.factors().get("tech") * 0.20
            + score.factors().get("bases") * 0.15;
        assertThat(score.overall()).isCloseTo(
            Math.max(-1.0, Math.min(1.0, expectedOverall)), offset(0.001));
    }

    @Test
    void assess_overallClampedToOne() {
        // Extreme advantage in all factors
        MultiFactorDominanceAssessor smallDelta = new MultiFactorDominanceAssessor(
            0.30, 0.35, 0.20, 0.15, 1.0, 100, 0.5, 1, 3);
        GameState state = gameState(200, 100, 15, 20,
            armyOf(10, UnitType.ZEALOT), List.of(nexus(), nexus(), nexus(), gateway(), roboFacility(), fleetBeacon()),
            List.of(probe()), List.of(nexus()));
        DominanceScore score = smallDelta.assess(state);
        assertThat(score.overall()).isLessThanOrEqualTo(1.0);
        assertThat(score.overall()).isGreaterThanOrEqualTo(-1.0);
    }

    // --- factor keys ---

    @Test
    void assess_containsAllFourFactors() {
        GameState state = gameState(200, 100, 15, 10,
            List.of(probe(), probe(), probe()), List.of(nexus()),
            List.of(probe(), probe(), probe()), List.of(nexus()));
        DominanceScore score = assessor.assess(state);
        assertThat(score.factors()).containsOnlyKeys("economy", "army", "tech", "bases");
    }

    // --- helpers ---

    private static GameState gameState(int minerals, int vespene, int supply, int supplyUsed,
            List<Unit> myUnits, List<Building> myBuildings,
            List<Unit> enemyUnits, List<Building> enemyBuildings) {
        return new GameState(minerals, vespene, supply, supplyUsed,
            myUnits, myBuildings, enemyUnits, enemyBuildings,
            List.of(), List.of(), List.of(), 5000);
    }

    private static Unit probe() { return unit(UnitType.PROBE); }
    private static Unit zealot() { return unit(UnitType.ZEALOT); }

    private static Unit unit(UnitType type) {
        return new Unit("tag-" + type, type, new Point2d(0, 0), 100, 100, 50, 50, 0, 0);
    }

    private static List<Unit> armyOf(int count, UnitType type) {
        return java.util.stream.IntStream.range(0, count).mapToObj(i -> unit(type)).toList();
    }

    private static Building nexus() { return building(BuildingType.NEXUS); }
    private static Building gateway() { return building(BuildingType.GATEWAY); }
    private static Building roboFacility() { return building(BuildingType.ROBOTICS_FACILITY); }
    private static Building fleetBeacon() { return building(BuildingType.FLEET_BEACON); }

    private static Building building(BuildingType type) {
        return new Building("tag-" + type, type, new Point2d(0, 0), 1000, 1000, true);
    }

    private static Building incompleteBuilding(BuildingType type) {
        return new Building("tag-" + type, type, new Point2d(0, 0), 500, 1000, false);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=MultiFactorDominanceAssessorTest -q`
Expected: FAIL — `MultiFactorDominanceAssessor` class not found

- [ ] **Step 3: Implement MultiFactorDominanceAssessor**

Create `src/main/java/io/quarkmind/agent/MultiFactorDominanceAssessor.java`:

```java
package io.quarkmind.agent;

import io.quarkmind.domain.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.OptionalInt;

@ApplicationScoped
public class MultiFactorDominanceAssessor implements DominanceAssessor {

    private final double economyWeight;
    private final double armyWeight;
    private final double techWeight;
    private final double basesWeight;
    private final double maxExpectedEconomyDelta;
    private final int maxExpectedArmyDelta;
    private final double maxExpectedTechDelta;
    private final int maxExpectedBaseDelta;
    private final int minEnemyVisibility;

    @Inject
    MultiFactorDominanceAssessor(MilestoneConfig config) {
        this(config.dominance().economyWeight(),
             config.dominance().armyWeight(),
             config.dominance().techWeight(),
             config.dominance().basesWeight(),
             config.dominance().maxExpectedEconomyDelta(),
             config.dominance().maxExpectedArmyDelta(),
             config.dominance().maxExpectedTechDelta(),
             config.dominance().maxExpectedBaseDelta(),
             config.dominance().minEnemyVisibility());
    }

    MultiFactorDominanceAssessor(
            double economyWeight, double armyWeight, double techWeight, double basesWeight,
            double maxExpectedEconomyDelta, int maxExpectedArmyDelta,
            double maxExpectedTechDelta, int maxExpectedBaseDelta,
            int minEnemyVisibility) {
        this.economyWeight = economyWeight;
        this.armyWeight = armyWeight;
        this.techWeight = techWeight;
        this.basesWeight = basesWeight;
        this.maxExpectedEconomyDelta = maxExpectedEconomyDelta;
        this.maxExpectedArmyDelta = maxExpectedArmyDelta;
        this.maxExpectedTechDelta = maxExpectedTechDelta;
        this.maxExpectedBaseDelta = maxExpectedBaseDelta;
        this.minEnemyVisibility = minEnemyVisibility;
    }

    private static final DominanceScore NEUTRAL = new DominanceScore(0.0,
        Map.of("economy", 0.0, "army", 0.0, "tech", 0.0, "bases", 0.0));

    @Override
    public DominanceScore assess(GameState state) {
        int totalEnemyVisible = state.enemyUnits().size() + state.enemyBuildings().size();
        if (totalEnemyVisible < minEnemyVisibility) {
            return NEUTRAL;
        }

        double economy = economyFactor(state);
        double army = armyFactor(state);
        double tech = techFactor(state);
        double bases = basesFactor(state);

        double overall = clamp(economy * economyWeight + army * armyWeight
            + tech * techWeight + bases * basesWeight);

        Map<String, Double> factors = new LinkedHashMap<>(4);
        factors.put("economy", economy);
        factors.put("army", army);
        factors.put("tech", tech);
        factors.put("bases", bases);

        return new DominanceScore(overall, factors);
    }

    private double economyFactor(GameState state) {
        long ownWorkers = state.myUnits().stream()
            .filter(u -> SC2Data.isWorker(u.type())).count();
        long enemyWorkers = state.enemyUnits().stream()
            .filter(u -> SC2Data.isWorker(u.type())).count();
        if (enemyWorkers == 0) return 0.0;

        double rate = SC2Data.MINERAL_TIER_RATES_PER_TICK[0];
        double delta = (ownWorkers - enemyWorkers) * rate;
        return clamp(delta / maxExpectedEconomyDelta);
    }

    private double armyFactor(GameState state) {
        if (state.enemyUnits().isEmpty()) return 0.0;

        int ownValue = state.myUnits().stream()
            .filter(u -> !SC2Data.isWorker(u.type()))
            .mapToInt(u -> SC2Data.mineralCost(u.type()) + SC2Data.gasCost(u.type()))
            .sum();
        int enemyValue = state.enemyUnits().stream()
            .filter(u -> !SC2Data.isWorker(u.type()))
            .mapToInt(u -> SC2Data.mineralCost(u.type()) + SC2Data.gasCost(u.type()))
            .sum();
        return clamp((double) (ownValue - enemyValue) / maxExpectedArmyDelta);
    }

    private double techFactor(GameState state) {
        if (state.enemyBuildings().isEmpty()) return 0.0;

        double ownScore = techScore(state.myBuildings());
        double enemyScore = techScore(state.enemyBuildings());
        return clamp((ownScore - enemyScore) / maxExpectedTechDelta);
    }

    static double techScore(List<Building> buildings) {
        int maxTier = 0;
        int breadth = 0;
        for (Building b : buildings) {
            if (!b.isComplete()) continue;
            OptionalInt tier = SC2Data.techTier(b.type());
            if (tier.isPresent()) {
                maxTier = Math.max(maxTier, tier.getAsInt());
                breadth++;
            }
        }
        return maxTier + 0.1 * breadth;
    }

    private double basesFactor(GameState state) {
        if (state.enemyBuildings().isEmpty()) return 0.0;

        long ownBases = state.myBuildings().stream()
            .filter(b -> b.isComplete() && SC2Data.isBase(b.type())).count();
        long enemyBases = state.enemyBuildings().stream()
            .filter(b -> b.isComplete() && SC2Data.isBase(b.type())).count();
        return clamp((double) (ownBases - enemyBases) / maxExpectedBaseDelta);
    }

    private static double clamp(double value) {
        return Math.max(-1.0, Math.min(1.0, value));
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -Dtest=MultiFactorDominanceAssessorTest -q`
Expected: PASS

- [ ] **Step 5: Delete SupplyDominanceAssessor and its test**

Delete `src/main/java/io/quarkmind/agent/SupplyDominanceAssessor.java` and
`src/test/java/io/quarkmind/agent/SupplyDominanceAssessorTest.java`.

- [ ] **Step 6: Run full test suite**

Run: `mvn test -q`
Expected: All tests pass (test count decreases by SupplyDominanceAssessorTest count,
increases by MultiFactorDominanceAssessorTest + DominanceScoreTest + SC2DataTest count)

- [ ] **Step 7: Commit**

```
feat(#223): MultiFactorDominanceAssessor replaces SupplyDominanceAssessor

Four factors (economy, army, tech, bases) with configurable weights
and two-layer fog-of-war guard. Each factor normalised to [-1, 1].
Deletes SupplyDominanceAssessor placeholder.

Refs #223
```

---

### Task 5: Update MilestoneOutcomeRecorderTest for DominanceScore

**Files:**
- Modify: `src/test/java/io/quarkmind/agent/MilestoneOutcomeRecorderTest.java`

**Interfaces:**
- Consumes: `DominanceScore` (Task 1)
- Produces: nothing new — verifies existing callers work with the new return type

- [ ] **Step 1: Update all DominanceAssessor lambdas in MilestoneOutcomeRecorderTest**

Every lambda `state -> 0.5` or `state -> 0.1` becomes
`state -> new DominanceScore(0.5, Map.of())` or
`state -> new DominanceScore(0.1, Map.of())`.

This was partially done in Task 1 Step 7 to fix compilation. Verify all instances
are updated and add `import io.quarkmind.domain.DominanceScore;` if missing.

- [ ] **Step 2: Run MilestoneOutcomeRecorderTest**

Run: `mvn test -Dtest=MilestoneOutcomeRecorderTest -q`
Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `mvn test -q`
Expected: All tests pass

- [ ] **Step 4: Commit**

```
test(#223): update MilestoneOutcomeRecorderTest for DominanceScore

Lambda DominanceAssessor stubs return DominanceScore records
instead of raw doubles.

Closes #223
```
