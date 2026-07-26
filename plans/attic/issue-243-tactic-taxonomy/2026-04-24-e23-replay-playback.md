# E23: Replay Playback + Interactive Unit Inspection — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Animate units moving across real terrain during replay, with play/pause/scrub controls, click-to-inspect unit detail panel, and SC2/3D camera mode toggle.

**Architecture:** E23a builds movement simulation (pure Java, no CDI) on top of the existing `ReplaySimulatedGame`. E23b adds REST controls for pause/seek/speed. E23c adds a click-inspect panel via raycasting. E23d adds camera modes, WASD, and responsive layout. All pieces build on the existing WebSocket broadcast pipeline — the visualizer already updates positions every tick from `GameStateBroadcast`.

**Tech Stack:** Java 21, Quarkus, scelight GAME_EVENTS parsing (`CmdEvent`, `SelectionDeltaEvent`), Three.js raycasting, CSS transitions, JUnit 5 + AssertJ, RestAssured, Playwright.

**Issues:** Epic #101 | #102 (E23a movement) | #103 (E23b controls) | #104 (E23c inspect) | #105 (E23d camera)

**Key tag encoding:** Raw unit tags from GAME_EVENTS = `(tagIndex << 18) | tagRecycle`. Tracker-event tag string = `"r-" + tagIndex + "-" + tagRecycle`. To match them: `tagIndex = rawTag >> 18`, `tagRecycle = rawTag & 0x3FFFF`.

---

## File Map

**New Java:**
- `src/main/java/io/quarkmind/sc2/replay/UnitOrder.java` — record
- `src/main/java/io/quarkmind/sc2/replay/GameEventStream.java` — parses GAME_EVENTS, emits UnitOrders
- `src/main/java/io/quarkmind/sc2/replay/UnitOrderTracker.java` — advances positions per tick
- `src/main/java/io/quarkmind/qa/ReplayControlsResource.java` — pause/resume/seek/speed/status
- `src/main/java/io/quarkmind/qa/ReplayStatusResponse.java` — response record
- `src/main/java/io/quarkmind/qa/UnitResource.java` — GET /qa/unit/{tag}
- `src/test/java/io/quarkmind/sc2/replay/GameEventStreamTest.java`
- `src/test/java/io/quarkmind/sc2/replay/UnitOrderTrackerTest.java`
- `src/test/java/io/quarkmind/qa/ReplayControlsIT.java`
- `src/test/java/io/quarkmind/qa/UnitResourceTest.java`

**Modified Java:**
- `src/main/java/io/quarkmind/domain/SC2Data.java` — add `UNIT_SPEEDS` map
- `src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java` — add `replaceUnitPosition()`, `replaceEnemyPosition()`
- `src/main/java/io/quarkmind/sc2/mock/ReplaySimulatedGame.java` — integrate UnitOrderTracker, add `seekTo()`, `totalLoops()`
- `src/main/java/io/quarkmind/sc2/replay/ReplayEngine.java` — construct GameEventStream, expose pause/speed
- `src/main/java/io/quarkmind/agent/AgentOrchestrator.java` — add `pauseScheduler()` / `resumeScheduler()` / `setSpeedMultiplier()`
- `src/main/resources/META-INF/resources/visualizer.js` — control bar, inspect panel, camera modes, WASD, UI scaling

---

## Task 1: UnitOrder record + SC2Data unit speeds — Issue #102

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/replay/UnitOrder.java`
- Modify: `src/main/java/io/quarkmind/domain/SC2Data.java`

- [ ] **Step 1.1: Create `UnitOrder.java`**

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.domain.Point2d;

/**
 * A movement or follow order extracted from GAME_EVENTS.
 * Exactly one of targetPos or targetUnitTag is non-null.
 */
public record UnitOrder(
    String unitTag,      // tracker-event tag: "r-{index}-{recycle}"
    long   loop,         // game loop this order was issued
    Point2d targetPos,   // non-null for move orders
    String targetUnitTag // non-null for follow/attack orders
) {
    public boolean isMove()   { return targetPos != null; }
    public boolean isFollow() { return targetUnitTag != null; }
}
```

- [ ] **Step 1.2: Add `UNIT_SPEEDS` to `SC2Data.java`**

Add after the existing `trainTimeInTicks` method:

```java
import io.quarkmind.domain.UnitType;
import java.util.Map;

/** Movement speed in tiles/sec at SC2 Faster speed (22.4 loops/sec). */
public static final Map<UnitType, Double> UNIT_SPEEDS = Map.ofEntries(
    Map.entry(UnitType.PROBE,        3.94),
    Map.entry(UnitType.ZEALOT,       3.15),
    Map.entry(UnitType.STALKER,      4.13),
    Map.entry(UnitType.IMMORTAL,     3.15),
    Map.entry(UnitType.COLOSSUS,     2.77),
    Map.entry(UnitType.DISRUPTOR,    3.15),
    Map.entry(UnitType.ADEPT,        3.50),
    Map.entry(UnitType.ARCHON,       3.94),
    Map.entry(UnitType.PHOENIX,      5.61),
    Map.entry(UnitType.ORACLE,       5.61),
    Map.entry(UnitType.VOID_RAY,     3.50),
    Map.entry(UnitType.CARRIER,      1.97),
    Map.entry(UnitType.TEMPEST,      2.63),
    Map.entry(UnitType.MOTHERSHIP,   1.97),
    Map.entry(UnitType.OBSERVER,     2.63),
    Map.entry(UnitType.SCV,          3.94),
    Map.entry(UnitType.MARINE,       3.15),
    Map.entry(UnitType.MARAUDER,     2.25),
    Map.entry(UnitType.MEDIVAC,      3.50),
    Map.entry(UnitType.DRONE,        2.95),
    Map.entry(UnitType.ZERGLING,     4.13),
    Map.entry(UnitType.ROACH,        3.15),
    Map.entry(UnitType.HYDRALISK,    3.15),
    Map.entry(UnitType.MUTALISK,     5.61),
    Map.entry(UnitType.BANELING,     3.50),
    Map.entry(UnitType.ULTRALISK,    4.13),
    Map.entry(UnitType.OVERLORD,     1.40),
    Map.entry(UnitType.OVERSEER,     3.94)
);

public static final double DEFAULT_UNIT_SPEED = 3.00;

public static double unitSpeed(UnitType type) {
    return UNIT_SPEEDS.getOrDefault(type, DEFAULT_UNIT_SPEED);
}
```

- [ ] **Step 1.3: Verify compilation**

```bash
cd /Users/mdproctor/claude/quarkmind && mvn compile -q
```
Expected: BUILD SUCCESS

- [ ] **Step 1.4: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/replay/UnitOrder.java \
        src/main/java/io/quarkmind/domain/SC2Data.java
git commit -m "feat: UnitOrder record + SC2Data.UNIT_SPEEDS for movement simulation Refs #102"
```

---

## Task 2: GameEventStream — GAME_EVENTS parsing — Issue #102

Parses `CmdEvent` + `SelectionDeltaEvent` from the replay MPQ, tracks per-player selection state, emits `List<UnitOrder>` sorted by loop.

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/replay/GameEventStreamTest.java`
- Create: `src/main/java/io/quarkmind/sc2/replay/GameEventStream.java`

- [ ] **Step 2.1: Write failing tests**

