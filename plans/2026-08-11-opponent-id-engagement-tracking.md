# Opponent Identity & Engagement Tracking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #271 — SC2 opponent identity extraction
**Issue group:** #271, #270

**Goal:** Extract opponent identity from SC2 protocol at game start and track per-engagement outcomes via army value delta FSM.

**Architecture:** GameStarted event enrichment carries opponent identity; GameStateTranslator observes it via AtomicReference. MomentDetectionTask gains a 3-state battle FSM (IDLE/IN_BATTLE/QUIESCENT) driven by own army value delta, emitting BATTLE_ENDED with typed EngagementOutcome metadata. SC2CbrRetentionObserver extracts engagement aggregates into CBR features.

**Tech Stack:** Quarkus CDI, Drools Rule Units, Java records, AtomicReference

## Global Constraints

- 1v1 mode only — single opponent assumed throughout
- Domain model (`domain/`) must remain plain Java — no CDI imports
- `EngagementOutcome` lives in `plugin/summarisation/` (not domain)
- All commits reference an issue: `Refs #271` or `Refs #270`
- No `@QuarkusTest` for tests that can be plain JUnit

---

### Task 1: GameStarted Enrichment & Opponent Identity (#271)

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/GameStarted.java`
- Modify: `src/main/java/io/quarkmind/agent/GameStateTranslator.java`
- Modify: `src/main/java/io/quarkmind/agent/AgentOrchestrator.java:92`
- Modify: `src/main/resources/application.properties:40-43`
- Create: `src/test/java/io/quarkmind/agent/GameStateTranslatorTest.java` (extend existing)
- Test: `src/test/java/io/quarkmind/agent/GameStateTranslatorTest.java`

**Interfaces:**
- Consumes: `SC2Data` (no changes), `QuarkMindCaseFile.OPPONENT_ID` (existing key)
- Produces: `GameStarted(String opponentRace, String opponentType, String opponentDifficulty)` — enriched event consumed by GameStateTranslator and all existing observers

- [ ] **Step 1: Write failing test — AI opponent identity hashing**

```java
// In GameStateTranslatorTest.java
@Test
void toMap_usesOpponentIdFromGameStartedEvent() {
    var translator = new GameStateTranslator();
    translator.onGameStarted(new GameStarted("ZERG", "COMPUTER", "VeryHard"));

    Map<String, Object> map = translator.toMap(minimalGameState());
    String opponentId = (String) map.get(QuarkMindCaseFile.OPPONENT_ID);

    assertThat(opponentId).isNotEqualTo("unknown");
    assertThat(opponentId).hasSize(64); // SHA-256 hex
}

@Test
void toMap_defaultsToUnknownBeforeGameStarted() {
    var translator = new GameStateTranslator();
    Map<String, Object> map = translator.toMap(minimalGameState());
    assertThat(map.get(QuarkMindCaseFile.OPPONENT_ID)).isEqualTo("unknown");
}

@Test
void computeOpponentId_aiOpponent_hashesRaceAndDifficulty() {
    String id = GameStateTranslator.computeOpponentId("ZERG", "COMPUTER", "VeryHard");
    String expected = GameStateTranslator.computeOpponentId("ZERG", "COMPUTER", "VeryHard");
    assertThat(id).isEqualTo(expected); // deterministic
    assertThat(id).hasSize(64);
}

