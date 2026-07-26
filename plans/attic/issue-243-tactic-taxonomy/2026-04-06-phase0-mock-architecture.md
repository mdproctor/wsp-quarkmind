# Phase 0 — Mock Architecture Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the complete StarCraft II agent architecture against a stateful mock SC2 implementation — no real StarCraft II installation required. All four package layers, CaseHub intelligence loop, dummy plugins, and QA REST endpoints fully operational.

**Architecture:** Single Quarkus module with four packages (`sc2`, `domain`, `agent`, `plugin`) plus a `qa` package for dev-only REST endpoints. The `sc2` package defines CDI interfaces; `sc2.mock` provides a stateful `SimulatedGame` that receives commands and mutates its own state. CaseHub's `CaseEngine` drives a decision cycle each game tick, activating dummy `TaskDefinition` plugins. An `IntentQueue` bridges the CaseEngine cycle to the `CommandDispatcher`.

**Tech Stack:** Java 17, Quarkus 3.32.2, CaseHub `io.casehub:casehub-core:1.0.0-SNAPSHOT` (local install), `quarkus-rest`, `quarkus-rest-jackson`, `quarkus-scheduler`, JUnit 5, AssertJ.

---

## Prerequisites

Before starting Task 1, install CaseHub to your local Maven repository:

```bash
cd /Users/mdproctor/claude/alpha
mvn install -DskipTests
```

Expected output ends with `BUILD SUCCESS`. Verify:

```bash
ls ~/.m2/repository/io/casehub/casehub-core/1.0.0-SNAPSHOT/
```

Expected: `casehub-core-1.0.0-SNAPSHOT.jar` present.

---

## File Map

```
starcraft/
  pom.xml
  NATIVE.md
  MODULES.md
  src/main/java/org/acme/starcraft/
    domain/
      Point2d.java
      UnitType.java
      BuildingType.java
      Unit.java
      Building.java
      Resource.java
      PlayerInfo.java
      GameMap.java
      GameState.java
    sc2/
      SC2Client.java
      GameObserver.java
      CommandDispatcher.java
      ScenarioRunner.java
      intent/
        Intent.java
        BuildIntent.java
        TrainIntent.java
        AttackIntent.java
        MoveIntent.java
      IntentQueue.java
    sc2/mock/
      SimulatedGame.java
      MockSC2Client.java
      MockGameObserver.java
      MockCommandDispatcher.java
      MockScenarioRunner.java
      scenario/
        Scenario.java
        ScenarioLibrary.java
        SpawnEnemyAttackScenario.java
        SetResourcesScenario.java
        SupplyAlmostCappedScenario.java
        EnemyExpandsScenario.java
    agent/
      StarCraftCaseFile.java
      GameStateTranslator.java
      AgentOrchestrator.java
      plugin/
        StrategyTask.java
        EconomicsTask.java
        TacticsTask.java
        ScoutingTask.java
    plugin/
      PassThroughStrategyTask.java
      PassThroughEconomicsTask.java
      PassThroughTacticsTask.java
      PassThroughScoutingTask.java
    qa/
      CaseFileResource.java
      IntentResource.java
      ScenarioResource.java
  src/main/resources/
    application.properties
  src/test/java/org/acme/starcraft/
    domain/
      GameStateTest.java
    sc2/
      IntentQueueTest.java
    sc2/mock/
      SimulatedGameTest.java
      MockPipelineTest.java
      ScenarioLibraryTest.java
    agent/
      GameStateTranslatorTest.java
    qa/
      QaEndpointsTest.java
```

---

## Task 1: Project Scaffold

**Files:**
- Create: `pom.xml`
- Create: `NATIVE.md`
- Create: `MODULES.md`
- Create: `src/main/resources/application.properties`

- [ ] **Step 1: Generate Quarkus project**

```bash
cd /Users/mdproctor/claude/starcraft
mvn io.quarkus.platform:quarkus-maven-plugin:3.32.2:create \
  -DprojectGroupId=org.acme.starcraft \
  -DprojectArtifactId=starcraft-agent \
  -DprojectVersion=1.0.0-SNAPSHOT \
  -Dextensions="rest,rest-jackson,scheduler" \
  -DnoCode
```

This generates `pom.xml`, `src/` structure, and `.gitignore`.

- [ ] **Step 2: Add CaseHub dependency to pom.xml**

Open `pom.xml`. Inside `<dependencies>`, add after the existing Quarkus entries:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-core</artifactId>
  <version>1.0.0-SNAPSHOT</version>
</dependency>
```

Also add AssertJ for tests:

```xml
<dependency>
  <groupId>org.assertj</groupId>
  <artifactId>assertj-core</artifactId>
  <scope>test</scope>
</dependency>
```

- [ ] **Step 3: Create package directories**

```bash
mkdir -p src/main/java/org/acme/starcraft/{domain,sc2/intent,sc2/mock/scenario,agent/plugin,plugin,qa}
mkdir -p src/test/java/org/acme/starcraft/{domain,sc2,sc2/mock,agent,qa}
```

- [ ] **Step 4: Write application.properties**

Replace the generated `src/main/resources/application.properties` with:

```properties
quarkus.application.name=starcraft-agent

# Game loop tick interval (mock mode)
starcraft.tick.interval=500ms

# SC2 connection (used in %sc2 profile only)
%sc2.starcraft.sc2.host=127.0.0.1
%sc2.starcraft.sc2.port=8168

# Disable scheduler in test unless explicitly enabled
%test.quarkus.scheduler.enabled=false
```

- [ ] **Step 5: Create NATIVE.md**

```bash
cat > NATIVE.md << 'EOF'
# Native Quarkus Compatibility Tracker

Status: JVM mode only (Phase 0)

## Dependencies

| Dependency | Version | Native Status | Notes |
|---|---|---|---|
| quarkus-rest | 3.32.2 | ✅ Supported | |
| quarkus-rest-jackson | 3.32.2 | ✅ Supported | |
| quarkus-scheduler | 3.32.2 | ✅ Supported | |
| casehub-core | 1.0.0-SNAPSHOT | 🔲 Not verified | Verify before native build |
| ocraft-s2client-bot | 0.4.21 | 🔲 Not added yet | Added in Phase 1; uses RxJava + Protobuf |

## Rules (enforce these always)
- No dynamic class loading or runtime code generation
- All CDI injection via constructor or field — no programmatic Arc.container() lookups
- Reflection usages → register in src/main/resources/reflection-config.json
- No raw use of Class.forName()

## Known Issues
(none yet — updated as issues are discovered)
EOF
```

- [ ] **Step 6: Create MODULES.md**

```bash
cat > MODULES.md << 'EOF'
# Intended Module Split

Currently a single Maven module. Extract when a plugin implementation matures.

## Planned Future Modules