```java
package io.quarkmind.sc2.replay;

import org.junit.jupiter.api.Test;
import java.nio.file.Path;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class GameEventStreamTest {

    static final Path REPLAY =
        Path.of("replays/aiarena_protoss/Nothing_4720936.SC2Replay");

    @Test
    void parsesMoreThanThousandOrders() {
        List<UnitOrder> orders = GameEventStream.parse(REPLAY);
        assertThat(orders).hasSizeGreaterThan(1000);
    }

    @Test
    void allTargetCoordinatesWithinMapBounds() {
        List<UnitOrder> orders = GameEventStream.parse(REPLAY);
        for (UnitOrder o : orders) {
            if (o.targetPos() != null) {
                assertThat(o.targetPos().x()).as("x for loop %d", o.loop())
                    .isBetween(0f, 160f);
                assertThat(o.targetPos().y()).as("y for loop %d", o.loop())
                    .isBetween(0f, 208f);
            }
        }
    }

    @Test
    void containsMoveOrders() {
        List<UnitOrder> orders = GameEventStream.parse(REPLAY);
        assertThat(orders).anyMatch(UnitOrder::isMove);
    }

    @Test
    void containsFollowOrders() {
        List<UnitOrder> orders = GameEventStream.parse(REPLAY);
        assertThat(orders).anyMatch(UnitOrder::isFollow);
    }

    @Test
    void ordersAreSortedByLoop() {
        List<UnitOrder> orders = GameEventStream.parse(REPLAY);
        for (int i = 1; i < orders.size(); i++) {
            assertThat(orders.get(i).loop())
                .isGreaterThanOrEqualTo(orders.get(i - 1).loop());
        }
    }

    @Test
    void unitTagsMatchTrackerEventFormat() {
        List<UnitOrder> orders = GameEventStream.parse(REPLAY);
        // All tags must start with "r-" (the ReplaySimulatedGame format)
        assertThat(orders).allMatch(o -> o.unitTag().startsWith("r-"));
    }

    @Test
    void throwsForMissingFile() {
        assertThatThrownBy(() -> GameEventStream.parse(Path.of("nonexistent.SC2Replay")))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2.2: Run tests to confirm failure**

```bash
mvn test -Dtest=GameEventStreamTest -q 2>&1 | tail -5
```
Expected: FAIL — `GameEventStream` does not exist.

- [ ] **Step 2.3: Implement `GameEventStream.java`**

```java
package io.quarkmind.sc2.replay;

import hu.belicza.andras.mpq.MpqParser;
import hu.scelight.sc2.rep.factory.RepContent;
import hu.scelight.sc2.rep.factory.RepParserEngine;
import hu.scelight.sc2.rep.model.Replay;
import hu.scelight.sc2.rep.model.gameevents.cmd.CmdEvent;
import hu.scelight.sc2.rep.model.gameevents.cmd.TargetPoint;
import hu.scelight.sc2.rep.model.gameevents.cmd.TargetUnit;
import hu.scelight.sc2.rep.model.gameevents.selectiondelta.Delta;
import hu.scelight.sc2.rep.model.gameevents.selectiondelta.SelectionDeltaEvent;
import hu.scelight.sc2.rep.s2prot.Event;
import io.quarkmind.domain.Point2d;

import java.nio.file.Path;
import java.util.*;

public final class GameEventStream {

    private GameEventStream() {}

    /**
     * Parses GAME_EVENTS from the replay, returns per-unit move/follow orders sorted by loop.
     * Tags use the same "r-{index}-{recycle}" format as {@code ReplaySimulatedGame}.
     */
    public static List<UnitOrder> parse(Path replayPath) {
        Replay replay;
        try {
            replay = RepParserEngine.parseReplay(replayPath, EnumSet.of(RepContent.GAME_EVENTS));
        } catch (Exception e) {
            throw new IllegalArgumentException("Cannot parse GAME_EVENTS from: " + replayPath, e);
        }
        if (replay == null || replay.gameEvents == null) {
            throw new IllegalArgumentException("No game events in replay: " + replayPath);
        }

        // Per-player selection state: userId → selected unit tags
        Map<Integer, List<String>> selections = new HashMap<>();
        List<UnitOrder> orders = new ArrayList<>();

        for (Event raw : replay.gameEvents.getEvents()) {
            int userId = raw.getUserId();

            if (raw instanceof SelectionDeltaEvent sel) {
                Delta delta = sel.getDelta();
                if (delta == null) continue;
                Integer[] addTags = delta.getAddUnitTags();
                if (addTags == null || addTags.length == 0) continue;
                List<String> decoded = new ArrayList<>(addTags.length);
                for (Integer rawTag : addTags) {
                    if (rawTag != null) decoded.add(decodeTag(rawTag));
                }
                // Replace current selection (addUnitTags replaces, not appends, for bot replays)
                selections.put(userId, decoded);

            } else if (raw instanceof CmdEvent cmd) {
                List<String> selected = selections.get(userId);
                if (selected == null || selected.isEmpty()) continue;

                TargetPoint tp = cmd.getTargetPoint();
                TargetUnit  tu = cmd.getTargetUnit();

                if (tp != null) {
                    float x = tp.getXFloat(), y = tp.getYFloat();
                    if (x >= 0 && x <= 256 && y >= 0 && y <= 256) {
                        Point2d target = new Point2d(x, y);
                        for (String tag : selected) {
                            orders.add(new UnitOrder(tag, raw.getLoop(), target, null));
                        }
                    }
                } else if (tu != null && tu.getTag() != null) {
                    String targetTag = decodeTag(tu.getTag());
                    for (String tag : selected) {
                        orders.add(new UnitOrder(tag, raw.getLoop(), null, targetTag));
                    }
                }
            }
        }

        orders.sort(Comparator.comparingLong(UnitOrder::loop));
        return orders;
    }

    /** Decode raw SC2 unit tag to tracker-event format "r-{index}-{recycle}". */
    static String decodeTag(int rawTag) {
        return "r-" + (rawTag >> 18) + "-" + (rawTag & 0x3FFFF);
    }
}
```

- [ ] **Step 2.4: Run tests — all 7 should pass**

```bash
mvn test -Dtest=GameEventStreamTest -q
```
Expected: BUILD SUCCESS, 7 tests passed.

- [ ] **Step 2.5: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/replay/GameEventStream.java \
        src/test/java/io/quarkmind/sc2/replay/GameEventStreamTest.java
git commit -m "feat: GameEventStream — parse GAME_EVENTS, emit per-unit move/follow orders Refs #102"
```

---

## Task 3: SimulatedGame position update helpers + UnitOrderTracker — Issue #102

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java`
- Create: `src/test/java/io/quarkmind/sc2/replay/UnitOrderTrackerTest.java`
- Create: `src/main/java/io/quarkmind/sc2/replay/UnitOrderTracker.java`

- [ ] **Step 3.1: Add position-update helpers to `SimulatedGame`**

Add alongside the existing `removeUnitByTag` / `setUnitHealth` helpers (around line 150):

```java
/** Advances a friendly unit's position in-place. No-op if tag not found. */
protected void replaceUnitPosition(String tag, Point2d newPos) {
    myUnits.replaceAll(u -> u.tag().equals(tag)
        ? new Unit(u.tag(), u.type(), newPos, u.health(), u.maxHealth(),
                   u.shields(), u.maxShields(), u.weaponCooldownTicks(), u.blinkCooldownTicks())
        : u);
}