@Test
void computeOpponentId_pvpOpponent_hashesPlayerId() {
    String id = GameStateTranslator.computeOpponentId("PROTOSS", "PARTICIPANT", null);
    assertThat(id).hasSize(64);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=GameStateTranslatorTest -q`
Expected: FAIL — `GameStarted` constructor doesn't accept args, `computeOpponentId` doesn't exist

- [ ] **Step 3: Enrich GameStarted record**

Replace `GameStarted.java` content:

```java
package io.quarkmind.sc2;

public record GameStarted(String opponentRace, String opponentType, String opponentDifficulty) {
    public GameStarted() {
        this("UNKNOWN", "UNKNOWN", null);
    }
}
```

The no-arg constructor preserves backward compatibility — all 15+ existing `@Observes GameStarted` handlers and the `new GameStarted()` call in `AgentOrchestrator` continue to work without changes.

- [ ] **Step 4: Refactor GameStateTranslator**

Replace `GameStateTranslator.java` content:

```java
package io.quarkmind.agent;

import io.quarkmind.domain.GameState;
import io.quarkmind.domain.Unit;
import io.quarkmind.sc2.GameStarted;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.HashMap;
import java.util.HexFormat;
import java.util.List;
import java.util.Map;
import java.util.concurrent.atomic.AtomicReference;

@ApplicationScoped
public class GameStateTranslator {

    private final AtomicReference<String> opponentId = new AtomicReference<>("unknown");

    void onGameStarted(@Observes GameStarted event) {
        opponentId.set(computeOpponentId(event.opponentRace(), event.opponentType(), event.opponentDifficulty()));
    }

    static String computeOpponentId(String race, String playerType, String difficulty) {
        if ("UNKNOWN".equals(race) || "UNKNOWN".equals(playerType)) {
            return "unknown";
        }
        String input = "COMPUTER".equals(playerType)
                ? race + ":" + difficulty
                : race + ":PARTICIPANT";
        return sha256(input);
    }

    public Map<String, Object> toMap(GameState state) {
        Map<String, Object> data = new HashMap<>();
        data.put(QuarkMindCaseFile.GAME_STATE, state);
        data.put(QuarkMindCaseFile.MINERALS, state.minerals());
        data.put(QuarkMindCaseFile.VESPENE, state.vespene());
        data.put(QuarkMindCaseFile.SUPPLY_CAP, state.supply());
        data.put(QuarkMindCaseFile.SUPPLY_USED, state.supplyUsed());
        data.put(QuarkMindCaseFile.GAME_FRAME, state.gameFrame());
        data.put(QuarkMindCaseFile.READY, Boolean.TRUE);
        data.put(QuarkMindCaseFile.OPPONENT_ID, opponentId.get());

        List<Unit> workers = state.myUnits().stream()
                                  .filter(u -> u.type().isWorker()).toList();
        List<Unit> army = state.myUnits().stream()
                               .filter(u -> !u.type().isWorker()).toList();

        data.put(QuarkMindCaseFile.WORKERS, workers);
        data.put(QuarkMindCaseFile.ARMY, army);
        data.put(QuarkMindCaseFile.MY_BUILDINGS, state.myBuildings());
        data.put(QuarkMindCaseFile.GEYSERS, state.geysers());
        data.put(QuarkMindCaseFile.ENEMY_UNITS, state.enemyUnits());
        data.put(QuarkMindCaseFile.RESOURCE_BUDGET, new ResourceBudget(state.minerals(), state.vespene()));
        return data;
    }

    private static String sha256(String input) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(input.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hash);
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException(e);
        }
    }
}
```

- [ ] **Step 5: Update application.properties — remove opponent.id entries**

Remove these four lines from `application.properties`:

```
%mock.quarkmind.opponent.id=mock-opponent
%emulated.quarkmind.opponent.id=PROTOSS_4GATE
%emulated-sc2.quarkmind.opponent.id=PROTOSS_4GATE
%test.quarkmind.opponent.id=mock-opponent
```

- [ ] **Step 6: Update AgentOrchestrator.startGame() to fire enriched GameStarted**

At line 92 in `AgentOrchestrator.java`, the current code is:
```java
gameStartedEvent.fire(new GameStarted());
```

Replace with:
```java
gameStartedEvent.fire(new GameStarted(
        engine.opponentRace(), engine.opponentType(), engine.opponentDifficulty()));
