# ocraft Action Translation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement `ActionTranslator` so `SC2BotAgent.onStep()` translates domain `Intent` objects into real ocraft `ActionInterface` commands, making the bot capable of acting in a live SC2 game.

**Architecture:** A pure static `ActionTranslator` class (mirrors `ObservationTranslator`) converts `List<Intent>` to `List<ResolvedCommand>` — a package-private record holding the Tag, Ability, and optional Point2d target. `SC2BotAgent.onStep()` applies each `ResolvedCommand` via the `ActionInterface`. Returning data rather than calling the interface directly keeps `ActionTranslator` a pure function, fully testable without mocking.

**Tech Stack:** Java 21, ocraft-s2client-bot 0.4.21, JUnit 5, AssertJ. No CDI, no mocking framework needed.

---

## File Map

| File | Change |
|---|---|
| `src/main/java/io/quarkmind/sc2/real/ResolvedCommand.java` | **Create** — package-private record: `(Tag tag, Abilities ability, Optional<ocraft.Point2d> target)` |
| `src/main/java/io/quarkmind/sc2/real/ActionTranslator.java` | **Create** — pure static translator; `translate()`, `mapBuildAbility()`, `mapTrainAbility()` |
| `src/main/java/io/quarkmind/sc2/real/SC2BotAgent.java` | **Modify** — replace no-op intent drain in `onStep()` with `ActionTranslator.translate()` |
| `src/test/java/io/quarkmind/sc2/real/ActionTranslatorTest.java` | **Create** — full test suite: mapping tables + dispatch + error/edge cases |

---

## Task 1: ResolvedCommand record

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/real/ResolvedCommand.java`

`ResolvedCommand` is the bridge between `ActionTranslator` (which produces commands) and `SC2BotAgent` (which dispatches them). It must exist before any test or translator code can compile. No behaviour to test-drive — it is a data carrier with no logic.

- [ ] **Step 1: Create ResolvedCommand.java**

```java
package io.quarkmind.sc2.real;

import com.github.ocraft.s2client.protocol.data.Abilities;
import com.github.ocraft.s2client.protocol.spatial.Point2d;
import com.github.ocraft.s2client.protocol.unit.Tag;

import java.util.Optional;

/**
 * A resolved SC2 command ready for dispatch via ActionInterface.
 * Package-private — only SC2BotAgent and ActionTranslator interact with this type.
 *
 * target is present for position-targeted commands (build, attack, move)
 * and empty for non-positional commands (train).
 */
record ResolvedCommand(Tag tag, Abilities ability, Optional<Point2d> target) {}
```

- [ ] **Step 2: Verify it compiles**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/real/ResolvedCommand.java
git commit -m "feat(sc2): add ResolvedCommand — resolved ocraft command ready to dispatch

Refs #12"
```

---

## Task 2: mapBuildAbility — tests then implementation

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/real/ActionTranslator.java` (stub)
- Create: `src/test/java/io/quarkmind/sc2/real/ActionTranslatorTest.java` (Group 1 build tests)

Write the build ability mapping tests against a stub translator, verify they fail, then implement the map.

- [ ] **Step 1: Create ActionTranslator stub**

```java
package io.quarkmind.sc2.real;

import com.github.ocraft.s2client.protocol.data.Abilities;
import com.github.ocraft.s2client.protocol.unit.Tag;
import io.quarkmind.domain.BuildingType;
import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.intent.Intent;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Optional;

/**
 * Translates domain Intent objects into ResolvedCommands for dispatch via ActionInterface.
 *
 * Pure static functions — no CDI, no instance state. All methods are static because
 * this class is called directly by SC2BotAgent: ActionTranslator.translate(intents).
 * Mirrors ObservationTranslator in design.
 *
 * mapBuildAbility and mapTrainAbility are package-private so ActionTranslatorTest
 * can call them directly without reflection.
 */
public final class ActionTranslator {

    private static final Logger log = Logger.getLogger(ActionTranslator.class);

    private ActionTranslator() {}

    public static List<ResolvedCommand> translate(List<Intent> intents) {
        throw new UnsupportedOperationException("not yet implemented");
    }

    static Abilities mapBuildAbility(BuildingType type) {
        throw new UnsupportedOperationException("not yet implemented");
    }

    static Abilities mapTrainAbility(UnitType type) {
        throw new UnsupportedOperationException("not yet implemented");
    }
}
```

- [ ] **Step 2: Create ActionTranslatorTest with Group 1 build tests**

```java
package io.quarkmind.sc2.real;

