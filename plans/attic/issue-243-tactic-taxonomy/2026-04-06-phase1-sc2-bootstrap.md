# Phase 1 — SC2 Bootstrap Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Connect the agent to a real StarCraft II process, prove the `sc2.real` implementations satisfy the same CDI interfaces as the mock, and verify the full pipeline (observe → CaseEngine → IntentQueue → SC2 commands) runs end-to-end.

**Architecture:** Two independent loops share `IntentQueue` as a thread-safe handoff. ocraft's `S2Agent.onStep()` callback stores the latest SC2 observation AND drains `IntentQueue` to send commands (commands MUST be sent within `onStep()`). Our `@Scheduled` intelligence pipeline reads observations via `RealGameObserver`, runs CaseEngine, and writes intents to `IntentQueue` — unaware of the ocraft internals. All mock beans gain `@UnlessBuildProfile("sc2")`; real beans gain `@IfBuildProfile("sc2")`. `AgentOrchestrator` uses `Instance<SimulatedGame>` so it compiles in both profiles.

**Tech Stack:** Java 17, Quarkus 3.34.2, `com.github.ocraft:ocraft-s2client-bot:0.4.21`, `quarkus-smallrye-fault-tolerance`, CaseHub 1.0.0-SNAPSHOT.

**Prerequisites:**
- StarCraft II installed locally (tested with macOS default install at `/Applications/StarCraft II/`)
- CaseHub installed: `cd /Users/mdproctor/claude/alpha && mvn install -DskipTests`
- Existing Phase 0 tests continue to pass throughout

---

## File Map

```
New files:
  src/main/java/org/acme/starcraft/sc2/real/
    ObservationTranslator.java    — pure fn: ocraft Observation → GameState (no CDI, unit-testable)
    SC2BotAgent.java              — ocraft S2Agent: stores latest obs, drains IntentQueue each frame
    RealSC2Client.java            — @IfBuildProfile("sc2"), implements SC2Client, wraps S2Coordinator
    RealGameObserver.java         — @IfBuildProfile("sc2"), implements GameObserver, reads SC2BotAgent
    RealCommandDispatcher.java    — @IfBuildProfile("sc2"), implements CommandDispatcher (no-op)
    SC2DebugScenarioRunner.java   — @IfBuildProfile("sc2"), implements ScenarioRunner via debug API

  src/test/java/org/acme/starcraft/sc2/real/
    ObservationTranslatorTest.java — unit tests for translation logic (no SC2 needed)

Modified files:
  pom.xml                                              — add ocraft + smallrye-fault-tolerance deps
  src/main/resources/application.properties            — add %sc2 profile config
  src/main/java/org/acme/starcraft/sc2/mock/
    SimulatedGame.java            — add @UnlessBuildProfile("sc2")
    MockSC2Client.java            — add @UnlessBuildProfile("sc2")
    MockGameObserver.java         — add @UnlessBuildProfile("sc2")
    MockCommandDispatcher.java    — add @UnlessBuildProfile("sc2")
    MockScenarioRunner.java       — add @UnlessBuildProfile("sc2")
  src/main/java/org/acme/starcraft/agent/AgentOrchestrator.java
                                  — SimulatedGame → Instance<SimulatedGame>
  NATIVE.md                       — update after first sc2 run with tracing agent findings
```

---

## Task 1: Add Dependencies + Profile-Guard All Mock Beans

**Files:**
- Modify: `pom.xml`
- Modify: `src/main/resources/application.properties`
- Modify: `src/main/java/org/acme/starcraft/sc2/mock/SimulatedGame.java`
- Modify: `src/main/java/org/acme/starcraft/sc2/mock/MockSC2Client.java`
- Modify: `src/main/java/org/acme/starcraft/sc2/mock/MockGameObserver.java`
- Modify: `src/main/java/org/acme/starcraft/sc2/mock/MockCommandDispatcher.java`
- Modify: `src/main/java/org/acme/starcraft/sc2/mock/MockScenarioRunner.java`

- [ ] **Step 1: Add ocraft and SmallRye Fault Tolerance to pom.xml**

In `pom.xml`, inside `<dependencies>`, add after the CaseHub entry:

```xml
<!-- Real SC2 API client — only needed at runtime in %sc2 profile -->
<dependency>
    <groupId>com.github.ocraft</groupId>
    <artifactId>ocraft-s2client-bot</artifactId>
    <version>0.4.21</version>
</dependency>

<!-- SC2 connection resilience: @CircuitBreaker, @Retry, @Fallback -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-fault-tolerance</artifactId>
</dependency>
```

- [ ] **Step 2: Add %sc2 profile config to application.properties**

Add to `src/main/resources/application.properties`:

```properties
# SC2 game settings (used in %sc2 profile)
%sc2.starcraft.sc2.map=Simple128
%sc2.starcraft.sc2.difficulty=VERY_EASY
%sc2.starcraft.sc2.opponent-race=RANDOM

# Disable scheduler in sc2 profile — ocraft's onStep() drives timing differently
# (keep it enabled so @Scheduled still runs the CaseEngine pipeline)
```

- [ ] **Step 3: Profile-guard all mock beans**

Add `@UnlessBuildProfile("sc2")` to the class-level annotation on each of these files. The import is `io.quarkus.arc.profile.UnlessBuildProfile`.

**`SimulatedGame.java`** — add above `@ApplicationScoped`:
```java
import io.quarkus.arc.profile.UnlessBuildProfile;

@UnlessBuildProfile("sc2")
@ApplicationScoped
public class SimulatedGame {
```

**`MockSC2Client.java`**:
```java
@UnlessBuildProfile("sc2")
@ApplicationScoped
public class MockSC2Client implements SC2Client {
```

