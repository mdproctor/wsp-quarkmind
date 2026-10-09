# Chrono Boost for Protoss Economy Calibration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #391 — Playbook: Chrono Boost for Protoss economy calibration
**Issue group:** #384, #391, #392, #393

**Goal:** Model Chrono Boost in EmulatedGame so the Protoss economy playbook produces ≥50 Probes at tick 305 (up from 44 without Chrono).

**Architecture:** Add `AbilityIntent` to the sealed Intent hierarchy. ProtossRaceModel tracks Nexus energy and Chrono Boost state internally (following ZergRaceModel's Queen energy pattern). EmulatedGame delegates ability handling to the race model and applies the speed modifier to training via a new `trainingSpeedMultiplier()` SPI method. SC2DeliveryHandler gets a new `ability` action for playbook-driven ability usage.

**Tech Stack:** Java 21, Quarkus (test profile), plain JUnit 5

## Global Constraints

- Domain model (`domain/`) stays plain Java — no framework deps
- `SC2Data` energy regen constant already exists: `QUEEN_ENERGY_REGEN_PER_LOOP = 0.5625 / GAME_LOOPS_PER_SECOND` (same rate for all SC2 casters)
- Sealed `Intent` hierarchy: update permits clause AND every switch expression over Intent in the same commit
- All commits reference #391

---

## Batch 1: Ability infrastructure + ProtossRaceModel energy

### Task 1: AbilityIntent and RaceModel SPI extensions

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/AbilityIntent.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/Intent.java:3` (permits clause)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/RaceModel.java` (add two default methods)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java:264-293` (two switch expressions over Intent)

**Interfaces:**
- Produces: `AbilityIntent(String casterTag, String ability, String targetTag) implements Intent`
- Produces: `RaceModel.trainingSpeedMultiplier(String buildingTag, long gameLoop)` → `double` (default 1.0)
- Produces: `RaceModel.handleAbility(PlayerState state, String casterTag, String ability, String targetTag, long gameLoop)` → `boolean` (default false)

- [ ] **Step 1: Create AbilityIntent record**

```java
package io.quarkmind.sc2.intent;

public record AbilityIntent(String casterTag, String ability, String targetTag) implements Intent {}
```

- [ ] **Step 2: Update Intent permits clause**

Change line 3 of `Intent.java` from:
```java
public sealed interface Intent permits BuildIntent, TrainIntent, AttackIntent, MoveIntent, BlinkIntent, MuleCalldownIntent, ResearchIntent, MorphIntent {
```
to:
```java
public sealed interface Intent permits BuildIntent, TrainIntent, AttackIntent, MoveIntent, BlinkIntent, MuleCalldownIntent, ResearchIntent, MorphIntent, AbilityIntent {
```

- [ ] **Step 3: Add two default methods to RaceModel**

Append before `workerType()` in `RaceModel.java`:

```java
default double trainingSpeedMultiplier(String buildingTag, long gameLoop) { return 1.0; }

default boolean handleAbility(PlayerState state, String casterTag, String ability,
                              String targetTag, long gameLoop) { return false; }
```

- [ ] **Step 4: Add AbilityIntent cases to both switch expressions in EmulatedGame**

In `applyIntent(TimedIntent ti)` (line 269), add before the closing `};`:
```java
case AbilityIntent a -> () -> handleAbility(a, friendly, friendlyPhysics, ti.loop());
```

In `applyIntent(Intent intent, PlayerState state, PhysicsState physics)` (line 283), add before the closing `};`:
```java
case AbilityIntent a -> () -> handleAbility(a, state, physics, gameFrame * SC2Data.LOOPS_PER_TICK);
```

- [ ] **Step 5: Add handleAbility stub in EmulatedGame**

Add after `handleMuleCalldown` method (after line 628):

```java
private void handleAbility(final AbilityIntent a, final PlayerState state,
                           final PhysicsState physics, final long absLoop) {
    final RaceModel model = (state == friendly) ? playerRaceModel : null;
    if (model == null) return;
    model.handleAbility(state, a.casterTag(), a.ability(), a.targetTag(), absLoop);
}
```

- [ ] **Step 6: Verify compilation**

Run: `mvn compile -pl quarkmind-sc2 -q`
Expected: BUILD SUCCESS

- [ ] **Step 7: Run existing tests to verify no regressions**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2PlaybookCalibrationTest -q`
Expected: 3 tests pass (all race playbooks unchanged)

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/AbilityIntent.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/intent/Intent.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/RaceModel.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git commit -m "feat: add AbilityIntent and RaceModel SPI for ability handling Refs #391"
```

### Task 2: ProtossRaceModel energy tracking and Chrono Boost handling

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/ProtossRaceModel.java` (energy maps, tickPassive, handleAbility, trainingSpeedMultiplier, onBuildingComplete, seedInitialState)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java` (add Chrono Boost constants)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ProtossRaceModelChronoTest.java`

**Interfaces:**
- Consumes: `RaceModel.handleAbility(...)`, `RaceModel.trainingSpeedMultiplier(...)` from Task 1
- Produces: ProtossRaceModel energy tracking (internal) — Chrono Boost castable when Nexus has ≥50 energy

- [ ] **Step 1: Add Chrono Boost constants to SC2Data**

Add to `SC2Data.java` alongside the existing `QUEEN_ENERGY_REGEN_PER_LOOP`:

```java
public static final double NEXUS_ENERGY_REGEN_PER_LOOP = 0.5625 / GAME_LOOPS_PER_SECOND;
public static final double CHRONO_BOOST_ENERGY_COST = 50.0;
public static final int    CHRONO_BOOST_DURATION_LOOPS = 448;
public static final double CHRONO_BOOST_MULTIPLIER = 0.5;
public static final double NEXUS_STARTING_ENERGY = 50.0;
public static final double MAX_CASTER_ENERGY = 200.0;
```

- [ ] **Step 2: Write failing test — energy initialisation**

Create `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ProtossRaceModelChronoTest.java`:

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.ArrayList;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class ProtossRaceModelChronoTest {

    private ProtossRaceModel model;
    private PlayerState state;

    @BeforeEach
    void setUp() {
        model = new ProtossRaceModel();
        state = new PlayerState();
        model.seedInitialState(state, new ArrayList<>());
    }

    @Test
    void chronoBoost_deductsEnergy() {
        boolean result = model.handleAbility(state, "nexus-0", "CHRONO_BOOST", "nexus-0", 0L);
        assertTrue(result, "Chrono Boost should succeed with starting energy");
    }

    @Test
    void chronoBoost_insufficientEnergy_fails() {
        model.handleAbility(state, "nexus-0", "CHRONO_BOOST", "nexus-0", 0L);
        boolean result = model.handleAbility(state, "nexus-0", "CHRONO_BOOST", "nexus-0", 1L);
        assertFalse(result, "Second Chrono should fail — no energy left");
    }

    @Test
    void chronoBoost_appliesSpeedMultiplier() {
        model.handleAbility(state, "nexus-0", "CHRONO_BOOST", "nexus-0", 0L);
        double mult = model.trainingSpeedMultiplier("nexus-0", 0L);
        assertEquals(SC2Data.CHRONO_BOOST_MULTIPLIER, mult, 0.001);
    }

    @Test
    void chronoBoost_expiresAfterDuration() {
        model.handleAbility(state, "nexus-0", "CHRONO_BOOST", "nexus-0", 0L);
        long afterExpiry = SC2Data.CHRONO_BOOST_DURATION_LOOPS + 1;
        double mult = model.trainingSpeedMultiplier("nexus-0", afterExpiry);
        assertEquals(1.0, mult, 0.001);
    }

    @Test
    void energyRegenerates_overTime() {
        model.handleAbility(state, "nexus-0", "CHRONO_BOOST", "nexus-0", 0L);
        long loopsFor50Energy = (long) (SC2Data.CHRONO_BOOST_ENERGY_COST
            / SC2Data.NEXUS_ENERGY_REGEN_PER_LOOP);
        for (long loop = SC2Data.LOOPS_PER_TICK; loop <= loopsFor50Energy + SC2Data.LOOPS_PER_TICK; loop += SC2Data.LOOPS_PER_TICK) {
            model.tickPassive(state, loop);
        }
        boolean result = model.handleAbility(state, "nexus-0", "CHRONO_BOOST", "nexus-0", loopsFor50Energy + SC2Data.LOOPS_PER_TICK);
        assertTrue(result, "Should have regenerated enough energy for second Chrono");
    }

    @Test
    void newNexus_getsStartingEnergy() {
        model.onBuildingComplete(state, BuildingType.NEXUS, "nexus-1");
        boolean result = model.handleAbility(state, "nexus-1", "CHRONO_BOOST", "nexus-1", 0L);
        assertTrue(result, "New Nexus should have starting energy");
    }

    @Test
    void unknownAbility_returnsFalse() {
        boolean result = model.handleAbility(state, "nexus-0", "UNKNOWN", "nexus-0", 0L);
        assertFalse(result);
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ProtossRaceModelChronoTest -q`
Expected: FAIL — handleAbility returns false (default no-op)

- [ ] **Step 4: Implement ProtossRaceModel energy and Chrono Boost**

Add fields and clear them in `seedInitialState`:

```java
private final Map<String, Double> nexusEnergyMap = new HashMap<>();
private final Map<String, Long> chronoBoostedUntil = new HashMap<>();
```

In `seedInitialState`, after adding nexus-0 building, add:
```java
nexusEnergyMap.clear();
chronoBoostedUntil.clear();
nexusEnergyMap.put("nexus-0", SC2Data.NEXUS_STARTING_ENERGY);
```

Implement `tickPassive`:
```java
@Override
public void tickPassive(final PlayerState state, final long gameLoop) {
    for (final Building b : state.buildings()) {
        if (b.type() != BuildingType.NEXUS || !b.isComplete()) continue;
        double energy = nexusEnergyMap.getOrDefault(b.tag(), 0.0);
        nexusEnergyMap.put(b.tag(), Math.min(SC2Data.MAX_CASTER_ENERGY,
            energy + SC2Data.NEXUS_ENERGY_REGEN_PER_LOOP * SC2Data.LOOPS_PER_TICK));
    }
}
```

Implement `handleAbility`:
```java
@Override
public boolean handleAbility(PlayerState state, String casterTag, String ability,
                             String targetTag, long gameLoop) {
    if (!"CHRONO_BOOST".equals(ability)) return false;
    double energy = nexusEnergyMap.getOrDefault(casterTag, 0.0);
    if (energy < SC2Data.CHRONO_BOOST_ENERGY_COST) return false;
    nexusEnergyMap.put(casterTag, energy - SC2Data.CHRONO_BOOST_ENERGY_COST);
    chronoBoostedUntil.put(targetTag, gameLoop + SC2Data.CHRONO_BOOST_DURATION_LOOPS);
    return true;
}
```

Implement `trainingSpeedMultiplier`:
```java
@Override
public double trainingSpeedMultiplier(String buildingTag, long gameLoop) {
    Long expiresAt = chronoBoostedUntil.get(buildingTag);
    if (expiresAt != null && gameLoop < expiresAt) return SC2Data.CHRONO_BOOST_MULTIPLIER;
    return 1.0;
}
```

Override `onBuildingComplete`:
```java
@Override
public void onBuildingComplete(PlayerState state, BuildingType type, String buildingTag) {
    if (type == BuildingType.NEXUS) {
        nexusEnergyMap.put(buildingTag, SC2Data.NEXUS_STARTING_ENERGY);
    }
}
```

Add necessary import:
```java
import java.util.HashMap;
import java.util.Map;
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ProtossRaceModelChronoTest -q`
Expected: 7 tests PASS

- [ ] **Step 6: Run existing tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2PlaybookCalibrationTest -q`
Expected: 3 tests PASS (playbook unchanged, tickPassive now runs but no Chrono steps yet)

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/ProtossRaceModel.java quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ProtossRaceModelChronoTest.java
git commit -m "feat: ProtossRaceModel energy tracking and Chrono Boost mechanics Refs #391"
```

---

## Batch 2: EmulatedGame speed modifier + SC2DeliveryHandler ability action + playbook

### Task 3: EmulatedGame startTraining speed modifier and mid-training adjustment

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java:393-421` (startTraining) and handleAbility stub (from Task 1)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameChronoTest.java`

**Interfaces:**
- Consumes: `RaceModel.trainingSpeedMultiplier(...)`, `RaceModel.handleAbility(...)` from Tasks 1-2
- Produces: EmulatedGame applies speed multiplier to training time; handleAbility adjusts in-progress training

- [ ] **Step 1: Write failing test — Chrono Boost reduces training time**

Create `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameChronoTest.java`:

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import io.quarkmind.sc2.intent.AbilityIntent;
import io.quarkmind.sc2.intent.TrainIntent;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class EmulatedGameChronoTest {

    private EmulatedGame game;

    @BeforeEach
    void setUp() {
        game = new EmulatedGame();
        game.setPlayerRaceModel(RaceModelFactory.forRace(Race.PROTOSS));
        game.reset();
    }

    @Test
    void chronoBoost_thenTrain_completesInHalfTime() {
        game.applyIntent(new AbilityIntent("nexus-0", "CHRONO_BOOST", "nexus-0"));
        game.applyIntent(new TrainIntent("nexus-0", UnitType.PROBE));

        int normalTicks = SC2Data.trainTimeInLoops(UnitType.PROBE) / SC2Data.LOOPS_PER_TICK;
        int halfTicks = normalTicks / 2;

        long probesBefore = game.snapshot().myUnits().stream()
            .filter(u -> u.type() == UnitType.PROBE).count();
        for (int i = 0; i < halfTicks + 2; i++) game.tick();
        long probesAfter = game.snapshot().myUnits().stream()
            .filter(u -> u.type() == UnitType.PROBE).count();

        assertTrue(probesAfter > probesBefore,
            "Chrono-boosted Probe should complete in ~" + halfTicks + " ticks, not " + normalTicks);
    }

    @Test
    void midTrainingChrono_halvesRemainingTime() {
        game.applyIntent(new TrainIntent("nexus-0", UnitType.PROBE));
        int normalTicks = SC2Data.trainTimeInLoops(UnitType.PROBE) / SC2Data.LOOPS_PER_TICK;
        for (int i = 0; i < 3; i++) game.tick();

        game.applyIntent(new AbilityIntent("nexus-0", "CHRONO_BOOST", "nexus-0"));

        long probesBefore = game.snapshot().myUnits().stream()
            .filter(u -> u.type() == UnitType.PROBE).count();
        int remainingHalved = (normalTicks - 3) / 2;
        for (int i = 0; i < remainingHalved + 2; i++) game.tick();
        long probesAfter = game.snapshot().myUnits().stream()
            .filter(u -> u.type() == UnitType.PROBE).count();

        assertTrue(probesAfter > probesBefore,
            "Mid-training Chrono should halve remaining time");
    }

    @Test
    void noChrono_normalTrainingTime() {
        game.applyIntent(new TrainIntent("nexus-0", UnitType.PROBE));
        int normalTicks = SC2Data.trainTimeInLoops(UnitType.PROBE) / SC2Data.LOOPS_PER_TICK;

        long probesBefore = game.snapshot().myUnits().stream()
            .filter(u -> u.type() == UnitType.PROBE).count();
        for (int i = 0; i < normalTicks / 2; i++) game.tick();
        long probesMid = game.snapshot().myUnits().stream()
            .filter(u -> u.type() == UnitType.PROBE).count();

        assertEquals(probesBefore, probesMid,
            "Without Chrono, Probe should NOT complete in half time");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameChronoTest -q`
Expected: FAIL — `chronoBoost_thenTrain_completesInHalfTime` fails (speed multiplier not applied yet)

- [ ] **Step 3: Implement startTraining speed modifier**

In `EmulatedGame.startTraining()` (line 393-421), change the `completesAt` calculation from:

```java
final long completesAt = gameFrame
    + (loopOffset + SC2Data.trainTimeInLoops(unitType)) / SC2Data.LOOPS_PER_TICK;
```

to:

```java
final double speedMult = (model != null)
    ? model.trainingSpeedMultiplier(buildingTag, absLoop) : 1.0;
final int effectiveTrainLoops = (int)(SC2Data.trainTimeInLoops(unitType) * speedMult);
final long completesAt = gameFrame
    + (loopOffset + effectiveTrainLoops) / SC2Data.LOOPS_PER_TICK;
```

Also update `buildingCompletionAtLoop` to use `effectiveTrainLoops`:
```java
physics.buildingCompletionAtLoop.put(buildingTag, absLoop + effectiveTrainLoops);
```

- [ ] **Step 4: Implement mid-training adjustment in handleAbility**

Replace the handleAbility stub (from Task 1) with:

```java
private void handleAbility(final AbilityIntent a, final PlayerState state,
                           final PhysicsState physics, final long absLoop) {
    final RaceModel model = (state == friendly) ? playerRaceModel : null;
    if (model == null) return;

    String casterTag = a.casterTag();
    String targetTag = a.targetTag();

    if (casterTag != null && casterTag.startsWith("r-")) {
        String abilityName = a.ability();
        Building caster = state.buildings().stream()
            .filter(b -> b.isComplete() && b.type() == BuildingType.NEXUS)
            .findFirst().orElse(null);
        if (caster != null) casterTag = caster.tag();
    }
    if (targetTag != null && targetTag.startsWith("r-")) {
        Building target = state.buildings().stream()
            .filter(b -> b.isComplete() && b.type() == BuildingType.NEXUS)
            .findFirst().orElse(null);
        if (target != null) targetTag = target.tag();
    }

    if (!model.handleAbility(state, casterTag, a.ability(), targetTag, absLoop)) {
        return;
    }

    final String resolvedTarget = targetTag;
    physics.pendingCompletions.replaceAll(pc -> {
        if (!physics.buildingTrainingUntil.containsKey(resolvedTarget)) return pc;
        Long trainingUntil = physics.buildingTrainingUntil.get(resolvedTarget);
        if (trainingUntil == null || pc.completesAtTick() != trainingUntil) return pc;
        long remaining = pc.completesAtTick() - gameFrame;
        if (remaining <= 0) return pc;
        long newCompletesAt = gameFrame + remaining / 2;
        physics.buildingTrainingUntil.put(resolvedTarget, newCompletesAt);
        return new PhysicsState.PendingCompletion(newCompletesAt, pc.action());
    });
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameChronoTest -q`
Expected: 3 tests PASS

- [ ] **Step 6: Run all existing tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2PlaybookCalibrationTest,ProtossRaceModelChronoTest -q`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameChronoTest.java
git commit -m "feat: EmulatedGame training speed modifier and mid-training Chrono adjustment Refs #391"
```

### Task 4: SC2DeliveryHandler ability action + playbook update + calibration

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/playbook/SC2DeliveryHandler.java:27-35` (add ability case)
- Modify: `quarkmind-sc2/src/test/resources/playbooks/economy-only-protoss.yaml` (add Chrono steps, tighten assertion)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/SC2DeliveryHandlerTest.java` (new)

**Interfaces:**
- Consumes: `AbilityIntent` from Task 1, EmulatedGame ability handling from Task 3

- [ ] **Step 1: Write failing test — ability action in handler**

Create `quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/SC2DeliveryHandlerTest.java`:

```java
package io.quarkmind.sc2.playbook;

import io.quarkmind.domain.Race;
import io.quarkmind.domain.SC2Data;
import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.emulated.EmulatedGame;
import io.quarkmind.sc2.emulated.RaceModelFactory;
import io.casehub.pages.playbook.StepOutcome;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class SC2DeliveryHandlerTest {

    private SC2DeliveryHandler handler;
    private EmulatedGame game;

    @BeforeEach
    void setUp() {
        game = new EmulatedGame();
        game.setPlayerRaceModel(RaceModelFactory.forRace(Race.PROTOSS));
        game.reset();
        handler = new SC2DeliveryHandler(game);
    }

    @Test
    void abilityAction_chronoBoost_succeeds() {
        StepOutcome outcome = handler.execute("chrono-1",
            Map.of("action", "ability", "ability", "CHRONO_BOOST", "target", "NEXUS"), null);
        assertTrue(outcome.success(), "Chrono Boost should succeed with starting energy");
    }

    @Test
    void abilityAction_chronoBoost_insufficientEnergy_fails() {
        handler.execute("chrono-1",
            Map.of("action", "ability", "ability", "CHRONO_BOOST", "target", "NEXUS"), null);
        StepOutcome outcome = handler.execute("chrono-2",
            Map.of("action", "ability", "ability", "CHRONO_BOOST", "target", "NEXUS"), null);
        assertFalse(outcome.success(), "Second Chrono should fail — no energy");
    }

    @Test
    void abilityAction_unknownAbility_fails() {
        StepOutcome outcome = handler.execute("unknown-1",
            Map.of("action", "ability", "ability", "UNKNOWN", "target", "NEXUS"), null);
        assertFalse(outcome.success());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2DeliveryHandlerTest -q`
Expected: FAIL — "Unknown SC2 action: ability"

- [ ] **Step 3: Implement doAbility in SC2DeliveryHandler**

Add the ability case to the switch in `execute()`:
```java
case "ability" -> doAbility(stepName, data);
```

Add the method:
```java
private StepOutcome doAbility(String stepName, Map<String, Object> data) {
    String ability = (String) data.get("ability");
    String targetType = (String) data.get("target");
    String casterTag = "r-ability";
    String targetTag = "r-" + targetType.toLowerCase();
    game.applyIntent(new AbilityIntent(casterTag, ability, targetTag));
    return StepOutcome.ok(stepName, Map.of("ability", ability));
}
```

Wait — the handler needs to know if the ability succeeded or failed (for `loop: continuous` retry). But `applyIntent` returns void. We need a way to detect success.

Revised approach — check energy before and after:

```java
private StepOutcome doAbility(String stepName, Map<String, Object> data) {
    String ability = (String) data.get("ability");
    String targetType = (String) data.get("target");
    String casterTag = "r-ability";
    String targetTag = "r-" + targetType.toLowerCase();
    AbilityIntent intent = new AbilityIntent(casterTag, ability, targetTag);
    boolean accepted = game.applyAbility(intent);
    if (!accepted) {
        return StepOutcome.fail(stepName, "ability rejected: " + ability);
    }
    return StepOutcome.ok(stepName, Map.of("ability", ability));
}
```

This requires adding `applyAbility(AbilityIntent)` → `boolean` to EmulatedGame.

Add to EmulatedGame:
```java
public boolean applyAbility(AbilityIntent a) {
    final RaceModel model = playerRaceModel;
    if (model == null) return false;

    String casterTag = a.casterTag();
    String targetTag = a.targetTag();

    if (casterTag != null && casterTag.startsWith("r-")) {
        Building caster = friendly.buildings().stream()
            .filter(b -> b.isComplete() && b.type() == BuildingType.NEXUS)
            .findFirst().orElse(null);
        if (caster != null) casterTag = caster.tag();
        else return false;
    }
    if (targetTag != null && targetTag.startsWith("r-")) {
        Building target = friendly.buildings().stream()
            .filter(b -> b.isComplete() && b.type() == BuildingType.NEXUS)
            .findFirst().orElse(null);
        if (target != null) targetTag = target.tag();
        else return false;
    }

    long absLoop = gameFrame * SC2Data.LOOPS_PER_TICK;
    if (!model.handleAbility(friendly, casterTag, a.ability(), targetTag, absLoop)) {
        return false;
    }

    final String resolvedTarget = targetTag;
    friendlyPhysics.pendingCompletions.replaceAll(pc -> {
        Long trainingUntil = friendlyPhysics.buildingTrainingUntil.get(resolvedTarget);
        if (trainingUntil == null || pc.completesAtTick() != trainingUntil) return pc;
        long remaining = pc.completesAtTick() - gameFrame;
        if (remaining <= 0) return pc;
        long newCompletesAt = gameFrame + remaining / 2;
        friendlyPhysics.buildingTrainingUntil.put(resolvedTarget, newCompletesAt);
        return new PhysicsState.PendingCompletion(newCompletesAt, pc.action());
    });
    return true;
}
```

Then update the existing `handleAbility(AbilityIntent, PlayerState, PhysicsState, long)` private method to call `applyAbility` when state is friendly (or keep both paths — the private method handles the generic case for both players, the public method is the playbook-facing API). Simplest: make the private handleAbility in the applyIntent switch delegate to applyAbility for friendly state:

```java
case AbilityIntent a -> () -> { applyAbility(a); };
```

And for the two-arg `applyIntent(Intent, PlayerState, PhysicsState)`, keep the existing handleAbility private method for non-friendly paths if needed, but for now the ability is always friendly. Use:
```java
case AbilityIntent a -> () -> { applyAbility(a); };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2DeliveryHandlerTest -q`
Expected: 3 tests PASS

- [ ] **Step 5: Add import for AbilityIntent in SC2DeliveryHandler**

```java
import io.quarkmind.sc2.intent.AbilityIntent;
```

- [ ] **Step 6: Update economy-only-protoss.yaml**

Add Chrono Boost step and tighten assertion:

```yaml
playbook: economy-only-protoss
schema: sc2
meta:
  race: PROTOSS
  duration: 310 ticks

steps:
  - train: PROBE
    loop: continuous

  - ability: CHRONO_BOOST
    target: NEXUS
    loop: continuous

  - build: PYLON
    at: 14 supply

  - build: NEXUS
    at: 20 supply

  - build: PYLON
    at: 22 supply

  - build: PYLON
    at: 30 supply

  - build: PYLON
    at: 38 supply

  - build: PYLON
    at: 46 supply

  - build: PYLON
    at: 54 supply

  - build: PYLON
    at: 62 supply

  - assert: tick-61
    at: 61 ticks
    expect:
      units:
        PROBE: { min: 15 }

  - assert: tick-122
    at: 122 ticks
    expect:
      units:
        PROBE: { min: 20 }

  - assert: tick-183
    at: 183 ticks
    expect:
      units:
        PROBE: { min: 25 }

  - assert: tick-244
    at: 244 ticks
    expect:
      units:
        PROBE: { min: 33 }

  - assert: tick-305
    at: 305 ticks
    expect:
      units:
        PROBE: { min: 50 }
```

- [ ] **Step 7: Run calibration test**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2PlaybookCalibrationTest#economyOnlyProtoss -q`
Expected: PASS with ≥50 Probes at tick 305

If the assertion fails (Probes < 50), check diagnostic output — may need to adjust intermediate assertions or verify energy regen timing. The early assertions (tick-61 through tick-244) may need slight increases too since Chrono adds extra Probes throughout.

- [ ] **Step 8: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2PlaybookCalibrationTest,SC2DeliveryHandlerTest,ProtossRaceModelChronoTest,EmulatedGameChronoTest -q`
Expected: all PASS

- [ ] **Step 9: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/playbook/SC2DeliveryHandler.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/SC2DeliveryHandlerTest.java quarkmind-sc2/src/test/resources/playbooks/economy-only-protoss.yaml
git commit -m "feat: SC2DeliveryHandler ability action, Chrono Boost playbook, ≥50 Probes at tick 305 Refs #391"
```

- [ ] **Step 10: Run broader tests to check for regressions**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all tests PASS

---

## References

- [2026-10-09-chrono-boost-protoss-design.md] — design spec this plan implements
- [EmulatedGame.java:393-421] — startTraining method (speed modifier target)
- [ProtossRaceModel.java] — energy tracking target
- [ZergRaceModel.java:29-98] — Queen energy pattern (reference implementation)
- [SC2DeliveryHandler.java:27-35] — action switch to extend
- [SC2PlaybookRunner.java:68-74] — PlaybookStep record and step parsing
- [PhysicsState.java:37-59] — PendingCompletion record
- [RaceModel.java:22-108] — SPI to extend
- [Intent.java:3] — sealed permits clause
- [SC2Data.java:95] — QUEEN_ENERGY_REGEN_PER_LOOP (reuse for Nexus)
- [PP-20260522-572156] — SC2 timing calibration protocol
- [GitHub #391] — acceptance criteria and SC2 constants
- [GitHub #384] — parent issue (playbook infrastructure)