```

This requires adding three methods to the `SC2Engine` interface. For the `SC2Engine` seam interface, add default methods returning "UNKNOWN":

```java
default String opponentRace()       { return "UNKNOWN"; }
default String opponentType()       { return "UNKNOWN"; }
default String opponentDifficulty() { return null; }
```

For `MockEngine` / `SimulatedGame` — override to return mock values:
```java
@Override public String opponentRace()       { return "PROTOSS"; }
@Override public String opponentType()       { return "COMPUTER"; }
@Override public String opponentDifficulty() { return "VeryEasy"; }
```

For `EmulatedEngine` — override to return the `EnemyStrategy` name as race:
```java
@Override public String opponentRace()       { return enemyBehavior.strategy().race().name(); }
@Override public String opponentType()       { return "COMPUTER"; }
@Override public String opponentDifficulty() { return enemyBehavior.strategy().name(); }
```

For `RealSC2Engine` — the fields are set in `SC2BotAgent.onGameStart(ResponseGameInfo)` and stored for retrieval. This is the actual protocol extraction point.

- [ ] **Step 7: Run all tests to verify nothing broke**

Run: `mvn test -q`
Expected: All tests pass. The no-arg `GameStarted()` constructor keeps existing code working. The `@ConfigProperty` removal is safe because all tests that relied on it either set `OPPONENT_ID` directly on CaseFile or will now use `GameStarted` enrichment.

- [ ] **Step 8: Fix any test that relied on the old @ConfigProperty injection**

Check `GameStateTranslatorTest.includesOpponentId` — this test constructed `GameStateTranslator` with the config property constructor. Update to:

```java
@Test
void includesOpponentId() {
    var translator = new GameStateTranslator();
    translator.onGameStarted(new GameStarted("PROTOSS", "COMPUTER", "VeryEasy"));
    Map<String, Object> map = translator.toMap(minimalGameState());
    assertThat(map.get(QuarkMindCaseFile.OPPONENT_ID)).isNotNull();
    assertThat((String) map.get(QuarkMindCaseFile.OPPONENT_ID)).hasSize(64);
}
```

- [ ] **Step 9: Run tests again**

Run: `mvn test -q`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/GameStarted.java \
        src/main/java/io/quarkmind/agent/GameStateTranslator.java \
        src/main/java/io/quarkmind/agent/AgentOrchestrator.java \
        src/main/resources/application.properties \
        src/test/java/io/quarkmind/agent/GameStateTranslatorTest.java
git commit -m "feat(#271): extract opponent identity from GameStarted event

GameStateTranslator observes GameStarted for opponent race/type/difficulty
instead of static @ConfigProperty. AI opponents hashed as race:difficulty,
PvP opponents hashed as player ID. Thread-safe via AtomicReference.

Refs #271"
```

---

### Task 2: EngagementOutcome Record (#270 foundation)

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/summarisation/EngagementOutcome.java`
- Create: `src/test/java/io/quarkmind/plugin/summarisation/EngagementOutcomeTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `EngagementOutcome(long startFrame, long endFrame, int ownUnitsLost, int enemyUnitsLost, int ownValueLost, int enemyValueLost, double unitTradeRatio, Outcome outcome)` — used by Task 3 (MomentDetectionTask) and Task 4 (CBR enrichment)

- [ ] **Step 1: Write failing tests — outcome determination**

```java
package io.quarkmind.plugin.summarisation;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class EngagementOutcomeTest {

    @Test
    void won_whenEnemyLostMoreValue() {
        var outcome = EngagementOutcome.of(100, 500, 2, 5, 200, 300);
        assertThat(outcome.outcome()).isEqualTo(EngagementOutcome.Outcome.WON);
        assertThat(outcome.unitTradeRatio()).isEqualTo(1.5);
    }

    @Test
    void lost_whenOwnLostMoreValue() {
        var outcome = EngagementOutcome.of(100, 500, 5, 2, 400, 100);
        assertThat(outcome.outcome()).isEqualTo(EngagementOutcome.Outcome.LOST);
        assertThat(outcome.unitTradeRatio()).isEqualTo(0.25);
    }

    @Test
    void even_whenValueLossWithinMargin() {
        var outcome = EngagementOutcome.of(100, 500, 3, 3, 200, 220);
        assertThat(outcome.outcome()).isEqualTo(EngagementOutcome.Outcome.EVEN);
    }

    @Test
    void won_whenNoOwnLosses() {
        var outcome = EngagementOutcome.of(100, 500, 0, 3, 0, 300);
        assertThat(outcome.outcome()).isEqualTo(EngagementOutcome.Outcome.WON);
        assertThat(outcome.unitTradeRatio()).isEqualTo(Double.MAX_VALUE);
    }

    @Test
    void even_whenNoLossesOnEitherSide() {
        var outcome = EngagementOutcome.of(100, 500, 0, 0, 0, 0);
        assertThat(outcome.outcome()).isEqualTo(EngagementOutcome.Outcome.EVEN);
        assertThat(outcome.unitTradeRatio()).isEqualTo(0.0);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=EngagementOutcomeTest -q`
Expected: FAIL — class doesn't exist

- [ ] **Step 3: Implement EngagementOutcome**

