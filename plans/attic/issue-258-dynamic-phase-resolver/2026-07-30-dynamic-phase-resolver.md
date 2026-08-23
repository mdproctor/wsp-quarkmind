# Dynamic PhaseResolver Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #258 — feat: dynamic PhaseResolver — state-based phase detection
**Issue group:** #258

**Goal:** Replace the time-only `PhaseResolver` interface with one that accepts `GameState`, implement a state-based resolver using expansion count, tech tier, and supply, and wire it via CDI producer with config-based selection.

**Architecture:** Change `PhaseResolver.resolve(double)` → `resolve(GameState)`. Add `GameState` as a CaseFile key. New `StateBasedPhaseResolver` examines buildings and supply with time floors. CDI producer selects between state-based (default) and time-based implementations via config property.

**Tech Stack:** Java 21, Quarkus CDI, JUnit 5, AssertJ

## Global Constraints

- `PhaseResolver` stays in `domain/` — plain Java, no CDI imports
- `GameState` stays in `domain/` — plain Java record
- Both resolver implementations are plain classes (no `@ApplicationScoped`) — producer owns scope
- Building signals count regardless of `isComplete` status
- All building type sets reference existing `BuildingType` enum values
- `SC2Data.GAME_LOOPS_PER_SECOND` (22.4) is the canonical FPS constant — no new constants

---