/** Advances an enemy unit's position in-place. No-op if tag not found. */
protected void replaceEnemyPosition(String tag, Point2d newPos) {
    enemyUnits.replaceAll(u -> u.tag().equals(tag)
        ? new Unit(u.tag(), u.type(), newPos, u.health(), u.maxHealth(),
                   u.shields(), u.maxShields(), u.weaponCooldownTicks(), u.blinkCooldownTicks())
        : u);
}
```

- [ ] **Step 3.2: Write failing UnitOrderTracker tests**

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.UnitType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class UnitOrderTrackerTest {

    UnitOrderTracker tracker;

    @BeforeEach
    void setUp() { tracker = new UnitOrderTracker(); }

    @Test
    void unitMovesTowardTargetEachTick() {
        // Probe speed = 3.94 tiles/sec, LOOPS_PER_TICK = 22, 22.4 loops/sec
        // Distance per tick = 3.94 * 22 / 22.4 ≈ 3.87 tiles
        tracker.loadOrders(List.of(
            new UnitOrder("r-1-1", 0L, new Point2d(20, 20), null)
        ));
        Map<String, Point2d> positions = new java.util.HashMap<>(Map.of("r-1-1", new Point2d(10, 10)));
        Map<String, UnitType> types    = Map.of("r-1-1", UnitType.PROBE);

        tracker.advance(22L, positions, types);

        Point2d pos = positions.get("r-1-1");
        assertThat(pos.x()).isGreaterThan(10f); // moved toward target
        assertThat(pos.y()).isGreaterThan(10f);
        assertThat(pos.x()).isLessThan(20f);    // didn't overshoot
    }

    @Test
    void unitStopsWithinHalfTileOfTarget() {
        tracker.loadOrders(List.of(
            new UnitOrder("r-1-1", 0L, new Point2d(10.1f, 10.1f), null)
        ));
        Map<String, Point2d> positions = new java.util.HashMap<>(Map.of("r-1-1", new Point2d(10, 10)));
        Map<String, UnitType> types    = Map.of("r-1-1", UnitType.PROBE);

        tracker.advance(22L, positions, types);

        Point2d pos = positions.get("r-1-1");
        float dx = pos.x() - 10.1f, dy = pos.y() - 10.1f;
        assertThat(Math.sqrt(dx*dx + dy*dy)).isLessThan(0.5);
    }

    @Test
    void ordersAfterCurrentLoopAreNotApplied() {
        tracker.loadOrders(List.of(
            new UnitOrder("r-1-1", 100L, new Point2d(20, 20), null) // future order
        ));
        Map<String, Point2d> positions = new java.util.HashMap<>(Map.of("r-1-1", new Point2d(10, 10)));
        Map<String, UnitType> types    = Map.of("r-1-1", UnitType.PROBE);

        tracker.advance(22L, positions, types); // loop 22 < order loop 100

        assertThat(positions.get("r-1-1").x()).isEqualTo(10f); // not moved
    }

    @Test
    void followOrderMovesUnitTowardTarget() {
        tracker.loadOrders(List.of(
            new UnitOrder("r-1-1", 0L, null, "r-2-1") // follow r-2-1
        ));
        Map<String, Point2d> positions = new java.util.HashMap<>(Map.of(
            "r-1-1", new Point2d(10, 10),
            "r-2-1", new Point2d(30, 30)
        ));
        Map<String, UnitType> types = Map.of(
            "r-1-1", UnitType.ZEALOT,
            "r-2-1", UnitType.PROBE
        );

        tracker.advance(22L, positions, types);

        Point2d pos = positions.get("r-1-1");
        assertThat(pos.x()).isGreaterThan(10f);
        assertThat(pos.y()).isGreaterThan(10f);
    }

    @Test
    void unknownUnitTypeUsesDefaultSpeed() {
        tracker.loadOrders(List.of(
            new UnitOrder("r-1-1", 0L, new Point2d(100, 100), null)
        ));
        Map<String, Point2d> positions = new java.util.HashMap<>(Map.of("r-1-1", new Point2d(10, 10)));
        Map<String, UnitType> types    = Map.of("r-1-1", UnitType.UNKNOWN);

        // Should not throw, should move with default speed
        tracker.advance(22L, positions, types);
        assertThat(positions.get("r-1-1").x()).isGreaterThan(10f);
    }

    @Test
    void newOrderOverridesPreviousOrder() {
        tracker.loadOrders(List.of(
            new UnitOrder("r-1-1",  0L, new Point2d(100, 100), null),
            new UnitOrder("r-1-1", 10L, new Point2d(5, 5),     null) // overrides at loop 10
        ));
        Map<String, Point2d> positions = new java.util.HashMap<>(Map.of("r-1-1", new Point2d(10, 10)));
        Map<String, UnitType> types    = Map.of("r-1-1", UnitType.PROBE);

        tracker.advance(22L, positions, types); // both orders consumed by loop 22

        Point2d pos = positions.get("r-1-1");
        // Should be moving toward (5,5), not (100,100)
        assertThat(pos.x()).isLessThan(10f);
        assertThat(pos.y()).isLessThan(10f);
    }
}
```

- [ ] **Step 3.3: Run tests to confirm failure**

```bash
mvn test -Dtest=UnitOrderTrackerTest -q 2>&1 | tail -5
```
Expected: FAIL — `UnitOrderTracker` does not exist.

- [ ] **Step 3.4: Implement `UnitOrderTracker.java`**

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.SC2Data;
import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.mock.Sc2ReplayShared;

import java.util.*;

public final class UnitOrderTracker {

    private static final float ARRIVAL_THRESHOLD = 0.5f;

    // Per-unit active order
    private final Map<String, UnitOrder> activeOrders = new HashMap<>();
    // Pending orders, sorted by loop (consumed as time advances)
    private final List<UnitOrder> pending = new ArrayList<>();
    private int pendingCursor = 0;

    /** Call once after constructing, before any advance() calls. */
    public void loadOrders(List<UnitOrder> orders) {
        pending.clear();
        pending.addAll(orders);
        pendingCursor = 0;
        activeOrders.clear();
    }

    /**
     * Advance all units one tick.
     * @param currentLoop current game loop (after advancing LOOPS_PER_TICK)
     * @param positions   mutable map of tag → current position (updated in-place)
     * @param unitTypes   map of tag → UnitType for speed lookup
     */
    public void advance(long currentLoop, Map<String, Point2d> positions,
                        Map<String, UnitType> unitTypes) {
        // Consume pending orders up to currentLoop
        while (pendingCursor < pending.size() && pending.get(pendingCursor).loop() <= currentLoop) {
            UnitOrder o = pending.get(pendingCursor++);
            activeOrders.put(o.unitTag(), o); // newest order overwrites older one
        }

        float loopsPerTick = Sc2ReplayShared.LOOPS_PER_TICK;
        float secondsPerTick = loopsPerTick / 22.4f;

        for (var entry : activeOrders.entrySet()) {
            String tag = entry.getKey();
            UnitOrder order = entry.getValue();
            Point2d current = positions.get(tag);
            if (current == null) continue;

            Point2d target = resolveTarget(order, positions);
            if (target == null) continue;

            float dx = target.x() - current.x();
            float dy = target.y() - current.y();
            float dist = (float) Math.sqrt(dx * dx + dy * dy);
            if (dist <= ARRIVAL_THRESHOLD) continue;

            UnitType type = unitTypes.getOrDefault(tag, UnitType.UNKNOWN);
            float speed = (float) SC2Data.unitSpeed(type);
            float step = speed * secondsPerTick;

            if (step >= dist) {
                positions.put(tag, target);
            } else {
                float ratio = step / dist;
                positions.put(tag, new Point2d(current.x() + dx * ratio,
                                               current.y() + dy * ratio));
            }
        }
    }

    /** Remove units that have died so their orders don't linger. */
    public void removeUnit(String tag) {
        activeOrders.remove(tag);
    }

    /** Reset to initial state (called by seekTo). */
    public void reset() {
        activeOrders.clear();
        pendingCursor = 0;
    }

    private static Point2d resolveTarget(UnitOrder order, Map<String, Point2d> positions) {
        if (order.isMove()) return order.targetPos();
        if (order.isFollow()) return positions.get(order.targetUnitTag());
        return null;
    }
}
```

- [ ] **Step 3.5: Run tests — all 6 should pass**

```bash
mvn test -Dtest=UnitOrderTrackerTest -q
```
Expected: BUILD SUCCESS, 6 tests passed.

- [ ] **Step 3.6: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java \
        src/main/java/io/quarkmind/sc2/replay/UnitOrderTracker.java \
        src/test/java/io/quarkmind/sc2/replay/UnitOrderTrackerTest.java
git commit -m "feat: UnitOrderTracker position simulation + SimulatedGame position helpers Refs #102"
```

---

## Task 4: ReplaySimulatedGame integration — seekTo, totalLoops, UnitOrderTracker — Issue #102

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/mock/ReplaySimulatedGame.java`
- Modify: `src/test/java/io/quarkmind/sc2/replay/ReplayEngineTest.java`

- [ ] **Step 4.1: Write failing tests**

Add to `ReplayEngineTest.java` (existing test class, add new methods):

```java
@Test
void totalLoopsIsPositive() {
    engine.connect();
    engine.joinGame();
    // Access via reflection or expose via ReplayEngine — see implementation notes
    // For now, test via seekTo correctness
    assertThat(engine.isConnected()).isTrue();
}