```java
package io.quarkmind.plugin.summarisation;

public record EngagementOutcome(
        long startFrame,
        long endFrame,
        int ownUnitsLost,
        int enemyUnitsLost,
        int ownValueLost,
        int enemyValueLost,
        double unitTradeRatio,
        Outcome outcome
) {
    public enum Outcome { WON, LOST, EVEN }

    static final double WIN_MARGIN = 1.2;

    public static EngagementOutcome of(long startFrame, long endFrame,
                                       int ownUnitsLost, int enemyUnitsLost,
                                       int ownValueLost, int enemyValueLost) {
        double ratio;
        if (ownValueLost == 0 && enemyValueLost == 0) {
            ratio = 0.0;
        } else if (ownValueLost == 0) {
            ratio = Double.MAX_VALUE;
        } else {
            ratio = (double) enemyValueLost / ownValueLost;
        }

        Outcome outcome;
        if (ownValueLost == 0 && enemyValueLost == 0) {
            outcome = Outcome.EVEN;
        } else if (ownValueLost == 0) {
            outcome = Outcome.WON;
        } else if (enemyValueLost > ownValueLost * WIN_MARGIN) {
            outcome = Outcome.WON;
        } else if (ownValueLost > enemyValueLost * WIN_MARGIN) {
            outcome = Outcome.LOST;
        } else {
            outcome = Outcome.EVEN;
        }

        return new EngagementOutcome(startFrame, endFrame,
                ownUnitsLost, enemyUnitsLost, ownValueLost, enemyValueLost,
                ratio, outcome);
    }
}
```

- [ ] **Step 4: Run tests**

Run: `mvn test -Dtest=EngagementOutcomeTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/quarkmind/plugin/summarisation/EngagementOutcome.java \
        src/test/java/io/quarkmind/plugin/summarisation/EngagementOutcomeTest.java
git commit -m "feat(#270): add EngagementOutcome record with outcome determination

Typed record for per-engagement battle results. Factory method computes
unitTradeRatio and WON/LOST/EVEN outcome with 20% margin threshold.

Refs #270"
```

---

### Task 3: Battle FSM in MomentDetectionTask (#270 core)

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/summarisation/MomentDetectionTask.java`
- Modify: `src/main/java/io/quarkmind/plugin/summarisation/MomentDetectionTask.java` — `requires()` to add ARMY and ENEMY_UNITS
- Create or extend: `src/test/java/io/quarkmind/plugin/summarisation/MomentDetectionTaskTest.java`

**Interfaces:**
- Consumes: `EngagementOutcome.of(...)` from Task 2, `QuarkMindCaseFile.ARMY`, `QuarkMindCaseFile.ENEMY_UNITS`, `SC2Data.mineralCost()`, `SC2Data.gasCost()`
- Produces: `GameMoment(BATTLE_ENDED, frame, Map.of("engagement", EngagementOutcome))` on the L2 moment bus

- [ ] **Step 1: Write failing test — IDLE to IN_BATTLE on army value drop**

```java
package io.quarkmind.plugin.summarisation;

import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.SC2Data;
import io.quarkmind.domain.Unit;
import io.quarkmind.domain.UnitType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class MomentDetectionBattleTest {

    private TestMomentDetectionTask task;
    private List<GameMoment> emittedMoments;

    @BeforeEach
    void setUp() {
        emittedMoments = new ArrayList<>();
        task = new TestMomentDetectionTask(emittedMoments);
    }

    @Test
    void battleStartsOnArmyValueDrop() {
        List<Unit> army10 = stalkers(10);
        List<Unit> enemy5 = zealots(5);

        // First tick establishes baseline
        task.tickBattle(100, army10, enemy5);
        assertThat(task.battleState()).isEqualTo(MomentDetectionTask.BattleState.IDLE);

        // Army drops >15% — battle detected
        List<Unit> army7 = stalkers(7); // lost 3 of 10 = 30% drop
        task.tickBattle(200, army7, enemy5);
        assertThat(task.battleState()).isEqualTo(MomentDetectionTask.BattleState.IN_BATTLE);
    }

    // Helper: create N Stalkers at origin
    static List<Unit> stalkers(int n) {
        List<Unit> units = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            units.add(new Unit("s" + i, UnitType.STALKER, new Point2d(10, 10),
                    100, 80, 0, false, false));
        }
        return units;
    }

    static List<Unit> zealots(int n) {
        List<Unit> units = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            units.add(new Unit("z" + i, UnitType.ZEALOT, new Point2d(20, 20),
                    100, 0, 0, false, false));
        }
        return units;
    }
}
```

The `TestMomentDetectionTask` is a test subclass that exposes `tickBattle()` to drive the FSM directly without Drools/CDI:

```java
// Inner class or separate file in the same test package
class TestMomentDetectionTask extends MomentDetectionTask {
    private final List<GameMoment> emitted;

    TestMomentDetectionTask(List<GameMoment> emitted) {
        super(null); // no Drools needed for battle FSM tests
        this.emitted = emitted;
    }