### Task 1: Interface change — PhaseResolver and GameState

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/PhaseResolver.java`
- Modify: `src/main/java/io/quarkmind/domain/GameState.java`
- Modify: `src/test/java/io/quarkmind/domain/GameStateTest.java`

**Interfaces:**
- Produces: `PhaseResolver.resolve(GameState) → GamePhase`
- Produces: `GameState.gameTimeMinutes() → double`

- [ ] **Step 1: Write failing test for GameState.gameTimeMinutes()**

```java
// In GameStateTest.java — add test method
@Test
void gameTimeMinutes_convertsFrameToMinutes() {
    GameState state = new GameState(0, 0, 0, 0, List.of(), List.of(), List.of(), List.of(), List.of(), List.of(), List.of(), (long)(5.0 * 60 * 22.4), null, PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY, Set.of(), Set.of());
    assertThat(state.gameTimeMinutes()).isCloseTo(5.0, within(0.01));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=GameStateTest#gameTimeMinutes_convertsFrameToMinutes -q`
Expected: FAIL — `gameTimeMinutes()` method does not exist

- [ ] **Step 3: Add gameTimeMinutes() to GameState**

Use `ide_insert_member` on `GameState.java` to add after the compact constructor:

```java
public double gameTimeMinutes() {
    return gameFrame() / SC2Data.GAME_LOOPS_PER_SECOND / 60.0;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -Dtest=GameStateTest#gameTimeMinutes_convertsFrameToMinutes -q`
Expected: PASS

- [ ] **Step 5: Change PhaseResolver interface**

Use `ide_edit_member` on `PhaseResolver.java`, member=`resolve`:

```java
GamePhase resolve(GameState gameState);
```

- [ ] **Step 6: Verify diagnostics — expect compile errors in consumers**

Run: `mvn compile 2>&1 | tail -20`
Expected: compile errors in `TimeBasedPhaseResolver` and `DroolsScoutingTask` (both still use old signature). This confirms the interface change propagated.

- [ ] **Step 7: Commit interface change**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/domain/PhaseResolver.java src/main/java/io/quarkmind/domain/GameState.java src/test/java/io/quarkmind/domain/GameStateTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#258): change PhaseResolver.resolve to accept GameState

Add gameTimeMinutes() convenience method to GameState.
Interface now requires GameState instead of double gameTimeMinutes.

Refs #258"
```

---

### Task 2: Update TimeBasedPhaseResolver to new interface

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/TimeBasedPhaseResolver.java`
- Modify: `src/test/java/io/quarkmind/agent/TimeBasedPhaseResolverTest.java`

**Interfaces:**
- Consumes: `PhaseResolver.resolve(GameState)`, `GameState.gameTimeMinutes()`
- Produces: `TimeBasedPhaseResolver` — plain class, same thresholds (5.0/12.0)

- [ ] **Step 1: Update test to use GameState**

Rewrite `TimeBasedPhaseResolverTest` to pass `GameState` instead of `double`:

```java
package io.quarkmind.agent;

import io.quarkmind.domain.Building;
import io.quarkmind.domain.GamePhase;
import io.quarkmind.domain.GameState;
import io.quarkmind.domain.MapInfo;
import io.quarkmind.domain.Resource;
import io.quarkmind.domain.SC2Data;
import io.quarkmind.domain.Unit;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class TimeBasedPhaseResolverTest {

    private final TimeBasedPhaseResolver resolver = new TimeBasedPhaseResolver();

    @ParameterizedTest
    @CsvSource({
        "0.0, EARLY", "2.5, EARLY", "4.99, EARLY",
        "5.0, MID", "8.0, MID", "11.99, MID",
        "12.0, LATE", "20.0, LATE"
    })
    void resolve_mapsTimeToPhase(double minutes, GamePhase expected) {
        long frame = (long) (minutes * 60 * SC2Data.GAME_LOOPS_PER_SECOND);
        GameState state = new GameState(0, 0, 0, 0, List.of(), List.of(), List.of(), List.of(), List.of(), List.of(), List.of(), frame, null, PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY, Set.of(), Set.of());
        assertThat(resolver.resolve(state)).isEqualTo(expected);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=TimeBasedPhaseResolverTest -q`
Expected: FAIL — `resolve(GameState)` not yet implemented

- [ ] **Step 3: Update TimeBasedPhaseResolver**

Use `ide_edit_member` on `TimeBasedPhaseResolver.java`, member=`TimeBasedPhaseResolver` (class declaration) to remove `@ApplicationScoped`:

```java
public class TimeBasedPhaseResolver implements PhaseResolver {
```

Use `ide_edit_member` on `TimeBasedPhaseResolver.java`, member=`resolve`:

```java
@Override
public GamePhase resolve(GameState gameState) {
    double gameTimeMinutes = gameState.gameTimeMinutes();
    if (gameTimeMinutes < EARLY_END) return GamePhase.EARLY;
    if (gameTimeMinutes < MID_END) return GamePhase.MID;
    return GamePhase.LATE;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -Dtest=TimeBasedPhaseResolverTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/agent/TimeBasedPhaseResolver.java src/test/java/io/quarkmind/agent/TimeBasedPhaseResolverTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#258): update TimeBasedPhaseResolver to resolve(GameState)

Remove @ApplicationScoped — producer will own scope.
Same 5.0/12.0 threshold behavior, new signature.

Refs #258"
```

---

### Task 3: Implement StateBasedPhaseResolver

**Files:**
- Create: `src/main/java/io/quarkmind/agent/StateBasedPhaseResolver.java`
- Create: `src/test/java/io/quarkmind/agent/StateBasedPhaseResolverTest.java`

**Interfaces:**
- Consumes: `PhaseResolver.resolve(GameState)`, `GameState.gameTimeMinutes()`, `BuildingType`, `SC2Data.GAME_LOOPS_PER_SECOND`
- Produces: `StateBasedPhaseResolver` — plain class

- [ ] **Step 1: Write failing tests**

Create `StateBasedPhaseResolverTest.java`:

```java
package io.quarkmind.agent;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class StateBasedPhaseResolverTest {

    private final StateBasedPhaseResolver resolver = new StateBasedPhaseResolver();

    static GameState state(long frame, int supplyUsed, List<Building> buildings) {
        return new GameState(0, 0, 200, supplyUsed, List.of(), buildings, List.of(), List.of(), List.of(), List.of(), List.of(), frame, null, PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY, Set.of(), Set.of());
    }

    static Building building(BuildingType type) {
        return new Building("b-1", type, new Point2d(0, 0), 100, 100, true);
    }

    static long minutesToFrames(double minutes) {
        return (long) (minutes * 60 * SC2Data.GAME_LOOPS_PER_SECOND);
    }

    // --- EARLY ---

    @Test
    void early_oneBase_noTech_lowSupply() {
        var gs = state(minutesToFrames(4), 30, List.of(building(BuildingType.NEXUS)));
        assertThat(resolver.resolve(gs)).isEqualTo(GamePhase.EARLY);
    }

    @Test
    void early_emptyBuildings_gameStart() {
        var gs = state(0, 0, List.of());
        assertThat(resolver.resolve(gs)).isEqualTo(GamePhase.EARLY);
    }

    // --- MID via expansion ---

    @Test
    void mid_twoNexus() {
        var gs = state(minutesToFrames(5), 40, List.of(
            building(BuildingType.NEXUS), building(BuildingType.NEXUS)));
        assertThat(resolver.resolve(gs)).isEqualTo(GamePhase.MID);
    }

    @Test
    void mid_terranTwoBases() {
        var gs = state(minutesToFrames(5), 40, List.of(
            building(BuildingType.COMMAND_CENTER), building(BuildingType.ORBITAL_COMMAND)));
        assertThat(resolver.resolve(gs)).isEqualTo(GamePhase.MID);
    }

    @Test
    void mid_zergTwoBases() {
        var gs = state(minutesToFrames(5), 40, List.of(
            building(BuildingType.HATCHERY), building(BuildingType.LAIR)));
        assertThat(resolver.resolve(gs)).isEqualTo(GamePhase.MID);
    }

    // --- MID via tech ---

    @Test
    void mid_oneBase_tier2Tech() {
        var gs = state(minutesToFrames(5), 30, List.of(
            building(BuildingType.NEXUS), building(BuildingType.ROBOTICS_FACILITY)));
        assertThat(resolver.resolve(gs)).isEqualTo(GamePhase.MID);
    }

    @Test
    void mid_oneBase_tier3Tech_noTier2() {
        var gs = state(minutesToFrames(5), 30, List.of(
            building(BuildingType.NEXUS), building(BuildingType.FLEET_BEACON)));
        assertThat(resolver.resolve(gs)).isEqualTo(GamePhase.MID);
    }

    // --- MID via supply ---

    @Test
    void mid_supply60() {
        var gs = state(minutesToFrames(5), 60, List.of(building(BuildingType.NEXUS)));
        assertThat(resolver.resolve(gs)).isEqualTo(GamePhase.MID);
    }

    // --- LATE via expansion + tech ---

    @Test
    void late_threeBasesAndTier3() {
        var gs = state(minutesToFrames(15), 120, List.of(
            building(BuildingType.NEXUS), building(BuildingType.NEXUS),
            building(BuildingType.NEXUS), building(BuildingType.FLEET_BEACON)));
        assertThat(resolver.resolve(gs)).isEqualTo(GamePhase.LATE);
    }

    // --- LATE via supply ---

    @Test
    void late_supply150() {
        var gs = state(minutesToFrames(15), 150, List.of(building(BuildingType.NEXUS)));
        assertThat(resolver.resolve(gs)).isEqualTo(GamePhase.LATE);
    }

    // --- Time floors ---

    @Test
    void timeFloor_midSignalsBefore3min_clampsToEarly() {
        var gs = state(minutesToFrames(2.5), 70, List.of(
            building(BuildingType.NEXUS), building(BuildingType.NEXUS)));
        assertThat(resolver.resolve(gs)).isEqualTo(GamePhase.EARLY);
    }

    @Test
    void timeFloor_lateSignalsBefore8min_clampsToMid() {
        var gs = state(minutesToFrames(6), 160, List.of(
            building(BuildingType.NEXUS), building(BuildingType.NEXUS),
            building(BuildingType.NEXUS), building(BuildingType.FLEET_BEACON)));
        assertThat(resolver.resolve(gs)).isEqualTo(GamePhase.MID);
    }

    // --- Zerg morph chain ---

    @Test
    void zergMorphChain_hiveNoTier2_isMid() {
        var gs = state(minutesToFrames(12), 80, List.of(
            building(BuildingType.HIVE)));
        assertThat(resolver.resolve(gs)).isEqualTo(GamePhase.MID);
    }

    // --- Under-construction buildings count ---

    @Test
    void underConstruction_countsTowardSignals() {
        Building incomplete = new Building("b-1", BuildingType.ROBOTICS_FACILITY,
            new Point2d(0, 0), 50, 100, false);
        var gs = state(minutesToFrames(5), 30, List.of(
            building(BuildingType.NEXUS), incomplete));
        assertThat(resolver.resolve(gs)).isEqualTo(GamePhase.MID);
    }

    // --- All races expansion detection ---

    @ParameterizedTest
    @EnumSource(value = BuildingType.class, names = {
        "NEXUS", "COMMAND_CENTER", "ORBITAL_COMMAND", "PLANETARY_FORTRESS",
        "HATCHERY", "LAIR", "HIVE"})
    void expansion_allRaceTypes(BuildingType baseType) {
        var gs = state(minutesToFrames(5), 40, List.of(
            building(baseType), building(baseType)));
        assertThat(resolver.resolve(gs)).isEqualTo(GamePhase.MID);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=StateBasedPhaseResolverTest -q`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement StateBasedPhaseResolver**

Create via `ide_create_file`:

```java
package io.quarkmind.agent;

import io.quarkmind.domain.BuildingType;
import io.quarkmind.domain.GamePhase;
import io.quarkmind.domain.GameState;
import io.quarkmind.domain.PhaseResolver;

import java.util.EnumSet;
import java.util.Set;

public class StateBasedPhaseResolver implements PhaseResolver {

    static final double MID_TIME_FLOOR = 3.0;
    static final double LATE_TIME_FLOOR = 8.0;
    static final int MID_SUPPLY_THRESHOLD = 60;
    static final int LATE_SUPPLY_THRESHOLD = 150;
    static final int MID_EXPANSION_THRESHOLD = 2;
    static final int LATE_EXPANSION_THRESHOLD = 3;

    private static final Set<BuildingType> EXPANSION_TYPES = EnumSet.of(
        BuildingType.NEXUS,
        BuildingType.COMMAND_CENTER, BuildingType.ORBITAL_COMMAND, BuildingType.PLANETARY_FORTRESS,
        BuildingType.HATCHERY, BuildingType.LAIR, BuildingType.HIVE);

    private static final Set<BuildingType> TIER_2 = EnumSet.of(
        BuildingType.ROBOTICS_FACILITY, BuildingType.STARGATE, BuildingType.TWILIGHT_COUNCIL,
        BuildingType.FACTORY, BuildingType.STARPORT,
        BuildingType.LAIR, BuildingType.HYDRALISK_DEN, BuildingType.ROACH_WARREN);

    private static final Set<BuildingType> TIER_3 = EnumSet.of(
        BuildingType.FLEET_BEACON, BuildingType.ROBOTICS_BAY, BuildingType.TEMPLAR_ARCHIVES,
        BuildingType.DARK_SHRINE,
        BuildingType.FUSION_CORE, BuildingType.GHOST_ACADEMY,
        BuildingType.HIVE, BuildingType.GREATER_SPIRE, BuildingType.ULTRALISK_CAVERN);

    @Override
    public GamePhase resolve(GameState gameState) {
        double minutes = gameState.gameTimeMinutes();

        long expansions = gameState.myBuildings().stream()
            .filter(b -> EXPANSION_TYPES.contains(b.type()))
            .count();
        boolean hasTier2 = gameState.myBuildings().stream()
            .anyMatch(b -> TIER_2.contains(b.type()));
        boolean hasTier3 = gameState.myBuildings().stream()
            .anyMatch(b -> TIER_3.contains(b.type()));
        int supply = gameState.supplyUsed();

        GamePhase raw;
        if ((expansions >= LATE_EXPANSION_THRESHOLD && hasTier3) || supply >= LATE_SUPPLY_THRESHOLD) {
            raw = GamePhase.LATE;
        } else if (expansions >= MID_EXPANSION_THRESHOLD || hasTier2 || hasTier3 || supply >= MID_SUPPLY_THRESHOLD) {
            raw = GamePhase.MID;
        } else {
            raw = GamePhase.EARLY;
        }

        if (raw == GamePhase.LATE && minutes < LATE_TIME_FLOOR) return GamePhase.MID;
        if (raw == GamePhase.MID && minutes < MID_TIME_FLOOR) return GamePhase.EARLY;
        return raw;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -Dtest=StateBasedPhaseResolverTest -q`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/agent/StateBasedPhaseResolver.java src/test/java/io/quarkmind/agent/StateBasedPhaseResolverTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#258): implement StateBasedPhaseResolver

State-based phase detection using expansion count, tech tier,
and supply with time floors (3min MID, 8min LATE).
Handles Zerg morph chain and under-construction buildings.

Refs #258"
```

---

### Task 4: CDI producer, CaseFile plumbing, and consumer wiring

**Files:**
- Create: `src/main/java/io/quarkmind/agent/PhaseResolverProducer.java`
- Create: `src/test/java/io/quarkmind/agent/PhaseResolverProducerTest.java`
- Modify: `src/main/java/io/quarkmind/agent/QuarkMindCaseFile.java`
- Modify: `src/main/java/io/quarkmind/agent/GameStateTranslator.java`
- Modify: `src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java`

**Interfaces:**
- Consumes: `StateBasedPhaseResolver`, `TimeBasedPhaseResolver`, `GameState`, `PhaseResolver.resolve(GameState)`
- Produces: `PhaseResolverProducer` (CDI), `QuarkMindCaseFile.GAME_STATE` constant

- [ ] **Step 1: Write failing test for PhaseResolverProducer**

Create `PhaseResolverProducerTest.java`:

```java
package io.quarkmind.agent;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class PhaseResolverProducerTest {

    private final PhaseResolverProducer producer = new PhaseResolverProducer();

    @Test
    void defaultStrategy_returnsStateBased() {
        var resolver = producer.phaseResolver("state-based");
        assertThat(resolver).isInstanceOf(StateBasedPhaseResolver.class);
    }

    @Test
    void timeBasedStrategy_returnsTimeBased() {
        var resolver = producer.phaseResolver("time-based");
        assertThat(resolver).isInstanceOf(TimeBasedPhaseResolver.class);
    }

    @Test
    void unknownStrategy_defaultsToStateBased() {
        var resolver = producer.phaseResolver("unknown");
        assertThat(resolver).isInstanceOf(StateBasedPhaseResolver.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=PhaseResolverProducerTest -q`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement PhaseResolverProducer**

Create via `ide_create_file`:

```java
package io.quarkmind.agent;

import io.quarkmind.domain.PhaseResolver;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
public class PhaseResolverProducer {

    @Produces
    @ApplicationScoped
    PhaseResolver phaseResolver(
            @ConfigProperty(name = "quarkmind.phase-resolver.strategy",
                           defaultValue = "state-based") String strategy) {
        return "time-based".equals(strategy)
            ? new TimeBasedPhaseResolver()
            : new StateBasedPhaseResolver();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -Dtest=PhaseResolverProducerTest -q`
Expected: PASS

- [ ] **Step 5: Add GAME_STATE to QuarkMindCaseFile**

Use `ide_insert_member` on `QuarkMindCaseFile.java` after `READY` (line 17):

```java
public static final String GAME_STATE     = "game.state";
```

Update `ALL_KEYS` list via `ide_edit_member` member=`ALL_KEYS` to add `GAME_STATE`:

```java
public static final List<String> ALL_KEYS = List.of(
        MINERALS, VESPENE, SUPPLY_USED, SUPPLY_CAP,
        WORKERS, ARMY, MY_BUILDINGS, GEYSERS, ENEMY_UNITS, GAME_FRAME, READY,
        GAME_STATE,
        RESOURCE_BUDGET, STRATEGY, CRISIS, ENEMY_ARMY_SIZE,
        ENEMY_BUILD_ORDER, TIMING_ATTACK_INCOMING, ENEMY_POSTURE, MOMENTS_LATEST, GAME_PHASE,
        STRATEGY_SELECTED_ID, STRATEGY_ROUTED_CONTEXT, STRATEGY_ROUTED_ARCHETYPE,
        STRATEGY_ROUTED_CONFIDENCE, STRATEGY_PIVOT_COUNT,
        COMMENTARY_TRIGGER, COMMENTARY_NARRATIVE_TRIGGER,
        GAME_MODE, COACHING_TRIGGER
                                                   );
```

- [ ] **Step 6: Add GameState to GameStateTranslator.toMap()**

Use `ide_replace_member` on `GameStateTranslator.java`, member=`toMap`:

```java
Map<String, Object> data = new HashMap<>();
data.put(QuarkMindCaseFile.GAME_STATE,  state);
data.put(QuarkMindCaseFile.MINERALS,    state.minerals());
data.put(QuarkMindCaseFile.VESPENE,     state.vespene());
data.put(QuarkMindCaseFile.SUPPLY_CAP,  state.supply());
data.put(QuarkMindCaseFile.SUPPLY_USED, state.supplyUsed());
data.put(QuarkMindCaseFile.GAME_FRAME,  state.gameFrame());
data.put(QuarkMindCaseFile.READY,       Boolean.TRUE);

List<Unit> workers = state.myUnits().stream()
    .filter(u -> u.type() == UnitType.PROBE).toList();
List<Unit> army = state.myUnits().stream()
    .filter(u -> u.type() != UnitType.PROBE).toList();

data.put(QuarkMindCaseFile.WORKERS,         workers);
data.put(QuarkMindCaseFile.ARMY,            army);
data.put(QuarkMindCaseFile.MY_BUILDINGS,    state.myBuildings());
data.put(QuarkMindCaseFile.GEYSERS,         state.geysers());
data.put(QuarkMindCaseFile.ENEMY_UNITS,     state.enemyUnits());
data.put(QuarkMindCaseFile.RESOURCE_BUDGET, new ResourceBudget(state.minerals(), state.vespene()));
return data;
```

- [ ] **Step 7: Update DroolsScoutingTask — remove FRAMES_PER_SECOND, update resolve call**

Use `ide_edit_member` on `DroolsScoutingTask.java`, member=`FRAMES_PER_SECOND`:

```java
static final double FRAMES_PER_SECOND = SC2Data.GAME_LOOPS_PER_SECOND;
```

Find the pattern classification block in `execute()` (around line 278-281). The lines:
```java
double gameTimeMin = gameTimeMs / 60000.0;
ctx.set(QuarkMindCaseFile.GAME_PHASE, phaseResolver.resolve(gameTimeMin).name());
```
Change to:
```java
GameState gameState = ctx.getAs(QuarkMindCaseFile.GAME_STATE, GameState.class);
double gameTimeMin = gameState.gameTimeMinutes();
ctx.set(QuarkMindCaseFile.GAME_PHASE, phaseResolver.resolve(gameState).name());
```

This requires using `ide_replace_text_in_file` or manual editing for the execute method body since it's within a large method.

- [ ] **Step 8: Verify compilation**

Run: `mvn compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 9: Run full test suite**

Run: `mvn test -q`
Expected: All tests pass. If any DroolsScoutingTask-related tests fail due to the signature change, they need to be updated in the next step.

- [ ] **Step 10: Fix any remaining test compilation**

Check `ide_diagnostics` for any test files that still reference `phaseResolver.resolve(double)`. Update them to pass `GameState`. Common pattern: construct a `GameState` with the appropriate `gameFrame` and pass it.

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/agent/PhaseResolverProducer.java src/test/java/io/quarkmind/agent/PhaseResolverProducerTest.java src/main/java/io/quarkmind/agent/QuarkMindCaseFile.java src/main/java/io/quarkmind/agent/GameStateTranslator.java src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#258): wire PhaseResolver via CDI producer, add GameState to CaseFile

PhaseResolverProducer selects state-based (default) or time-based
via quarkmind.phase-resolver.strategy config property.
GameState added as CaseFile key alongside decomposed fields.
DroolsScoutingTask updated to pass GameState to resolver.
FRAMES_PER_SECOND consolidated to SC2Data.GAME_LOOPS_PER_SECOND.

Refs #258"
```

---

### Task 5: Constant consolidation and final verification

**Files:**
- Modify: `src/test/java/io/quarkmind/plugin/scouting/PatternClassificationCalibrationTest.java`

**Interfaces:**
- Consumes: `SC2Data.GAME_LOOPS_PER_SECOND`

- [ ] **Step 1: Update PatternClassificationCalibrationTest.FRAMES_PER_SECOND**

Use `ide_edit_member` on `PatternClassificationCalibrationTest.java`, member=`FRAMES_PER_SECOND`:

```java
private static final double FRAMES_PER_SECOND = SC2Data.GAME_LOOPS_PER_SECOND;
```

- [ ] **Step 2: Run full test suite**

Run: `mvn test -q`
Expected: ALL PASS

- [ ] **Step 3: Run benchmark to verify no performance regression**

Run: `mvn test -Pbenchmark -Dtest=GameLoopBenchmarkTest -q`
Verify tick timings are within normal range.

- [ ] **Step 4: Verify no remaining compile warnings**

Run: `mvn compile -q`
Expected: BUILD SUCCESS, no warnings

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/test/java/io/quarkmind/plugin/scouting/PatternClassificationCalibrationTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "refactor(#258): consolidate FRAMES_PER_SECOND to SC2Data constant

PatternClassificationCalibrationTest now references
SC2Data.GAME_LOOPS_PER_SECOND instead of local duplicate.

Refs #258"
```