**`MockGameObserver.java`**:
```java
@UnlessBuildProfile("sc2")
@ApplicationScoped
public class MockGameObserver implements GameObserver {
```

**`MockCommandDispatcher.java`**:
```java
@UnlessBuildProfile("sc2")
@ApplicationScoped
public class MockCommandDispatcher implements CommandDispatcher {
```

**`MockScenarioRunner.java`**:
```java
@UnlessBuildProfile("sc2")
@ApplicationScoped
public class MockScenarioRunner implements ScenarioRunner {
```

- [ ] **Step 4: Verify mock tests still pass**

```bash
cd /Users/mdproctor/claude/starcraft
mvn test -q
```

Expected: BUILD SUCCESS, all existing tests pass. If CDI ambiguity errors appear, the `@UnlessBuildProfile` annotations are working correctly — the real beans don't exist yet so there is no ambiguity yet.

- [ ] **Step 5: Commit**

```bash
git add pom.xml src/main/resources/application.properties \
  src/main/java/org/acme/starcraft/sc2/mock/
git commit -m "feat: add ocraft + fault-tolerance deps; profile-guard mock beans with @UnlessBuildProfile(sc2)"
```

---

## Task 2: Update AgentOrchestrator for Dual-Profile Operation

**Files:**
- Modify: `src/main/java/org/acme/starcraft/agent/AgentOrchestrator.java`

`SimulatedGame` will not exist as a CDI bean in the `%sc2` profile (Task 1 added `@UnlessBuildProfile("sc2")`). `AgentOrchestrator` currently injects it directly — this fails at build time in sc2 profile. Fix: use `Instance<SimulatedGame>`.

- [ ] **Step 1: Update AgentOrchestrator**

Replace the full file content:

```java
package org.acme.starcraft.agent;

import io.casehub.coordination.CaseEngine;
import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.acme.starcraft.sc2.CommandDispatcher;
import org.acme.starcraft.sc2.GameObserver;
import org.acme.starcraft.sc2.SC2Client;
import org.acme.starcraft.sc2.mock.SimulatedGame;
import org.jboss.logging.Logger;
import java.time.Duration;
import java.util.Map;

@ApplicationScoped
public class AgentOrchestrator {
    private static final Logger log = Logger.getLogger(AgentOrchestrator.class);

    @Inject SC2Client sc2Client;
    @Inject GameObserver gameObserver;
    @Inject GameStateTranslator translator;
    @Inject CaseEngine caseEngine;
    @Inject CommandDispatcher commandDispatcher;
    @Inject Instance<SimulatedGame> simulatedGame; // Optional — absent in %sc2 profile

    public void startGame() {
        if (simulatedGame.isResolvable()) {
            simulatedGame.get().reset();
        }
        sc2Client.connect();
        sc2Client.joinGame();
        log.info("Game started");
    }

    public void stopGame() {
        sc2Client.leaveGame();
        log.info("Game stopped");
    }

    @Scheduled(every = "${starcraft.tick.interval:500ms}", concurrentExecution = Scheduled.ConcurrentExecution.SKIP)
    public void gameTick() {
        if (!sc2Client.isConnected()) return;

        // Advance mock game clock — skipped in %sc2 profile (ocraft drives timing)
        if (simulatedGame.isResolvable()) {
            simulatedGame.get().tick();
        }

        var gameState = gameObserver.observe();
        Map<String, Object> caseData = translator.toMap(gameState);

        try {
            caseEngine.createAndSolve("starcraft-game", caseData, Duration.ofSeconds(5));
        } catch (Exception e) {
            log.errorf("CaseEngine decision cycle failed at frame %d: %s", gameState.gameFrame(), e.getMessage());
        }

        commandDispatcher.dispatch();
        log.debugf("Tick complete — frame=%d minerals=%d supplyUsed=%d/%d",
            gameState.gameFrame(), gameState.minerals(), gameState.supplyUsed(), gameState.supply());
    }
}
```

- [ ] **Step 2: Verify all tests still pass**

```bash
cd /Users/mdproctor/claude/starcraft
mvn test -q
```

Expected: BUILD SUCCESS, all tests pass. `FullMockPipelineIT` calls `orchestrator.gameTick()` — it must still work since `SimulatedGame` is resolvable in the `%test` profile.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/org/acme/starcraft/agent/AgentOrchestrator.java
git commit -m "feat: use Instance<SimulatedGame> in AgentOrchestrator for sc2/mock dual-profile compatibility"
```

---

## Task 3: ObservationTranslator — ocraft Observation → GameState

**Files:**
- Create: `src/main/java/org/acme/starcraft/sc2/real/ObservationTranslator.java`
- Create: `src/test/java/org/acme/starcraft/sc2/real/ObservationTranslatorTest.java`

This is a pure function with no CDI annotations — fully unit-testable without ocraft connected to SC2. It translates an ocraft `ObservationInterface` snapshot into our `GameState` domain object.

- [ ] **Step 1: Write the failing test**

```java
// src/test/java/org/acme/starcraft/sc2/real/ObservationTranslatorTest.java
package org.acme.starcraft.sc2.real;