    void tickBattle(long frame, List<Unit> ownArmy, List<Unit> enemyUnits) {
        updateBattleFSM(frame, ownArmy, enemyUnits, emitted);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=MomentDetectionBattleTest -q`
Expected: FAIL — `BattleState`, `tickBattle()`, `updateBattleFSM()` don't exist

- [ ] **Step 3: Add battle FSM state to MomentDetectionTask**

Add these fields after `lastSupplyBlockFrame` (line 41):

```java
enum BattleState { IDLE, IN_BATTLE, QUIESCENT }
private static final double BATTLE_START_THRESHOLD = 0.15;
private static final double BATTLE_STABLE_THRESHOLD = 0.05;
static final int QUIESCENCE_FRAMES = 224;

private BattleState battleState = BattleState.IDLE;
private long quiescenceStartFrame = -1;
private long battleStartFrame = -1;
private int previousOwnArmyValue = 0;
private int battleStartOwnValue = 0;
private int battleStartEnemyValue = 0;
private int battleStartOwnCount = 0;
private int battleStartEnemyCount = 0;
```

Add the `updateBattleFSM` method:

```java
void updateBattleFSM(long frame, List<Unit> ownArmy, List<Unit> enemyUnits,
                     List<GameMoment> moments) {
    int currentOwnValue = armyValue(ownArmy);
    int currentEnemyValue = armyValue(enemyUnits);

    switch (battleState) {
        case IDLE -> {
            if (previousOwnArmyValue > 0 && currentOwnValue < previousOwnArmyValue * (1 - BATTLE_START_THRESHOLD)) {
                battleState = BattleState.IN_BATTLE;
                battleStartFrame = frame;
                battleStartOwnValue = previousOwnArmyValue;
                battleStartEnemyValue = currentEnemyValue + armyValueDelta(previousOwnArmyValue, currentOwnValue);
                battleStartOwnCount = ownArmy.size() + (int)((previousOwnArmyValue - currentOwnValue) / avgUnitValue(ownArmy));
                battleStartEnemyCount = enemyUnits.size();
            }
        }
        case IN_BATTLE -> {
            if (previousOwnArmyValue > 0
                    && Math.abs(currentOwnValue - previousOwnArmyValue) <= previousOwnArmyValue * BATTLE_STABLE_THRESHOLD) {
                battleState = BattleState.QUIESCENT;
                quiescenceStartFrame = frame;
            }
        }
        case QUIESCENT -> {
            if (previousOwnArmyValue > 0
                    && currentOwnValue < previousOwnArmyValue * (1 - BATTLE_START_THRESHOLD)) {
                battleState = BattleState.IN_BATTLE;
            } else if (frame - quiescenceStartFrame >= QUIESCENCE_FRAMES) {
                int ownValueLost = Math.max(0, battleStartOwnValue - currentOwnValue);
                int enemyValueLost = Math.max(0, battleStartEnemyValue - currentEnemyValue);
                int ownUnitsLost = Math.max(0, battleStartOwnCount - ownArmy.size());
                int enemyUnitsLost = Math.max(0, battleStartEnemyCount - enemyUnits.size());

                EngagementOutcome engagement = EngagementOutcome.of(
                        battleStartFrame, frame,
                        ownUnitsLost, enemyUnitsLost, ownValueLost, enemyValueLost);
                moments.add(new GameMoment(GameMomentType.BATTLE_ENDED, frame,
                        Map.of("engagement", engagement)));
                battleState = BattleState.IDLE;
            }
        }
    }
    previousOwnArmyValue = currentOwnValue;
}

BattleState battleState() { return battleState; }

static int armyValue(List<Unit> units) {
    return units.stream()
            .mapToInt(u -> SC2Data.mineralCost(u.type()) + SC2Data.gasCost(u.type()))
            .sum();
}

private static int armyValueDelta(int prev, int current) {
    return Math.max(0, prev - current);
}

private static double avgUnitValue(List<Unit> units) {
    if (units.isEmpty()) return 1.0;
    return (double) armyValue(units) / units.size();
}
```

- [ ] **Step 4: Wire FSM into execute() method**

In `MomentDetectionTask.execute()` (line 86), after the existing `fireRules()` call, add the battle FSM tick:

```java
@Override
public void execute(CaseContext ctx) {
    Long    frameL     = ctx.getAs(QuarkMindCaseFile.GAME_FRAME, Long.class);
    long    frame      = frameL != null ? frameL : 0L;
    Integer supplyUsed = ctx.getAs(QuarkMindCaseFile.SUPPLY_USED, Integer.class);
    Integer supplyCap  = ctx.getAs(QuarkMindCaseFile.SUPPLY_CAP, Integer.class);
    List<GameMoment> moments = fireRules(frame,
                                         supplyUsed != null ? supplyUsed : 0,
                                         supplyCap != null ? supplyCap : 0);

    @SuppressWarnings("unchecked")
    List<Unit> ownArmy = (List<Unit>) ctx.get(QuarkMindCaseFile.ARMY);
    @SuppressWarnings("unchecked")
    List<Unit> enemyUnits = (List<Unit>) ctx.get(QuarkMindCaseFile.ENEMY_UNITS);
    if (ownArmy != null && enemyUnits != null) {
        updateBattleFSM(frame, ownArmy, enemyUnits, moments);
    }

    if (!moments.isEmpty()) {
        ctx.set(QuarkMindCaseFile.MOMENTS_LATEST, moments);
    }
}
```

Update `requires()` to include ARMY and ENEMY_UNITS:

```java
@Override
public Set<String> requires() {
    return Set.of(
        QuarkMindCaseFile.ENEMY_UNITS,
        QuarkMindCaseFile.ENEMY_POSTURE,
        QuarkMindCaseFile.TIMING_ATTACK_INCOMING,
        QuarkMindCaseFile.ARMY);
}
```

- [ ] **Step 5: Reset battle state in onGameStarted**

Add to `onGameStarted()` (line 146):

```java
void onGameStarted(@Observes GameStarted event) {
    pendingIntel.clear();
    firstContactFired    = false;
    previousArmyValue    = 0;
    previousPosture      = null;
    lastSupplyBlockFrame = -1;
    battleState          = BattleState.IDLE;
    quiescenceStartFrame = -1;
    battleStartFrame     = -1;
    previousOwnArmyValue = 0;
    battleStartOwnValue  = 0;
    battleStartEnemyValue = 0;
    battleStartOwnCount  = 0;
    battleStartEnemyCount = 0;
}
```

- [ ] **Step 6: Run the initial test**

Run: `mvn test -Dtest=MomentDetectionBattleTest -q`
Expected: PASS

- [ ] **Step 7: Write additional FSM tests**

```java
@Test
void quiescenceThenBattleEnded() {
    List<Unit> army10 = stalkers(10);
    List<Unit> enemy5 = zealots(5);

    task.tickBattle(100, army10, enemy5);             // baseline
    task.tickBattle(200, stalkers(7), enemy5);         // battle starts (30% drop)
    assertThat(task.battleState()).isEqualTo(MomentDetectionTask.BattleState.IN_BATTLE);

    task.tickBattle(300, stalkers(7), enemy5);         // stable → quiescent
    assertThat(task.battleState()).isEqualTo(MomentDetectionTask.BattleState.QUIESCENT);

    task.tickBattle(300 + 224, stalkers(7), zealots(3)); // 224 frames later → BATTLE_ENDED
    assertThat(task.battleState()).isEqualTo(MomentDetectionTask.BattleState.IDLE);
    assertThat(emittedMoments).hasSize(1);
    assertThat(emittedMoments.get(0).type()).isEqualTo(GameMomentType.BATTLE_ENDED);

    EngagementOutcome engagement = (EngagementOutcome) emittedMoments.get(0).context().get("engagement");
    assertThat(engagement).isNotNull();
    assertThat(engagement.ownUnitsLost()).isEqualTo(3);
    assertThat(engagement.enemyUnitsLost()).isEqualTo(2);
    assertThat(engagement.outcome()).isEqualTo(EngagementOutcome.Outcome.LOST); // stalkers cost more
}

@Test
void quiescenceInterruptedByNewFighting() {
    List<Unit> army10 = stalkers(10);
    List<Unit> enemy5 = zealots(5);

    task.tickBattle(100, army10, enemy5);             // baseline
    task.tickBattle(200, stalkers(7), enemy5);         // battle starts
    task.tickBattle(300, stalkers(7), enemy5);         // stable → quiescent

    task.tickBattle(350, stalkers(5), enemy5);         // fighting resumes (>15% drop)
    assertThat(task.battleState()).isEqualTo(MomentDetectionTask.BattleState.IN_BATTLE);
    assertThat(emittedMoments).isEmpty(); // no BATTLE_ENDED emitted
}

@Test
void multipleSequentialEngagements() {
    List<Unit> army10 = stalkers(10);
    List<Unit> enemy5 = zealots(5);

    // First engagement
    task.tickBattle(100, army10, enemy5);
    task.tickBattle(200, stalkers(7), enemy5);
    task.tickBattle(300, stalkers(7), enemy5);
    task.tickBattle(300 + 224, stalkers(7), zealots(3));
    assertThat(emittedMoments).hasSize(1);

    // Second engagement
    task.tickBattle(600, stalkers(7), zealots(3));     // new baseline
    task.tickBattle(700, stalkers(5), zealots(3));     // battle starts
    task.tickBattle(800, stalkers(5), zealots(3));     // stable
    task.tickBattle(800 + 224, stalkers(5), zealots(1));
    assertThat(emittedMoments).hasSize(2);
}

@Test
void noBattleWhenArmyValueDropsBelowThreshold() {
    List<Unit> army10 = stalkers(10);
    List<Unit> enemy5 = zealots(5);

    task.tickBattle(100, army10, enemy5);
    task.tickBattle(200, stalkers(9), enemy5); // 10% drop — below 15% threshold
    assertThat(task.battleState()).isEqualTo(MomentDetectionTask.BattleState.IDLE);
}
```

- [ ] **Step 8: Run all battle tests**

Run: `mvn test -Dtest=MomentDetectionBattleTest -q`
Expected: PASS

- [ ] **Step 9: Run full test suite**

Run: `mvn test -q`
Expected: PASS — existing MomentDetectionTask tests unaffected (battle FSM is additive)

- [ ] **Step 10: Commit**

```bash
git add src/main/java/io/quarkmind/plugin/summarisation/MomentDetectionTask.java \
        src/test/java/io/quarkmind/plugin/summarisation/MomentDetectionBattleTest.java
git commit -m "feat(#270): battle FSM in MomentDetectionTask — army value delta detection

3-state FSM (IDLE/IN_BATTLE/QUIESCENT) detects engagements via own army
value delta (>15% drop). Emits BATTLE_ENDED with EngagementOutcome metadata
after 224-frame quiescence window. Independent of BATTLE_STARTED.

Refs #270"
```

---

### Task 4: CBR Engagement Enrichment (#270 CBR)

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/cbr/EnrichedGameData.java`
- Modify: `src/main/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserver.java:103-161`
- Modify: `src/main/java/io/quarkmind/agent/cbr/SC2GameCbrCase.java:54-90`
- Modify: `src/test/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserverTest.java`
- Modify: `src/test/java/io/quarkmind/agent/cbr/SC2GameCbrCaseTest.java`

**Interfaces:**
- Consumes: `EngagementOutcome` from Task 2, `GameMomentType.BATTLE_ENDED` moments from Task 3
- Produces: CBR case features `engagements_won`, `engagements_lost`, `unit_trade_ratio`

- [ ] **Step 1: Write failing test — EnrichedGameData with engagement fields**

```java
// In SC2GameCbrCaseTest.java — add test
@Test
void buildForGameEnriched_includesEngagementFeatures() {
    var enrichment = new EnrichedGameData(
            List.of("PASSIVE"), 3, "arc", 10.0,
            2, 0.6, 0.5, 1, 12, 0.4, 0,
            OptionalDouble.of(2.0), OptionalDouble.of(1.0), 0.8, "opp-hash",
            2, 1, 1.5);  // engagementsWon=2, engagementsLost=1, unitTradeRatio=1.5

    var cbrCase = SC2GameCbrCase.buildForGameEnriched(
            "ZERG_ROACH_RUSH", "ZERG", "PvZ", 0.8, "adaptive", enrichment);

    assertThat(cbrCase.features().get("engagements_won").numberValue()).isEqualTo(2);
    assertThat(cbrCase.features().get("engagements_lost").numberValue()).isEqualTo(1);
    assertThat(cbrCase.features().get("unit_trade_ratio").numberValue()).isEqualTo(1.5);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=SC2GameCbrCaseTest#buildForGameEnriched_includesEngagementFeatures -q`
Expected: FAIL — `EnrichedGameData` constructor doesn't accept engagement fields

- [ ] **Step 3: Add engagement fields to EnrichedGameData**

```java
package io.quarkmind.agent.cbr;

import java.util.List;
import java.util.OptionalDouble;

public record EnrichedGameData(
        List<String> phaseSequence,
        int momentCount,
        String arcNarrative,
        double gameDurationMinutes,
        int battleCount,
        double dominanceArmy,
        double dominanceOverall,
        int expansionCount,
        int workerCountFinal,
        double dominanceEconomy,
        int supplyBlockCount,
        OptionalDouble firstContactMinute,
        OptionalDouble scoutDispatchMinute,
        double archetypeConfidence,
        String opponentId,
        int engagementsWon,
        int engagementsLost,
        double unitTradeRatio
) {
    public EnrichedGameData {
        phaseSequence = List.copyOf(phaseSequence);
    }
}
```

- [ ] **Step 4: Update SC2GameCbrCase.buildForGameEnriched — add engagement features**

After line 85 (`features.put("opponent_id", ...)`), add:

```java
features.put("engagements_won", FeatureValue.number(e.engagementsWon()));
features.put("engagements_lost", FeatureValue.number(e.engagementsLost()));
features.put("unit_trade_ratio", FeatureValue.number(e.unitTradeRatio()));
```

- [ ] **Step 5: Update SC2CbrRetentionObserver.onOutcome — extract engagement data**

After `battleCount` computation (line 130), add:

```java
List<EngagementOutcome> engagements = moments.stream()
        .filter(m -> m.type() == GameMomentType.BATTLE_ENDED)
        .map(m -> (EngagementOutcome) m.context().get("engagement"))
        .filter(java.util.Objects::nonNull)
        .toList();
int engagementsWon = (int) engagements.stream()
        .filter(e -> e.outcome() == EngagementOutcome.Outcome.WON).count();
int engagementsLost = (int) engagements.stream()
        .filter(e -> e.outcome() == EngagementOutcome.Outcome.LOST).count();
int totalOwnValueLost = engagements.stream().mapToInt(EngagementOutcome::ownValueLost).sum();
int totalEnemyValueLost = engagements.stream().mapToInt(EngagementOutcome::enemyValueLost).sum();
double unitTradeRatio = totalOwnValueLost == 0
        ? (totalEnemyValueLost > 0 ? Double.MAX_VALUE : 0.0)
        : (double) totalEnemyValueLost / totalOwnValueLost;
```

Then update the `EnrichedGameData` construction (line 154) to include the three new fields:

```java
var enrichment = new EnrichedGameData(
        phaseSequence, momentCount, arcNarrative, gameDurationMinutes,
        battleCount, dominance.factors().getOrDefault("army", 0.0), dominance.overall(),
        expansionCount, workerCountFinal,
        dominance.factors().getOrDefault("economy", 0.0), supplyBlockCount,
        firstContactMinute, scoutDispatchMinute,
        confidence != null ? confidence : 0.0,
        opponentId,
        engagementsWon, engagementsLost, unitTradeRatio);
```

- [ ] **Step 6: Fix all existing EnrichedGameData construction sites**

Search for all `new EnrichedGameData(` calls in test code. Each needs three additional trailing arguments: `0, 0, 0.0` (zero engagements for existing tests).

Run: `ide_search_text("new EnrichedGameData(", filePattern="*Test*.java")` to find all sites.

Update each to append `, 0, 0, 0.0)` before the closing paren.

- [ ] **Step 7: Write test — SC2CbrRetentionObserver extracts engagement data**

```java
// In SC2CbrRetentionObserverTest.java — add test
@Test
void onOutcome_includesEngagementDataInCbrCase() {
    // Set up moments including BATTLE_ENDED with EngagementOutcome
    var engagement = EngagementOutcome.of(100, 500, 2, 4, 300, 600);
    observer.collectMoment(new LevelEvent<>(
            new GameMoment(GameMomentType.BATTLE_ENDED, 500, Map.of("engagement", engagement)),
            500, LEVEL_2));
    observer.collectMoment(new LevelEvent<>(
            new GameMoment(GameMomentType.BATTLE_STARTED, 100, Map.of()),
            100, LEVEL_2));

    // Fire outcome and verify enrichment
    // ... (follow existing test pattern for firing CaseOutcomeEvent)
    // Assert the stored case has engagement features
}
```

- [ ] **Step 8: Run all tests**

Run: `mvn test -q`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add src/main/java/io/quarkmind/agent/cbr/EnrichedGameData.java \
        src/main/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserver.java \
        src/main/java/io/quarkmind/agent/cbr/SC2GameCbrCase.java \
        src/test/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserverTest.java \
        src/test/java/io/quarkmind/agent/cbr/SC2GameCbrCaseTest.java
git commit -m "feat(#270): CBR enrichment with per-engagement outcomes

EnrichedGameData gains engagementsWon, engagementsLost, unitTradeRatio.
SC2CbrRetentionObserver extracts EngagementOutcome from BATTLE_ENDED
moments. SC2GameCbrCase includes engagement features for case retrieval.

Refs #270"
```