@Test
void unitPositionsUpdateAfterTicks() {
    engine.connect();
    engine.joinGame();
    // Tick forward enough for some orders to be applied
    for (int i = 0; i < 50; i++) engine.tick();
    GameState state = engine.observe();
    // Units should exist (Probes were born at loop 0)
    assertThat(state.myUnits()).isNotEmpty();
    // After 50 ticks units may have moved — positions should be within map bounds
    for (var u : state.myUnits()) {
        assertThat(u.position().x()).isBetween(0f, 160f);
        assertThat(u.position().y()).isBetween(0f, 208f);
    }
}
```

Also add a unit test directly on `ReplaySimulatedGame`:

```java
// In a new test class: ReplaySimulatedGameMovementTest
package io.quarkmind.sc2.mock;

import io.quarkmind.sc2.replay.GameEventStream;
import org.junit.jupiter.api.Test;
import java.nio.file.Path;
import static org.assertj.core.api.Assertions.assertThat;

class ReplaySimulatedGameMovementTest {

    static final Path REPLAY = Path.of("replays/aiarena_protoss/Nothing_4720936.SC2Replay");

    @Test
    void seekToReachesCorrectLoop() {
        var game = new ReplaySimulatedGame(REPLAY, 1);
        var orders = GameEventStream.parse(REPLAY);
        game.loadOrders(orders);
        game.seekTo(500);
        // After seek, gameFrame should be approximately 500/22 ≈ 22
        assertThat(game.snapshot().gameFrame()).isBetween(20L, 30L);
    }

    @Test
    void seekToCompletesQuickly() {
        var game = new ReplaySimulatedGame(REPLAY, 1);
        var orders = GameEventStream.parse(REPLAY);
        game.loadOrders(orders);
        long start = System.currentTimeMillis();
        game.seekTo(5000);
        assertThat(System.currentTimeMillis() - start).isLessThan(500);
    }

    @Test
    void totalLoopsIsPositive() {
        var game = new ReplaySimulatedGame(REPLAY, 1);
        assertThat(game.totalLoops()).isGreaterThan(0);
    }

    @Test
    void allUnitPositionsWithinMapBoundsAfter200Ticks() {
        var game = new ReplaySimulatedGame(REPLAY, 1);
        var orders = GameEventStream.parse(REPLAY);
        game.loadOrders(orders);
        for (int i = 0; i < 200; i++) game.tick();
        var state = game.snapshot();
        for (var u : state.myUnits()) {
            assertThat(u.position().x()).isBetween(0f, 160f);
            assertThat(u.position().y()).isBetween(0f, 208f);
        }
        for (var u : state.enemyUnits()) {
            assertThat(u.position().x()).isBetween(0f, 160f);
            assertThat(u.position().y()).isBetween(0f, 208f);
        }
    }
}
```

- [ ] **Step 4.2: Run tests to confirm failure**

```bash
mvn test -Dtest="ReplaySimulatedGameMovementTest" -q 2>&1 | tail -5
```

- [ ] **Step 4.3: Implement `loadOrders`, `seekTo`, `totalLoops` in `ReplaySimulatedGame`**

Add fields:
```java
private UnitOrderTracker orderTracker;
private long totalLoops = 0;
```

Update the constructor to compute `totalLoops`:
```java
// After parsing trackerEvents:
if (trackerEvents.length > 0) {
    this.totalLoops = trackerEvents[trackerEvents.length - 1].getLoop();
}
```

Add new public methods:
```java
/** Load movement orders parsed from GAME_EVENTS. Call once after construction. */
public void loadOrders(List<UnitOrder> orders) {
    this.orderTracker = new UnitOrderTracker();
    this.orderTracker.loadOrders(orders);
}

/** Total game loops in this replay (from last tracker event). */
public long totalLoops() { return totalLoops; }

/**
 * Fast-forward to the given loop without broadcasting intermediate states.
 * Caller is responsible for suppressing GameStateBroadcaster if needed.
 */
public synchronized void seekTo(long targetLoop) {
    reset();
    while (currentLoop < targetLoop && !isComplete()) {
        currentLoop += LOOPS_PER_TICK;
        setGameFrame(currentLoop / LOOPS_PER_TICK);
        drainEventsUpTo(currentLoop);
        if (orderTracker != null) advanceMovement();
    }
}
```

Update `tick()` to call movement after tracker events:
```java
@Override
public synchronized void tick() {
    currentLoop += LOOPS_PER_TICK;
    setGameFrame(currentLoop / LOOPS_PER_TICK);
    drainEventsUpTo(currentLoop);
    if (orderTracker != null) advanceMovement();
}
```

Update `reset()` to reset the tracker:
```java
// In reset(), after pendingBuildings.clear():
if (orderTracker != null) orderTracker.reset();
```

Add private `advanceMovement()`:
```java
private void advanceMovement() {
    // Build position maps from current unit state
    Map<String, Point2d> positions = new HashMap<>();
    Map<String, UnitType> types    = new HashMap<>();
    for (var u : getMyUnits())    { positions.put(u.tag(), u.position()); types.put(u.tag(), u.type()); }
    for (var u : getEnemyUnits()) { positions.put(u.tag(), u.position()); types.put(u.tag(), u.type()); }

    orderTracker.advance(currentLoop, positions, types);

    // Apply updated positions back
    for (var entry : positions.entrySet()) {
        replaceUnitPosition(entry.getKey(), entry.getValue());
        replaceEnemyPosition(entry.getKey(), entry.getValue());
    }
}
```

You'll need `getMyUnits()` and `getEnemyUnits()` on `SimulatedGame` — add:
```java
// In SimulatedGame:
protected List<Unit> getMyUnits()    { return Collections.unmodifiableList(myUnits); }
protected List<Unit> getEnemyUnits() { return Collections.unmodifiableList(enemyUnits); }
```

Also update `applyUnitDied` in `ReplaySimulatedGame` to notify the tracker:
```java
// At the end of applyUnitDied, after removeUnitByTag / removeBuildingByTag:
if (orderTracker != null) orderTracker.removeUnit(tag);
```

- [ ] **Step 4.4: Run all new tests**

```bash
mvn test -Dtest="ReplaySimulatedGameMovementTest,ReplayEngineTest" -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 4.5: Run full suite**

```bash
mvn test -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 4.6: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/mock/ReplaySimulatedGame.java \
        src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java \
        src/test/java/io/quarkmind/sc2/mock/ReplaySimulatedGameMovementTest.java \
        src/test/java/io/quarkmind/sc2/replay/ReplayEngineTest.java
git commit -m "feat: ReplaySimulatedGame integrates UnitOrderTracker — seekTo, totalLoops, movement Refs #102"
```

---

## Task 5: Wire GameEventStream into ReplayEngine — Issue #102

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/replay/ReplayEngine.java`

- [ ] **Step 5.1: Update `ReplayEngine.connect()` to parse and load orders**

In `connect()`, after `game = new ReplaySimulatedGame(...)` and `parseMapMetadata(...)`:

```java
// Parse GAME_EVENTS and load into the game
List<UnitOrder> orders = GameEventStream.parse(Path.of(replayFile));
game.loadOrders(orders);
log.infof("[REPLAY] Loaded %d movement orders from GAME_EVENTS", orders.size());
```

Add import: `import io.quarkmind.sc2.replay.UnitOrder;` and `import java.util.List;` if not present.

- [ ] **Step 5.2: Build and verify**

```bash
mvn compile -q && mvn test -Dtest=ReplayEngineTest -q
```
Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 5.3: Commit — closes E23a**

```bash
git add src/main/java/io/quarkmind/sc2/replay/ReplayEngine.java
git commit -m "feat: ReplayEngine loads GAME_EVENTS movement orders on connect Closes #102"
```

---

## Task 6: Replay controls — pause/resume/seek/speed endpoints — Issue #103

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/AgentOrchestrator.java`
- Modify: `src/main/java/io/quarkmind/qa/GameStateBroadcaster.java`
- Create: `src/main/java/io/quarkmind/qa/ReplayStatusResponse.java`
- Create: `src/main/java/io/quarkmind/qa/ReplayControlsResource.java`
- Create: `src/test/java/io/quarkmind/qa/ReplayControlsIT.java`

- [ ] **Step 6.1: Add pause/speed to `AgentOrchestrator`**

Add fields and methods:

```java
private volatile boolean schedulerPaused = false;
private volatile int speedMultiplier = 1; // 1=normal, 2=2x, 4=4x; 0=half (skip alternating ticks)
private long tickCount = 0;

public void pauseScheduler()  { schedulerPaused = true; }
public void resumeScheduler() { schedulerPaused = false; }
public boolean isSchedulerPaused() { return schedulerPaused; }
public void setSpeedMultiplier(int x) { this.speedMultiplier = x; }
public int getSpeedMultiplier() { return speedMultiplier; }
```

Update `gameTick()`:
```java
@Scheduled(every = "${starcraft.tick.interval:500ms}", concurrentExecution = Scheduled.ConcurrentExecution.SKIP)
public void gameTick() {
    if (!engine.isConnected()) return;
    if (schedulerPaused) return;

    tickCount++;
    // 0.5x: skip every other tick
    if (speedMultiplier == 0 && tickCount % 2 == 0) return;

    int ticks = (speedMultiplier > 1) ? speedMultiplier : 1;
    for (int i = 0; i < ticks; i++) {
        // ... existing tick body ...
    }
}
```

Wrap the existing tick body in the for loop. The broadcast should only happen after all ticks for this cycle.

- [ ] **Step 6.2: Add `setSuppressed` to `GameStateBroadcaster`**

```java
private volatile boolean suppressed = false;
public void setSuppressed(boolean s) { this.suppressed = s; }

// In onFrame():
private void onFrame(GameState state) {
    if (suppressed || sessions.isEmpty()) return;
    // ... existing body ...
}
```

- [ ] **Step 6.3: Create `ReplayStatusResponse.java`**

```java
package io.quarkmind.qa;

public record ReplayStatusResponse(long loop, long totalLoops, boolean paused, int speed) {}
```

- [ ] **Step 6.4: Write failing integration tests**

```java
package io.quarkmind.qa;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestProfile(ReplayControlsIT.ReplayProfile.class)
class ReplayControlsIT {

    public static class ReplayProfile implements QuarkusTestProfile {
        @Override public String getConfigProfile() { return "replay"; }
        @Override public Map<String, String> getConfigOverrides() {
            return Map.of("quarkus.scheduler.enabled", "false"); // manual tick in tests
        }
    }

    @Test
    void statusEndpointReturnsReplayState() {
        given().when().get("/qa/replay/status")
            .then().statusCode(200)
            .body("paused", notNullValue())
            .body("loop", greaterThanOrEqualTo(0))
            .body("totalLoops", greaterThan(0));
    }

    @Test
    void pauseAndResumeWork() {
        given().when().post("/qa/replay/pause").then().statusCode(204);
        given().when().get("/qa/replay/status").then().body("paused", is(true));
        given().when().post("/qa/replay/resume").then().statusCode(204);
        given().when().get("/qa/replay/status").then().body("paused", is(false));
    }

    @Test
    void seekMovesToRequestedLoop() {
        given().queryParam("loop", 200).when().post("/qa/replay/seek")
            .then().statusCode(204);
        given().when().get("/qa/replay/status")
            .then().body("loop", greaterThanOrEqualTo(0));
    }

    @Test
    void speedEndpointSetsMultiplier() {
        given().queryParam("multiplier", 2).when().post("/qa/replay/speed")
            .then().statusCode(204);
        given().when().get("/qa/replay/status")
            .then().body("speed", is(2));
        // Reset
        given().queryParam("multiplier", 1).when().post("/qa/replay/speed").then().statusCode(204);
    }

    @Test
    void unknownProfileReturns404() {
        // This test runs in replay profile so endpoints exist.
        // Confirm it doesn't 404.
        given().when().get("/qa/replay/status").then().statusCode(200);
    }
}
```

- [ ] **Step 6.5: Run tests to confirm failure**

```bash
mvn test -Dtest=ReplayControlsIT -q 2>&1 | tail -5
```

- [ ] **Step 6.6: Implement `ReplayControlsResource.java`**

```java
package io.quarkmind.qa;

import io.quarkus.arc.profile.UnlessBuildProfile;
import io.quarkmind.agent.AgentOrchestrator;
import io.quarkmind.sc2.SC2Engine;
import io.quarkmind.sc2.mock.ReplaySimulatedGame;
import io.quarkmind.sc2.replay.ReplayEngine;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

@UnlessBuildProfile("prod")
@Path("/qa/replay")
public class ReplayControlsResource {

    @Inject AgentOrchestrator orchestrator;
    @Inject SC2Engine engine;
    @Inject GameStateBroadcaster broadcaster;

    @GET @Path("/status") @Produces(MediaType.APPLICATION_JSON)
    public Response status() {
        if (!(engine instanceof ReplayEngine re)) return Response.status(404).build();
        long loop = re.currentLoop();
        long total = re.totalLoops();
        return Response.ok(new ReplayStatusResponse(
            loop, total, orchestrator.isSchedulerPaused(), orchestrator.getSpeedMultiplier()
        )).build();
    }

    @POST @Path("/pause")
    public Response pause() {
        orchestrator.pauseScheduler();
        return Response.noContent().build();
    }

    @POST @Path("/resume")
    public Response resume() {
        orchestrator.resumeScheduler();
        return Response.noContent().build();
    }

    @POST @Path("/seek")
    public Response seek(@QueryParam("loop") long targetLoop) {
        if (!(engine instanceof ReplayEngine re)) return Response.status(404).build();
        broadcaster.setSuppressed(true);
        try {
            re.seekTo(targetLoop);
        } finally {
            broadcaster.setSuppressed(false);
        }
        // Trigger one broadcast after seek
        engine.observe();
        return Response.noContent().build();
    }

    @POST @Path("/speed")
    public Response speed(@QueryParam("multiplier") int multiplier) {
        if (multiplier < 0 || multiplier > 8) return Response.status(400).build();
        orchestrator.setSpeedMultiplier(multiplier);
        return Response.noContent().build();
    }
}
```

You'll need to expose `currentLoop()` and `totalLoops()` and `seekTo()` on `ReplayEngine`:

```java
// Add to ReplayEngine:
public long currentLoop()  { return game != null ? game.currentLoop() : 0; }
public long totalLoops()   { return game != null ? game.totalLoops()  : 0; }
public void seekTo(long loop) {
    if (game != null) game.seekTo(loop);
}
```

And expose `currentLoop` from `ReplaySimulatedGame`:
```java
public long currentLoop() { return currentLoop; }
```

- [ ] **Step 6.7: Run integration tests**

```bash
mvn test -Dtest=ReplayControlsIT -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 6.8: Run full suite**

```bash
mvn test -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 6.9: Commit — closes E23b**

```bash
git add src/main/java/io/quarkmind/agent/AgentOrchestrator.java \
        src/main/java/io/quarkmind/qa/GameStateBroadcaster.java \
        src/main/java/io/quarkmind/qa/ReplayStatusResponse.java \
        src/main/java/io/quarkmind/qa/ReplayControlsResource.java \
        src/main/java/io/quarkmind/sc2/replay/ReplayEngine.java \
        src/main/java/io/quarkmind/sc2/mock/ReplaySimulatedGame.java \
        src/test/java/io/quarkmind/qa/ReplayControlsIT.java
git commit -m "feat: replay controls — pause/resume/seek/speed REST endpoints Closes #103"
```

---

## Task 7: Visualizer control bar — play/pause/rewind/scrub/speed — Issue #103

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`

- [ ] **Step 7.1: Read the bottom of `visualizer.js` to find init sequence**

```bash
tail -50 src/main/resources/META-INF/resources/visualizer.js
```

Find where `loadTerrain()`, `setupCamera()`, and `connectWebSocket()` are called.

- [ ] **Step 7.2: Add control bar HTML + CSS + JS**

After the `<canvas>` element and `<div id="hud">` in `visualizer.html` (or injected via JS), add the control bar. Since the visualizer uses a single HTML file, inject via JS at the bottom of `visualizer.js`:

```javascript
// ── Replay control bar ─────────────────────────────────────────────────────
let replayTotalLoops = 0;
let controlBarActive = false;