import com.github.ocraft.s2client.protocol.data.Abilities;
import com.github.ocraft.s2client.protocol.spatial.Point2d;
import com.github.ocraft.s2client.protocol.unit.Tag;
import io.quarkmind.domain.BuildingType;
import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.intent.AttackIntent;
import io.quarkmind.sc2.intent.BuildIntent;
import io.quarkmind.sc2.intent.MoveIntent;
import io.quarkmind.sc2.intent.TrainIntent;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class ActionTranslatorTest {

    // ─── Group 1: Ability maps ────────────────────────────────────────────────

    @Test
    void allProtossBuildAbilitiesMapCorrectly() {
        assertThat(ActionTranslator.mapBuildAbility(BuildingType.NEXUS))
            .isEqualTo(Abilities.BUILD_NEXUS);
        assertThat(ActionTranslator.mapBuildAbility(BuildingType.PYLON))
            .isEqualTo(Abilities.BUILD_PYLON);
        assertThat(ActionTranslator.mapBuildAbility(BuildingType.GATEWAY))
            .isEqualTo(Abilities.BUILD_GATEWAY);
        assertThat(ActionTranslator.mapBuildAbility(BuildingType.CYBERNETICS_CORE))
            .isEqualTo(Abilities.BUILD_CYBERNETICS_CORE);
        assertThat(ActionTranslator.mapBuildAbility(BuildingType.ASSIMILATOR))
            .isEqualTo(Abilities.BUILD_ASSIMILATOR);
        assertThat(ActionTranslator.mapBuildAbility(BuildingType.ROBOTICS_FACILITY))
            .isEqualTo(Abilities.BUILD_ROBOTICS_FACILITY);
        assertThat(ActionTranslator.mapBuildAbility(BuildingType.STARGATE))
            .isEqualTo(Abilities.BUILD_STARGATE);
        assertThat(ActionTranslator.mapBuildAbility(BuildingType.FORGE))
            .isEqualTo(Abilities.BUILD_FORGE);
        assertThat(ActionTranslator.mapBuildAbility(BuildingType.TWILIGHT_COUNCIL))
            .isEqualTo(Abilities.BUILD_TWILIGHT_COUNCIL);
    }

    @Test
    void unknownBuildingTypeHasNullBuildAbility() {
        assertThat(ActionTranslator.mapBuildAbility(BuildingType.UNKNOWN)).isNull();
    }
}
```

- [ ] **Step 3: Run tests — verify they fail**

```bash
mvn test -Dtest=ActionTranslatorTest -q
```

Expected: FAIL — `UnsupportedOperationException: not yet implemented`

- [ ] **Step 4: Implement mapBuildAbility**

Replace the stub `mapBuildAbility` body in `ActionTranslator.java`:

```java
static Abilities mapBuildAbility(BuildingType type) {
    return switch (type) {
        case NEXUS             -> Abilities.BUILD_NEXUS;
        case PYLON             -> Abilities.BUILD_PYLON;
        case GATEWAY           -> Abilities.BUILD_GATEWAY;
        case CYBERNETICS_CORE  -> Abilities.BUILD_CYBERNETICS_CORE;
        case ASSIMILATOR       -> Abilities.BUILD_ASSIMILATOR;
        case ROBOTICS_FACILITY -> Abilities.BUILD_ROBOTICS_FACILITY;
        case STARGATE          -> Abilities.BUILD_STARGATE;
        case FORGE             -> Abilities.BUILD_FORGE;
        case TWILIGHT_COUNCIL  -> Abilities.BUILD_TWILIGHT_COUNCIL;
        case UNKNOWN           -> null;
    };
}
```

- [ ] **Step 5: Run tests — verify they pass**

```bash
mvn test -Dtest=ActionTranslatorTest -q
```

Expected: BUILD SUCCESS — 2 tests pass

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/real/ActionTranslator.java \
        src/test/java/io/quarkmind/sc2/real/ActionTranslatorTest.java
git commit -m "feat(sc2): ActionTranslator — mapBuildAbility covers all Protoss buildings

Refs #12"
```

---

