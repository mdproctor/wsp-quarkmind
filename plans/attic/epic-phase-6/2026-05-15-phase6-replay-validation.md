# Phase 6 — Replay-Accurate Forward Simulation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Validate `EmulatedGame` economic accuracy against real SC2 replay data by feeding the human player's commands from GAME_EVENTS into the emulated engine and comparing state tick-by-tick to the replay ground truth.

**Architecture:** `GameEventStream` is refactored to a thin MPQ reader (`events(Path) → List<Event>`). A new stateful `AbilityMapping` owns per-player selection state and maps `CmdEvent` ability IDs to `Intent` objects. `ReplayCommandExtractor` orchestrates both into a `ReplayCommandStream`. `ReplayValidationHarness` runs `EmulatedGame` and `ReplaySimulatedGame` in parallel, producing a `DivergenceReport` consumed by both a regression JUnit test and a human-readable report test.

**Tech Stack:** Java 21, Quarkus, Scelight (`RepParserEngine`, `CmdEvent`, `SelectionDeltaEvent`), JUnit 5 + AssertJ, Maven

**Spec correction (supersedes design doc):** `AbilityMapping.process()` returns `List<ReplayCommand>` (not `Optional`) — a movement command targets all selected units simultaneously, requiring one `UnitOrder` per selected tag.

**Issues tracked:** #137 (this epic), #138 (Terran/Zerg EmulatedGame), #140 (Terran replay acquisition for ability discovery)

---

## File Map

| Action | File |
|--------|------|
| Modify | `src/main/java/io/quarkmind/sc2/replay/GameEventStream.java` |
| Modify | `src/main/java/io/quarkmind/sc2/replay/ReplayEngine.java` |
| Modify | `src/test/java/io/quarkmind/sc2/replay/GameEventStreamTest.java` |
| Modify | `pom.xml` |
| Create | `src/main/java/io/quarkmind/sc2/replay/TimedIntent.java` |
| Create | `src/main/java/io/quarkmind/sc2/replay/ReplayCommand.java` |
| Create | `src/main/java/io/quarkmind/sc2/replay/ReplayCommandStream.java` |
| Create | `src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java` |
| Create | `src/main/java/io/quarkmind/sc2/replay/ReplayCommandExtractor.java` |
| Create | `src/main/java/io/quarkmind/sc2/replay/DivergenceReport.java` |
| Create | `src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java` |
| Create | `src/test/java/io/quarkmind/sc2/replay/AbilityDiscoveryTest.java` |
| Create | `src/test/java/io/quarkmind/sc2/replay/AbilityMappingTest.java` |
| Create | `src/test/java/io/quarkmind/sc2/replay/ReplayCommandExtractorTest.java` |
| Create | `src/test/java/io/quarkmind/sc2/replay/ReplayValidationTest.java` |
| Create | `src/test/java/io/quarkmind/sc2/replay/ReplayValidationReportTest.java` |

---

## Task 1: Refactor `GameEventStream` and update call sites

`parse(Path) → List<UnitOrder>` is replaced by `events(Path) → List<Event>`. All extraction logic moves out. `ReplayEngine` is updated atomically — these three files change together.

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/replay/GameEventStream.java`
- Modify: `src/main/java/io/quarkmind/sc2/replay/ReplayEngine.java`
- Modify: `src/test/java/io/quarkmind/sc2/replay/GameEventStreamTest.java`

- [ ] **Replace `GameEventStream.java` entirely:**

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.factory.RepContent;
import hu.scelight.sc2.rep.factory.RepParserEngine;
import hu.scelight.sc2.rep.model.Replay;
import hu.scelight.sc2.rep.s2prot.Event;

import java.nio.file.Path;
import java.util.Arrays;
import java.util.EnumSet;
import java.util.List;

public final class GameEventStream {

    private GameEventStream() {}

    /**
     * Parses GAME_EVENTS from a replay file and returns the raw event list.
     * No filtering or extraction — callers decide what to do with events.
     */
    public static List<Event> events(Path replayPath) {
        Replay replay;
        try {
            replay = RepParserEngine.parseReplay(replayPath, EnumSet.of(RepContent.GAME_EVENTS));
        } catch (Exception e) {
            throw new IllegalArgumentException("Cannot parse GAME_EVENTS from: " + replayPath, e);
        }
        if (replay == null || replay.gameEvents == null) {
            throw new IllegalArgumentException("No game events in replay: " + replayPath);
        }
        return Arrays.asList(replay.gameEvents.getEvents());
    }

    /** Decode raw SC2 unit tag integer to "r-{index}-{recycle}" format. */
    static String decodeTag(int rawTag) {
        return "r-" + (rawTag >> 18) + "-" + (rawTag & 0x3FFFF);
    }
}
```

- [ ] **Replace `GameEventStreamTest.java` entirely:**

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.model.gameevents.cmd.CmdEvent;
import hu.scelight.sc2.rep.model.gameevents.selectiondelta.SelectionDeltaEvent;
import hu.scelight.sc2.rep.s2prot.Event;
import org.junit.jupiter.api.Test;