import org.acme.starcraft.domain.*;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ObservationTranslatorTest {

    @Test
    void mapsProtossUnitTypes() {
        assertThat(ObservationTranslator.mapUnitType(com.github.ocraft.s2client.protocol.data.Units.PROTOSS_PROBE))
            .isEqualTo(UnitType.PROBE);
        assertThat(ObservationTranslator.mapUnitType(com.github.ocraft.s2client.protocol.data.Units.PROTOSS_ZEALOT))
            .isEqualTo(UnitType.ZEALOT);
        assertThat(ObservationTranslator.mapUnitType(com.github.ocraft.s2client.protocol.data.Units.PROTOSS_STALKER))
            .isEqualTo(UnitType.STALKER);
        assertThat(ObservationTranslator.mapUnitType(com.github.ocraft.s2client.protocol.data.Units.PROTOSS_NEXUS))
            .isEqualTo(UnitType.UNKNOWN); // buildings go through mapBuildingType, not mapUnitType
    }

    @Test
    void mapsProtossBuildingTypes() {
        assertThat(ObservationTranslator.mapBuildingType(com.github.ocraft.s2client.protocol.data.Units.PROTOSS_NEXUS))
            .isEqualTo(BuildingType.NEXUS);
        assertThat(ObservationTranslator.mapBuildingType(com.github.ocraft.s2client.protocol.data.Units.PROTOSS_PYLON))
            .isEqualTo(BuildingType.PYLON);
        assertThat(ObservationTranslator.mapBuildingType(com.github.ocraft.s2client.protocol.data.Units.PROTOSS_PROBE))
            .isEqualTo(BuildingType.UNKNOWN); // units go through mapUnitType
    }

    @Test
    void knowsWhichTypesAreBuildings() {
        assertThat(ObservationTranslator.isBuilding(com.github.ocraft.s2client.protocol.data.Units.PROTOSS_NEXUS))
            .isTrue();
        assertThat(ObservationTranslator.isBuilding(com.github.ocraft.s2client.protocol.data.Units.PROTOSS_PYLON))
            .isTrue();
        assertThat(ObservationTranslator.isBuilding(com.github.ocraft.s2client.protocol.data.Units.PROTOSS_PROBE))
            .isFalse();
        assertThat(ObservationTranslator.isBuilding(com.github.ocraft.s2client.protocol.data.Units.PROTOSS_ZEALOT))
            .isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd /Users/mdproctor/claude/starcraft
mvn test -Dtest=ObservationTranslatorTest -q 2>&1 | tail -5
```

Expected: compilation error — `ObservationTranslator` not defined.

- [ ] **Step 3: Write ObservationTranslator**

```java
// src/main/java/org/acme/starcraft/sc2/real/ObservationTranslator.java
package org.acme.starcraft.sc2.real;

import com.github.ocraft.s2client.bot.gateway.ObservationInterface;
import com.github.ocraft.s2client.protocol.data.Units;
import com.github.ocraft.s2client.protocol.unit.Alliance;
import org.acme.starcraft.domain.*;

import java.util.List;
import java.util.Set;

/**
 * Pure function — translates an ocraft ObservationInterface snapshot into our GameState.
 * No CDI, no framework dependencies. Unit-testable without a live SC2 connection.
 */
public final class ObservationTranslator {

    // Protoss building types (used to distinguish units from structures in the API)
    private static final Set<Units> PROTOSS_BUILDINGS = Set.of(
        Units.PROTOSS_NEXUS, Units.PROTOSS_PYLON, Units.PROTOSS_GATEWAY,
        Units.PROTOSS_CYBERNETICSCORE, Units.PROTOSS_ASSIMILATOR,
        Units.PROTOSS_ROBOTICSFACILITY, Units.PROTOSS_STARGATE,
        Units.PROTOSS_FORGE, Units.PROTOSS_TWILIGHTCOUNCIL,
        Units.PROTOSS_DARKSHRINE, Units.PROTOSS_TEMPLARARCHIVE,
        Units.PROTOSS_FLEETBEACON, Units.PROTOSS_ROBOTICSBAY
    );

    private ObservationTranslator() {}

    public static GameState translate(ObservationInterface obs) {
        var allUnits  = obs.getUnits();
        var selfUnits = allUnits.stream()
            .filter(u -> u.getAlliance() == Alliance.SELF).toList();
        var enemyUnits = allUnits.stream()
            .filter(u -> u.getAlliance() == Alliance.ENEMY).toList();

        List<Unit> myUnits = selfUnits.stream()
            .filter(u -> !isBuilding(u.getType().toEnum()))
            .map(ObservationTranslator::toUnit)
            .toList();

        List<Building> myBuildings = selfUnits.stream()
            .filter(u -> isBuilding(u.getType().toEnum()))
            .map(ObservationTranslator::toBuilding)
            .toList();

        List<Unit> enemies = enemyUnits.stream()
            .map(ObservationTranslator::toUnit)
            .toList();

        return new GameState(
            (int) obs.getMinerals(),
            (int) obs.getVespene(),
            (int) obs.getFoodCap(),
            (int) obs.getFoodUsed(),
            myUnits,
            myBuildings,
            enemies,
            obs.getGameLoop()
        );
    }

    static boolean isBuilding(Units type) {
        return PROTOSS_BUILDINGS.contains(type);
    }

    static UnitType mapUnitType(Units type) {
        return switch (type) {
            case PROTOSS_PROBE         -> UnitType.PROBE;
            case PROTOSS_ZEALOT        -> UnitType.ZEALOT;
            case PROTOSS_STALKER       -> UnitType.STALKER;
            case PROTOSS_IMMORTAL      -> UnitType.IMMORTAL;
            case PROTOSS_COLOSSUS      -> UnitType.COLOSSUS;
            case PROTOSS_CARRIER       -> UnitType.CARRIER;
            case PROTOSS_DARK_TEMPLAR  -> UnitType.DARK_TEMPLAR;
            case PROTOSS_HIGH_TEMPLAR  -> UnitType.HIGH_TEMPLAR;
            case PROTOSS_ARCHON        -> UnitType.ARCHON;
            case PROTOSS_OBSERVER      -> UnitType.OBSERVER;
            case PROTOSS_VOIDRAY       -> UnitType.VOID_RAY;
            default                    -> UnitType.UNKNOWN;
        };
    }

    static BuildingType mapBuildingType(Units type) {
        return switch (type) {
            case PROTOSS_NEXUS            -> BuildingType.NEXUS;
            case PROTOSS_PYLON            -> BuildingType.PYLON;
            case PROTOSS_GATEWAY          -> BuildingType.GATEWAY;
            case PROTOSS_CYBERNETICSCORE  -> BuildingType.CYBERNETICS_CORE;
            case PROTOSS_ASSIMILATOR      -> BuildingType.ASSIMILATOR;
            case PROTOSS_ROBOTICSFACILITY -> BuildingType.ROBOTICS_FACILITY;
            case PROTOSS_STARGATE         -> BuildingType.STARGATE;
            case PROTOSS_FORGE            -> BuildingType.FORGE;
            case PROTOSS_TWILIGHTCOUNCIL  -> BuildingType.TWILIGHT_COUNCIL;
            default                       -> BuildingType.UNKNOWN;
        };
    }

    private static Unit toUnit(com.github.ocraft.s2client.protocol.unit.Unit u) {
        var pos = u.getPosition();
        return new Unit(
            String.valueOf(u.getTag()),
            mapUnitType(u.getType().toEnum()),
            new Point2d(pos.getX(), pos.getY()),
            (int) u.getHealth(),
            (int) u.getHealthMax()
        );
    }

    private static Building toBuilding(com.github.ocraft.s2client.protocol.unit.Unit u) {
        var pos = u.getPosition();
        return new Building(
            String.valueOf(u.getTag()),
            mapBuildingType(u.getType().toEnum()),
            new Point2d(pos.getX(), pos.getY()),
            (int) u.getHealth(),
            (int) u.getHealthMax(),
            u.getBuildProgress() >= 1.0f
        );
    }
}
```

**If ocraft API differs from above:** Check the ocraft Javadoc at `~/.m2/repository/com/github/ocraft/ocraft-s2client-bot/0.4.21/`. Key classes to verify:
- `ObservationInterface` — methods for minerals, vespene, food, gameLoop, units
- `Unit` (ocraft's) — `getType()`, `getAlliance()`, `getPosition()`, `getHealth()`, `getHealthMax()`, `getBuildProgress()`, `getTag()`
- `UnitType` vs `Units` — `getType()` may return a `UnitType` wrapper; call `.toEnum()` to get the `Units` constant
- `Alliance` — enum constants SELF, ALLY, NEUTRAL, ENEMY

Adjust method calls to match the actual API. The translation logic (switch statements, filtering) stays the same.

- [ ] **Step 4: Run test to verify it passes**

```bash
cd /Users/mdproctor/claude/starcraft
mvn test -Dtest=ObservationTranslatorTest -q
```

Expected: BUILD SUCCESS, 3 tests passed.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/org/acme/starcraft/sc2/real/ObservationTranslator.java \
        src/test/java/org/acme/starcraft/sc2/real/ObservationTranslatorTest.java
git commit -m "feat: ObservationTranslator — ocraft Observation to GameState with Protoss unit/building type mapping"
```

---

## Task 4: SC2BotAgent — ocraft S2Agent Bridge

**Files:**
- Create: `src/main/java/org/acme/starcraft/sc2/real/SC2BotAgent.java`

`SC2BotAgent` is NOT a CDI bean — ocraft owns its lifecycle, not Quarkus. It is created by `RealSC2Client` and passed references to `IntentQueue`. Its `onStep()` callback is called by ocraft each SC2 frame.

No test for this task — it requires a live SC2 connection. It is tested implicitly in Task 9 (integration smoke test).

- [ ] **Step 1: Create SC2BotAgent**

```java
// src/main/java/org/acme/starcraft/sc2/real/SC2BotAgent.java
package org.acme.starcraft.sc2.real;

import com.github.ocraft.s2client.bot.S2Agent;
import org.acme.starcraft.domain.GameState;
import org.acme.starcraft.sc2.IntentQueue;
import org.acme.starcraft.sc2.intent.*;
import org.jboss.logging.Logger;
import java.util.concurrent.atomic.AtomicReference;

/**
 * ocraft S2Agent bridge. NOT a CDI bean — lifecycle owned by S2Coordinator.
 *
 * Two responsibilities:
 *  1. Store the latest SC2 observation for RealGameObserver to poll.
 *  2. Drain IntentQueue and send commands to SC2 each frame (MUST happen within onStep()).
 */
public class SC2BotAgent extends S2Agent {
    private static final Logger log = Logger.getLogger(SC2BotAgent.class);

    private final IntentQueue intentQueue;
    private final AtomicReference<GameState> latestGameState = new AtomicReference<>(null);

    public SC2BotAgent(IntentQueue intentQueue) {
        this.intentQueue = intentQueue;
    }

    @Override
    public void onGameStart() {
        log.info("[SC2] Game started");
    }

    @Override
    public void onStep() {
        // 1. Translate and store the current observation for our @Scheduled pipeline to read
        try {
            GameState state = ObservationTranslator.translate(observation());
            latestGameState.set(state);
        } catch (Exception e) {
            log.warnf("[SC2] Observation translation failed at frame %d: %s",
                observation().getGameLoop(), e.getMessage());
        }

        // 2. Drain IntentQueue and send actions to SC2
        //    Commands MUST be sent within onStep() — this is an ocraft/SC2 API requirement.
        //    For Phase 1, dummy plugins produce no intents — this drains an empty queue.
        //    Phase 3+: translate typed Intents to ocraft action calls here.
        intentQueue.drainAll().forEach(intent -> {
            log.debugf("[SC2] Intent received (Phase 1 no-op): %s", intent);
            // Phase 3+ implementation:
            // if (intent instanceof TrainIntent t) {
            //     actions().unitCommand(Long.parseLong(t.unitTag()), Abilities.TRAIN_ZEALOT, false);
            // }
        });
    }

    @Override
    public void onGameEnd() {
        log.info("[SC2] Game ended");
    }

    /** Called by RealGameObserver — returns null until first onStep() fires. */
    public GameState getLatestGameState() {
        return latestGameState.get();
    }
}
```

- [ ] **Step 2: Verify it compiles**

```bash
cd /Users/mdproctor/claude/starcraft
mvn compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/org/acme/starcraft/sc2/real/SC2BotAgent.java
git commit -m "feat: SC2BotAgent — ocraft S2Agent that stores observations and dispatches intents each frame"
```

---

## Task 5: RealGameObserver + RealCommandDispatcher

**Files:**
- Create: `src/main/java/org/acme/starcraft/sc2/real/RealGameObserver.java`
- Create: `src/main/java/org/acme/starcraft/sc2/real/RealCommandDispatcher.java`

Both are `@IfBuildProfile("sc2")` CDI beans. `RealGameObserver` reads from `SC2BotAgent`. `RealCommandDispatcher` is a no-op — commands are sent by `SC2BotAgent.onStep()` directly.

No separate test — covered by the integration smoke test in Task 9.

- [ ] **Step 1: Create RealGameObserver**

```java
// src/main/java/org/acme/starcraft/sc2/real/RealGameObserver.java
package org.acme.starcraft.sc2.real;

import io.quarkus.arc.profile.IfBuildProfile;
import jakarta.enterprise.context.ApplicationScoped;
import org.acme.starcraft.domain.GameState;
import org.acme.starcraft.sc2.GameObserver;
import org.jboss.logging.Logger;
import java.util.ArrayList;
import java.util.List;
import java.util.function.Consumer;

/**
 * Reads the latest GameState from SC2BotAgent (which stores it each ocraft frame).
 * SC2BotAgent is set via setter by RealSC2Client after the agent is created.
 */
@IfBuildProfile("sc2")
@ApplicationScoped
public class RealGameObserver implements GameObserver {
    private static final Logger log = Logger.getLogger(RealGameObserver.class);

    private SC2BotAgent botAgent;
    private final List<Consumer<GameState>> frameListeners = new ArrayList<>();

    /** Called by RealSC2Client after creating the SC2BotAgent. */
    public void setBotAgent(SC2BotAgent botAgent) {
        this.botAgent = botAgent;
    }

    @Override
    public GameState observe() {
        if (botAgent == null) {
            log.warn("[SC2] observe() called before SC2BotAgent was set — returning empty state");
            return emptyState();
        }
        GameState state = botAgent.getLatestGameState();
        if (state == null) {
            log.debug("[SC2] No observation yet — game not started or first frame pending");
            return emptyState();
        }
        frameListeners.forEach(l -> l.accept(state));
        return state;
    }

    @Override
    public void addFrameListener(Consumer<GameState> listener) {
        frameListeners.add(listener);
    }

    private static GameState emptyState() {
        return new GameState(0, 0, 0, 0, List.of(), List.of(), List.of(), 0L);
    }
}
```

- [ ] **Step 2: Create RealCommandDispatcher**

```java
// src/main/java/org/acme/starcraft/sc2/real/RealCommandDispatcher.java
package org.acme.starcraft.sc2.real;

import io.quarkus.arc.profile.IfBuildProfile;
import jakarta.enterprise.context.ApplicationScoped;
import org.acme.starcraft.sc2.CommandDispatcher;
import org.jboss.logging.Logger;

/**
 * No-op implementation for %sc2 profile.
 *
 * Commands are sent by SC2BotAgent.onStep() which drains IntentQueue directly.
 * SC2 requires all actions to be submitted within the onStep() callback —
 * dispatching from our @Scheduled pipeline is too late. AgentOrchestrator
 * still calls dispatch() but this implementation does nothing.
 */
@IfBuildProfile("sc2")
@ApplicationScoped
public class RealCommandDispatcher implements CommandDispatcher {
    private static final Logger log = Logger.getLogger(RealCommandDispatcher.class);

    @Override
    public void dispatch() {
        // No-op: SC2BotAgent.onStep() handles command dispatch within the SC2 frame callback.
        log.trace("[SC2] CommandDispatcher.dispatch() called — no-op, SC2BotAgent handles dispatch");
    }
}
```

- [ ] **Step 3: Verify compile**

```bash
cd /Users/mdproctor/claude/starcraft
mvn compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
git add src/main/java/org/acme/starcraft/sc2/real/RealGameObserver.java \
        src/main/java/org/acme/starcraft/sc2/real/RealCommandDispatcher.java
git commit -m "feat: RealGameObserver (reads SC2BotAgent) and RealCommandDispatcher (no-op) for %sc2 profile"
```

---

## Task 6: RealSC2Client with SmallRye Fault Tolerance

**Files:**
- Create: `src/main/java/org/acme/starcraft/sc2/real/RealSC2Client.java`

This is the most complex class. It wraps `S2Coordinator`, creates `SC2BotAgent`, wires it to `RealGameObserver`, and starts ocraft's game loop in a background thread. SmallRye Fault Tolerance annotations protect the connection.

- [ ] **Step 1: Create RealSC2Client**

```java
// src/main/java/org/acme/starcraft/sc2/real/RealSC2Client.java
package org.acme.starcraft.sc2.real;

import com.github.ocraft.s2client.bot.S2Coordinator;
import com.github.ocraft.s2client.protocol.game.BattlenetMap;
import com.github.ocraft.s2client.protocol.game.Difficulty;
import com.github.ocraft.s2client.protocol.game.Race;
import io.quarkus.arc.profile.IfBuildProfile;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.acme.starcraft.sc2.IntentQueue;
import org.acme.starcraft.sc2.SC2Client;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.eclipse.microprofile.faulttolerance.CircuitBreaker;
import org.eclipse.microprofile.faulttolerance.Fallback;
import org.eclipse.microprofile.faulttolerance.Retry;
import org.jboss.logging.Logger;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.atomic.AtomicBoolean;

@IfBuildProfile("sc2")
@ApplicationScoped
public class RealSC2Client implements SC2Client {
    private static final Logger log = Logger.getLogger(RealSC2Client.class);

    @Inject IntentQueue intentQueue;
    @Inject RealGameObserver gameObserver;

    @ConfigProperty(name = "starcraft.sc2.map", defaultValue = "Simple128")
    String mapName;

    @ConfigProperty(name = "starcraft.sc2.difficulty", defaultValue = "VERY_EASY")
    String difficulty;

    private S2Coordinator coordinator;
    private SC2BotAgent botAgent;
    private final AtomicBoolean connected = new AtomicBoolean(false);
    private CompletableFuture<Void> gameLoop;

    @Override
    @Retry(maxRetries = 3, delay = 2000)
    @CircuitBreaker(requestVolumeThreshold = 4, failureRatio = 0.5, delay = 10000)
    @Fallback(fallbackMethod = "connectFallback")
    public void connect() {
        log.info("[SC2] Connecting to StarCraft II...");
        botAgent = new SC2BotAgent(intentQueue);
        gameObserver.setBotAgent(botAgent);

        coordinator = S2Coordinator.setup()
            .loadSettings(new String[]{})
            .setParticipants(
                S2Coordinator.createParticipant(Race.PROTOSS, botAgent),
                S2Coordinator.createComputer(Race.fromString(difficulty).orElse(Race.RANDOM),
                    Difficulty.fromString(difficulty).orElse(Difficulty.VERY_EASY))
            )
            .launchStarcraft()
            .create();

        connected.set(true);
        log.info("[SC2] Connected — coordinator ready");
    }

    public void connectFallback() {
        log.error("[SC2] Failed to connect after retries — bot will run without SC2");
    }

    @Override
    public void joinGame() {
        if (coordinator == null) {
            log.error("[SC2] Cannot join game — coordinator not initialised");
            return;
        }
        log.infof("[SC2] Starting game on map: %s", mapName);
        coordinator.startGame(BattlenetMap.of(mapName));

        // Run ocraft's game loop in a background thread.
        // coordinator.update() returns false when the game ends.
        gameLoop = CompletableFuture.runAsync(() -> {
            log.info("[SC2] Game loop started");
            while (coordinator.update()) {
                // onStep() called here by ocraft each frame
            }
            connected.set(false);
            log.info("[SC2] Game loop ended");
        }).exceptionally(e -> {
            log.errorf("[SC2] Game loop error: %s", e.getMessage());
            connected.set(false);
            return null;
        });
    }

    @Override
    public void leaveGame() {
        connected.set(false);
        if (coordinator != null) {
            coordinator.quit();
        }
        if (gameLoop != null) {
            gameLoop.cancel(true);
        }
        log.info("[SC2] Left game");
    }

    @Override
    public boolean isConnected() {
        return connected.get();
    }
}
```

**If `S2Coordinator` API differs:** Check the ocraft samples at `~/.m2/repository/com/github/ocraft/ocraft-s2client-sample/`. Key methods:
- `S2Coordinator.setup()` — builder entry point
- `.loadSettings(String[] args)` — loads ocraft settings; pass empty array for defaults
- `.setParticipants(...)` — takes `S2Coordinator.createParticipant(race, agent)` and optionally `S2Coordinator.createComputer(race, difficulty)`
- `.launchStarcraft()` — auto-detects SC2 install; on macOS checks `/Applications/StarCraft II/`
- `.create()` — builds the coordinator
- `.startGame(BattlenetMap.of(mapName))` — starts a game on the named ladder map
- `.update()` — processes one game tick; returns false when game over
- `.quit()` — shuts down SC2

`Race` and `Difficulty` enums are in `com.github.ocraft.s2client.protocol.game`. `BattlenetMap` is in the same package.

- [ ] **Step 2: Verify compile**

```bash
cd /Users/mdproctor/claude/starcraft
mvn compile -q
```

Expected: BUILD SUCCESS. If `Race.fromString()` or `Difficulty.fromString()` don't exist, replace with direct enum constants: `Race.RANDOM` and `Difficulty.VERY_EASY`.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/org/acme/starcraft/sc2/real/RealSC2Client.java
git commit -m "feat: RealSC2Client — ocraft S2Coordinator with @Retry/@CircuitBreaker, auto-launches SC2, starts game loop in background thread"
```

---

## Task 7: SC2DebugScenarioRunner

**Files:**
- Create: `src/main/java/org/acme/starcraft/sc2/real/SC2DebugScenarioRunner.java`

Implements the same named scenarios as `MockScenarioRunner` but triggers them via SC2's debug API. Requires access to `SC2BotAgent` (which holds the ocraft agent with access to `debug()`).

- [ ] **Step 1: Create SC2DebugScenarioRunner**

```java
// src/main/java/org/acme/starcraft/sc2/real/SC2DebugScenarioRunner.java
package org.acme.starcraft.sc2.real;

import com.github.ocraft.s2client.protocol.debug.DebugCommand;
import com.github.ocraft.s2client.protocol.debug.DebugCreateUnit;
import com.github.ocraft.s2client.protocol.data.Units;
import com.github.ocraft.s2client.protocol.game.Race;
import com.github.ocraft.s2client.protocol.spatial.Point2d;
import io.quarkus.arc.profile.IfBuildProfile;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.acme.starcraft.sc2.ScenarioRunner;
import org.jboss.logging.Logger;
import java.util.List;
import java.util.Set;

/**
 * Triggers named scenarios against a live SC2 game via the SC2 Debug API.
 * Same scenario names as MockScenarioRunner — integration parity by design.
 *
 * Note: Debug API calls are queued and sent on the next onStep() callback.
 * Scenarios are visible in-game on the following frame after calling run().
 */
@IfBuildProfile("sc2")
@ApplicationScoped
public class SC2DebugScenarioRunner implements ScenarioRunner {
    private static final Logger log = Logger.getLogger(SC2DebugScenarioRunner.class);

    @Inject RealGameObserver gameObserver; // holds ref to SC2BotAgent

    private static final Set<String> AVAILABLE = Set.of(
        "spawn-enemy-attack", "set-resources-500", "supply-almost-capped", "enemy-expands"
    );

    @Override
    public void run(String scenarioName) {
        SC2BotAgent agent = gameObserver.getBotAgent();
        if (agent == null) {
            throw new IllegalStateException("SC2 not connected — cannot run scenario: " + scenarioName);
        }
        log.infof("[SC2-DEBUG] Running scenario: %s", scenarioName);
        switch (scenarioName) {
            case "spawn-enemy-attack" -> spawnEnemyAttack(agent);
            case "set-resources-500" -> setResources(agent, 500, 200);
            case "supply-almost-capped" -> supplyAlmostCapped(agent);
            case "enemy-expands" -> enemyExpands(agent);
            default -> throw new IllegalArgumentException(
                "Unknown scenario: " + scenarioName + ". Available: " + AVAILABLE);
        }
    }

    @Override
    public Set<String> availableScenarios() {
        return AVAILABLE;
    }

    private void spawnEnemyAttack(SC2BotAgent agent) {
        // Spawn 2 Zealots + 1 Stalker near our base for enemy player (player 2)
        agent.debug().debugCreateUnit(Units.PROTOSS_ZEALOT, 1,
            Point2d.of(20.0f, 20.0f), 2);
        agent.debug().debugCreateUnit(Units.PROTOSS_ZEALOT, 1,
            Point2d.of(21.0f, 20.0f), 2);
        agent.debug().debugCreateUnit(Units.PROTOSS_STALKER, 1,
            Point2d.of(22.0f, 20.0f), 2);
        agent.debug().sendDebug();
    }

    private void setResources(SC2BotAgent agent, int minerals, int vespene) {
        agent.debug().debugSetMinerals(minerals);
        agent.debug().debugSetVespene(vespene);
        agent.debug().sendDebug();
    }

    private void supplyAlmostCapped(SC2BotAgent agent) {
        // Create units to fill supply to cap - 1
        // Simpler: spawn probes to fill supply (each costs 1)
        agent.debug().debugCreateUnit(Units.PROTOSS_PROBE, 2,
            Point2d.of(10.0f, 10.0f), 1);
        agent.debug().sendDebug();
    }

    private void enemyExpands(SC2BotAgent agent) {
        // Spawn an enemy probe near expansion location
        agent.debug().debugCreateUnit(Units.PROTOSS_PROBE, 1,
            Point2d.of(50.0f, 50.0f), 2);
        agent.debug().sendDebug();
    }
}
```

**Important:** Expose `getBotAgent()` on `RealGameObserver`:

Add this method to `RealGameObserver.java`:
```java
public SC2BotAgent getBotAgent() {
    return botAgent;
}
```

**If ocraft debug API differs:** Check the ocraft `DebugInterface`. Common methods:
- `agent.debug().debugCreateUnit(unitType, count, position, playerId)` — spawns units for a player (1=self, 2=enemy)
- `agent.debug().debugSetMinerals(amount)` — sets mineral count
- `agent.debug().debugSetVespene(amount)` — sets vespene count
- `agent.debug().sendDebug()` — flushes all queued debug commands

The debug API requires `agent.debug()` to be called from within `onStep()` or the commands must be queued and sent in the next step. If the API doesn't support calling outside `onStep()`, queue the commands in `SC2BotAgent` and send them on the next frame.

- [ ] **Step 2: Verify compile**

```bash
cd /Users/mdproctor/claude/starcraft
mvn compile -q
```

Expected: BUILD SUCCESS. Fix any ocraft debug API method name differences.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/org/acme/starcraft/sc2/real/SC2DebugScenarioRunner.java \
        src/main/java/org/acme/starcraft/sc2/real/RealGameObserver.java
git commit -m "feat: SC2DebugScenarioRunner — same 4 scenario names triggered via SC2 debug API"
```

---

## Task 8: Create sc2/real Package Directory + Verify Existing Tests Pass

**Files:**
- Create: `src/main/java/org/acme/starcraft/sc2/real/` (already created by Tasks 3–7)
- Create: `src/test/java/org/acme/starcraft/sc2/real/` (already created by Task 3)

- [ ] **Step 1: Run full test suite to confirm no regressions**

```bash
cd /Users/mdproctor/claude/starcraft
mvn test -q
```

Expected: BUILD SUCCESS, all existing tests pass. The `%test` profile uses the mock beans (which still exist). No sc2.real beans are active in tests.

If any test fails, check:
- `FullMockPipelineIT` — calls `orchestrator.gameTick()` and uses `Instance<SimulatedGame>`. Verify `SimulatedGame` is still resolvable in `%test` profile (it should be — `@UnlessBuildProfile("sc2")` is not `@UnlessBuildProfile("test")`).
- `QaEndpointsTest` — injects `ScenarioRunner`. In `%test`, `MockScenarioRunner` should still be the active bean.

- [ ] **Step 2: Create package-info to document the real package**

Create `src/main/java/org/acme/starcraft/sc2/real/package-info.java`:

```java
/**
 * Real StarCraft II API implementation — active only in the %sc2 profile.
 * All classes annotated @IfBuildProfile("sc2").
 *
 * Requires StarCraft II installed locally. ocraft-s2client-bot auto-detects
 * the SC2 binary on macOS at /Applications/StarCraft II/.
 *
 * @see org.acme.starcraft.sc2.mock for the mock implementations used in development and testing.
 */
package org.acme.starcraft.sc2.real;
```

- [ ] **Step 3: Commit**

```bash
git add src/main/java/org/acme/starcraft/sc2/real/package-info.java
git commit -m "docs: add package-info for sc2.real package documenting profile and SC2 dependency"
```

---

## Task 9: Integration Smoke Test — Connect, Observe, Disconnect

This task requires a running StarCraft II installation. It cannot be automated in CI without Docker SC2 (Phase 4+ concern). Run manually.

**Prerequisites:**
- SC2 is installed at `/Applications/StarCraft II/` (default macOS location)
- SC2 may launch automatically via ocraft's `launchStarcraft()` — allow it to update if prompted
- Internet connection not required once SC2 is installed

- [ ] **Step 1: Start the agent in sc2 profile**

```bash
cd /Users/mdproctor/claude/starcraft
mvn quarkus:dev -Dquarkus.profile=sc2
```

Expected:
- SC2 launches automatically (may take 30–60 seconds)
- Log lines appear: `[SC2] Connecting to StarCraft II...`, `[SC2] Connected`, `[SC2] Game started`
- After game starts: repeated `Tick complete — frame=N minerals=M supplyUsed=X/Y` log lines
- No CDI ambiguity errors (both `SC2Client` and `GameObserver` should have exactly one active bean)

- [ ] **Step 2: Verify game state visible via QA endpoint**

In a second terminal:

```bash
curl -s http://localhost:8080/sc2/casefile | python3 -m json.tool
```

Expected: JSON showing real game state with non-zero values:
```json
{
  "minerals": 50,
  "supply": 15,
  "supplyUsed": 12,
  ...
}
```

If minerals are 0 and supply is 0, the observation is not being translated correctly — check `ObservationTranslator` against the actual ocraft API.

- [ ] **Step 3: Verify frame counter advances**

```bash
# Run twice, 2 seconds apart — gameFrame should increase
curl -s http://localhost:8080/sc2/frame
sleep 2
curl -s http://localhost:8080/sc2/frame
```

Expected: `gameFrame` is larger in the second response.

- [ ] **Step 4: Trigger a debug scenario**

```bash
curl -s -X POST http://localhost:8080/sc2/debug/scenario/spawn-enemy-attack
```

Expected: HTTP 204. Watch the SC2 window — 3 enemy Protoss units should appear near your base.

```bash
curl -s http://localhost:8080/sc2/casefile | python3 -m json.tool | grep enemyUnits
```

Expected: `enemyUnits` list is non-empty.

- [ ] **Step 5: Stop the agent and update NATIVE.md**

Press `Ctrl+C` to stop quarkus:dev.

Update `NATIVE.md` with findings from this run. Note any warnings related to ocraft, Netty, RxJava, or Protobuf that appeared in the log output — these are candidates for the `reflect-config.json` tracing agent run.

**To run the GraalVM tracing agent** (capture reflection config for future native build):

```bash
# Export JAVA_HOME if not already set (requires GraalVM JDK)
export JAVA_HOME=$(/usr/libexec/java_home)

# Build the runnable JAR first
mvn package -Dquarkus.profile=sc2 -DskipTests -q

# Run with tracing agent — play through an entire game
$JAVA_HOME/bin/java \
  -agentlib:native-image-agent=config-merge-dir=src/main/resources/META-INF/native-image \
  -jar target/quarkus-app/quarkus-run.jar \
  -Dquarkus.profile=sc2
```

Let the game run to completion (or at least 60+ seconds) to capture all code paths. The generated files (`reflect-config.json`, `proxy-config.json`, etc.) accumulate in `src/main/resources/META-INF/native-image/`. Run multiple times on different maps to improve coverage.

- [ ] **Step 6: Commit final state**

```bash
cd /Users/mdproctor/claude/starcraft
git add NATIVE.md src/main/resources/META-INF/native-image/ 2>/dev/null || true
git commit -m "feat: Phase 1 complete — real SC2 integration verified, NATIVE.md updated with ocraft tracing findings"
```

---

## Done When

- [ ] `mvn test -q` passes (all Phase 0 tests still green, `ObservationTranslatorTest` passes)
- [ ] `mvn quarkus:dev -Dquarkus.profile=sc2` starts SC2, joins a game, logs frame ticks
- [ ] `GET /sc2/casefile` returns real game state with non-zero minerals and units
- [ ] `GET /sc2/frame` shows `gameFrame` advancing over time
- [ ] `POST /sc2/debug/scenario/spawn-enemy-attack` triggers 3 enemy units in the SC2 window
- [ ] `NATIVE.md` updated with ocraft/Netty/Protobuf findings from the run