## Task 3: mapTrainAbility — tests then implementation

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/real/ActionTranslatorTest.java` (add Group 1 train tests)
- Modify: `src/main/java/io/quarkmind/sc2/real/ActionTranslator.java` (implement mapTrainAbility)

- [ ] **Step 1: Add train ability tests to ActionTranslatorTest**

Add these three test methods inside `ActionTranslatorTest`, after the existing two tests:

```java
    @Test
    void allProtossTrainAbilitiesMapCorrectly() {
        assertThat(ActionTranslator.mapTrainAbility(UnitType.PROBE))
            .isEqualTo(Abilities.TRAIN_PROBE);
        assertThat(ActionTranslator.mapTrainAbility(UnitType.ZEALOT))
            .isEqualTo(Abilities.TRAIN_ZEALOT);
        assertThat(ActionTranslator.mapTrainAbility(UnitType.STALKER))
            .isEqualTo(Abilities.TRAIN_STALKER);
        assertThat(ActionTranslator.mapTrainAbility(UnitType.IMMORTAL))
            .isEqualTo(Abilities.TRAIN_IMMORTAL);
        assertThat(ActionTranslator.mapTrainAbility(UnitType.COLOSSUS))
            .isEqualTo(Abilities.TRAIN_COLOSSUS);
        assertThat(ActionTranslator.mapTrainAbility(UnitType.CARRIER))
            .isEqualTo(Abilities.TRAIN_CARRIER);
        assertThat(ActionTranslator.mapTrainAbility(UnitType.DARK_TEMPLAR))
            .isEqualTo(Abilities.TRAIN_DARK_TEMPLAR);
        assertThat(ActionTranslator.mapTrainAbility(UnitType.HIGH_TEMPLAR))
            .isEqualTo(Abilities.TRAIN_HIGH_TEMPLAR);
        assertThat(ActionTranslator.mapTrainAbility(UnitType.OBSERVER))
            .isEqualTo(Abilities.TRAIN_OBSERVER);
        // Note: ocraft spells this TRAIN_VOIDRAY (no underscore), not TRAIN_VOID_RAY
        assertThat(ActionTranslator.mapTrainAbility(UnitType.VOID_RAY))
            .isEqualTo(Abilities.TRAIN_VOIDRAY);
    }

    @Test
    void archonTrainAbilityIsNullBecauseItRequiresTwoTemplar() {
        // ARCHON is formed by merging two High or Dark Templar — the single-tag
        // TrainIntent model cannot express a two-unit merge. Log warn and skip.
        assertThat(ActionTranslator.mapTrainAbility(UnitType.ARCHON)).isNull();
    }

    @Test
    void enemyAndUnknownUnitTypesHaveNullTrainAbility() {
        // Zerg and Terran types exist in UnitType for scouting recognition only.
        // They are not trainable by a Protoss player.
        assertThat(ActionTranslator.mapTrainAbility(UnitType.ZERGLING)).isNull();
        assertThat(ActionTranslator.mapTrainAbility(UnitType.ROACH)).isNull();
        assertThat(ActionTranslator.mapTrainAbility(UnitType.HYDRALISK)).isNull();
        assertThat(ActionTranslator.mapTrainAbility(UnitType.MARINE)).isNull();
        assertThat(ActionTranslator.mapTrainAbility(UnitType.MARAUDER)).isNull();
        assertThat(ActionTranslator.mapTrainAbility(UnitType.MEDIVAC)).isNull();
        assertThat(ActionTranslator.mapTrainAbility(UnitType.UNKNOWN)).isNull();
    }