import java.nio.file.Path;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class GameEventStreamTest {

    static final Path REPLAY =
        Path.of("replays/aiarena_protoss/Nothing_4720936.SC2Replay");

    @Test
    void returnsNonEmptyEventList() {
        List<Event> events = GameEventStream.events(REPLAY);
        assertThat(events).isNotEmpty();
    }

    @Test
    void containsCmdEvents() {
        List<Event> events = GameEventStream.events(REPLAY);
        assertThat(events).anyMatch(e -> e instanceof CmdEvent);
    }

    @Test
    void containsSelectionDeltaEvents() {
        List<Event> events = GameEventStream.events(REPLAY);
        assertThat(events).anyMatch(e -> e instanceof SelectionDeltaEvent);
    }

    @Test
    void allEventsHaveNonNegativeLoops() {
        List<Event> events = GameEventStream.events(REPLAY);
        assertThat(events).allMatch(e -> e.getLoop() >= 0);
    }

    @Test
    void throwsForMissingFile() {
        assertThatThrownBy(() -> GameEventStream.events(Path.of("nonexistent.SC2Replay")))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Update `ReplayEngine.connect()`** — replace the `GameEventStream.parse()` call with a forward reference to `ReplayCommandExtractor` (which doesn't exist yet — the code won't compile until Task 5, so add a `// TODO Task 5` placeholder that keeps the intent clear):

```java
@Override
public void connect() {
    log.infof("[REPLAY] Loading replay: %s (player %d)", replayFile, watchedPlayerId);
    game = new ReplaySimulatedGame(Path.of(replayFile), watchedPlayerId);
    parseMapMetadata(Path.of(replayFile));
    // TODO Task 5: replace with ReplayCommandExtractor.extract(...)
    // game.loadOrders(commands.movementOrders());
    connected = true;
    log.infof("[REPLAY] Replay loaded — %d tracker events ready, map=%s (%dx%d)",
            game.eventCount(), mapName, mapWidth, mapHeight);
}
```

Note: `ReplayEngine` temporarily loses movement order loading. This is restored in Task 5. The replay profile still works — it just won't load movement orders until then.

- [ ] **Verify the old `GameEventStreamTest` tests pass and compilation succeeds:**

```bash
mvn test -Dtest=GameEventStreamTest -q
```

Expected: all 5 tests pass.

- [ ] **Commit:**

```bash
git add src/main/java/io/quarkmind/sc2/replay/GameEventStream.java \
        src/main/java/io/quarkmind/sc2/replay/ReplayEngine.java \
        src/test/java/io/quarkmind/sc2/replay/GameEventStreamTest.java
git commit -m "refactor(replay): replace GameEventStream.parse() with thin events() reader

Extraction logic moves to ReplayCommandExtractor (Task 5).
ReplayEngine movement loading temporarily disabled pending Task 5.

Refs #137"
```

---

## Task 2: Write and run `AbilityDiscoveryTest` — record ability IDs

This test has no assertions — it scans replays and prints all observed ability ID tuples. Run it, paste the output into a comment in `AbilityMapping.java` (Task 4). It stays in the suite permanently as a coverage instrument.

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/replay/AbilityDiscoveryTest.java`

- [ ] **Create `AbilityDiscoveryTest.java`:**

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.model.gameevents.cmd.CmdEvent;
import hu.scelight.sc2.rep.model.gameevents.selectiondelta.Delta;
import hu.scelight.sc2.rep.model.gameevents.selectiondelta.SelectionDeltaEvent;
import hu.scelight.sc2.rep.s2prot.Event;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.MethodSource;

import java.nio.file.Path;
import java.util.*;
import java.util.stream.Stream;

/**
 * Scans replays and prints all observed (abilLink, abilCmdIndex) tuples with counts.
 * No assertions — output is used to populate AbilityMapping's static table.
 * Covers Protoss (userId=0) and Zerg (userId=1) from PvZ aiarena replays.
 * Terran coverage requires .SC2Replay files — see issue #140.
 */
class AbilityDiscoveryTest {

    @ParameterizedTest(name = "{0}")
    @MethodSource("replayFiles")
    void discoverAbilityIds(Path replayPath) {
        List<Event> events = GameEventStream.events(replayPath);
        Map<Integer, List<String>> selections = new HashMap<>();
        Map<String, Long> counts = new TreeMap<>();

        for (Event raw : events) {
            if (raw instanceof SelectionDeltaEvent sel) {
                Delta delta = sel.getDelta();
                if (delta == null || delta.getAddUnitTags() == null) continue;
                List<String> tags = Arrays.stream(delta.getAddUnitTags())
                    .filter(Objects::nonNull)
                    .map(GameEventStream::decodeTag)
                    .toList();
                selections.put(sel.getUserId(), tags);
            } else if (raw instanceof CmdEvent cmd) {
                Integer abilLink = cmd.getAbilLink();
                if (abilLink == null) continue;
                int selSize = selections.getOrDefault(cmd.getUserId(), List.of()).size();
                String key = String.format(
                    "userId=%d  abilLink=%5d  abilCmdIndex=%d  selSize=%2d  hasTP=%-5s  hasTU=%-5s",
                    cmd.getUserId(), abilLink,
                    Objects.requireNonNullElse(cmd.getAbilCmdIndex(), 0),
                    selSize,
                    cmd.getTargetPoint() != null,
                    cmd.getTargetUnit() != null);
                counts.merge(key, 1L, Long::sum);
            }
        }

        System.out.println("\n=== Ability IDs: " + replayPath.getFileName() + " ===");
        counts.entrySet().stream()
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .forEach(e -> System.out.printf("  count=%5d  %s%n", e.getValue(), e.getKey()));
        System.out.println();
    }

    static Stream<Path> replayFiles() {
        return Stream.of(
            // PvZ — userId 0 = Protoss (player 1), userId 1 = Zerg (player 2)
            Path.of("replays/aiarena_protoss/Nothing_4720936.SC2Replay"),
            Path.of("replays/aiarena_protoss/ArgoBot_4721222.SC2Replay")
        );
    }
}
```

- [ ] **Run the discovery test and capture all output:**

```bash
mvn test -Dtest=AbilityDiscoveryTest -q 2>/dev/null | grep -v "^\[" | head -200
```

Expected: printed tables of `abilLink` values per replay. You will see entries like:
```
  count=  312  userId=0  abilLink=   16  abilCmdIndex=0  selSize= 1  hasTP=true   hasTU=false
  count=   12  userId=0  abilLink= 1006  abilCmdIndex=0  selSize= 1  hasTP=false  hasTU=false
```

- [ ] **From the output, identify and record the following ability IDs** (userId=0 is Protoss, userId=1 is Zerg). You need these for Tasks 4 and 5:

```
# Paste the relevant lines here as a reference — you'll hardcode these into AbilityMapping

MOVE abilLink        = <value from output: hasTP=true, high count, selSize varies>
PROTOSS TRAIN:
  Probe  abilLink    = <value: userId=0, selSize=1, hasTU=false, hasTP=false>
  Zealot abilLink    = <value>
  Stalker abilLink   = <value>
  Immortal abilLink  = <value>
  Observer abilLink  = <value>
PROTOSS BUILD:
  Pylon  abilLink    = <value: userId=0, hasTP=true>
  Gateway abilLink   = <value>
  Assimilator abilLink = <value>
  CyberneticsCore abilLink = <value>
  RoboticsFacility abilLink = <value>
ZERG TRAIN:
  Drone abilLink     = <value: userId=1>
  Zergling abilLink  = <value>
  Roach abilLink     = <value>
  etc.
```

- [ ] **Commit the discovery test (with discovered values documented in a comment at the top):**

```bash
git add src/test/java/io/quarkmind/sc2/replay/AbilityDiscoveryTest.java
git commit -m "test(replay): add AbilityDiscoveryTest for ability ID coverage

Scans PvZ replays to discover SC2 ability IDs for AbilityMapping.
Output recorded in AbilityMapping javadoc.
Terran coverage pending issue #140.

Refs #137"
```

---

## Task 3: Create core types — `TimedIntent`, `ReplayCommand`, `ReplayCommandStream`

Three small, pure data types. Write them all at once and verify compilation.

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/replay/TimedIntent.java`
- Create: `src/main/java/io/quarkmind/sc2/replay/ReplayCommand.java`
- Create: `src/main/java/io/quarkmind/sc2/replay/ReplayCommandStream.java`

- [ ] **Create `TimedIntent.java`:**

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.sc2.intent.Intent;

/** An Intent extracted from replay GAME_EVENTS, tagged with its raw SC2 game loop. */
public record TimedIntent(long loop, Intent intent) {}
```

- [ ] **Create `ReplayCommand.java`:**

```java
package io.quarkmind.sc2.replay;

/** Discriminated union of what a single CmdEvent produces. */
public sealed interface ReplayCommand permits ReplayCommand.Movement, ReplayCommand.IntentCommand {
    record Movement(UnitOrder order)          implements ReplayCommand {}
    record IntentCommand(TimedIntent intent)  implements ReplayCommand {}
}
```

- [ ] **Create `ReplayCommandStream.java`:**

```java
package io.quarkmind.sc2.replay;

import java.util.List;

/** Full extraction result from a replay's GAME_EVENTS for one player. */
public record ReplayCommandStream(
    List<UnitOrder>   movementOrders,
    List<TimedIntent> intents) {}
```

- [ ] **Verify compilation:**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS, no errors.

- [ ] **Commit:**

```bash
git add src/main/java/io/quarkmind/sc2/replay/TimedIntent.java \
        src/main/java/io/quarkmind/sc2/replay/ReplayCommand.java \
        src/main/java/io/quarkmind/sc2/replay/ReplayCommandStream.java
git commit -m "feat(replay): add TimedIntent, ReplayCommand, ReplayCommandStream

Core data types for the ability extraction pipeline.

Refs #137"
```

---

## Task 4: `AbilityMapping` — stateful command interpreter

Owns per-player selection state. Maps `CmdEvent` ability IDs (discovered in Task 2) to `ReplayCommand` entries. Movement commands produce one entry per selected unit tag. Build/train commands produce a single entry.

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/replay/AbilityMappingTest.java`
- Create: `src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java`

- [ ] **Write the failing test first.** Fill `MOVE_ABIL`, `PROBE_ABIL`, etc. from Task 2's discovered values. These are integer constants at the top of the test class.

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.model.gameevents.cmd.CmdEvent;
import hu.scelight.sc2.rep.model.gameevents.selectiondelta.SelectionDeltaEvent;
import io.quarkmind.domain.BuildingType;
import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.intent.BuildIntent;
import io.quarkmind.sc2.intent.TrainIntent;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class AbilityMappingTest {

    // Fill these from AbilityDiscoveryTest output (Task 2):
    static final int MOVE_ABIL       = /* from Task 2 */ 0;
    static final int PROBE_ABIL      = /* from Task 2 */ 0;
    static final int ZEALOT_ABIL     = /* from Task 2 */ 0;
    static final int STALKER_ABIL    = /* from Task 2 */ 0;
    static final int PYLON_ABIL      = /* from Task 2 */ 0;
    static final int GATEWAY_ABIL    = /* from Task 2 */ 0;
    static final int ASSIMILATOR_ABIL = /* from Task 2 */ 0;

    AbilityMapping mapping;

    @BeforeEach
    void setUp() {
        mapping = new AbilityMapping(1); // player 1 = userId 0
    }

    @Test
    void unknownAbilLinkReturnsEmptyList() {
        List<ReplayCommand> result = mapping.process(fakeCmdEvent(99999, 0, 100, null, null));
        assertThat(result).isEmpty();
    }

    @Test
    void noSelectionReturnsEmptyList() {
        // No prior SelectionDeltaEvent — selection is empty
        List<ReplayCommand> result = mapping.process(fakeCmdEvent(MOVE_ABIL, 0, 100, new float[]{50f, 60f}, null));
        assertThat(result).isEmpty();
    }

    @Test
    void moveCommandProducesOneOrderPerSelectedUnit() {
        primeSelection(List.of("r-1-1", "r-2-1", "r-3-1"), 0);
        List<ReplayCommand> result = mapping.process(
            fakeCmdEvent(MOVE_ABIL, 0, 200, new float[]{45f, 55f}, null));
        assertThat(result).hasSize(3);
        assertThat(result).allMatch(r -> r instanceof ReplayCommand.Movement);
        List<UnitOrder> orders = result.stream()
            .map(r -> ((ReplayCommand.Movement) r).order())
            .toList();
        assertThat(orders).allMatch(o -> o.loop() == 200);
        assertThat(orders).allMatch(o -> o.targetPos() != null
            && o.targetPos().x() == 45f && o.targetPos().y() == 55f);
        assertThat(orders.stream().map(UnitOrder::unitTag).toList())
            .containsExactlyInAnyOrder("r-1-1", "r-2-1", "r-3-1");
    }

    @Test
    void trainProbeProducesSingleTrainIntent() {
        primeSelection(List.of("r-10-1"), 0); // nexus tag
        List<ReplayCommand> result = mapping.process(
            fakeCmdEvent(PROBE_ABIL, 0, 300, null, null));
        assertThat(result).hasSize(1);
        ReplayCommand.IntentCommand ic = (ReplayCommand.IntentCommand) result.get(0);
        assertThat(ic.intent().loop()).isEqualTo(300);
        assertThat(ic.intent().intent()).isInstanceOf(TrainIntent.class);
        TrainIntent t = (TrainIntent) ic.intent().intent();
        assertThat(t.unitType()).isEqualTo(UnitType.PROBE);
        assertThat(t.buildingTag()).isEqualTo("r-10-1");
    }

    @Test
    void trainZealotProducesSingleTrainIntent() {
        primeSelection(List.of("r-20-1"), 0);
        List<ReplayCommand> result = mapping.process(
            fakeCmdEvent(ZEALOT_ABIL, 0, 400, null, null));
        assertThat(result).hasSize(1);
        TrainIntent t = (TrainIntent) ((ReplayCommand.IntentCommand) result.get(0)).intent().intent();
        assertThat(t.unitType()).isEqualTo(UnitType.ZEALOT);
    }

    @Test
    void trainStalkerProducesSingleTrainIntent() {
        primeSelection(List.of("r-21-1"), 0);
        List<ReplayCommand> result = mapping.process(
            fakeCmdEvent(STALKER_ABIL, 0, 500, null, null));
        TrainIntent t = (TrainIntent) ((ReplayCommand.IntentCommand) result.get(0)).intent().intent();
        assertThat(t.unitType()).isEqualTo(UnitType.STALKER);
    }

    @Test
    void buildPylonProducesBuildIntent() {
        primeSelection(List.of("r-5-1"), 0); // probe tag
        List<ReplayCommand> result = mapping.process(
            fakeCmdEvent(PYLON_ABIL, 0, 600, new float[]{30f, 40f}, null));
        assertThat(result).hasSize(1);
        BuildIntent b = (BuildIntent) ((ReplayCommand.IntentCommand) result.get(0)).intent().intent();
        assertThat(b.buildingType()).isEqualTo(BuildingType.PYLON);
        assertThat(b.location()).isEqualTo(new Point2d(30f, 40f));
        assertThat(b.unitTag()).isEqualTo("r-5-1");
    }

    @Test
    void resetClearsSelection() {
        primeSelection(List.of("r-1-1"), 0);
        mapping.reset();
        List<ReplayCommand> result = mapping.process(
            fakeCmdEvent(MOVE_ABIL, 0, 100, new float[]{10f, 10f}, null));
        assertThat(result).isEmpty();
    }

    @Test
    void otherPlayersCommandsIgnored() {
        // userId 1 = player 2; mapping is for player 1 (userId 0)
        primeSelection(List.of("r-1-1"), 1); // player 2 selection
        List<ReplayCommand> result = mapping.process(
            fakeCmdEvent(MOVE_ABIL, 1, 100, new float[]{10f, 10f}, null)); // userId=1
        assertThat(result).isEmpty();
    }

    // --- Helpers ---

    private void primeSelection(List<String> tags, int userId) {
        // SelectionDeltaEvent is a concrete class — construct via raw event map approach
        // Use a real replay parse instead of mocking: prime via the real event objects
        // obtained from GameEventStream.events() in integration tests.
        // For unit tests, inject selection directly using package-private helper:
        mapping.setSelectionForTest(userId, tags);
    }

    /** Minimal fake CmdEvent using raw struct map. */
    private CmdEvent fakeCmdEvent(int abilLink, int userId, long loop,
                                   float[] targetPoint, Integer targetUnitRawTag) {
        java.util.Map<String, Object> struct = new java.util.HashMap<>();
        java.util.Map<String, Object> abil = new java.util.HashMap<>();
        abil.put("abilLink", abilLink);
        abil.put("abilCmdIndex", 0);
        struct.put("abil", abil);
        struct.put("cmdFlags", 0);
        if (targetPoint != null) {
            java.util.Map<String, Object> tp = new java.util.HashMap<>();
            tp.put("x", (int)(targetPoint[0] * (1 << 16)));
            tp.put("y", (int)(targetPoint[1] * (1 << 16)));
            tp.put("z", 0);
            hu.sllauncher.util.Pair<String, Object> data = new hu.sllauncher.util.Pair<>("TargetPoint", tp);
            struct.put("data", data);
        } else if (targetUnitRawTag != null) {
            java.util.Map<String, Object> tu = new java.util.HashMap<>();
            tu.put("tag", targetUnitRawTag);
            tu.put("targetUnitFlags", 0);
            tu.put("timer", 0);
            tu.put("snapshotUnitLink", 0);
            tu.put("snapshotPlayerId", 0);
            hu.sllauncher.util.Pair<String, Object> data = new hu.sllauncher.util.Pair<>("TargetUnit", tu);
            struct.put("data", data);
        }
        return new CmdEvent(struct, 27, "BasicCommandEvent", (int) loop, userId, 99999, null);
    }
}
```

**Note on `fakeCmdEvent`:** If `CmdEvent` construction proves brittle (the Scelight constructor is complex), use a real parsed event from the replay as a fixture instead. See the approach in `ReplayCommandExtractorTest` (Task 5) which uses a real replay file.

- [ ] **Run the test to confirm it fails (class not found):**

```bash
mvn test -Dtest=AbilityMappingTest -q 2>&1 | tail -5
```

Expected: compilation error — `AbilityMapping` does not exist.

- [ ] **Create `AbilityMapping.java`.** Fill ability link constants from Task 2's discovery output. The `/* FILL: ... */` markers below are data-dependency annotations, not design deferrals — replace each one with the integer value from Task 2's printed output.

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.model.gameevents.cmd.CmdEvent;
import hu.scelight.sc2.rep.model.gameevents.selectiondelta.Delta;
import hu.scelight.sc2.rep.model.gameevents.selectiondelta.SelectionDeltaEvent;
import io.quarkmind.domain.BuildingType;
import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.intent.BuildIntent;
import io.quarkmind.sc2.intent.TrainIntent;
import org.jboss.logging.Logger;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Objects;

/**
 * Stateful SC2 command interpreter scoped to one player.
 * Tracks the player's current selection across SelectionDeltaEvents and maps
 * each CmdEvent to zero or more ReplayCommands based on abilLink.
 *
 * Ability IDs discovered via AbilityDiscoveryTest from aiarena_protoss PvZ replays.
 * Protoss = userId 0 (player 1), Zerg = userId 1 (player 2).
 * Terran coverage deferred pending issue #140.
 *
 * Discovered ability ID table:
 * [paste AbilityDiscoveryTest output here after Task 2]
 */
public class AbilityMapping {

    private static final Logger log = Logger.getLogger(AbilityMapping.class);

    // --- Ability link constants (fill from AbilityDiscoveryTest output) ---

    // Movement (both races share the same move abilLink)
    private static final int ABIL_MOVE             = /* FILL: from Task 2, high count, hasTP=true */ 0;

    // Protoss train
    private static final int ABIL_TRAIN_PROBE      = /* FILL: from Task 2, userId=0, hasTP=false */ 0;
    private static final int ABIL_TRAIN_ZEALOT      = /* FILL */ 0;
    private static final int ABIL_TRAIN_STALKER     = /* FILL */ 0;
    private static final int ABIL_TRAIN_IMMORTAL    = /* FILL */ 0;
    private static final int ABIL_TRAIN_OBSERVER    = /* FILL */ 0;
    private static final int ABIL_TRAIN_COLOSSUS    = /* FILL */ 0;
    private static final int ABIL_TRAIN_ADEPT       = /* FILL */ 0;
    private static final int ABIL_TRAIN_DISRUPTOR   = /* FILL */ 0;
    private static final int ABIL_TRAIN_PHOENIX     = /* FILL */ 0;
    private static final int ABIL_TRAIN_ORACLE      = /* FILL */ 0;
    private static final int ABIL_TRAIN_VOID_RAY    = /* FILL */ 0;

    // Protoss build
    private static final int ABIL_BUILD_PYLON            = /* FILL: userId=0, hasTP=true */ 0;
    private static final int ABIL_BUILD_GATEWAY           = /* FILL */ 0;
    private static final int ABIL_BUILD_ASSIMILATOR       = /* FILL */ 0;
    private static final int ABIL_BUILD_CYBERNETICS_CORE  = /* FILL */ 0;
    private static final int ABIL_BUILD_ROBOTICS_FACILITY = /* FILL */ 0;
    private static final int ABIL_BUILD_STARGATE          = /* FILL */ 0;
    private static final int ABIL_BUILD_FORGE             = /* FILL */ 0;
    private static final int ABIL_BUILD_SHIELD_BATTERY    = /* FILL */ 0;
    private static final int ABIL_BUILD_PHOTON_CANNON     = /* FILL */ 0;
    private static final int ABIL_BUILD_TWILIGHT_COUNCIL  = /* FILL */ 0;
    private static final int ABIL_BUILD_TEMPLAR_ARCHIVES  = /* FILL */ 0;
    private static final int ABIL_BUILD_DARK_SHRINE       = /* FILL */ 0;
    private static final int ABIL_BUILD_FLEET_BEACON      = /* FILL */ 0;
    private static final int ABIL_BUILD_ROBOTICS_BAY      = /* FILL */ 0;

    // Zerg train (from userId=1 in PvZ replays)
    private static final int ABIL_TRAIN_DRONE       = /* FILL: userId=1 */ 0;
    private static final int ABIL_TRAIN_OVERLORD    = /* FILL */ 0;
    private static final int ABIL_TRAIN_ZERGLING    = /* FILL */ 0;
    private static final int ABIL_TRAIN_ROACH       = /* FILL */ 0;
    private static final int ABIL_TRAIN_HYDRALISK   = /* FILL */ 0;
    private static final int ABIL_TRAIN_MUTALISK    = /* FILL */ 0;
    private static final int ABIL_TRAIN_BANELING    = /* FILL */ 0;
    private static final int ABIL_TRAIN_ULTRALISK   = /* FILL */ 0;

    // Zerg build (morphs — probe equivalent is drone for buildings)
    private static final int ABIL_BUILD_SPAWNING_POOL     = /* FILL */ 0;
    private static final int ABIL_BUILD_HATCHERY          = /* FILL */ 0;
    private static final int ABIL_BUILD_EVOLUTION_CHAMBER = /* FILL */ 0;
    private static final int ABIL_BUILD_ROACH_WARREN      = /* FILL */ 0;
    private static final int ABIL_BUILD_HYDRALISK_DEN     = /* FILL */ 0;
    private static final int ABIL_BUILD_SPIRE             = /* FILL */ 0;
    private static final int ABIL_BUILD_EXTRACTOR         = /* FILL */ 0;

    // ---

    private final int userId; // 0-indexed: playerId - 1
    private List<String> currentSelection = List.of();

    public AbilityMapping(int playerId) {
        this.userId = playerId - 1;
    }

    public void onSelection(SelectionDeltaEvent event) {
        if (event.getUserId() != userId) return;
        Delta delta = event.getDelta();
        if (delta == null || delta.getAddUnitTags() == null) {
            currentSelection = List.of();
            return;
        }
        currentSelection = Arrays.stream(delta.getAddUnitTags())
            .filter(Objects::nonNull)
            .map(GameEventStream::decodeTag)
            .toList();
    }

    public List<ReplayCommand> process(CmdEvent event) {
        if (event.getUserId() != userId) return List.of();
        if (currentSelection.isEmpty()) return List.of();
        Integer abilLink = event.getAbilLink();
        if (abilLink == null) return List.of();
        return dispatch(abilLink, event);
    }

    public void reset() {
        currentSelection = List.of();
    }

    /** Package-private — used by AbilityMappingTest to prime selection without parsing a replay. */
    void setSelectionForTest(int forUserId, List<String> tags) {
        if (forUserId == this.userId) this.currentSelection = List.copyOf(tags);
    }

    private List<ReplayCommand> dispatch(int abilLink, CmdEvent event) {
        long loop = event.getLoop();

        if (abilLink == ABIL_MOVE) {
            return moveOrders(event, loop);
        }

        // Protoss train
        UnitType trainUnit = switch (abilLink) {
            case ABIL_TRAIN_PROBE     -> UnitType.PROBE;
            case ABIL_TRAIN_ZEALOT    -> UnitType.ZEALOT;
            case ABIL_TRAIN_STALKER   -> UnitType.STALKER;
            case ABIL_TRAIN_IMMORTAL  -> UnitType.IMMORTAL;
            case ABIL_TRAIN_OBSERVER  -> UnitType.OBSERVER;
            case ABIL_TRAIN_COLOSSUS  -> UnitType.COLOSSUS;
            case ABIL_TRAIN_ADEPT     -> UnitType.ADEPT;
            case ABIL_TRAIN_DISRUPTOR -> UnitType.DISRUPTOR;
            case ABIL_TRAIN_PHOENIX   -> UnitType.PHOENIX;
            case ABIL_TRAIN_ORACLE    -> UnitType.ORACLE;
            case ABIL_TRAIN_VOID_RAY  -> UnitType.VOID_RAY;
            // Zerg train
            case ABIL_TRAIN_DRONE     -> UnitType.DRONE;
            case ABIL_TRAIN_OVERLORD  -> UnitType.OVERLORD;
            case ABIL_TRAIN_ZERGLING  -> UnitType.ZERGLING;
            case ABIL_TRAIN_ROACH     -> UnitType.ROACH;
            case ABIL_TRAIN_HYDRALISK -> UnitType.HYDRALISK;
            case ABIL_TRAIN_MUTALISK  -> UnitType.MUTALISK;
            case ABIL_TRAIN_BANELING  -> UnitType.BANELING;
            case ABIL_TRAIN_ULTRALISK -> UnitType.ULTRALISK;
            default -> null;
        };
        if (trainUnit != null) {
            String buildingTag = currentSelection.get(0);
            return List.of(new ReplayCommand.IntentCommand(
                new TimedIntent(loop, new TrainIntent(buildingTag, trainUnit))));
        }

        // Protoss build
        BuildingType buildType = switch (abilLink) {
            case ABIL_BUILD_PYLON             -> BuildingType.PYLON;
            case ABIL_BUILD_GATEWAY           -> BuildingType.GATEWAY;
            case ABIL_BUILD_ASSIMILATOR       -> BuildingType.ASSIMILATOR;
            case ABIL_BUILD_CYBERNETICS_CORE  -> BuildingType.CYBERNETICS_CORE;
            case ABIL_BUILD_ROBOTICS_FACILITY -> BuildingType.ROBOTICS_FACILITY;
            case ABIL_BUILD_STARGATE          -> BuildingType.STARGATE;
            case ABIL_BUILD_FORGE             -> BuildingType.FORGE;
            case ABIL_BUILD_SHIELD_BATTERY    -> BuildingType.SHIELD_BATTERY;
            case ABIL_BUILD_PHOTON_CANNON     -> BuildingType.PHOTON_CANNON;
            case ABIL_BUILD_TWILIGHT_COUNCIL  -> BuildingType.TWILIGHT_COUNCIL;
            case ABIL_BUILD_TEMPLAR_ARCHIVES  -> BuildingType.TEMPLAR_ARCHIVES;
            case ABIL_BUILD_DARK_SHRINE       -> BuildingType.DARK_SHRINE;
            case ABIL_BUILD_FLEET_BEACON      -> BuildingType.FLEET_BEACON;
            case ABIL_BUILD_ROBOTICS_BAY      -> BuildingType.ROBOTICS_BAY;
            // Zerg build
            case ABIL_BUILD_SPAWNING_POOL     -> BuildingType.SPAWNING_POOL;
            case ABIL_BUILD_HATCHERY          -> BuildingType.HATCHERY;
            case ABIL_BUILD_EVOLUTION_CHAMBER -> BuildingType.EVOLUTION_CHAMBER;
            case ABIL_BUILD_ROACH_WARREN      -> BuildingType.ROACH_WARREN;
            case ABIL_BUILD_HYDRALISK_DEN     -> BuildingType.HYDRALISK_DEN;
            case ABIL_BUILD_SPIRE             -> BuildingType.SPIRE;
            case ABIL_BUILD_EXTRACTOR         -> BuildingType.EXTRACTOR;
            default -> null;
        };
        if (buildType != null) {
            var tp = event.getTargetPoint();
            if (tp == null) return List.of();
            String probeTag = currentSelection.get(0);
            Point2d loc = new Point2d(tp.getXFloat(), tp.getYFloat());
            return List.of(new ReplayCommand.IntentCommand(
                new TimedIntent(loop, new BuildIntent(probeTag, buildType, loc))));
        }

        log.debugf("[ABILITY] Unknown abilLink=%d — skipped", abilLink);
        return List.of();
    }

    private List<ReplayCommand> moveOrders(CmdEvent event, long loop) {
        var tp = event.getTargetPoint();
        var tu = event.getTargetUnit();
        List<ReplayCommand> orders = new ArrayList<>(currentSelection.size());
        for (String tag : currentSelection) {
            if (tp != null) {
                float x = tp.getXFloat(), y = tp.getYFloat();
                if (x >= 0 && x <= 256 && y >= 0 && y <= 256) {
                    orders.add(new ReplayCommand.Movement(
                        new UnitOrder(tag, loop, new Point2d(x, y), null)));
                }
            } else if (tu != null && tu.getTag() != null) {
                orders.add(new ReplayCommand.Movement(
                    new UnitOrder(tag, loop, null, GameEventStream.decodeTag(tu.getTag()))));
            }
        }
        return orders;
    }
}
```

**Note:** `BuildingType` must include Zerg building types (`SPAWNING_POOL`, `HATCHERY`, `EVOLUTION_CHAMBER`, `ROACH_WARREN`, `HYDRALISK_DEN`, `SPIRE`, `EXTRACTOR`). Check `BuildingType.java` — if any are missing, add them as part of this task.

- [ ] **Run the test:**

```bash
mvn test -Dtest=AbilityMappingTest -q
```

Expected: tests pass (once ability link constants are filled from Task 2). Tests with `abilLink=0` will trivially pass the "unknown returns empty" test until constants are filled — verify that `trainProbeProducesSingleTrainIntent` and other specific tests actually exercise non-zero constants.

- [ ] **Commit:**

```bash
git add src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java \
        src/test/java/io/quarkmind/sc2/replay/AbilityMappingTest.java
git commit -m "feat(replay): add AbilityMapping — stateful CmdEvent → Intent converter

Maps SC2 ability IDs (discovered from PvZ replays) to TrainIntent/BuildIntent.
Covers Protoss and Zerg. Terran pending #140.

Refs #137"
```

---

## Task 5: `ReplayCommandExtractor` — orchestrating facade

Combines `GameEventStream.events()` and `AbilityMapping` into a `ReplayCommandStream`. Also restores `ReplayEngine.connect()` to load movement orders.

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/replay/ReplayCommandExtractorTest.java`
- Create: `src/main/java/io/quarkmind/sc2/replay/ReplayCommandExtractor.java`
- Modify: `src/main/java/io/quarkmind/sc2/replay/ReplayEngine.java`

- [ ] **Write the failing test:**

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.s2prot.Event;
import io.quarkmind.domain.SC2Data;
import io.quarkmind.sc2.intent.BuildIntent;
import io.quarkmind.sc2.intent.TrainIntent;
import org.junit.jupiter.api.Test;

import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;

class ReplayCommandExtractorTest {

    static final Path REPLAY =
        Path.of("replays/aiarena_protoss/Nothing_4720936.SC2Replay");

    @Test
    void extractsNonEmptyIntentListForProtossPlayer() {
        ReplayCommandStream stream = ReplayCommandExtractor.extract(REPLAY, 1);
        assertThat(stream.intents()).isNotEmpty();
    }

    @Test
    void extractsNonEmptyMovementOrdersForProtossPlayer() {
        ReplayCommandStream stream = ReplayCommandExtractor.extract(REPLAY, 1);
        assertThat(stream.movementOrders()).isNotEmpty();
    }

    @Test
    void intentsAreOrderedByLoop() {
        ReplayCommandStream stream = ReplayCommandExtractor.extract(REPLAY, 1);
        var intents = stream.intents();
        for (int i = 1; i < intents.size(); i++) {
            assertThat(intents.get(i).loop())
                .isGreaterThanOrEqualTo(intents.get(i - 1).loop());
        }
    }

    @Test
    void movementOrdersAreOrderedByLoop() {
        ReplayCommandStream stream = ReplayCommandExtractor.extract(REPLAY, 1);
        var orders = stream.movementOrders();
        for (int i = 1; i < orders.size(); i++) {
            assertThat(orders.get(i).loop())
                .isGreaterThanOrEqualTo(orders.get(i - 1).loop());
        }
    }

    @Test
    void allIntentLoopsArePositive() {
        ReplayCommandStream stream = ReplayCommandExtractor.extract(REPLAY, 1);
        assertThat(stream.intents()).allMatch(ti -> ti.loop() >= 0);
    }

    @Test
    void trainIntentsReferenceKnownUnitTypes() {
        ReplayCommandStream stream = ReplayCommandExtractor.extract(REPLAY, 1);
        stream.intents().stream()
            .filter(ti -> ti.intent() instanceof TrainIntent)
            .map(ti -> (TrainIntent) ti.intent())
            .forEach(t -> assertThat(SC2Data.trainTimeInTicks(t.unitType()))
                .as("trainTimeInTicks must be defined for %s", t.unitType())
                .isGreaterThan(0));
    }

    @Test
    void buildIntentsHaveNonNullLocation() {
        ReplayCommandStream stream = ReplayCommandExtractor.extract(REPLAY, 1);
        stream.intents().stream()
            .filter(ti -> ti.intent() instanceof BuildIntent)
            .map(ti -> (BuildIntent) ti.intent())
            .forEach(b -> assertThat(b.location()).isNotNull());
    }

    @Test
    void allMovementOrderTagsMatchTrackerFormat() {
        ReplayCommandStream stream = ReplayCommandExtractor.extract(REPLAY, 1);
        assertThat(stream.movementOrders()).allMatch(o -> o.unitTag().startsWith("r-"));
    }

    @Test
    void listsAreUnmodifiable() {
        ReplayCommandStream stream = ReplayCommandExtractor.extract(REPLAY, 1);
        assertThat(stream.movementOrders()).isUnmodifiable();
        assertThat(stream.intents()).isUnmodifiable();
    }
}
```

- [ ] **Run to confirm it fails (class not found):**

```bash
mvn test -Dtest=ReplayCommandExtractorTest -q 2>&1 | tail -3
```

- [ ] **Create `ReplayCommandExtractor.java`:**

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.model.gameevents.cmd.CmdEvent;
import hu.scelight.sc2.rep.model.gameevents.selectiondelta.SelectionDeltaEvent;
import hu.scelight.sc2.rep.s2prot.Event;

import java.nio.file.Path;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public final class ReplayCommandExtractor {

    private ReplayCommandExtractor() {}

    /**
     * Parses GAME_EVENTS for the given player and returns movement orders and
     * build/train intents in loop-ascending order.
     *
     * @param replayPath  path to the .SC2Replay file
     * @param playerId    1-indexed player ID (matches starcraft.replay.player config)
     */
    public static ReplayCommandStream extract(Path replayPath, int playerId) {
        List<Event> events = GameEventStream.events(replayPath);
        AbilityMapping mapping = new AbilityMapping(playerId);
        List<UnitOrder>   orders  = new ArrayList<>();
        List<TimedIntent> intents = new ArrayList<>();

        for (Event raw : events) {
            if (raw instanceof SelectionDeltaEvent sel) {
                mapping.onSelection(sel);
            } else if (raw instanceof CmdEvent cmd) {
                for (ReplayCommand rc : mapping.process(cmd)) {
                    switch (rc) {
                        case ReplayCommand.Movement      m -> orders.add(m.order());
                        case ReplayCommand.IntentCommand i -> intents.add(i.intent());
                    }
                }
            }
        }

        return new ReplayCommandStream(
            Collections.unmodifiableList(orders),
            Collections.unmodifiableList(intents));
    }
}
```

- [ ] **Run the test:**

```bash
mvn test -Dtest=ReplayCommandExtractorTest -q
```

Expected: all tests pass.

- [ ] **Restore `ReplayEngine.connect()`** — replace the `// TODO Task 5` comment with the real call:

```java
@Override
public void connect() {
    log.infof("[REPLAY] Loading replay: %s (player %d)", replayFile, watchedPlayerId);
    game = new ReplaySimulatedGame(Path.of(replayFile), watchedPlayerId);
    parseMapMetadata(Path.of(replayFile));
    ReplayCommandStream commands = ReplayCommandExtractor.extract(Path.of(replayFile), watchedPlayerId);
    game.loadOrders(commands.movementOrders());
    log.infof("[REPLAY] Loaded %d movement orders, %d intents from GAME_EVENTS",
              commands.movementOrders().size(), commands.intents().size());
    connected = true;
    log.infof("[REPLAY] Replay loaded — %d tracker events ready, map=%s (%dx%d)",
            game.eventCount(), mapName, mapWidth, mapHeight);
}
```

- [ ] **Run the existing `ReplayEngineTest` to confirm nothing regressed:**

```bash
mvn test -Dtest=ReplayEngineTest -q
```

Expected: all tests pass.

- [ ] **Commit:**

```bash
git add src/main/java/io/quarkmind/sc2/replay/ReplayCommandExtractor.java \
        src/test/java/io/quarkmind/sc2/replay/ReplayCommandExtractorTest.java \
        src/main/java/io/quarkmind/sc2/replay/ReplayEngine.java
git commit -m "feat(replay): add ReplayCommandExtractor and restore ReplayEngine movement loading

Refs #137"
```

---

## Task 6: Maven `report` profile

Adds the `@Tag("report")` infrastructure so `ReplayValidationReportTest` (Task 9) can be excluded from default runs and activated on demand.

**Files:**
- Modify: `pom.xml`

- [ ] **Add `report` to the default `excludedGroups`** (line ~204, currently `benchmark,browser`):

Find:
```xml
<excludedGroups>benchmark,browser</excludedGroups>
```

Replace with:
```xml
<excludedGroups>benchmark,browser,report</excludedGroups>
```

- [ ] **Add the `report` profile** after the `benchmark` profile block:

```xml
<profile>
    <id>report</id>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <groups>report</groups>
                    <excludedGroups/>
                </configuration>
            </plugin>
        </plugins>
    </build>
</profile>
```

- [ ] **Verify default run still excludes `report` tag:**

```bash
mvn test -q 2>&1 | grep -i "report" | head -5
```

Expected: no `ReplayValidationReportTest` in output (it doesn't exist yet, but the tag exclusion is now in place).

- [ ] **Commit:**

```bash
git add pom.xml
git commit -m "build: add report Maven profile for @Tag(\"report\") tests

Refs #137"
```

---

## Task 7: `DivergenceReport`

Pure data record with a human-readable render method.

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/replay/DivergenceReport.java`

- [ ] **Create `DivergenceReport.java`:**

```java
package io.quarkmind.sc2.replay;

import java.util.List;

public record DivergenceReport(List<TickSnapshot> ticks, Summary summary) {

    public record TickSnapshot(
        int tick,
        int groundTruthUnits,     int emulatedUnits,
        int groundTruthBuildings, int emulatedBuildings,
        int groundTruthMinerals,  int emulatedMinerals,
        int groundTruthVespene,   int emulatedVespene) {

        public int unitDelta()     { return Math.abs(emulatedUnits     - groundTruthUnits); }
        public int buildingDelta() { return Math.abs(emulatedBuildings - groundTruthBuildings); }
        public int mineralDelta()  { return Math.abs(emulatedMinerals  - groundTruthMinerals); }
        public int vespeneDelta()  { return Math.abs(emulatedVespene   - groundTruthVespene); }
        public boolean hasUnitDivergence()     { return emulatedUnits     != groundTruthUnits; }
        public boolean hasBuildingDivergence() { return emulatedBuildings != groundTruthBuildings; }
    }

    public record Summary(
        int firstUnitDivergenceTick,
        int firstBuildingDivergenceTick,
        int maxMineralDelta,
        int maxVespeneDelta,
        boolean economicallyAccurate) {}

    public static DivergenceReport from(List<TickSnapshot> ticks) {
        int firstUnit     = -1;
        int firstBuilding = -1;
        int maxMineral    = 0;
        int maxVespene    = 0;

        for (TickSnapshot t : ticks) {
            if (firstUnit     == -1 && t.hasUnitDivergence())     firstUnit     = t.tick();
            if (firstBuilding == -1 && t.hasBuildingDivergence()) firstBuilding = t.tick();
            maxMineral = Math.max(maxMineral, t.mineralDelta());
            maxVespene = Math.max(maxVespene, t.vespeneDelta());
        }

        boolean accurate = (firstUnit == -1) && (firstBuilding == -1);
        return new DivergenceReport(ticks, new Summary(firstUnit, firstBuilding, maxMineral, maxVespene, accurate));
    }

    public String renderReport() {
        var sb = new StringBuilder();
        sb.append(String.format("=== Divergence Report ===%n"));
        sb.append(String.format("Ticks: %d  |  Economically accurate: %s%n",
            ticks.size(), summary.economicallyAccurate() ? "YES" : "NO"));
        sb.append(String.format("First unit divergence:    tick %s%n",
            summary.firstUnitDivergenceTick()     == -1 ? "none" : summary.firstUnitDivergenceTick()));
        sb.append(String.format("First building divergence: tick %s%n",
            summary.firstBuildingDivergenceTick() == -1 ? "none" : summary.firstBuildingDivergenceTick()));
        sb.append(String.format("Max mineral delta:  %d%n", summary.maxMineralDelta()));
        sb.append(String.format("Max vespene delta:  %d%n", summary.maxVespeneDelta()));
        sb.append(String.format("%n%-6s  %-10s  %-12s  %-18s  %-18s%n",
            "Tick", "Units GT/EM", "Bldgs GT/EM", "Minerals GT/EM", "Vespene GT/EM"));
        for (TickSnapshot t : ticks) {
            boolean diverges = t.hasUnitDivergence() || t.hasBuildingDivergence()
                || t.mineralDelta() > 100 || t.vespeneDelta() > 50;
            if (diverges) {
                sb.append(String.format("%-6d  %5d/%-5d  %6d/%-6d  %8d/%-8d  %8d/%-8d%n",
                    t.tick(),
                    t.groundTruthUnits(),     t.emulatedUnits(),
                    t.groundTruthBuildings(), t.emulatedBuildings(),
                    t.groundTruthMinerals(),  t.emulatedMinerals(),
                    t.groundTruthVespene(),   t.emulatedVespene()));
            }
        }
        if (ticks.stream().noneMatch(t -> t.hasUnitDivergence() || t.hasBuildingDivergence()
                || t.mineralDelta() > 100 || t.vespeneDelta() > 50)) {
            sb.append("(no divergent ticks)").append(System.lineSeparator());
        }
        return sb.toString();
    }
}
```

- [ ] **Verify compilation:**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Commit:**

```bash
git add src/main/java/io/quarkmind/sc2/replay/DivergenceReport.java
git commit -m "feat(replay): add DivergenceReport — per-tick economic divergence record

Refs #137"
```

---

## Task 8: `ReplayValidationHarness` — the parallel-run engine

Runs `EmulatedGame` and `ReplaySimulatedGame` tick-by-tick, applying `TimedIntent`s at the correct tick, and produces a `DivergenceReport`.

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/replay/ReplayValidationTest.java`
- Create: `src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java`

- [ ] **Write the failing test first:**

```java
package io.quarkmind.sc2.replay;

import org.junit.jupiter.api.Test;

import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Regression test — validates EmulatedGame economic accuracy against
 * Nothing_4720936.SC2Replay (PvZ, Nothing = Protoss, player 1) to 3 minutes.
 * Refs #137.
 */
class ReplayValidationTest {

    static final Path REPLAY =
        Path.of("replays/aiarena_protoss/Nothing_4720936.SC2Replay");
    static final int THREE_MINUTES_TICKS = 180; // 3 * 60 seconds * (22 loops/tick ÷ 22.4/sec) ≈ 180

    @Test
    void unitCountMatchesGroundTruthExactlyForThreeMinutes() {
        DivergenceReport report = ReplayValidationHarness.run(REPLAY, 1, THREE_MINUTES_TICKS);

        assertThat(report.summary().firstUnitDivergenceTick())
            .as("Unit counts must match ground truth exactly at every tick. " +
                "First divergence at tick %d. Full report:%n%s",
                report.summary().firstUnitDivergenceTick(),
                report.renderReport())
            .isEqualTo(-1);
    }

    @Test
    void buildingCountMatchesGroundTruthExactlyForThreeMinutes() {
        DivergenceReport report = ReplayValidationHarness.run(REPLAY, 1, THREE_MINUTES_TICKS);

        assertThat(report.summary().firstBuildingDivergenceTick())
            .as("Building counts must match ground truth exactly at every tick. " +
                "First divergence at tick %d. Full report:%n%s",
                report.summary().firstBuildingDivergenceTick(),
                report.renderReport())
            .isEqualTo(-1);
    }

    @Test
    void mineralDeltaWithinToleranceForThreeMinutes() {
        DivergenceReport report = ReplayValidationHarness.run(REPLAY, 1, THREE_MINUTES_TICKS);

        assertThat(report.summary().maxMineralDelta())
            .as("Mineral delta must stay ≤ 100 at every tick. " +
                "Max delta was %d. Full report:%n%s",
                report.summary().maxMineralDelta(),
                report.renderReport())
            .isLessThanOrEqualTo(100);
    }

    @Test
    void vespeneDeltaWithinToleranceForThreeMinutes() {
        DivergenceReport report = ReplayValidationHarness.run(REPLAY, 1, THREE_MINUTES_TICKS);

        assertThat(report.summary().maxVespeneDelta())
            .as("Vespene delta must stay ≤ 50 at every tick. " +
                "Max delta was %d. Full report:%n%s",
                report.summary().maxVespeneDelta(),
                report.renderReport())
            .isLessThanOrEqualTo(50);
    }
}
```

- [ ] **Run to confirm it fails (class not found):**

```bash
mvn test -Dtest=ReplayValidationTest -q 2>&1 | tail -3
```

- [ ] **Create `ReplayValidationHarness.java`:**

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.domain.GameState;
import io.quarkmind.sc2.emulated.EmulatedGame;
import io.quarkmind.sc2.mock.ReplaySimulatedGame;
import io.quarkmind.sc2.mock.Sc2ReplayShared;

import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;

/**
 * Runs EmulatedGame and ReplaySimulatedGame in parallel, feeding the replay player's
 * commands into the emulated engine and comparing state tick-by-tick.
 *
 * Economic validation only: unit counts, building counts, minerals, vespene.
 * EmulatedGame runs without terrain or enemy AI — economic layer only.
 */
public final class ReplayValidationHarness {

    private ReplayValidationHarness() {}

    /**
     * @param replayPath  path to a .SC2Replay file
     * @param playerId    1-indexed player to validate (matches starcraft.replay.player)
     * @param tickLimit   maximum number of ticks to run (use Integer.MAX_VALUE for full replay)
     */
    public static DivergenceReport run(Path replayPath, int playerId, int tickLimit) {
        ReplaySimulatedGame replayGame = new ReplaySimulatedGame(replayPath, playerId);
        EmulatedGame        emulated   = new EmulatedGame();

        replayGame.reset();
        emulated.reset();

        assertInitialStateMatch(replayGame.snapshot(), emulated.snapshot(), replayPath, playerId);

        ReplayCommandStream commands = ReplayCommandExtractor.extract(replayPath, playerId);
        List<TimedIntent>   intents  = commands.intents();
        int                 cursor   = 0;

        List<DivergenceReport.TickSnapshot> snapshots = new ArrayList<>(tickLimit);

        for (int tick = 0; tick < tickLimit && !replayGame.isComplete(); tick++) {
            long windowStart = (long) tick       * Sc2ReplayShared.LOOPS_PER_TICK;
            long windowEnd   = (long) (tick + 1) * Sc2ReplayShared.LOOPS_PER_TICK;

            // Apply all intents whose loop falls in [windowStart, windowEnd)
            while (cursor < intents.size() && intents.get(cursor).loop() < windowEnd) {
                if (intents.get(cursor).loop() >= windowStart) {
                    emulated.applyIntent(intents.get(cursor).intent());
                }
                cursor++;
            }

            emulated.tick();
            replayGame.tick();

            GameState gt = replayGame.snapshot();
            GameState em = emulated.snapshot();

            snapshots.add(new DivergenceReport.TickSnapshot(
                tick,
                gt.myUnits().size(),     em.myUnits().size(),
                gt.myBuildings().size(), em.myBuildings().size(),
                gt.minerals(),           em.minerals(),
                gt.vespene(),            em.vespene()));
        }

        return DivergenceReport.from(snapshots);
    }

    private static void assertInitialStateMatch(GameState gt, GameState em,
                                                 Path replayPath, int playerId) {
        if (gt.myUnits().size() != em.myUnits().size()
                || gt.myBuildings().size() != em.myBuildings().size()
                || Math.abs(gt.minerals() - em.minerals()) > 10
                || Math.abs(gt.vespene()  - em.vespene())  > 5) {
            throw new IllegalStateException(String.format(
                "Initial state mismatch for %s player %d — " +
                "replay: %d units, %d buildings, %d minerals, %d vespene; " +
                "emulated: %d units, %d buildings, %d minerals, %d vespene",
                replayPath.getFileName(), playerId,
                gt.myUnits().size(), gt.myBuildings().size(), gt.minerals(), gt.vespene(),
                em.myUnits().size(), em.myBuildings().size(), em.minerals(), em.vespene()));
        }
    }
}
```

- [ ] **Run the test:**

```bash
mvn test -Dtest=ReplayValidationTest -q
```

Expected: tests pass once ability IDs are correctly filled. If tests fail, `renderReport()` output will be included in the assertion message — read it to identify which tick diverges and which ability ID mapping is incorrect. Iterate on `AbilityMapping` constants until green.

- [ ] **Commit (tests green):**

```bash
git add src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java \
        src/test/java/io/quarkmind/sc2/replay/ReplayValidationTest.java
git commit -m "feat(replay): add ReplayValidationHarness and ReplayValidationTest

Economic validation: exact unit/building counts, mineral delta ≤ 100.
Runs Nothing_4720936.SC2Replay player 1 to 3 minutes.

Closes #137"
```

---

## Task 9: `ReplayValidationReportTest` — full-replay human-readable dump

`@Tag("report")`, excluded from default runs, activated with `mvn test -Preport`.

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/replay/ReplayValidationReportTest.java`

- [ ] **Create `ReplayValidationReportTest.java`:**

```java
package io.quarkmind.sc2.replay;

import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.nio.file.Path;

/**
 * Full-replay divergence report — run with: mvn test -Preport
 * No assertions. Human-readable output to stdout.
 */
@Tag("report")
class ReplayValidationReportTest {

    @Test
    void fullReplayDivergenceReport() {
        Path replay = Path.of("replays/aiarena_protoss/Nothing_4720936.SC2Replay");
        DivergenceReport report = ReplayValidationHarness.run(replay, 1, Integer.MAX_VALUE);
        System.out.println(report.renderReport());
    }
}
```

- [ ] **Verify it's excluded from the default run:**

```bash
mvn test -q 2>&1 | grep -i "ReplayValidationReport" | head -3
```

Expected: no output — the test is skipped by the `report` exclusion added in Task 6.

- [ ] **Verify it runs via the profile:**

```bash
mvn test -Preport -q 2>&1 | tail -20
```

Expected: `ReplayValidationReportTest` runs and prints the divergence report to stdout.

- [ ] **Run the full test suite to confirm no regressions:**

```bash
mvn test -q
```

Expected: BUILD SUCCESS, all default tests pass.

- [ ] **Commit:**

```bash
git add src/test/java/io/quarkmind/sc2/replay/ReplayValidationReportTest.java
git commit -m "test(replay): add ReplayValidationReportTest for full-replay divergence dump

Run with: mvn test -Preport

Refs #137"
```

---

## Self-Review Notes

**Spec coverage:**
- `GameEventStream` refactored ✅ (Task 1)
- `AbilityDiscoveryTest` permanent coverage instrument ✅ (Task 2)
- `TimedIntent`, `ReplayCommand`, `ReplayCommandStream` ✅ (Task 3)
- `AbilityMapping` stateful, all three races (Protoss + Zerg from replays, Terran #140) ✅ (Task 4)
- `ReplayCommandExtractor` static facade ✅ (Task 5)
- `ReplayEngine.connect()` updated ✅ (Task 5)
- Maven `report` profile ✅ (Task 6)
- `DivergenceReport` with `renderReport()` ✅ (Task 7)
- `ReplayValidationHarness` with initial-state assert ✅ (Task 8)
- `ReplayValidationTest` exact unit/building + toleranced minerals ✅ (Task 8)
- `ReplayValidationReportTest` excluded from default, activated via `-Preport` ✅ (Task 9)

**Spec correction applied:** `AbilityMapping.process()` returns `List<ReplayCommand>` (spec said `Optional`) — movement commands produce one entry per selected unit.

**Known gap:** Terran ability IDs in `AbilityMapping` are stubs (all `0`) until Terran `.SC2Replay` files are acquired (#140). The stub constants don't affect correctness for the Protoss validation test (Task 8).