| Module | Extract when | Contains |
|---|---|---|
| starcraft-sc2 | Phase 1 completes | sc2/, sc2/mock/, sc2/real/ |
| starcraft-domain | Phase 1 completes | domain/ |
| starcraft-agent | Phase 3 — first real plugin | agent/ + plugin interfaces |
| starcraft-agent-drools | Drools plugin matures | Drools TaskDefinition implementations |
| starcraft-agent-flow | Flow plugin matures | Quarkus Flow Worker implementations |
EOF
```

- [ ] **Step 7: Verify compile**

```bash
mvn compile
```

Expected: `BUILD SUCCESS`. Fix any dependency resolution errors before proceeding.

- [ ] **Step 8: Commit**

```bash
git init
git add .
git commit -m "feat: project scaffold — Quarkus 3.32.2 + CaseHub, package structure, NATIVE.md, MODULES.md"
```

---

## Task 2: Domain Model

**Files:**
- Create: `src/main/java/org/acme/starcraft/domain/Point2d.java`
- Create: `src/main/java/org/acme/starcraft/domain/UnitType.java`
- Create: `src/main/java/org/acme/starcraft/domain/BuildingType.java`
- Create: `src/main/java/org/acme/starcraft/domain/Unit.java`
- Create: `src/main/java/org/acme/starcraft/domain/Building.java`
- Create: `src/main/java/org/acme/starcraft/domain/Resource.java`
- Create: `src/main/java/org/acme/starcraft/domain/PlayerInfo.java`
- Create: `src/main/java/org/acme/starcraft/domain/GameMap.java`
- Create: `src/main/java/org/acme/starcraft/domain/GameState.java`
- Test: `src/test/java/org/acme/starcraft/domain/GameStateTest.java`

- [ ] **Step 1: Write the failing test**

```java
// src/test/java/org/acme/starcraft/domain/GameStateTest.java
package org.acme.starcraft.domain;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class GameStateTest {

    @Test
    void initialProtossGameState() {
        var probe = new Unit("tag-1", UnitType.PROBE, new Point2d(10.0f, 10.0f), 45, 45);
        var nexus = new Building("tag-100", BuildingType.NEXUS, new Point2d(8.0f, 8.0f), 1500, 1500, true);
        var state = new GameState(50, 0, 15, 1, List.of(probe), List.of(nexus), List.of(), 0L);

        assertThat(state.minerals()).isEqualTo(50);
        assertThat(state.supply()).isEqualTo(15);
        assertThat(state.supplyUsed()).isEqualTo(1);
        assertThat(state.myUnits()).hasSize(1);
        assertThat(state.myUnits().get(0).type()).isEqualTo(UnitType.PROBE);
        assertThat(state.myBuildings()).hasSize(1);
        assertThat(state.myBuildings().get(0).type()).isEqualTo(BuildingType.NEXUS);
    }

    @Test
    void probeCount() {
        var probes = List.of(
            new Unit("t1", UnitType.PROBE, new Point2d(0, 0), 45, 45),
            new Unit("t2", UnitType.PROBE, new Point2d(1, 1), 45, 45),
            new Unit("t3", UnitType.ZEALOT, new Point2d(2, 2), 100, 100)
        );
        var state = new GameState(50, 0, 15, 3, probes, List.of(), List.of(), 0L);
        long probeCount = state.myUnits().stream().filter(u -> u.type() == UnitType.PROBE).count();
        assertThat(probeCount).isEqualTo(2);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl . -Dtest=GameStateTest -q
```

Expected: FAIL — `GameState` not defined.

- [ ] **Step 3: Write domain classes**

```java
// src/main/java/org/acme/starcraft/domain/Point2d.java
package org.acme.starcraft.domain;
public record Point2d(float x, float y) {}
```

```java
// src/main/java/org/acme/starcraft/domain/UnitType.java
package org.acme.starcraft.domain;
public enum UnitType {
    PROBE, ZEALOT, STALKER, IMMORTAL, COLOSSUS, CARRIER,
    DARK_TEMPLAR, HIGH_TEMPLAR, ARCHON, OBSERVER, VOID_RAY,
    UNKNOWN
}
```

```java
// src/main/java/org/acme/starcraft/domain/BuildingType.java
package org.acme.starcraft.domain;
public enum BuildingType {
    NEXUS, PYLON, GATEWAY, CYBERNETICS_CORE, ASSIMILATOR,
    ROBOTICS_FACILITY, STARGATE, FORGE, TWILIGHT_COUNCIL,
    UNKNOWN
}
```

```java
// src/main/java/org/acme/starcraft/domain/Unit.java
package org.acme.starcraft.domain;
public record Unit(String tag, UnitType type, Point2d position, int health, int maxHealth) {}
```

```java
// src/main/java/org/acme/starcraft/domain/Building.java
package org.acme.starcraft.domain;
public record Building(String tag, BuildingType type, Point2d position, int health, int maxHealth, boolean isComplete) {}
```

```java
// src/main/java/org/acme/starcraft/domain/Resource.java
package org.acme.starcraft.domain;
public record Resource(String tag, Point2d position, int remaining) {}
```

```java
// src/main/java/org/acme/starcraft/domain/PlayerInfo.java
package org.acme.starcraft.domain;
public record PlayerInfo(int playerId, String race, int minerals, int vespene) {}
```

```java
// src/main/java/org/acme/starcraft/domain/GameMap.java
package org.acme.starcraft.domain;
import java.util.List;
public record GameMap(String name, int width, int height, List<Point2d> expansionLocations) {}
```

```java
// src/main/java/org/acme/starcraft/domain/GameState.java
package org.acme.starcraft.domain;
import java.util.List;
public record GameState(
    int minerals,
    int vespene,
    int supply,
    int supplyUsed,
    List<Unit> myUnits,
    List<Building> myBuildings,
    List<Unit> enemyUnits,
    long gameFrame
) {}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -pl . -Dtest=GameStateTest -q
```

Expected: `BUILD SUCCESS`, 2 tests passed.

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "feat: domain model — GameState, Unit, Building, UnitType, BuildingType, Point2d"
```

---

## Task 3: SC2 Interfaces + Intent System

**Files:**
- Create: `src/main/java/org/acme/starcraft/sc2/SC2Client.java`
- Create: `src/main/java/org/acme/starcraft/sc2/GameObserver.java`
- Create: `src/main/java/org/acme/starcraft/sc2/CommandDispatcher.java`
- Create: `src/main/java/org/acme/starcraft/sc2/ScenarioRunner.java`
- Create: `src/main/java/org/acme/starcraft/sc2/intent/Intent.java`
- Create: `src/main/java/org/acme/starcraft/sc2/intent/BuildIntent.java`
- Create: `src/main/java/org/acme/starcraft/sc2/intent/TrainIntent.java`
- Create: `src/main/java/org/acme/starcraft/sc2/intent/AttackIntent.java`
- Create: `src/main/java/org/acme/starcraft/sc2/intent/MoveIntent.java`
- Create: `src/main/java/org/acme/starcraft/sc2/IntentQueue.java`
- Test: `src/test/java/org/acme/starcraft/sc2/IntentQueueTest.java`

- [ ] **Step 1: Write the failing test**

```java
// src/test/java/org/acme/starcraft/sc2/IntentQueueTest.java
package org.acme.starcraft.sc2;

import org.acme.starcraft.domain.Point2d;
import org.acme.starcraft.domain.UnitType;
import org.acme.starcraft.sc2.intent.AttackIntent;
import org.acme.starcraft.sc2.intent.MoveIntent;
import org.acme.starcraft.sc2.intent.TrainIntent;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class IntentQueueTest {

    @Test
    void addAndDrainIntents() {
        var queue = new IntentQueue();
        queue.add(new AttackIntent("tag-1", new Point2d(50, 50)));
        queue.add(new TrainIntent("tag-100", UnitType.ZEALOT));
        queue.add(new MoveIntent("tag-2", new Point2d(30, 30)));

        assertThat(queue.pending()).hasSize(3);
        var drained = queue.drainAll();
        assertThat(drained).hasSize(3);
        assertThat(queue.pending()).isEmpty();
    }

    @Test
    void drainEmptyQueueReturnsEmptyList() {
        var queue = new IntentQueue();
        assertThat(queue.drainAll()).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl . -Dtest=IntentQueueTest -q
```

Expected: FAIL — `IntentQueue` not defined.

- [ ] **Step 3: Write SC2 interfaces and intent classes**

```java
// src/main/java/org/acme/starcraft/sc2/SC2Client.java
package org.acme.starcraft.sc2;
public interface SC2Client {
    void connect();
    void joinGame();
    void leaveGame();
    boolean isConnected();
}
```

```java
// src/main/java/org/acme/starcraft/sc2/GameObserver.java
package org.acme.starcraft.sc2;
import org.acme.starcraft.domain.GameState;
import java.util.function.Consumer;
public interface GameObserver {
    GameState observe();
    void addFrameListener(Consumer<GameState> listener);
}
```

```java
// src/main/java/org/acme/starcraft/sc2/CommandDispatcher.java
package org.acme.starcraft.sc2;
public interface CommandDispatcher {
    void dispatch();
}
```

```java
// src/main/java/org/acme/starcraft/sc2/ScenarioRunner.java
package org.acme.starcraft.sc2;
import java.util.Set;
public interface ScenarioRunner {
    void run(String scenarioName);
    Set<String> availableScenarios();
}
```

```java
// src/main/java/org/acme/starcraft/sc2/intent/Intent.java
package org.acme.starcraft.sc2.intent;
public interface Intent {
    String unitTag();
}
```

```java
// src/main/java/org/acme/starcraft/sc2/intent/BuildIntent.java
package org.acme.starcraft.sc2.intent;
import org.acme.starcraft.domain.BuildingType;
import org.acme.starcraft.domain.Point2d;
public record BuildIntent(String unitTag, BuildingType buildingType, Point2d location) implements Intent {}
```

```java
// src/main/java/org/acme/starcraft/sc2/intent/TrainIntent.java
package org.acme.starcraft.sc2.intent;
import org.acme.starcraft.domain.UnitType;
public record TrainIntent(String unitTag, UnitType unitType) implements Intent {}
```

```java
// src/main/java/org/acme/starcraft/sc2/intent/AttackIntent.java
package org.acme.starcraft.sc2.intent;
import org.acme.starcraft.domain.Point2d;
public record AttackIntent(String unitTag, Point2d targetLocation) implements Intent {}
```

```java
// src/main/java/org/acme/starcraft/sc2/intent/MoveIntent.java
package org.acme.starcraft.sc2.intent;
import org.acme.starcraft.domain.Point2d;
public record MoveIntent(String unitTag, Point2d targetLocation) implements Intent {}
```

```java
// src/main/java/org/acme/starcraft/sc2/IntentQueue.java
package org.acme.starcraft.sc2;

import jakarta.enterprise.context.ApplicationScoped;
import org.acme.starcraft.sc2.intent.Intent;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ConcurrentLinkedQueue;

@ApplicationScoped
public class IntentQueue {
    private final ConcurrentLinkedQueue<Intent> queue = new ConcurrentLinkedQueue<>();
    private final List<Intent> dispatched = new ArrayList<>();
    private static final int DISPATCHED_BUFFER_SIZE = 100;

    public void add(Intent intent) {
        queue.add(intent);
    }

    public List<Intent> drainAll() {
        List<Intent> drained = new ArrayList<>();
        Intent intent;
        while ((intent = queue.poll()) != null) {
            drained.add(intent);
        }
        synchronized (dispatched) {
            dispatched.addAll(drained);
            if (dispatched.size() > DISPATCHED_BUFFER_SIZE) {
                dispatched.subList(0, dispatched.size() - DISPATCHED_BUFFER_SIZE).clear();
            }
        }
        return drained;
    }

    public List<Intent> pending() {
        return List.copyOf(queue);
    }

    public List<Intent> recentlyDispatched() {
        synchronized (dispatched) {
            return List.copyOf(dispatched);
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -pl . -Dtest=IntentQueueTest -q
```

Expected: `BUILD SUCCESS`, 2 tests passed.

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "feat: SC2 interfaces (SC2Client, GameObserver, CommandDispatcher, ScenarioRunner) and intent system"
```

---

## Task 4: SimulatedGame

**Files:**
- Create: `src/main/java/org/acme/starcraft/sc2/mock/SimulatedGame.java`
- Test: `src/test/java/org/acme/starcraft/sc2/mock/SimulatedGameTest.java`

- [ ] **Step 1: Write the failing test**

```java
// src/test/java/org/acme/starcraft/sc2/mock/SimulatedGameTest.java
package org.acme.starcraft.sc2.mock;

import org.acme.starcraft.domain.*;
import org.acme.starcraft.sc2.intent.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class SimulatedGameTest {

    SimulatedGame game;

    @BeforeEach
    void setUp() {
        game = new SimulatedGame();
        game.reset();
    }

    @Test
    void initialStateIsStandardProtossOpener() {
        GameState state = game.snapshot();
        assertThat(state.minerals()).isEqualTo(50);
        assertThat(state.vespene()).isEqualTo(0);
        assertThat(state.supply()).isEqualTo(15);
        assertThat(state.supplyUsed()).isEqualTo(12);
        assertThat(state.myUnits().stream().filter(u -> u.type() == UnitType.PROBE).count()).isEqualTo(12);
        assertThat(state.myBuildings().stream().filter(b -> b.type() == BuildingType.NEXUS).count()).isEqualTo(1);
        assertThat(state.enemyUnits()).isEmpty();
        assertThat(state.gameFrame()).isEqualTo(0L);
    }

    @Test
    void tickAdvancesGameFrame() {
        game.tick();
        game.tick();
        assertThat(game.snapshot().gameFrame()).isEqualTo(2L);
    }

    @Test
    void applyTrainIntentIncreasesSupplyUsed() {
        String nexusTag = game.snapshot().myBuildings().get(0).tag();
        game.applyIntent(new TrainIntent(nexusTag, UnitType.ZEALOT));
        game.tick(); // complete training
        GameState state = game.snapshot();
        assertThat(state.supplyUsed()).isEqualTo(14); // +2 for zealot
    }

    @Test
    void applyBuildIntentAddsPylon() {
        String probeTag = game.snapshot().myUnits().get(0).tag();
        game.applyIntent(new BuildIntent(probeTag, BuildingType.PYLON, new Point2d(15, 15)));
        game.tick();
        GameState state = game.snapshot();
        assertThat(state.myBuildings().stream().anyMatch(b -> b.type() == BuildingType.PYLON)).isTrue();
        assertThat(state.supply()).isEqualTo(23); // +8 from pylon
    }

    @Test
    void spawnEnemyUnitsAreVisible() {
        game.spawnEnemyUnit(UnitType.ZEALOT, new Point2d(100, 100));
        assertThat(game.snapshot().enemyUnits()).hasSize(1);
        assertThat(game.snapshot().enemyUnits().get(0).type()).isEqualTo(UnitType.ZEALOT);
    }

    @Test
    void setMineralsDirectly() {
        game.setMinerals(500);
        assertThat(game.snapshot().minerals()).isEqualTo(500);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl . -Dtest=SimulatedGameTest -q
```

Expected: FAIL — `SimulatedGame` not defined.

- [ ] **Step 3: Write SimulatedGame**

```java
// src/main/java/org/acme/starcraft/sc2/mock/SimulatedGame.java
package org.acme.starcraft.sc2.mock;

import jakarta.enterprise.context.ApplicationScoped;
import org.acme.starcraft.domain.*;
import org.acme.starcraft.sc2.intent.*;
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.atomic.AtomicLong;

@ApplicationScoped
public class SimulatedGame {

    private volatile int minerals;
    private volatile int vespene;
    private volatile int supply;
    private volatile int supplyUsed;
    private final AtomicLong gameFrame = new AtomicLong(0);
    private final List<Unit> myUnits = new CopyOnWriteArrayList<>();
    private final List<Building> myBuildings = new CopyOnWriteArrayList<>();
    private final List<Unit> enemyUnits = new CopyOnWriteArrayList<>();
    private final Queue<Runnable> buildQueue = new LinkedList<>();
    private int nextTag = 200;

    public void reset() {
        minerals = 50;
        vespene = 0;
        supply = 15;
        supplyUsed = 12;
        gameFrame.set(0);
        myUnits.clear();
        myBuildings.clear();
        enemyUnits.clear();
        buildQueue.clear();
        nextTag = 200;

        // 12 Probes
        for (int i = 0; i < 12; i++) {
            myUnits.add(new Unit("probe-" + i, UnitType.PROBE, new Point2d(9 + i * 0.5f, 9), 45, 45));
        }
        // 1 Nexus
        myBuildings.add(new Building("nexus-0", BuildingType.NEXUS, new Point2d(8, 8), 1500, 1500, true));
    }

    public synchronized void tick() {
        gameFrame.incrementAndGet();
        minerals = Math.min(minerals + 5, 9999); // rough mineral trickle
        Runnable completion = buildQueue.poll();
        if (completion != null) completion.run();
    }

    public synchronized void applyIntent(Intent intent) {
        if (intent instanceof TrainIntent t) {
            int cost = supplyCost(t.unitType());
            buildQueue.add(() -> {
                supplyUsed += cost;
                myUnits.add(new Unit("unit-" + nextTag++, t.unitType(), new Point2d(9, 9), maxHealth(t.unitType()), maxHealth(t.unitType())));
            });
        } else if (intent instanceof BuildIntent b) {
            buildQueue.add(() -> {
                int supplyBonus = supplyBonus(b.buildingType());
                supply += supplyBonus;
                myBuildings.add(new Building("bldg-" + nextTag++, b.buildingType(), b.location(), maxBuildingHealth(b.buildingType()), maxBuildingHealth(b.buildingType()), true));
            });
        }
        // AttackIntent and MoveIntent: positions updated in future phases
    }

    public GameState snapshot() {
        return new GameState(minerals, vespene, supply, supplyUsed,
            List.copyOf(myUnits), List.copyOf(myBuildings), List.copyOf(enemyUnits), gameFrame.get());
    }

    public void spawnEnemyUnit(UnitType type, Point2d position) {
        enemyUnits.add(new Unit("enemy-" + nextTag++, type, position, maxHealth(type), maxHealth(type)));
    }

    public void setMinerals(int amount) { this.minerals = amount; }
    public void setVespene(int amount) { this.vespene = amount; }
    public void setSupply(int cap) { this.supply = cap; }
    public void setSupplyUsed(int used) { this.supplyUsed = used; }

    private int supplyCost(UnitType type) {
        return switch (type) {
            case PROBE -> 1;
            case ZEALOT -> 2;
            case STALKER -> 2;
            case IMMORTAL -> 4;
            default -> 2;
        };
    }

    private int supplyBonus(BuildingType type) {
        return type == BuildingType.PYLON ? 8 : 0;
    }

    private int maxHealth(UnitType type) {
        return switch (type) {
            case PROBE -> 45;
            case ZEALOT -> 100;
            case STALKER -> 80;
            default -> 100;
        };
    }

    private int maxBuildingHealth(BuildingType type) {
        return switch (type) {
            case NEXUS -> 1500;
            case PYLON -> 200;
            case GATEWAY -> 500;
            default -> 500;
        };
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -pl . -Dtest=SimulatedGameTest -q
```

Expected: `BUILD SUCCESS`, 6 tests passed.

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "feat: SimulatedGame — stateful Protoss mock with tick, intent application, enemy spawning"
```

---

## Task 5: Mock SC2 Adapter

**Files:**
- Create: `src/main/java/org/acme/starcraft/sc2/mock/MockSC2Client.java`
- Create: `src/main/java/org/acme/starcraft/sc2/mock/MockGameObserver.java`
- Create: `src/main/java/org/acme/starcraft/sc2/mock/MockCommandDispatcher.java`
- Test: `src/test/java/org/acme/starcraft/sc2/mock/MockPipelineTest.java`

- [ ] **Step 1: Write the failing test**

```java
// src/test/java/org/acme/starcraft/sc2/mock/MockPipelineTest.java
package org.acme.starcraft.sc2.mock;

import org.acme.starcraft.domain.UnitType;
import org.acme.starcraft.sc2.IntentQueue;
import org.acme.starcraft.sc2.intent.TrainIntent;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class MockPipelineTest {

    SimulatedGame game;
    MockGameObserver observer;
    MockCommandDispatcher dispatcher;
    IntentQueue intentQueue;

    @BeforeEach
    void setUp() {
        game = new SimulatedGame();
        game.reset();
        intentQueue = new IntentQueue();
        observer = new MockGameObserver(game);
        dispatcher = new MockCommandDispatcher(game, intentQueue);
    }

    @Test
    void observerReflectsInitialGameState() {
        var state = observer.observe();
        assertThat(state.minerals()).isEqualTo(50);
        assertThat(state.myUnits().stream().filter(u -> u.type() == UnitType.PROBE).count()).isEqualTo(12);
    }

    @Test
    void dispatchDrainsIntentQueueAndMutatesGame() {
        String nexusTag = game.snapshot().myBuildings().get(0).tag();
        intentQueue.add(new TrainIntent(nexusTag, UnitType.ZEALOT));
        assertThat(intentQueue.pending()).hasSize(1);

        dispatcher.dispatch();
        game.tick(); // complete training

        assertThat(intentQueue.pending()).isEmpty();
        assertThat(intentQueue.recentlyDispatched()).hasSize(1);
        assertThat(game.snapshot().supplyUsed()).isEqualTo(14);
    }

    @Test
    void observerFrameListenerCalledOnObserve() {
        var captured = new java.util.ArrayList<>();
        observer.addFrameListener(captured::add);
        observer.observe();
        assertThat(captured).hasSize(1);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl . -Dtest=MockPipelineTest -q
```

Expected: FAIL — `MockGameObserver` not defined.

- [ ] **Step 3: Write mock adapter classes**

```java
// src/main/java/org/acme/starcraft/sc2/mock/MockSC2Client.java
package org.acme.starcraft.sc2.mock;

import jakarta.enterprise.context.ApplicationScoped;
import org.acme.starcraft.sc2.SC2Client;
import org.jboss.logging.Logger;

@ApplicationScoped
public class MockSC2Client implements SC2Client {
    private static final Logger log = Logger.getLogger(MockSC2Client.class);
    private boolean connected = false;

    @Override
    public void connect() {
        connected = true;
        log.info("[MOCK] SC2Client connected");
    }

    @Override
    public void joinGame() {
        log.info("[MOCK] Joined game");
    }

    @Override
    public void leaveGame() {
        connected = false;
        log.info("[MOCK] Left game");
    }

    @Override
    public boolean isConnected() {
        return connected;
    }
}
```

```java
// src/main/java/org/acme/starcraft/sc2/mock/MockGameObserver.java
package org.acme.starcraft.sc2.mock;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.acme.starcraft.domain.GameState;
import org.acme.starcraft.sc2.GameObserver;
import java.util.ArrayList;
import java.util.List;
import java.util.function.Consumer;

@ApplicationScoped
public class MockGameObserver implements GameObserver {
    private final SimulatedGame game;
    private final List<Consumer<GameState>> frameListeners = new ArrayList<>();

    @Inject
    public MockGameObserver(SimulatedGame game) {
        this.game = game;
    }

    @Override
    public GameState observe() {
        GameState state = game.snapshot();
        frameListeners.forEach(l -> l.accept(state));
        return state;
    }

    @Override
    public void addFrameListener(Consumer<GameState> listener) {
        frameListeners.add(listener);
    }
}
```

```java
// src/main/java/org/acme/starcraft/sc2/mock/MockCommandDispatcher.java
package org.acme.starcraft.sc2.mock;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.acme.starcraft.sc2.CommandDispatcher;
import org.acme.starcraft.sc2.IntentQueue;
import org.acme.starcraft.sc2.intent.Intent;
import org.jboss.logging.Logger;
import java.util.List;

@ApplicationScoped
public class MockCommandDispatcher implements CommandDispatcher {
    private static final Logger log = Logger.getLogger(MockCommandDispatcher.class);

    private final SimulatedGame game;
    private final IntentQueue intentQueue;

    @Inject
    public MockCommandDispatcher(SimulatedGame game, IntentQueue intentQueue) {
        this.game = game;
        this.intentQueue = intentQueue;
    }

    @Override
    public void dispatch() {
        List<Intent> intents = intentQueue.drainAll();
        intents.forEach(intent -> {
            log.debugf("[MOCK] Dispatching intent: %s", intent);
            game.applyIntent(intent);
        });
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -pl . -Dtest=MockPipelineTest -q
```

Expected: `BUILD SUCCESS`, 3 tests passed.

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "feat: mock SC2 adapter — MockSC2Client, MockGameObserver, MockCommandDispatcher"
```

---

## Task 6: ScenarioLibrary

**Files:**
- Create: `src/main/java/org/acme/starcraft/sc2/mock/scenario/Scenario.java`
- Create: `src/main/java/org/acme/starcraft/sc2/mock/scenario/ScenarioLibrary.java`
- Create: `src/main/java/org/acme/starcraft/sc2/mock/scenario/SpawnEnemyAttackScenario.java`
- Create: `src/main/java/org/acme/starcraft/sc2/mock/scenario/SetResourcesScenario.java`
- Create: `src/main/java/org/acme/starcraft/sc2/mock/scenario/SupplyAlmostCappedScenario.java`
- Create: `src/main/java/org/acme/starcraft/sc2/mock/scenario/EnemyExpandsScenario.java`
- Create: `src/main/java/org/acme/starcraft/sc2/mock/MockScenarioRunner.java`
- Test: `src/test/java/org/acme/starcraft/sc2/mock/ScenarioLibraryTest.java`

- [ ] **Step 1: Write the failing test**

```java
// src/test/java/org/acme/starcraft/sc2/mock/ScenarioLibraryTest.java
package org.acme.starcraft.sc2.mock;

import org.acme.starcraft.domain.UnitType;
import org.acme.starcraft.sc2.mock.scenario.ScenarioLibrary;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ScenarioLibraryTest {

    SimulatedGame game;
    ScenarioLibrary library;
    MockScenarioRunner runner;

    @BeforeEach
    void setUp() {
        game = new SimulatedGame();
        game.reset();
        library = new ScenarioLibrary();
        runner = new MockScenarioRunner(game, library);
    }

    @Test
    void availableScenariosContainsAllFour() {
        assertThat(runner.availableScenarios()).containsExactlyInAnyOrder(
            "spawn-enemy-attack", "set-resources-500", "supply-almost-capped", "enemy-expands"
        );
    }

    @Test
    void spawnEnemyAttackAddsEnemyUnits() {
        runner.run("spawn-enemy-attack");
        assertThat(game.snapshot().enemyUnits()).isNotEmpty();
        assertThat(game.snapshot().enemyUnits().stream().anyMatch(u -> u.type() == UnitType.ZEALOT)).isTrue();
    }

    @Test
    void setResources500SetsMinerals() {
        runner.run("set-resources-500");
        assertThat(game.snapshot().minerals()).isEqualTo(500);
    }

    @Test
    void supplyAlmostCappedSetsSupplyUsedNearCap() {
        runner.run("supply-almost-capped");
        GameState state = game.snapshot();
        assertThat(state.supplyUsed()).isGreaterThanOrEqualTo(state.supply() - 2);
    }

    @Test
    void unknownScenarioThrows() {
        assertThatThrownBy(() -> runner.run("nonexistent"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("nonexistent");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl . -Dtest=ScenarioLibraryTest -q
```

Expected: FAIL — `ScenarioLibrary` not defined.

- [ ] **Step 3: Write scenario classes**

```java
// src/main/java/org/acme/starcraft/sc2/mock/scenario/Scenario.java
package org.acme.starcraft.sc2.mock.scenario;
import org.acme.starcraft.sc2.mock.SimulatedGame;
@FunctionalInterface
public interface Scenario {
    void apply(SimulatedGame game);
}
```

```java
// src/main/java/org/acme/starcraft/sc2/mock/scenario/SpawnEnemyAttackScenario.java
package org.acme.starcraft.sc2.mock.scenario;
import org.acme.starcraft.domain.Point2d;
import org.acme.starcraft.domain.UnitType;
import org.acme.starcraft.sc2.mock.SimulatedGame;
public class SpawnEnemyAttackScenario implements Scenario {
    @Override
    public void apply(SimulatedGame game) {
        game.spawnEnemyUnit(UnitType.ZEALOT, new Point2d(20, 20));
        game.spawnEnemyUnit(UnitType.ZEALOT, new Point2d(21, 20));
        game.spawnEnemyUnit(UnitType.STALKER, new Point2d(22, 20));
    }
}
```

```java
// src/main/java/org/acme/starcraft/sc2/mock/scenario/SetResourcesScenario.java
package org.acme.starcraft.sc2.mock.scenario;
import org.acme.starcraft.sc2.mock.SimulatedGame;
public class SetResourcesScenario implements Scenario {
    @Override
    public void apply(SimulatedGame game) {
        game.setMinerals(500);
        game.setVespene(200);
    }
}
```

```java
// src/main/java/org/acme/starcraft/sc2/mock/scenario/SupplyAlmostCappedScenario.java
package org.acme.starcraft.sc2.mock.scenario;
import org.acme.starcraft.sc2.mock.SimulatedGame;
public class SupplyAlmostCappedScenario implements Scenario {
    @Override
    public void apply(SimulatedGame game) {
        // Set supply used to cap - 1 (tight supply)
        game.setSupplyUsed(game.snapshot().supply() - 1);
    }
}
```

```java
// src/main/java/org/acme/starcraft/sc2/mock/scenario/EnemyExpandsScenario.java
package org.acme.starcraft.sc2.mock.scenario;
import org.acme.starcraft.domain.Point2d;
import org.acme.starcraft.domain.UnitType;
import org.acme.starcraft.sc2.mock.SimulatedGame;
public class EnemyExpandsScenario implements Scenario {
    @Override
    public void apply(SimulatedGame game) {
        // Enemy probe heading to expansion location
        game.spawnEnemyUnit(UnitType.PROBE, new Point2d(50, 50));
    }
}
```

```java
// src/main/java/org/acme/starcraft/sc2/mock/scenario/ScenarioLibrary.java
package org.acme.starcraft.sc2.mock.scenario;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.Map;
import java.util.Set;

@ApplicationScoped
public class ScenarioLibrary {
    private final Map<String, Scenario> scenarios = Map.of(
        "spawn-enemy-attack", new SpawnEnemyAttackScenario(),
        "set-resources-500", new SetResourcesScenario(),
        "supply-almost-capped", new SupplyAlmostCappedScenario(),
        "enemy-expands", new EnemyExpandsScenario()
    );

    public Scenario get(String name) {
        Scenario scenario = scenarios.get(name);
        if (scenario == null) {
            throw new IllegalArgumentException("Unknown scenario: " + name + ". Available: " + scenarios.keySet());
        }
        return scenario;
    }

    public Set<String> names() {
        return scenarios.keySet();
    }
}
```

```java
// src/main/java/org/acme/starcraft/sc2/mock/MockScenarioRunner.java
package org.acme.starcraft.sc2.mock;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.acme.starcraft.sc2.ScenarioRunner;
import org.acme.starcraft.sc2.mock.scenario.ScenarioLibrary;
import org.jboss.logging.Logger;
import java.util.Set;

@ApplicationScoped
public class MockScenarioRunner implements ScenarioRunner {
    private static final Logger log = Logger.getLogger(MockScenarioRunner.class);
    private final SimulatedGame game;
    private final ScenarioLibrary library;

    @Inject
    public MockScenarioRunner(SimulatedGame game, ScenarioLibrary library) {
        this.game = game;
        this.library = library;
    }

    @Override
    public void run(String scenarioName) {
        log.infof("[MOCK] Running scenario: %s", scenarioName);
        library.get(scenarioName).apply(game);
    }

    @Override
    public Set<String> availableScenarios() {
        return library.names();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -pl . -Dtest=ScenarioLibraryTest -q
```

Expected: `BUILD SUCCESS`, 5 tests passed.

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "feat: ScenarioLibrary with 4 initial Protoss scenarios + MockScenarioRunner"
```

---

## Task 7: CaseHub Integration

**Files:**
- Create: `src/main/java/org/acme/starcraft/agent/StarCraftCaseFile.java`
- Create: `src/main/java/org/acme/starcraft/agent/GameStateTranslator.java`
- Create: `src/main/java/org/acme/starcraft/agent/AgentOrchestrator.java`
- Test: `src/test/java/org/acme/starcraft/agent/GameStateTranslatorTest.java`

- [ ] **Step 1: Write the failing test**

```java
// src/test/java/org/acme/starcraft/agent/GameStateTranslatorTest.java
package org.acme.starcraft.agent;

import org.acme.starcraft.domain.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class GameStateTranslatorTest {

    GameStateTranslator translator = new GameStateTranslator();

    @Test
    void translatesResourcesCorrectly() {
        var state = new GameState(150, 75, 23, 14, List.of(), List.of(), List.of(), 42L);
        Map<String, Object> map = translator.toMap(state);
        assertThat(map.get(StarCraftCaseFile.MINERALS)).isEqualTo(150);
        assertThat(map.get(StarCraftCaseFile.VESPENE)).isEqualTo(75);
        assertThat(map.get(StarCraftCaseFile.SUPPLY_CAP)).isEqualTo(23);
        assertThat(map.get(StarCraftCaseFile.SUPPLY_USED)).isEqualTo(14);
        assertThat(map.get(StarCraftCaseFile.GAME_FRAME)).isEqualTo(42L);
        assertThat(map.get(StarCraftCaseFile.READY)).isEqualTo(Boolean.TRUE);
    }

    @Test
    void separatesWorkersFromArmy() {
        var probe = new Unit("p1", UnitType.PROBE, new Point2d(0,0), 45, 45);
        var zealot = new Unit("z1", UnitType.ZEALOT, new Point2d(1,1), 100, 100);
        var state = new GameState(50, 0, 15, 3, List.of(probe, zealot), List.of(), List.of(), 0L);
        Map<String, Object> map = translator.toMap(state);
        assertThat((List<?>) map.get(StarCraftCaseFile.WORKERS)).hasSize(1);
        assertThat((List<?>) map.get(StarCraftCaseFile.ARMY)).hasSize(1);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl . -Dtest=GameStateTranslatorTest -q
```

Expected: FAIL — `GameStateTranslator` not defined.

- [ ] **Step 3: Write CaseHub integration classes**

```java
// src/main/java/org/acme/starcraft/agent/StarCraftCaseFile.java
package org.acme.starcraft.agent;

public final class StarCraftCaseFile {
    // Observation state — written by GameStateTranslator
    public static final String MINERALS      = "game.resources.minerals";
    public static final String VESPENE       = "game.resources.vespene";
    public static final String SUPPLY_USED   = "game.resources.supply.used";
    public static final String SUPPLY_CAP    = "game.resources.supply.cap";
    public static final String WORKERS       = "game.units.workers";
    public static final String ARMY          = "game.units.army";
    public static final String ENEMY_UNITS   = "game.intel.enemy.units";
    public static final String GAME_FRAME    = "game.frame";
    public static final String READY         = "game.ready";

    // Agent state — written by plugins
    public static final String STRATEGY      = "agent.strategy.current";
    public static final String CRISIS        = "agent.intent.crisis";

    private StarCraftCaseFile() {}
}
```

```java
// src/main/java/org/acme/starcraft/agent/GameStateTranslator.java
package org.acme.starcraft.agent;

import jakarta.enterprise.context.ApplicationScoped;
import org.acme.starcraft.domain.*;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class GameStateTranslator {

    public Map<String, Object> toMap(GameState state) {
        Map<String, Object> data = new HashMap<>();
        data.put(StarCraftCaseFile.MINERALS,    state.minerals());
        data.put(StarCraftCaseFile.VESPENE,     state.vespene());
        data.put(StarCraftCaseFile.SUPPLY_CAP,  state.supply());
        data.put(StarCraftCaseFile.SUPPLY_USED, state.supplyUsed());
        data.put(StarCraftCaseFile.GAME_FRAME,  state.gameFrame());
        data.put(StarCraftCaseFile.READY,       Boolean.TRUE);

        List<Unit> workers = state.myUnits().stream()
            .filter(u -> u.type() == UnitType.PROBE).toList();
        List<Unit> army = state.myUnits().stream()
            .filter(u -> u.type() != UnitType.PROBE).toList();

        data.put(StarCraftCaseFile.WORKERS,     workers);
        data.put(StarCraftCaseFile.ARMY,        army);
        data.put(StarCraftCaseFile.ENEMY_UNITS, state.enemyUnits());
        return data;
    }
}
```

```java
// src/main/java/org/acme/starcraft/agent/AgentOrchestrator.java
package org.acme.starcraft.agent;

import io.casehub.coordination.CaseEngine;
import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
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
    @Inject SimulatedGame simulatedGame;

    public void startGame() {
        simulatedGame.reset();
        sc2Client.connect();
        sc2Client.joinGame();
        log.info("Game started — mock SC2 ready");
    }

    public void stopGame() {
        sc2Client.leaveGame();
        log.info("Game stopped");
    }

    @Scheduled(every = "${starcraft.tick.interval:500ms}")
    void gameTick() {
        if (!sc2Client.isConnected()) return;

        simulatedGame.tick();
        var gameState = gameObserver.observe();
        Map<String, Object> caseData = translator.toMap(gameState);

        try {
            caseEngine.createAndSolve("starcraft-game", caseData, Duration.ofSeconds(5));
        } catch (Exception e) {
            // CaseCreationException and timeout exceptions both caught here.
            // Check io.casehub.coordination.CaseEngine source for exact checked exceptions
            // and narrow this catch once confirmed.
            log.errorf("CaseEngine decision cycle failed at frame %d: %s", gameState.gameFrame(), e.getMessage());
        }

        commandDispatcher.dispatch();
        log.debugf("Tick complete — frame=%d minerals=%d supplyUsed=%d/%d",
            gameState.gameFrame(), gameState.minerals(), gameState.supplyUsed(), gameState.supply());
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -pl . -Dtest=GameStateTranslatorTest -q
```

Expected: `BUILD SUCCESS`, 2 tests passed.

- [ ] **Step 5: Verify the project compiles with CaseHub wired**

```bash
mvn compile
```

Expected: `BUILD SUCCESS`. If CaseEngine's `CaseCreationException` import fails, check the exact exception class name in the CaseHub source: `ls /Users/mdproctor/claude/alpha/casehub-core/src/main/java/io/casehub/` and adjust the import.

- [ ] **Step 6: Commit**

```bash
git add src/
git commit -m "feat: CaseHub integration — StarCraftCaseFile keys, GameStateTranslator, AgentOrchestrator game loop"
```

---

## Task 8: Plugin Seams + Dummy Implementations

**Files:**
- Create: `src/main/java/org/acme/starcraft/agent/plugin/StrategyTask.java`
- Create: `src/main/java/org/acme/starcraft/agent/plugin/EconomicsTask.java`
- Create: `src/main/java/org/acme/starcraft/agent/plugin/TacticsTask.java`
- Create: `src/main/java/org/acme/starcraft/agent/plugin/ScoutingTask.java`
- Create: `src/main/java/org/acme/starcraft/plugin/PassThroughStrategyTask.java`
- Create: `src/main/java/org/acme/starcraft/plugin/PassThroughEconomicsTask.java`
- Create: `src/main/java/org/acme/starcraft/plugin/PassThroughTacticsTask.java`
- Create: `src/main/java/org/acme/starcraft/plugin/PassThroughScoutingTask.java`

There is no test for this task using `@QuarkusTest` as it requires Quarkus to boot. That integration is covered in Task 10. Here we write unit-testable pass-through logic.

- [ ] **Step 1: Write plugin seam interfaces**

```java
// src/main/java/org/acme/starcraft/agent/plugin/StrategyTask.java
package org.acme.starcraft.agent.plugin;
import io.casehub.core.TaskDefinition;
public interface StrategyTask extends TaskDefinition {}
```

```java
// src/main/java/org/acme/starcraft/agent/plugin/EconomicsTask.java
package org.acme.starcraft.agent.plugin;
import io.casehub.core.TaskDefinition;
public interface EconomicsTask extends TaskDefinition {}
```

```java
// src/main/java/org/acme/starcraft/agent/plugin/TacticsTask.java
package org.acme.starcraft.agent.plugin;
import io.casehub.core.TaskDefinition;
public interface TacticsTask extends TaskDefinition {}
```

```java
// src/main/java/org/acme/starcraft/agent/plugin/ScoutingTask.java
package org.acme.starcraft.agent.plugin;
import io.casehub.core.TaskDefinition;
public interface ScoutingTask extends TaskDefinition {}
```

- [ ] **Step 2: Write dummy implementations**

```java
// src/main/java/org/acme/starcraft/plugin/PassThroughStrategyTask.java
package org.acme.starcraft.plugin;

import io.casehub.core.CaseFile;
import jakarta.enterprise.context.ApplicationScoped;
import org.acme.starcraft.agent.StarCraftCaseFile;
import org.acme.starcraft.agent.plugin.StrategyTask;
import org.jboss.logging.Logger;
import java.util.Set;

@ApplicationScoped
public class PassThroughStrategyTask implements StrategyTask {
    private static final Logger log = Logger.getLogger(PassThroughStrategyTask.class);

    @Override public String getId() { return "strategy.passthrough"; }
    @Override public String getName() { return "PassThrough Strategy"; }
    @Override public Set<String> entryCriteria() { return Set.of(StarCraftCaseFile.READY); }
    @Override public Set<String> producedKeys() { return Set.of(); }

    @Override
    public void execute(CaseFile caseFile) {
        log.debugf("[STRATEGY] PassThrough activated at frame=%s minerals=%s",
            caseFile.get(StarCraftCaseFile.GAME_FRAME, Long.class).orElse(-1L),
            caseFile.get(StarCraftCaseFile.MINERALS, Integer.class).orElse(0));
    }
}
```

```java
// src/main/java/org/acme/starcraft/plugin/PassThroughEconomicsTask.java
package org.acme.starcraft.plugin;

import io.casehub.core.CaseFile;
import jakarta.enterprise.context.ApplicationScoped;
import org.acme.starcraft.agent.StarCraftCaseFile;
import org.acme.starcraft.agent.plugin.EconomicsTask;
import org.jboss.logging.Logger;
import java.util.Set;

@ApplicationScoped
public class PassThroughEconomicsTask implements EconomicsTask {
    private static final Logger log = Logger.getLogger(PassThroughEconomicsTask.class);

    @Override public String getId() { return "economics.passthrough"; }
    @Override public String getName() { return "PassThrough Economics"; }
    @Override public Set<String> entryCriteria() { return Set.of(StarCraftCaseFile.READY); }
    @Override public Set<String> producedKeys() { return Set.of(); }

    @Override
    public void execute(CaseFile caseFile) {
        log.debugf("[ECONOMICS] PassThrough activated — supply=%s/%s",
            caseFile.get(StarCraftCaseFile.SUPPLY_USED, Integer.class).orElse(0),
            caseFile.get(StarCraftCaseFile.SUPPLY_CAP, Integer.class).orElse(0));
    }
}
```

```java
// src/main/java/org/acme/starcraft/plugin/PassThroughTacticsTask.java
package org.acme.starcraft.plugin;

import io.casehub.core.CaseFile;
import jakarta.enterprise.context.ApplicationScoped;
import org.acme.starcraft.agent.StarCraftCaseFile;
import org.acme.starcraft.agent.plugin.TacticsTask;
import org.jboss.logging.Logger;
import java.util.List;
import java.util.Set;

@ApplicationScoped
public class PassThroughTacticsTask implements TacticsTask {
    private static final Logger log = Logger.getLogger(PassThroughTacticsTask.class);

    @Override public String getId() { return "tactics.passthrough"; }
    @Override public String getName() { return "PassThrough Tactics"; }
    @Override public Set<String> entryCriteria() { return Set.of(StarCraftCaseFile.READY); }
    @Override public Set<String> producedKeys() { return Set.of(); }

    @Override
    public void execute(CaseFile caseFile) {
        List<?> army = caseFile.get(StarCraftCaseFile.ARMY, List.class).orElse(List.of());
        log.debugf("[TACTICS] PassThrough activated — army size=%d", army.size());
    }
}
```

```java
// src/main/java/org/acme/starcraft/plugin/PassThroughScoutingTask.java
package org.acme.starcraft.plugin;

import io.casehub.core.CaseFile;
import jakarta.enterprise.context.ApplicationScoped;
import org.acme.starcraft.agent.StarCraftCaseFile;
import org.acme.starcraft.agent.plugin.ScoutingTask;
import org.jboss.logging.Logger;
import java.util.List;
import java.util.Set;

@ApplicationScoped
public class PassThroughScoutingTask implements ScoutingTask {
    private static final Logger log = Logger.getLogger(PassThroughScoutingTask.class);

    @Override public String getId() { return "scouting.passthrough"; }
    @Override public String getName() { return "PassThrough Scouting"; }
    @Override public Set<String> entryCriteria() { return Set.of(StarCraftCaseFile.READY); }
    @Override public Set<String> producedKeys() { return Set.of(); }

    @Override
    public void execute(CaseFile caseFile) {
        List<?> enemies = caseFile.get(StarCraftCaseFile.ENEMY_UNITS, List.class).orElse(List.of());
        log.debugf("[SCOUTING] PassThrough activated — visible enemy units=%d", enemies.size());
    }
}
```

- [ ] **Step 3: Compile**

```bash
mvn compile
```

Expected: `BUILD SUCCESS`. If `TaskDefinition` method signatures don't match (e.g. `getId()` not in the interface), check `/Users/mdproctor/claude/alpha/casehub-core/src/main/java/io/casehub/core/TaskDefinition.java` and adjust to match the actual interface.

- [ ] **Step 4: Commit**

```bash
git add src/
git commit -m "feat: plugin seam interfaces (StrategyTask, EconomicsTask, TacticsTask, ScoutingTask) + pass-through dummy implementations"
```

---

## Task 9: QA REST Endpoints

**Files:**
- Create: `src/main/java/org/acme/starcraft/qa/CaseFileResource.java`
- Create: `src/main/java/org/acme/starcraft/qa/IntentResource.java`
- Create: `src/main/java/org/acme/starcraft/qa/ScenarioResource.java`
- Test: `src/test/java/org/acme/starcraft/qa/QaEndpointsTest.java`

- [ ] **Step 1: Write the failing test**

```java
// src/test/java/org/acme/starcraft/qa/QaEndpointsTest.java
package org.acme.starcraft.qa;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;
import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class QaEndpointsTest {

    @Test
    void getGameStateReturnsJson() {
        given()
            .when().get("/sc2/casefile")
            .then()
            .statusCode(200)
            .contentType("application/json")
            .body("minerals", notNullValue())
            .body("supply", notNullValue());
    }

    @Test
    void getFrameReturnsFrameInfo() {
        given()
            .when().get("/sc2/frame")
            .then()
            .statusCode(200)
            .contentType("application/json")
            .body("gameFrame", notNullValue())
            .body("connected", notNullValue());
    }

    @Test
    void getIntentsPendingReturnsEmptyInitially() {
        given()
            .when().get("/sc2/intents/pending")
            .then()
            .statusCode(200)
            .contentType("application/json")
            .body("$", hasSize(0));
    }

    @Test
    void runKnownScenarioReturns204() {
        given()
            .when().post("/sc2/debug/scenario/set-resources-500")
            .then()
            .statusCode(204);
    }

    @Test
    void runUnknownScenarioReturns400() {
        given()
            .when().post("/sc2/debug/scenario/nonexistent")
            .then()
            .statusCode(400);
    }
}
```

Also add RestAssured to `pom.xml` test scope (if not already present from Quarkus generation):

```xml
<dependency>
  <groupId>io.rest-assured</groupId>
  <artifactId>rest-assured</artifactId>
  <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl . -Dtest=QaEndpointsTest -q
```

Expected: FAIL — endpoints not defined.

- [ ] **Step 3: Write QA resource classes**

```java
// src/main/java/org/acme/starcraft/qa/CaseFileResource.java
package org.acme.starcraft.qa;

import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import org.acme.starcraft.sc2.mock.SimulatedGame;
import org.acme.starcraft.domain.GameState;
import java.util.Map;

@Path("/sc2")
@Produces(MediaType.APPLICATION_JSON)
public class CaseFileResource {

    @Inject SimulatedGame simulatedGame;

    @GET
    @Path("/casefile")
    public GameState getGameState() {
        return simulatedGame.snapshot();
    }
}
```

```java
// src/main/java/org/acme/starcraft/qa/IntentResource.java
package org.acme.starcraft.qa;

import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import org.acme.starcraft.sc2.IntentQueue;
import org.acme.starcraft.sc2.mock.SimulatedGame;
import org.acme.starcraft.sc2.intent.Intent;
import java.util.List;
import java.util.Map;

@Path("/sc2")
@Produces(MediaType.APPLICATION_JSON)
public class IntentResource {

    @Inject IntentQueue intentQueue;
    @Inject SimulatedGame simulatedGame;

    @GET
    @Path("/intents/pending")
    public List<Intent> pending() {
        return intentQueue.pending();
    }

    @GET
    @Path("/intents/dispatched")
    public List<Intent> dispatched() {
        return intentQueue.recentlyDispatched();
    }

    @GET
    @Path("/frame")
    public Map<String, Object> frame() {
        var state = simulatedGame.snapshot();
        return Map.of(
            "gameFrame", state.gameFrame(),
            "connected", true,
            "pendingIntents", intentQueue.pending().size()
        );
    }
}
```

```java
// src/main/java/org/acme/starcraft/qa/ScenarioResource.java
package org.acme.starcraft.qa;

import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import org.acme.starcraft.sc2.ScenarioRunner;
import java.util.Map;

@Path("/sc2/debug")
@Produces(MediaType.APPLICATION_JSON)
public class ScenarioResource {

    @Inject ScenarioRunner scenarioRunner;

    @POST
    @Path("/scenario/{name}")
    public Response runScenario(@PathParam("name") String name) {
        try {
            scenarioRunner.run(name);
            return Response.noContent().build();
        } catch (IllegalArgumentException e) {
            return Response.status(Response.Status.BAD_REQUEST)
                .entity(Map.of("error", e.getMessage()))
                .type(MediaType.APPLICATION_JSON)
                .build();
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -pl . -Dtest=QaEndpointsTest -q
```

Expected: `BUILD SUCCESS`, 5 tests passed.

If Jackson cannot serialize `Intent` (sealed interface/records), add `@JsonTypeInfo` or register subtypes. The simplest fix: add `@JsonSubTypes` to `Intent` for Jackson polymorphism, or make the QA endpoints return `List<Map<String,Object>>` by converting intents to maps.

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "feat: QA REST endpoints — /sc2/casefile, /sc2/frame, /sc2/intents, /sc2/debug/scenario"
```

---

## Task 10: Integration Test — Full Mock Pipeline

**Files:**
- Test: `src/test/java/org/acme/starcraft/sc2/mock/FullMockPipelineIT.java`

- [ ] **Step 1: Write the integration test**

```java
// src/test/java/org/acme/starcraft/sc2/mock/FullMockPipelineIT.java
package org.acme.starcraft.sc2.mock;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.acme.starcraft.agent.AgentOrchestrator;
import org.acme.starcraft.domain.UnitType;
import org.acme.starcraft.sc2.IntentQueue;
import org.acme.starcraft.sc2.ScenarioRunner;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class FullMockPipelineIT {

    @Inject AgentOrchestrator orchestrator;
    @Inject SimulatedGame simulatedGame;
    @Inject IntentQueue intentQueue;
    @Inject ScenarioRunner scenarioRunner;

    @BeforeEach
    void setUp() {
        simulatedGame.reset();
        orchestrator.startGame();
    }

    @Test
    void gameTickAdvancesFrameAndDispatchesPipeline() throws InterruptedException {
        long frameBefore = simulatedGame.snapshot().gameFrame();
        // Manually trigger one tick (scheduler is disabled in %test)
        orchestrator.gameTick();
        long frameAfter = simulatedGame.snapshot().gameFrame();
        assertThat(frameAfter).isGreaterThan(frameBefore);
    }

    @Test
    void scenarioChangesGameStateAndPipelineReacts() throws Exception {
        // Apply scenario
        scenarioRunner.run("spawn-enemy-attack");
        assertThat(simulatedGame.snapshot().enemyUnits()).isNotEmpty();

        // Run a tick — pipeline should observe the enemy
        orchestrator.gameTick();

        // Dummy plugins produce no intents but pipeline should complete without error
        assertThat(intentQueue.pending()).isEmpty();
        assertThat(simulatedGame.snapshot().enemyUnits()).isNotEmpty(); // still visible
    }

    @Test
    void setResourcesScenarioVisibleInNextTick() throws Exception {
        scenarioRunner.run("set-resources-500");
        assertThat(simulatedGame.snapshot().minerals()).isEqualTo(500);

        orchestrator.gameTick();
        // Verify no crash — minerals observed correctly by pipeline
        assertThat(simulatedGame.snapshot().minerals()).isGreaterThanOrEqualTo(500); // trickle may have added more
    }

    @Test
    void allScenariosRunWithoutException() {
        scenarioRunner.availableScenarios().forEach(name -> {
            simulatedGame.reset();
            orchestrator.startGame();
            scenarioRunner.run(name);
            orchestrator.gameTick(); // pipeline reacts without error
        });
    }
}
```

- [ ] **Step 2: Run test to verify it passes**

```bash
mvn test -pl . -Dtest=FullMockPipelineIT -q
```

Expected: `BUILD SUCCESS`, 4 tests passed.

If CaseEngine throws because no TaskDefinitions are registered for `"starcraft-game"` case type: check whether CaseHub uses `@CaseType` CDI qualifier on TaskDefinitions. If so, annotate all four pass-through implementations with `@io.casehub.annotation.CaseType("starcraft-game")` and re-run.

- [ ] **Step 3: Run full test suite**

```bash
mvn test
```

Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 4: Commit**

```bash
git add src/
git commit -m "feat: full mock pipeline integration test — all 4 dummy plugins, all scenarios, CaseEngine cycling verified"
```

---

## Done When

- [ ] `mvn test` passes with zero failures
- [ ] `mvn quarkus:dev` starts without errors, structured logs show CaseEngine cycling with dummy plugins
- [ ] `GET http://localhost:8080/sc2/casefile` returns current SimulatedGame state as JSON
- [ ] `GET http://localhost:8080/sc2/frame` returns frame counter
- [ ] `POST http://localhost:8080/sc2/debug/scenario/spawn-enemy-attack` returns 204 and enemy units appear in `/sc2/casefile`
- [ ] `NATIVE.md` updated with any issues found during implementation