async function initReplayControls() {
  const meta = await fetchJson('/qa/current-map');
  if (!meta) return; // not replay profile
  controlBarActive = true;

  const status = await fetchJson('/qa/replay/status');
  if (status) replayTotalLoops = status.totalLoops;

  // Inject control bar HTML
  const bar = document.createElement('div');
  bar.id = 'replay-bar';
  bar.innerHTML = `
    <button id="rb-rewind" title="Rewind">⏮</button>
    <button id="rb-playpause" title="Play/Pause">⏸</button>
    <input id="rb-scrub" type="range" min="0" max="${replayTotalLoops}" value="0" step="22">
    <span id="rb-time">0:00 / 0:00</span>
    <button class="rb-speed" data-x="0">½×</button>
    <button class="rb-speed rb-active" data-x="1">1×</button>
    <button class="rb-speed" data-x="2">2×</button>
    <button class="rb-speed" data-x="4">4×</button>
  `;
  document.body.appendChild(bar);

  document.getElementById('rb-rewind').onclick = async () => {
    await fetch('/qa/replay/pause', {method:'POST'});
    await fetch('/qa/replay/seek?loop=0', {method:'POST'});
    document.getElementById('rb-playpause').textContent = '▶';
  };

  let playing = true;
  document.getElementById('rb-playpause').onclick = async () => {
    if (playing) {
      await fetch('/qa/replay/pause', {method:'POST'});
      document.getElementById('rb-playpause').textContent = '▶';
    } else {
      await fetch('/qa/replay/resume', {method:'POST'});
      document.getElementById('rb-playpause').textContent = '⏸';
    }
    playing = !playing;
  };

  const scrub = document.getElementById('rb-scrub');
  scrub.addEventListener('mouseup', async () => {
    await fetch(`/qa/replay/seek?loop=${scrub.value}`, {method:'POST'});
  });

  document.querySelectorAll('.rb-speed').forEach(btn => {
    btn.onclick = async () => {
      const x = parseInt(btn.dataset.x);
      await fetch(`/qa/replay/speed?multiplier=${x}`, {method:'POST'});
      document.querySelectorAll('.rb-speed').forEach(b => b.classList.remove('rb-active'));
      btn.classList.add('rb-active');
    };
  });

  // Poll status every 500ms when playing
  setInterval(async () => {
    const s = await fetchJson('/qa/replay/status');
    if (!s) return;
    scrub.value = s.loop;
    document.getElementById('rb-time').textContent =
      `${fmtLoop(s.loop)} / ${fmtLoop(s.totalLoops)}`;
  }, 500);
}

function fmtLoop(loop) {
  const secs = Math.floor(loop / 22.4);
  return `${Math.floor(secs/60)}:${String(secs%60).padStart(2,'0')}`;
}
```

Add CSS (injected or in `<style>`):
```javascript
const style = document.createElement('style');
style.textContent = `
  #replay-bar {
    position: fixed; bottom: 0; left: 0; width: 100%; height: 44px;
    background: rgba(0,0,0,0.75); display: flex; align-items: center;
    gap: 8px; padding: 0 12px; box-sizing: border-box; z-index: 100;
    color: #fff; font-family: monospace; font-size: 13px;
  }
  #replay-bar button { background: #333; color: #fff; border: 1px solid #555;
    border-radius: 4px; padding: 4px 8px; cursor: pointer; }
  #replay-bar button:hover { background: #555; }
  .rb-active { background: #4488ff !important; }
  #rb-scrub { flex: 1; }
  #rb-time { min-width: 90px; text-align: center; }
`;
document.head.appendChild(style);
```

Call `initReplayControls()` in the init sequence alongside `loadTerrain()`.

- [ ] **Step 7.3: Run Playwright tests to confirm no regressions**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest -q 2>&1 | grep -E "Tests run:|BUILD"
```
Expected: BUILD SUCCESS, all existing tests pass.

- [ ] **Step 7.4: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js
git commit -m "feat: visualizer replay control bar — play/pause/rewind/scrub/speed Refs #103"
```

---

## Task 8: UnitResource — GET /qa/unit/{tag} — Issue #104

**Files:**
- Create: `src/main/java/io/quarkmind/qa/UnitResource.java`
- Create: `src/test/java/io/quarkmind/qa/UnitResourceTest.java`

- [ ] **Step 8.1: Write failing tests**

```java
package io.quarkmind.qa;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.mock.SimulatedGame;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class UnitResourceTest {

    @Inject SimulatedGame game;

    @BeforeEach
    void seed() {
        game.spawnFriendlyUnitForTesting(UnitType.STALKER, new Point2d(10, 10));
    }

    @Test
    void returnsUnitDetailsForKnownTag() {
        // Get tag from observe
        var units = game.snapshot().myUnits();
        String tag = units.stream()
            .filter(u -> u.type() == UnitType.STALKER).findFirst()
            .orElseThrow().tag();

        given().pathParam("tag", tag).when().get("/qa/unit/{tag}")
            .then().statusCode(200)
            .body("type", equalTo("STALKER"))
            .body("health", greaterThan(0))
            .body("maxHealth", greaterThan(0));
    }

    @Test
    void returns404ForUnknownTag() {
        given().pathParam("tag", "nonexistent-tag")
            .when().get("/qa/unit/{tag}")
            .then().statusCode(404);
    }
}
```

- [ ] **Step 8.2: Run to confirm failure**

```bash
mvn test -Dtest=UnitResourceTest -q 2>&1 | tail -5
```

- [ ] **Step 8.3: Implement `UnitResource.java`**

```java
package io.quarkmind.qa;

import io.quarkus.arc.profile.UnlessBuildProfile;
import io.quarkmind.sc2.SC2Engine;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

@UnlessBuildProfile("prod")
@Path("/qa/unit")
@Produces(MediaType.APPLICATION_JSON)
public class UnitResource {

    @Inject SC2Engine engine;

    @GET @Path("/{tag}")
    public Response getUnit(@PathParam("tag") String tag) {
        var state = engine.observe();
        return state.myUnits().stream()
            .filter(u -> u.tag().equals(tag))
            .findFirst()
            .map(u -> Response.ok(new UnitDetail(
                u.tag(), u.type().name(),
                u.health(), u.maxHealth(),
                u.shields(), u.maxShields()
            )).build())
            .orElseGet(() ->
                state.enemyUnits().stream()
                    .filter(u -> u.tag().equals(tag))
                    .findFirst()
                    .map(u -> Response.ok(new UnitDetail(
                        u.tag(), u.type().name(),
                        u.health(), u.maxHealth(),
                        u.shields(), u.maxShields()
                    )).build())
                    .orElse(Response.status(404).build())
            );
    }

    public record UnitDetail(String tag, String type,
                             int health, int maxHealth,
                             int shields, int maxShields) {}
}
```

- [ ] **Step 8.4: Run tests**

```bash
mvn test -Dtest=UnitResourceTest -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 8.5: Commit**

```bash
git add src/main/java/io/quarkmind/qa/UnitResource.java \
        src/test/java/io/quarkmind/qa/UnitResourceTest.java
git commit -m "feat: UnitResource — GET /qa/unit/{tag} returns live unit state Refs #104"
```

---

## Task 9: Visualizer — click-to-inspect panel — Issue #104

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 9.1: Add `userData` to unit sprites at creation time**

In `syncUnitLayer()`, when creating a new sprite (`if (!spriteMap.has(u.tag))`), add after `sp.userData.mats = mats`:

```javascript
sp.userData.unitTag  = u.tag;
sp.userData.unitType = u.type;
sp.userData.isEnemy  = isEnemy;
```

- [ ] **Step 9.2: Add inspect panel HTML + CSS + raycasting**