```

- [ ] **Step 2: Run tests — verify new ones fail**

```bash
mvn test -Dtest=ActionTranslatorTest -q
```

Expected: FAIL — `UnsupportedOperationException: not yet implemented` (mapTrainAbility)

- [ ] **Step 3: Implement mapTrainAbility**

Replace the stub `mapTrainAbility` body in `ActionTranslator.java`:

```java
static Abilities mapTrainAbility(UnitType type) {
    return switch (type) {
        case PROBE        -> Abilities.TRAIN_PROBE;
        case ZEALOT       -> Abilities.TRAIN_ZEALOT;
        case STALKER      -> Abilities.TRAIN_STALKER;
        case IMMORTAL     -> Abilities.TRAIN_IMMORTAL;
        case COLOSSUS     -> Abilities.TRAIN_COLOSSUS;
        case CARRIER      -> Abilities.TRAIN_CARRIER;
        case DARK_TEMPLAR -> Abilities.TRAIN_DARK_TEMPLAR;
        case HIGH_TEMPLAR -> Abilities.TRAIN_HIGH_TEMPLAR;
        case OBSERVER     -> Abilities.TRAIN_OBSERVER;
        case VOID_RAY     -> Abilities.TRAIN_VOIDRAY; // ocraft: no underscore
        // Archon requires two Templar — single-tag TrainIntent cannot express this
        case ARCHON       -> null;
        // Zerg and Terran types exist for scouting recognition, not Protoss training
        case ZERGLING, ROACH, HYDRALISK,
             MARINE, MARAUDER, MEDIVAC,
             UNKNOWN      -> null;
    };
}
```

- [ ] **Step 4: Run tests — verify all 5 pass**

```bash
mvn test -Dtest=ActionTranslatorTest -q
```

Expected: BUILD SUCCESS — 5 tests pass

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/real/ActionTranslator.java \
        src/test/java/io/quarkmind/sc2/real/ActionTranslatorTest.java
git commit -m "feat(sc2): ActionTranslator — mapTrainAbility covers full UnitType roster

Refs #12"
```

---

## Task 4: translate() happy paths — tests then implementation

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/real/ActionTranslatorTest.java` (add Group 2 dispatch tests)
- Modify: `src/main/java/io/quarkmind/sc2/real/ActionTranslator.java` (implement translate())

- [ ] **Step 1: Add dispatch tests (Group 2) to ActionTranslatorTest**

Add a `// ─── Group 2: Intent dispatch` section after the existing tests:

```java
    // ─── Group 2: Intent dispatch ─────────────────────────────────────────────

    @Test
    void buildIntentProducesPositionTargetedCommandWithCorrectTagAbilityAndLocation() {
        var intent = new BuildIntent("456", BuildingType.GATEWAY,
                                    new io.quarkmind.domain.Point2d(30f, 40f));

        var commands = ActionTranslator.translate(List.of(intent));

        assertThat(commands).hasSize(1);
        var cmd = commands.get(0);
        assertThat(cmd.tag()).isEqualTo(Tag.of(456L));
        assertThat(cmd.ability()).isEqualTo(Abilities.BUILD_GATEWAY);
        assertThat(cmd.target()).contains(Point2d.of(30f, 40f));
    }

    @Test
    void trainIntentProducesCommandWithCorrectTagAndAbilityAndNoPosition() {
        var intent = new TrainIntent("789", UnitType.PROBE);

        var commands = ActionTranslator.translate(List.of(intent));

        assertThat(commands).hasSize(1);
        var cmd = commands.get(0);
        assertThat(cmd.tag()).isEqualTo(Tag.of(789L));
        assertThat(cmd.ability()).isEqualTo(Abilities.TRAIN_PROBE);
        assertThat(cmd.target()).isEmpty();
    }

    @Test
    void attackIntentProducesAttackCommandAtTargetLocation() {
        var intent = new AttackIntent("111", new io.quarkmind.domain.Point2d(50f, 60f));

        var commands = ActionTranslator.translate(List.of(intent));

        assertThat(commands).hasSize(1);
        var cmd = commands.get(0);
        assertThat(cmd.tag()).isEqualTo(Tag.of(111L));
        assertThat(cmd.ability()).isEqualTo(Abilities.ATTACK);
        assertThat(cmd.target()).contains(Point2d.of(50f, 60f));
    }

    @Test
    void moveIntentProducesMoveCommandAtTargetLocation() {
        var intent = new MoveIntent("222", new io.quarkmind.domain.Point2d(10f, 20f));

        var commands = ActionTranslator.translate(List.of(intent));

        assertThat(commands).hasSize(1);
        var cmd = commands.get(0);
        assertThat(cmd.tag()).isEqualTo(Tag.of(222L));
        assertThat(cmd.ability()).isEqualTo(Abilities.MOVE);
        assertThat(cmd.target()).contains(Point2d.of(10f, 20f));
    }
```

- [ ] **Step 2: Run tests — verify new ones fail**

```bash
mvn test -Dtest=ActionTranslatorTest -q
```

Expected: FAIL — `UnsupportedOperationException: not yet implemented` (translate)

- [ ] **Step 3: Implement translate() and private helpers**

Replace the stub `translate()` method and add private helpers in `ActionTranslator.java`. The full updated class body (keeping the existing map methods unchanged):

```java
    public static List<ResolvedCommand> translate(List<Intent> intents) {
        List<ResolvedCommand> commands = new ArrayList<>();
        for (Intent intent : intents) {
            try {
                ResolvedCommand cmd = switch (intent) {
                    case BuildIntent  b -> build(b);
                    case TrainIntent  t -> train(t);
                    case AttackIntent a -> attack(a);
                    case MoveIntent   m -> move(m);
                    default -> {
                        log.warnf("[SC2] Unknown intent type: %s — skipped",
                                  intent.getClass().getSimpleName());
                        yield null;
                    }
                };
                if (cmd != null) commands.add(cmd);
            } catch (Exception e) {
                log.warnf("[SC2] Failed to translate intent %s: %s", intent, e.getMessage());
            }
        }
        return commands;
    }

    private static ResolvedCommand build(BuildIntent intent) {
        Abilities ability = mapBuildAbility(intent.buildingType());
        if (ability == null) {
            log.warnf("[SC2] No build ability for BuildingType.%s — intent skipped",
                      intent.buildingType());
            return null;
        }
        return new ResolvedCommand(
            toTag(intent.unitTag()),
            ability,
            Optional.of(toOcraft(intent.location()))
        );
    }

    private static ResolvedCommand train(TrainIntent intent) {
        Abilities ability = mapTrainAbility(intent.unitType());
        if (ability == null) {
            log.warnf("[SC2] No train ability for UnitType.%s — intent skipped",
                      intent.unitType());
            return null;
        }
        return new ResolvedCommand(
            toTag(intent.unitTag()),
            ability,
            Optional.empty()
        );
    }

    private static ResolvedCommand attack(AttackIntent intent) {
        return new ResolvedCommand(
            toTag(intent.unitTag()),
            Abilities.ATTACK,
            Optional.of(toOcraft(intent.targetLocation()))
        );
    }

    private static ResolvedCommand move(MoveIntent intent) {
        return new ResolvedCommand(
            toTag(intent.unitTag()),
            Abilities.MOVE,
            Optional.of(toOcraft(intent.targetLocation()))
        );
    }

    private static Tag toTag(String tagStr) {
        return Tag.of(Long.parseLong(tagStr));
    }

    private static com.github.ocraft.s2client.protocol.spatial.Point2d toOcraft(
            io.quarkmind.domain.Point2d p) {
        return com.github.ocraft.s2client.protocol.spatial.Point2d.of(p.x(), p.y());
    }
```

Also add the missing `ArrayList` import at the top of `ActionTranslator.java`:

```java
import java.util.ArrayList;
```

And the intent type imports (add after existing intent import if missing):

```java
import io.quarkmind.sc2.intent.AttackIntent;
import io.quarkmind.sc2.intent.BuildIntent;
import io.quarkmind.sc2.intent.MoveIntent;
import io.quarkmind.sc2.intent.TrainIntent;
```

- [ ] **Step 4: Run tests — verify all 9 pass**

```bash
mvn test -Dtest=ActionTranslatorTest -q
```

Expected: BUILD SUCCESS — 9 tests pass

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/real/ActionTranslator.java \
        src/test/java/io/quarkmind/sc2/real/ActionTranslatorTest.java
git commit -m "feat(sc2): ActionTranslator — translate() dispatches all four intent types

Refs #12"
```

---

## Task 5: Error and edge cases — tests then implementation

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/real/ActionTranslatorTest.java` (add Group 3 edge case tests)

The error handling paths (null ability, malformed tag, exception isolation) are already wired in from Task 4's implementation. These tests verify the contracts are upheld.

- [ ] **Step 1: Add error/edge case tests (Group 3) to ActionTranslatorTest**

Add a `// ─── Group 3: Error and edge cases` section after Group 2:

```java
    // ─── Group 3: Error and edge cases ───────────────────────────────────────

    @Test
    void archonTrainIntentProducesNoCommandAndDoesNotThrow() {
        // ARCHON merge is not expressible as a single-tag TrainIntent.
        // The translator logs a warning and returns an empty list — no crash.
        var intent = new TrainIntent("333", UnitType.ARCHON);

        var commands = ActionTranslator.translate(List.of(intent));

        assertThat(commands).isEmpty();
    }

    @Test
    void unknownBuildingTypeProducesNoCommand() {
        var intent = new BuildIntent("444", BuildingType.UNKNOWN,
                                    new io.quarkmind.domain.Point2d(0f, 0f));

        var commands = ActionTranslator.translate(List.of(intent));

        assertThat(commands).isEmpty();
    }

    @Test
    void malformedTagProducesNoCommandAndDoesNotThrow() {
        // Tags must be numeric (Long). A non-numeric tag is logged and skipped —
        // one bad intent must not abort the rest of the frame's dispatch.
        var intent = new BuildIntent("not-a-number", BuildingType.PYLON,
                                     new io.quarkmind.domain.Point2d(0f, 0f));

        var commands = ActionTranslator.translate(List.of(intent));

        assertThat(commands).isEmpty();
    }

    @Test
    void validAndInvalidIntentsMixed_onlyValidOnesProduceCommands() {
        var valid   = new BuildIntent("100", BuildingType.PYLON,
                                     new io.quarkmind.domain.Point2d(15f, 25f));
        var archon  = new TrainIntent("200", UnitType.ARCHON);         // null ability
        var badTag  = new MoveIntent("oops", new io.quarkmind.domain.Point2d(0f, 0f)); // bad tag

        var commands = ActionTranslator.translate(List.of(valid, archon, badTag));

        assertThat(commands).hasSize(1);
        assertThat(commands.get(0).ability()).isEqualTo(Abilities.BUILD_PYLON);
        assertThat(commands.get(0).tag()).isEqualTo(Tag.of(100L));
    }

    @Test
    void emptyIntentListProducesEmptyCommandList() {
        assertThat(ActionTranslator.translate(List.of())).isEmpty();
    }
```

- [ ] **Step 2: Run tests — verify all 14 pass**

```bash
mvn test -Dtest=ActionTranslatorTest -q
```

Expected: BUILD SUCCESS — 14 tests pass. The error handling was already implemented in Task 4; these tests confirm the contracts.

If any test fails, the likely cause is the `try-catch` in `translate()` not catching `NumberFormatException` from `toTag()`. Verify the private `build()` / `train()` / `move()` methods call `toTag()` inside the per-intent try-catch block (they do, as written in Task 4).

- [ ] **Step 3: Commit**

```bash
git add src/test/java/io/quarkmind/sc2/real/ActionTranslatorTest.java
git commit -m "test(sc2): ActionTranslatorTest — error and edge cases

Refs #12"
```

---

## Task 6: Wire into SC2BotAgent — integration and full regression

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/real/SC2BotAgent.java`

Replace the no-op intent drain with `ActionTranslator.translate()`. This is the final wiring step.

- [ ] **Step 1: Read the current SC2BotAgent.onStep()**

Open `src/main/java/io/quarkmind/sc2/real/SC2BotAgent.java` and locate the intent drain block (around line 53–55):

```java
        // 2. Drain IntentQueue — commands MUST be sent within onStep().
        //    Phase 1: dummy plugins produce no intents — queue is always empty.
        //    Phase 3+: translate Intent types to ocraft actions here.
        intentQueue.drainAll().forEach(intent ->
            log.debugf("[SC2] Intent (Phase 1 no-op): %s", intent));
```

- [ ] **Step 2: Replace the no-op drain with ActionTranslator**

Replace those lines with:

```java
        // 2. Drain IntentQueue and dispatch commands.
        //    Commands MUST be issued within onStep() — ocraft enforces this.
        //    ActionTranslator resolves each Intent to a Tag + Ability + optional Point2d.
        List<ResolvedCommand> commands = ActionTranslator.translate(intentQueue.drainAll());
        commands.forEach(cmd ->
            cmd.target().ifPresentOrElse(
                pos -> actions().unitCommand(cmd.tag(), cmd.ability(), pos, false),
                ()  -> actions().unitCommand(cmd.tag(), cmd.ability(), false)
            )
        );
```

Also add the missing import at the top of `SC2BotAgent.java`:

```java
import java.util.List;
```

- [ ] **Step 3: Compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS

- [ ] **Step 4: Run full test suite**

```bash
mvn test -q
```

Expected: BUILD SUCCESS — all 173+ tests pass. `ActionTranslatorTest` contributes 14 new tests. The `@IfBuildProfile("sc2")` annotation on `RealSC2Engine` and `SC2BotAgent` means they are excluded from unit and `@QuarkusTest` runs — no test touches `SC2BotAgent.onStep()` directly.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/real/SC2BotAgent.java
git commit -m "feat(sc2): wire ActionTranslator into SC2BotAgent.onStep() — intents now dispatch

Closes #12"
```