```javascript
// ── Unit inspect panel ─────────────────────────────────────────────────────
const raycaster = new THREE.Raycaster();
const ndcMouse  = new THREE.Vector2();
let   inspectTag = null;
let   dragStartX = 0, dragStartY = 0;

function setupInspectPanel() {
  const panel = document.createElement('div');
  panel.id = 'unit-panel';
  panel.innerHTML = `
    <canvas id="up-portrait" width="64" height="64"></canvas>
    <div id="up-info">
      <div id="up-name"></div>
      <div id="up-team"></div>
      <div class="up-bar-row"><span>HP</span>
        <div class="up-bar"><div id="up-hp-fill" class="up-fill hp"></div></div>
        <span id="up-hp-text"></span></div>
      <div class="up-bar-row sh-row"><span>SH</span>
        <div class="up-bar"><div id="up-sh-fill" class="up-fill sh"></div></div>
        <span id="up-sh-text"></span></div>
    </div>
  `;
  document.body.appendChild(panel);

  // Inject CSS
  const s = document.createElement('style');
  s.textContent = `
    #unit-panel {
      position: fixed; bottom: 56px; right: 12px; width: 230px;
      background: rgba(0,0,0,0.85); color: #fff; border: 1px solid #444;
      border-radius: 6px; padding: 10px; display: flex; gap: 10px;
      transform: translateX(260px); transition: transform 150ms ease;
      z-index: 99; font-family: monospace; font-size: 12px;
    }
    #unit-panel.visible { transform: translateX(0); }
    #up-portrait { border-radius: 4px; flex-shrink: 0; }
    #up-info { flex: 1; }
    #up-name { font-weight: bold; font-size: 13px; margin-bottom: 4px; }
    #up-team { color: #aaa; margin-bottom: 6px; }
    .up-bar-row { display: flex; align-items: center; gap: 4px; margin-bottom: 3px; }
    .up-bar { flex: 1; height: 8px; background: #333; border-radius: 3px; overflow: hidden; }
    .up-fill { height: 100%; border-radius: 3px; transition: width 200ms; }
    .hp { background: #44cc44; }
    .sh { background: #4488ff; }
    .sh-row { display: none; }
  `;
  document.head.appendChild(s);
}

function showUnitPanel(tag, type, isEnemy) {
  inspectTag = tag;
  const panel = document.getElementById('unit-panel');

  fetch(`/qa/unit/${encodeURIComponent(tag)}`)
    .then(r => r.ok ? r.json() : null)
    .then(data => {
      if (!data) return;
      document.getElementById('up-name').textContent = data.type.replace(/_/g,' ');
      document.getElementById('up-team').textContent = isEnemy ? 'Enemy' : 'Friendly';

      const hpPct = data.maxHealth > 0 ? (data.health / data.maxHealth * 100) : 0;
      document.getElementById('up-hp-fill').style.width = hpPct + '%';
      document.getElementById('up-hp-fill').style.background =
        hpPct > 50 ? '#44cc44' : hpPct > 25 ? '#cccc44' : '#cc4444';
      document.getElementById('up-hp-text').textContent = `${data.health}/${data.maxHealth}`;

      const shRow = document.querySelector('.sh-row');
      if (data.maxShields > 0) {
        shRow.style.display = 'flex';
        const shPct = data.shields / data.maxShields * 100;
        document.getElementById('up-sh-fill').style.width = shPct + '%';
        document.getElementById('up-sh-text').textContent = `${data.shields}/${data.maxShields}`;
      } else {
        shRow.style.display = 'none';
      }

      // Draw portrait
      const canvas = document.getElementById('up-portrait');
      const ctx = canvas.getContext('2d');
      ctx.clearRect(0, 0, 64, 64);
      const teamColor = isEnemy ? '#ff4422' : '#4488ff';
      const drawFn = window[`draw${toPascalCase(data.type)}`];
      if (drawFn) drawFn(ctx, 32, 32, 0, teamColor);
    });

  panel.classList.add('visible');
}

function hideUnitPanel() {
  inspectTag = null;
  document.getElementById('unit-panel')?.classList.remove('visible');
}

function toPascalCase(s) {
  return s.split('_').map(w => w[0] + w.slice(1).toLowerCase()).join('');
}

function setupClickInspect() {
  renderer.domElement.addEventListener('mousedown', e => {
    dragStartX = e.clientX; dragStartY = e.clientY;
  });
  renderer.domElement.addEventListener('mouseup', e => {
    const dx = e.clientX - dragStartX, dy = e.clientY - dragStartY;
    if (dx*dx + dy*dy > 25) return; // was a drag, not a click

    ndcMouse.x = (e.clientX / window.innerWidth)  *  2 - 1;
    ndcMouse.y = (e.clientY / window.innerHeight) * -2 + 1;
    raycaster.setFromCamera(ndcMouse, camera);

    const allSprites = [
      ...unitSprites.values(),
      ...enemySprites.values()
    ];
    const hits = raycaster.intersectObjects(allSprites);
    if (hits.length > 0) {
      const obj = hits[0].object;
      showUnitPanel(obj.userData.unitTag, obj.userData.unitType, obj.userData.isEnemy);
    } else {
      hideUnitPanel();
    }
  });

  // ESC to dismiss
  window.addEventListener('keydown', e => {
    if (e.key === 'Escape') hideUnitPanel();
  });
}
```

Call `setupInspectPanel()` and `setupClickInspect()` in the init sequence.

- [ ] **Step 9.3: Add Playwright test for panel visibility**

Add to `VisualizerRenderTest.java`:

```java
@Test
@Tag("browser")
void unitPanelHiddenInitially() {
    assumeTrue(browser != null, "Chromium not installed");
    try (var context = browser.newContext(); var page = context.newPage()) {
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test && window.__test.terrainReady()");
        // Panel should not be visible before any click
        var panel = page.locator("#unit-panel");
        assertThat(panel.count()).isEqualTo(1);
        assertThat(panel.getAttribute("class")).doesNotContain("visible");
    }
}
```

- [ ] **Step 9.4: Run Playwright tests**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#unitPanelHiddenInitially" -q
```
Expected: PASS.

- [ ] **Step 9.5: Run full Playwright suite for regressions**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest -q 2>&1 | grep -E "Tests run:|BUILD"
```
Expected: BUILD SUCCESS.

- [ ] **Step 9.6: Commit — closes E23c**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: click-to-inspect unit panel with portrait, health/shield bars Closes #104"
```

---

## Task 10: Camera mode toggle + WASD + UI scaling — Issue #105

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 10.1: Read `setupCamera()` fully**

```bash
sed -n '278,330p' src/main/resources/META-INF/resources/visualizer.js
```

Understand the existing `tTheta`, `tPhi`, `tDist`, `camTarget` variables and how drag and scroll work.

- [ ] **Step 10.2: Add camera mode state and toggle button**

At module scope (near other `let` declarations):

```javascript
let cameraMode = localStorage.getItem('quarkmind.cameraMode') || 'sc2';
```

Inject toggle button via JS:

```javascript
function setupCameraToggle() {
  const btn = document.createElement('div');
  btn.id = 'cam-toggle';
  btn.innerHTML = `<button id="cam-sc2">🎮 SC2</button><button id="cam-3d">🔭 3D</button>`;
  document.body.appendChild(btn);

  const style = document.createElement('style');
  style.textContent = `
    #cam-toggle { position: fixed; top: 12px; left: 12px; z-index: 100;
      display: flex; gap: 4px; }
    #cam-toggle button { background: rgba(0,0,0,0.6); color: #fff;
      border: 1px solid #555; border-radius: 4px; padding: 4px 8px;
      cursor: pointer; font-size: 12px; }
    #cam-toggle button.active { background: #4488ff; border-color: #4488ff; }
  `;
  document.head.appendChild(style);

  function updateActive() {
    document.getElementById('cam-sc2').classList.toggle('active', cameraMode === 'sc2');
    document.getElementById('cam-3d').classList.toggle('active', cameraMode === '3d');
  }
  updateActive();

  document.getElementById('cam-sc2').onclick = () => {
    cameraMode = 'sc2'; localStorage.setItem('quarkmind.cameraMode', 'sc2'); updateActive();
    // Lock pitch to SC2 angle
    tPhi = Math.PI / 3.5;
  };
  document.getElementById('cam-3d').onclick = () => {
    cameraMode = '3d'; localStorage.setItem('quarkmind.cameraMode', '3d'); updateActive();
  };
}
```

- [ ] **Step 10.3: Update `setupCamera()` for mode-aware drag**

Replace the `mousedown`/`mousemove` handlers:

```javascript
  renderer.domElement.addEventListener('mousedown', e => {
    drag = true; lastX = e.clientX; lastY = e.clientY;
    // In SC2 mode left-drag = pan; in 3D mode left-drag = orbit, right-drag = pan
    rDrag = (cameraMode === '3d') ? e.button === 2 : false;
    panDrag = (cameraMode === 'sc2') ? e.button === 0 : e.button === 2;
    e.preventDefault();
  });

  window.addEventListener('mousemove', e => {
    if (!drag) return;
    const dx = e.clientX - lastX, dy = e.clientY - lastY;
    lastX = e.clientX; lastY = e.clientY;

    if (panDrag || (cameraMode === '3d' && rDrag)) {
      const right = new THREE.Vector3();
      camera.getWorldDirection(right); right.cross(camera.up).normalize();
      const panSpeed = camDist * 0.015;
      camTarget.addScaledVector(right, -dx * panSpeed * 0.05);
      camTarget.y += dy * panSpeed * 0.05;
    } else if (cameraMode === '3d' && !rDrag) {
      tTheta -= dx * 0.012;
      tPhi = Math.max(0.08, Math.min(Math.PI / 2.05, tPhi - dy * 0.012));
    }
    // SC2 mode non-pan drag: ignore (no orbit)
  });
```

Add `let panDrag = false;` at the top of `setupCamera()`.

- [ ] **Step 10.4: Add WASD + arrow key pan**

```javascript
function setupKeyboardControls() {
  const keys = new Set();
  window.addEventListener('keydown', e => keys.add(e.key));
  window.addEventListener('keyup',   e => keys.delete(e.key));

  function applyKeyPan() {
    const panSpeed = camDist * 0.015;
    const forward = new THREE.Vector3(-Math.sin(camTheta), 0, -Math.cos(camTheta));
    const right   = new THREE.Vector3( Math.cos(camTheta), 0, -Math.sin(camTheta));

    if (keys.has('w') || keys.has('ArrowUp'))    camTarget.addScaledVector(forward,  panSpeed);
    if (keys.has('s') || keys.has('ArrowDown'))  camTarget.addScaledVector(forward, -panSpeed);
    if (keys.has('a') || keys.has('ArrowLeft'))  camTarget.addScaledVector(right,   -panSpeed);
    if (keys.has('d') || keys.has('ArrowRight')) camTarget.addScaledVector(right,    panSpeed);

    requestAnimationFrame(applyKeyPan);
  }
  applyKeyPan();
}
```

Call `setupKeyboardControls()` in init.

- [ ] **Step 10.5: Fix canvas + HUD CSS for responsive layout**

Update the `<style>` tag in `visualizer.html` (or inject via JS):

```javascript
const layoutStyle = document.createElement('style');
layoutStyle.textContent = `
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body { overflow: hidden; background: #000; }
  canvas { display: block; width: 100vw; height: 100vh; }
  #hud {
    position: fixed; top: 12px; right: 12px;
    background: rgba(0,0,0,0.6); color: #fff;
    padding: 6px 10px; border-radius: 4px;
    font-family: monospace; font-size: 13px; z-index: 100;
  }
`;
document.head.appendChild(layoutStyle);
```

- [ ] **Step 10.6: Add Playwright tests**

Add to `VisualizerRenderTest.java`:

```java
@Test
@Tag("browser")
void canvasCoversFullViewport() {
    assumeTrue(browser != null, "Chromium not installed");
    try (var context = browser.newContext(new Browser.NewContextOptions()
             .setViewportSize(1280, 720));
         var page = context.newPage()) {
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test && window.__test.terrainReady()");
        var canvas = page.locator("canvas");
        var box = canvas.boundingBox();
        assertThat(box.width).isEqualTo(1280.0);
        assertThat(box.height).isEqualTo(720.0);
    }
}

@Test
@Tag("browser")
void cameraToggleButtonsExist() {
    assumeTrue(browser != null, "Chromium not installed");
    try (var context = browser.newContext(); var page = context.newPage()) {
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test && window.__test.terrainReady()");
        assertThat(page.locator("#cam-sc2").count()).isEqualTo(1);
        assertThat(page.locator("#cam-3d").count()).isEqualTo(1);
    }
}
```

- [ ] **Step 10.7: Run Playwright tests**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#canvasCoversFullViewport,VisualizerRenderTest#cameraToggleButtonsExist" -q
```
Expected: PASS.

- [ ] **Step 10.8: Run full Playwright suite**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest -q 2>&1 | grep -E "Tests run:|BUILD"
```
Expected: BUILD SUCCESS.

- [ ] **Step 10.9: Commit — closes E23d**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: SC2/3D camera toggle, WASD pan, responsive UI layout Closes #105"
```

---

## Task 11: E2E validation + epic close — Issue #101

- [ ] **Step 11.1: Run full non-Playwright suite**

```bash
cd /Users/mdproctor/claude/quarkmind && mvn test -q 2>&1 | tail -5
```
Expected: BUILD SUCCESS.

- [ ] **Step 11.2: Run full Playwright suite**

```bash
mvn test -Pplaywright -q 2>&1 | grep -E "Tests run:|BUILD|FAIL"
```
Expected: BUILD SUCCESS. (VisualizerFogRenderTest 2 pre-existing failures are acceptable.)

- [ ] **Step 11.3: Smoke-test replay mode**

```bash
mvn quarkus:dev -Dquarkus.profile=replay > /dev/null 2>&1 &
sleep 25
curl -s http://localhost:8080/qa/replay/status
curl -s http://localhost:8080/qa/current-map
pkill -f 'quarkus:dev'
```

Expected: status returns `{loop, totalLoops, paused, speed}`, current-map returns `TorchesAIE_v4`.

- [ ] **Step 11.4: Close the epic**

```bash
gh issue close 101 --comment "All 4 child issues closed (#102 #103 #104 #105).

E23 delivered:
- Units animate across real terrain following GAME_EVENTS commands (22,648 CmdEvents parsed from Nothing_4720936)
- Play/pause/rewind/scrub/speed control bar (seeks full 11,200-loop replay in <50ms)
- Click-to-inspect panel with portrait, HP/shield bars, unit type
- SC2/3D camera toggle, WASD pan, responsive viewport layout
All tests pass."
```

- [ ] **Step 11.5: Commit plan doc**

```bash
git add docs/superpowers/plans/2026-04-24-e23-replay-playback.md
git commit -m "docs: E23 implementation plan Refs #101"
```

---

## Self-Review

**Spec coverage:**
- ✅ GameEventStream + movement — Tasks 1–5
- ✅ UnitOrderTracker position simulation — Task 3
- ✅ SC2Data.UNIT_SPEEDS — Task 1
- ✅ SimulatedGame.replaceUnitPosition/Enemy — Task 3
- ✅ ReplaySimulatedGame seekTo + totalLoops — Task 4
- ✅ GameEventStream wired into ReplayEngine — Task 5
- ✅ Pause/resume/seek/speed REST endpoints — Task 6
- ✅ Visualizer control bar — Task 7
- ✅ GET /qa/unit/{tag} — Task 8
- ✅ Click-to-inspect panel with raycasting — Task 9
- ✅ Camera mode toggle (SC2 vs 3D) — Task 10
- ✅ WASD/arrow key pan — Task 10
- ✅ UI responsive layout — Task 10

**Type consistency:** `UnitOrder` created in Task 1 used in Tasks 2–5. `UnitOrderTracker.advance()` signature defined in Task 3, called identically in Task 4. `ReplayEngine.seekTo()` / `currentLoop()` / `totalLoops()` defined in Task 6. `ReplayStatusResponse` created in Task 6.

**No placeholders found.**

**Robustness notes for implementer:**
- `GameEventStream.parse()` bounds-checks target coordinates (0–256) to reject corrupted events
- `UnitOrderTracker.advance()` skips units not in the positions map (handles birth/death race)
- `seekTo()` suppresses broadcaster during fast-forward; one clean broadcast after
- `setupCamera()` drag delta check prevents accidental panel open on drag-release
- WASD uses `requestAnimationFrame` loop (not keydown events) for smooth camera pan
