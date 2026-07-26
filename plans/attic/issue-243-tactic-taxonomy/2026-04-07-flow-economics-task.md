# FlowEconomicsTask Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace `BasicEconomicsTask` with `FlowEconomicsTask` — a Quarkus Flow–backed plugin that manages Protoss economics (probes, pylons, assimilators, expansion) as a stateful multi-phase workflow persisting across the whole game.

**Architecture:** `FlowEconomicsTask.execute()` emits a `GameStateTick` event on a SmallRye in-memory channel. A long-lived `EconomicsFlow` instance listens for tick events, runs four decision functions per tick, and writes intents directly to `IntentQueue`. One-tick lag is accepted by design (ADR-0001).

**Tech Stack:** Quarkus Flow 0.7.1 (`io.quarkiverse.flow:quarkus-flow`), SmallRye Reactive Messaging (`io.quarkus:quarkus-messaging`), CDI events, plain JUnit + AssertJ for unit tests, `@QuarkusTest` for integration.

**Spec:** `docs/superpowers/specs/2026-04-07-flow-economics-design.md`
**ADR:** `docs/adr/0001-quarkus-flow-placement.md`

---

## File Map

| Action | File | Responsibility |
|---|---|---|
| Create | `src/main/java/org/acme/starcraft/domain/GameStateTick.java` | Tick event payload record — raw game state snapshot |
| Create | `src/main/java/org/acme/starcraft/domain/EconomicsContext.java` | Flow workflow context record + Phase enum |
| Create | `src/main/java/org/acme/starcraft/sc2/GameStarted.java` | CDI event record — fired when game begins |
| Create | `src/main/java/org/acme/starcraft/sc2/GameStopped.java` | CDI event record — fired when game ends |
| Create | `src/main/java/org/acme/starcraft/plugin/FlowEconomicsTask.java` | CaseHub shim: emits `GameStateTick` on channel |
| Create | `src/main/java/org/acme/starcraft/plugin/flow/EconomicsDecisionService.java` | All decision logic: supply, probes, gas, expansion |
| Create | `src/main/java/org/acme/starcraft/plugin/flow/EconomicsFlow.java` | Quarkus Flow workflow descriptor |
| Create | `src/main/java/org/acme/starcraft/plugin/flow/EconomicsLifecycle.java` | Starts/stops flow instance on GameStarted/GameStopped |
| Modify | `pom.xml` | Add quarkus-flow + quarkus-messaging dependencies |
| Modify | `src/main/java/org/acme/starcraft/plugin/BasicEconomicsTask.java` | Remove `@ApplicationScoped` + `@CaseType` (demote from CDI) |
| Modify | `src/main/java/org/acme/starcraft/plugin/DroolsStrategyTask.java` | Remove ASSIMILATOR dispatch branch |
| Modify | `src/main/java/org/acme/starcraft/agent/AgentOrchestrator.java` | Fire `GameStarted`/`GameStopped` CDI events |
| Modify | `src/main/resources/application.properties` | Configure in-memory SmallRye channel |
| Create | `src/test/java/org/acme/starcraft/plugin/EconomicsDecisionServiceTest.java` | Plain JUnit unit tests for decision logic |
| Create | `src/test/java/org/acme/starcraft/plugin/EconomicsFlowTest.java` | `@QuarkusTest` end-to-end pipeline test |

---

## Task 1: Add Dependencies

**Files:**
- Modify: `pom.xml`

- [ ] **Add quarkus-flow and quarkus-messaging to `pom.xml`**

In the `<dependencies>` section, after the `quarkus-smallrye-fault-tolerance` dependency:

```xml
        <!-- Quarkus Flow — stateful workflow engine (CNCF Serverless Workflow) -->
        <!-- See docs/adr/0001-quarkus-flow-placement.md and NATIVE.md -->
        <dependency>
            <groupId>io.quarkiverse.flow</groupId>
            <artifactId>quarkus-flow</artifactId>
            <version>0.7.1</version>
        </dependency>

        <!-- SmallRye Reactive Messaging — in-memory channel bridge to EconomicsFlow -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-messaging</artifactId>
        </dependency>
```

- [ ] **Verify compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS (no output).

- [ ] **Commit**

```bash
git add pom.xml
git commit -m "feat(deps): add quarkus-flow 0.7.1 and quarkus-messaging"
```

---

## Task 2: Domain Records — GameStateTick and EconomicsContext

**Files:**
- Create: `src/main/java/org/acme/starcraft/domain/GameStateTick.java`
- Create: `src/main/java/org/acme/starcraft/domain/EconomicsContext.java`

These are plain Java — no CDI, no framework deps. They live in `domain/` per project convention.

- [ ] **Create `GameStateTick.java`**

```java
package org.acme.starcraft.domain;

import org.acme.starcraft.agent.ResourceBudget;

import java.util.List;

/**
 * Snapshot of game state emitted once per CaseHub tick by FlowEconomicsTask.
 *
 * <p>The budget is a per-tick snapshot copy — independent of the CaseHub shared
 * ResourceBudget, which has already been partially consumed by other plugins
 * by the time the flow processes this tick (one-tick lag; see ADR-0001).
 *
 * <p>gasReady is computed in FlowEconomicsTask from buildings: true when a Gateway
 * exists. When Drools CEP lands it becomes StarCraftCaseFile.SIGNAL_GAS_READY.
 */
public record GameStateTick(
        int minerals,
        int vespene,
        int supplyUsed,
        int supplyCap,
        List<Unit> workers,
        List<Building> buildings,
        List<Resource> geysers,
        ResourceBudget budget,
        String strategy,
        boolean gasReady
) {}
```

- [ ] **Create `EconomicsContext.java`**

```java
package org.acme.starcraft.domain;

/**
 * Quarkus Flow workflow context for the economics workflow.
 *
 * <p>Carries phase state and nexus count across ticks. Phase drives which
 * decisions are active; nexusCount determines probeTarget (22 × nexusCount)
 * for multi-base probe management (initial implementation uses 1 base).
 */
public record EconomicsContext(Phase phase, int nexusCount) {

    public enum Phase {
        /** Normal operation: probe saturation + supply management. */
        SATURATION,
        /** Saturation reached: expansion nexus queued, awaiting build. */
        EXPANDING
    }

    /** Probe target scales with active nexuses. */
    public int probeTarget() {
        return nexusCount * 22;
    }

    public static EconomicsContext initial() {
        return new EconomicsContext(Phase.SATURATION, 1);
    }
}
```

- [ ] **Verify compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Commit**

```bash
git add src/main/java/org/acme/starcraft/domain/GameStateTick.java \
        src/main/java/org/acme/starcraft/domain/EconomicsContext.java
git commit -m "feat(domain): add GameStateTick and EconomicsContext records"
```

---

## Task 3: CDI Lifecycle Events + Update AgentOrchestrator

**Files:**
- Create: `src/main/java/org/acme/starcraft/sc2/GameStarted.java`
- Create: `src/main/java/org/acme/starcraft/sc2/GameStopped.java`
- Modify: `src/main/java/org/acme/starcraft/agent/AgentOrchestrator.java`

- [ ] **Create `GameStarted.java`**

```java
package org.acme.starcraft.sc2;

/** CDI event fired when a game begins. Observed by EconomicsLifecycle. */
public record GameStarted() {}
```

- [ ] **Create `GameStopped.java`**

```java
package org.acme.starcraft.sc2;

/** CDI event fired when a game ends. Observed by EconomicsLifecycle. */
public record GameStopped() {}
```

- [ ] **Update `AgentOrchestrator.java` to fire lifecycle events**

Add the CDI event injections and fire them in `startGame()`/`stopGame()`. The full updated file:

```java
package org.acme.starcraft.agent;

import io.casehub.coordination.CaseEngine;
import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import org.acme.starcraft.sc2.GameStarted;
import org.acme.starcraft.sc2.GameStopped;
import org.acme.starcraft.sc2.SC2Engine;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.util.Map;

@ApplicationScoped
public class AgentOrchestrator {
    private static final Logger log = Logger.getLogger(AgentOrchestrator.class);

    @Inject SC2Engine engine;
    @Inject GameStateTranslator translator;
    @Inject CaseEngine caseEngine;
    @Inject Event<GameStarted> gameStartedEvent;
    @Inject Event<GameStopped> gameStoppedEvent;

    public void startGame() {
        engine.connect();
        engine.joinGame();
        gameStartedEvent.fire(new GameStarted());
        log.info("Game started");
    }

    public void stopGame() {
        engine.leaveGame();
        gameStoppedEvent.fire(new GameStopped());
        log.info("Game stopped");
    }

    @Scheduled(every = "${starcraft.tick.interval:500ms}", concurrentExecution = Scheduled.ConcurrentExecution.SKIP)
    public void gameTick() {
        if (!engine.isConnected()) return;

        engine.tick();

        var gameState = engine.observe();
        Map<String, Object> caseData = translator.toMap(gameState);

        try {
            caseEngine.createAndSolve("starcraft-game", caseData, Duration.ofSeconds(5));
        } catch (Exception e) {
            log.errorf("CaseEngine decision cycle failed at frame %d: %s", gameState.gameFrame(), e.getMessage());
        }

        engine.dispatch();
        log.debugf("Tick complete — frame=%d minerals=%d supplyUsed=%d/%d",
            gameState.gameFrame(), gameState.minerals(), gameState.supplyUsed(), gameState.supply());
    }
}
```

- [ ] **Run existing tests to confirm nothing broke**

```bash
mvn test -q
```

Expected: All 107 tests pass.

- [ ] **Commit**

```bash
git add src/main/java/org/acme/starcraft/sc2/GameStarted.java \
        src/main/java/org/acme/starcraft/sc2/GameStopped.java \
        src/main/java/org/acme/starcraft/agent/AgentOrchestrator.java
git commit -m "feat(lifecycle): add GameStarted/GameStopped events; AgentOrchestrator fires them"
```

---

## Task 4: Demote BasicEconomicsTask + Remove ASSIMILATOR from DroolsStrategyTask

**Files:**
- Modify: `src/main/java/org/acme/starcraft/plugin/BasicEconomicsTask.java`
- Modify: `src/main/java/org/acme/starcraft/plugin/DroolsStrategyTask.java`

Assimilator building moves from `DroolsStrategyTask` to `FlowEconomicsTask` (economics decision, not strategy). The Drools rule still fires and writes "ASSIMILATOR" to `buildDecisions`, but `DroolsStrategyTask.dispatchBuildDecisions()` no longer acts on it.

- [ ] **Demote `BasicEconomicsTask` from CDI and make `pylonPosition` public**

Remove `@ApplicationScoped` and `@CaseType("starcraft-game")` annotations. Make `pylonPosition` public so `EconomicsDecisionService` (in sub-package `plugin.flow`) can call it. Update the Javadoc status line:

```java
// OLD:
@ApplicationScoped
@CaseType("starcraft-game")
public class BasicEconomicsTask implements EconomicsTask {
```

```java
// NEW — remove both annotations, update Javadoc status line:
/**
 * Basic Protoss economics: probe production and pylon supply management.
 * ...
 * <p><b>Status:</b> superseded by {@link FlowEconomicsTask} as the active CaseHub plugin.
 * Retained as a plain class for direct-instantiation tests and as a reference implementation.
 */
public class BasicEconomicsTask implements EconomicsTask {
```

Also remove the `@Inject` annotation from the constructor, and change `pylonPosition` visibility from package-private to public:

```java
// OLD:
    @Inject
    public BasicEconomicsTask(IntentQueue intentQueue) {
    ...
    static Point2d pylonPosition(int buildingCount) {

// NEW:
    public BasicEconomicsTask(IntentQueue intentQueue) {
    ...
    public static Point2d pylonPosition(int buildingCount) {
```

- [ ] **Remove ASSIMILATOR dispatch from `DroolsStrategyTask`**

In `DroolsStrategyTask.dispatchBuildDecisions()`, remove the `else if (decision.equals("ASSIMILATOR") ...)` branch entirely:

```java
// REMOVE this block:
            } else if (decision.equals("ASSIMILATOR") && budget.spendMinerals(75)) {
                Set<Point2d> occupied = buildings.stream()
                    .filter(b -> b.type() == BuildingType.ASSIMILATOR)
                    .map(Building::position).collect(Collectors.toSet());
                workers.stream().findFirst().ifPresent(p ->
                    geysers.stream()
                        .filter(g -> !occupied.contains(g.position()))
                        .findFirst()
                        .ifPresent(g -> intentQueue.add(
                            new BuildIntent(p.tag(), BuildingType.ASSIMILATOR, g.position()))));
```

Also remove any unused imports that result (check for `Collectors` — it may still be used for the STALKER branch, keep it if so). Update the method Javadoc to remove "ASSIMILATOR" from the listed decisions.

- [ ] **Run tests**

```bash
mvn test -q
```

Expected: All 107 tests pass. If any `DroolsStrategyTaskTest` tests assert assimilator building via `StrategyTask`, they will now fail — those tests should be removed (assimilator is now `FlowEconomicsTask`'s responsibility). Check and remove any such tests.

- [ ] **Commit**

```bash
git add src/main/java/org/acme/starcraft/plugin/BasicEconomicsTask.java \
        src/main/java/org/acme/starcraft/plugin/DroolsStrategyTask.java
git commit -m "refactor(plugin): demote BasicEconomicsTask; move assimilator ownership to FlowEconomicsTask"
```

---

## Task 5: TDD EconomicsDecisionService

**Files:**
- Create: `src/test/java/org/acme/starcraft/plugin/EconomicsDecisionServiceTest.java`
- Create: `src/main/java/org/acme/starcraft/plugin/flow/EconomicsDecisionService.java`

Plain JUnit — no Quarkus boot. All four decision functions tested in isolation.

- [ ] **Write the failing test class**

```java
package org.acme.starcraft.plugin;

import org.acme.starcraft.agent.ResourceBudget;
import org.acme.starcraft.domain.*;
import org.acme.starcraft.plugin.flow.EconomicsDecisionService;
import org.acme.starcraft.sc2.IntentQueue;
import org.acme.starcraft.sc2.intent.BuildIntent;
import org.acme.starcraft.sc2.intent.TrainIntent;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.stream.IntStream;

import static org.assertj.core.api.Assertions.assertThat;

class EconomicsDecisionServiceTest {

    IntentQueue intentQueue;
    EconomicsDecisionService svc;

    @BeforeEach
    void setUp() {
        intentQueue = new IntentQueue();
        svc = new EconomicsDecisionService(intentQueue);
        intentQueue.drainAll();
    }

    // ─── checkSupply ────────────────────────────────────────────────────────

    @Test
    void buildsPylonWhenSupplyHeadroomLow() {
        // supplyUsed=13, supplyCap=15 → headroom=2, below threshold of 4
        var tick = tick(300, 13, 15, workers(6), buildings(nexus()), List.of());
        svc.checkSupply(tick);
        assertThat(intentQueue.pending())
            .hasSize(1)
            .first().isInstanceOf(BuildIntent.class);
        assertThat(((BuildIntent) intentQueue.pending().get(0)).buildingType())
            .isEqualTo(BuildingType.PYLON);
    }

    @Test
    void noPylonWhenSupplyHeadroomSufficient() {
        // supplyUsed=8, supplyCap=15 → headroom=7, above threshold
        var tick = tick(300, 8, 15, workers(6), buildings(nexus()), List.of());
        svc.checkSupply(tick);
        assertThat(intentQueue.pending()).isEmpty();
    }

    @Test
    void noPylonAtMaxSupply() {
        var tick = tick(500, 196, 200, workers(6), buildings(nexus()), List.of());
        svc.checkSupply(tick);
        assertThat(intentQueue.pending()).isEmpty();
    }

    @Test
    void noPylonWithoutMinerals() {
        var tick = tick(50, 13, 15, workers(6), buildings(nexus()), List.of());
        svc.checkSupply(tick);
        assertThat(intentQueue.pending()).isEmpty();
    }

    // ─── checkProbes ─────────────────────────────────────────────────────────

    @Test
    void trainsProbeWhenUnderCap() {
        // supply headroom=9 (well above threshold) so only probe rule fires
        var tick = tick(200, 6, 15, workers(6), buildings(nexus()), List.of());
        svc.checkProbes(tick);
        assertThat(intentQueue.pending())
            .hasSize(1)
            .first().isInstanceOf(TrainIntent.class);
        assertThat(((TrainIntent) intentQueue.pending().get(0)).unitType())
            .isEqualTo(UnitType.PROBE);
    }

    @Test
    void noProbeWhenAtCap() {
        var tick = tick(500, 22, 50, workers(22), buildings(nexus()), List.of());
        svc.checkProbes(tick);
        assertThat(intentQueue.pending()).isEmpty();
    }

    @Test
    void noProbeWithoutNexus() {
        var tick = tick(200, 6, 15, workers(6), List.of(), List.of());
        svc.checkProbes(tick);
        assertThat(intentQueue.pending()).isEmpty();
    }

    @Test
    void noProbeWithoutMinerals() {
        var tick = tick(40, 6, 15, workers(6), buildings(nexus()), List.of());
        svc.checkProbes(tick);
        assertThat(intentQueue.pending()).isEmpty();
    }

    @Test
    void noProbeFromIncompleteNexus() {
        Building incomplete = new Building("n-0", BuildingType.NEXUS, new Point2d(8, 8), 1500, 1500, false);
        var tick = tick(200, 6, 15, workers(6), List.of(incomplete), List.of());
        svc.checkProbes(tick);
        assertThat(intentQueue.pending()).isEmpty();
    }

    // ─── checkGas ────────────────────────────────────────────────────────────

    @Test
    void buildsAssimilatorWhenGatewayExistsAndNoneBuilt() {
        // Resource(tag, position, remaining) — no ResourceType field
        Resource geyser = new Resource("g-0", new Point2d(20, 20), 2250);
        var tick = tick(200, 6, 15, workers(6),
            buildings(nexus(), completeGateway()), List.of(geyser));
        svc.checkGas(tick);
        assertThat(intentQueue.pending())
            .hasSize(1)
            .first().isInstanceOf(BuildIntent.class);
        assertThat(((BuildIntent) intentQueue.pending().get(0)).buildingType())
            .isEqualTo(BuildingType.ASSIMILATOR);
    }

    @Test
    void noAssimilatorWithoutGateway() {
        Resource geyser = new Resource("g-0", new Point2d(20, 20), 2250);
        // gasReady=false in tick (no gateway in buildings)
        var tick = tick(200, 6, 15, workers(6), buildings(nexus()), List.of(geyser));
        svc.checkGas(tick);
        assertThat(intentQueue.pending()).isEmpty();
    }

    @Test
    void noAssimilatorWhenBothAlreadyBuilt() {
        Resource g1 = new Resource("g-0", new Point2d(20, 20), 2250);
        Resource g2 = new Resource("g-1", new Point2d(22, 20), 2250);
        Building a1 = new Building("a-0", BuildingType.ASSIMILATOR, new Point2d(20, 20), 450, 450, true);
        Building a2 = new Building("a-1", BuildingType.ASSIMILATOR, new Point2d(22, 20), 450, 450, true);
        var tick = tick(500, 6, 15, workers(6),
            buildings(nexus(), completeGateway(), a1, a2), List.of(g1, g2));
        svc.checkGas(tick);
        assertThat(intentQueue.pending()).isEmpty();
    }

    @Test
    void noAssimilatorWithoutGeysers() {
        var tick = tick(200, 6, 15, workers(6), buildings(nexus(), completeGateway()), List.of());
        svc.checkGas(tick);
        assertThat(intentQueue.pending()).isEmpty();
    }

    // ─── checkExpansion ──────────────────────────────────────────────────────

    @Test
    void queuesNexusWhenSaturatedAndAffordable() {
        // 22 workers, 1 nexus, 400 minerals
        var tick = tick(500, 22, 50, workers(22), buildings(nexus()), List.of());
        svc.checkExpansion(tick);
        assertThat(intentQueue.pending())
            .hasSize(1)
            .first().isInstanceOf(BuildIntent.class);
        assertThat(((BuildIntent) intentQueue.pending().get(0)).buildingType())
            .isEqualTo(BuildingType.NEXUS);
    }

    @Test
    void noExpansionWhenWorkersUnderCap() {
        var tick = tick(500, 15, 50, workers(15), buildings(nexus()), List.of());
        svc.checkExpansion(tick);
        assertThat(intentQueue.pending()).isEmpty();
    }

    @Test
    void noExpansionWhenSecondNexusAlreadyExists() {
        Building nexus2 = new Building("n-1", BuildingType.NEXUS, new Point2d(32, 32), 1500, 1500, false);
        var tick = tick(500, 22, 50, workers(22), buildings(nexus(), nexus2), List.of());
        svc.checkExpansion(tick);
        assertThat(intentQueue.pending()).isEmpty();
    }

    @Test
    void noExpansionWithoutMinerals() {
        var tick = tick(350, 22, 50, workers(22), buildings(nexus()), List.of());
        svc.checkExpansion(tick);
        assertThat(intentQueue.pending()).isEmpty();
    }

    // ─── Helpers ─────────────────────────────────────────────────────────────

    private GameStateTick tick(int minerals, int supplyUsed, int supplyCap,
                               List<Unit> workers, List<Building> buildings,
                               List<Resource> geysers) {
        return new GameStateTick(minerals, 0, supplyUsed, supplyCap,
            workers, buildings, geysers,
            new ResourceBudget(minerals, 0), "MACRO", false);
    }

    private List<Unit> workers(int count) {
        return IntStream.range(0, count)
            .mapToObj(i -> new Unit("probe-" + i, UnitType.PROBE, new Point2d(9, 9), 45, 45))
            .toList();
    }

    private List<Building> buildings(Building... bs) { return List.of(bs); }

    private Building nexus() {
        return new Building("n-0", BuildingType.NEXUS, new Point2d(8, 8), 1500, 1500, true);
    }

    private Building completeGateway() {
        return new Building("gw-0", BuildingType.GATEWAY, new Point2d(17, 18), 550, 550, true);
    }
}
```

- [ ] **Run tests — expect compilation failure** (EconomicsDecisionService doesn't exist yet)

```bash
mvn test -Dtest=EconomicsDecisionServiceTest -q 2>&1 | head -20
```

Expected: compile error — `EconomicsDecisionService` not found.

- [ ] **Create `EconomicsDecisionService.java`**

```java
package org.acme.starcraft.plugin.flow;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.acme.starcraft.domain.*;
import org.acme.starcraft.plugin.BasicEconomicsTask;
import org.acme.starcraft.sc2.IntentQueue;
import org.acme.starcraft.sc2.intent.BuildIntent;
import org.acme.starcraft.sc2.intent.TrainIntent;
import org.jboss.logging.Logger;

import java.util.Set;
import java.util.stream.Collectors;

/**
 * All per-tick economics decisions for the Flow workflow.
 *
 * <p>Called by {@link EconomicsFlow} on each tick event. Methods are side-effecting
 * — they add to {@link IntentQueue} when conditions are met. The budget in
 * {@link GameStateTick} is a per-tick snapshot independent of the CaseHub shared budget.
 *
 * <p>Assimilator ownership transferred here from DroolsStrategyTask (economics decision,
 * not strategy). Pylon position logic reused from {@link BasicEconomicsTask#pylonPosition}.
 */
@ApplicationScoped
public class EconomicsDecisionService {

    static final int PROBE_CAP       = 22;
    static final int SUPPLY_HEADROOM = 4;
    static final int MAX_SUPPLY      = 200;
    static final int PYLON_COST      = 100;
    static final int PROBE_COST      = 50;
    static final int ASSIMILATOR_COST = 75;
    static final int NEXUS_COST      = 400;

    // Hardcoded natural expansion position — Phase 3+ will use spatial analysis
    static final Point2d NATURAL_EXPANSION_POS = new Point2d(32, 32);

    private static final Logger log = Logger.getLogger(EconomicsDecisionService.class);

    private final IntentQueue intentQueue;

    @Inject
    public EconomicsDecisionService(IntentQueue intentQueue) {
        this.intentQueue = intentQueue;
    }

    /** Builds a Pylon when supply headroom drops to {@link #SUPPLY_HEADROOM} or below. */
    public void checkSupply(GameStateTick tick) {
        if (tick.supplyCap() >= MAX_SUPPLY) return;
        if (tick.supplyUsed() < tick.supplyCap() - SUPPLY_HEADROOM) return;
        if (!tick.budget().spendMinerals(PYLON_COST)) return;
        tick.workers().stream().findFirst().ifPresent(probe -> {
            Point2d pos = BasicEconomicsTask.pylonPosition(tick.buildings().size());
            intentQueue.add(new BuildIntent(probe.tag(), BuildingType.PYLON, pos));
            log.debugf("[FLOW-ECONOMICS] Pylon at %s (supply %d/%d)",
                pos, tick.supplyUsed(), tick.supplyCap());
        });
    }

    /** Trains a Probe when worker count is below {@link #PROBE_CAP} and budget allows. */
    public void checkProbes(GameStateTick tick) {
        if (tick.workers().size() >= PROBE_CAP) return;
        if (!tick.budget().spendMinerals(PROBE_COST)) return;
        tick.buildings().stream()
            .filter(b -> b.type() == BuildingType.NEXUS && b.isComplete())
            .findFirst()
            .ifPresent(nexus -> {
                intentQueue.add(new TrainIntent(nexus.tag(), UnitType.PROBE));
                log.debugf("[FLOW-ECONOMICS] Probe (workers=%d/%d)", tick.workers().size(), PROBE_CAP);
            });
    }

    /**
     * Builds Assimilators (up to 2) once a Gateway exists.
     * Uses tick.gasReady() — set by FlowEconomicsTask when a Gateway is in buildings.
     * Ownership transferred from DroolsStrategyTask (see ADR-0001, spec §assimilator ownership).
     */
    public void checkGas(GameStateTick tick) {
        if (!tick.gasReady()) return;

        long assimilatorCount = tick.buildings().stream()
            .filter(b -> b.type() == BuildingType.ASSIMILATOR).count();
        if (assimilatorCount >= 2) return;

        Set<Point2d> occupied = tick.buildings().stream()
            .filter(b -> b.type() == BuildingType.ASSIMILATOR)
            .map(Building::position)
            .collect(Collectors.toSet());

        tick.geysers().stream()
            .filter(g -> !occupied.contains(g.position()))
            .findFirst()
            .ifPresent(geyser -> {
                if (!tick.budget().spendMinerals(ASSIMILATOR_COST)) return;
                tick.workers().stream().findFirst().ifPresent(probe -> {
                    intentQueue.add(new BuildIntent(probe.tag(), BuildingType.ASSIMILATOR,
                        geyser.position()));
                    log.debugf("[FLOW-ECONOMICS] Assimilator at %s", geyser.position());
                });
            });
    }

    /**
     * Queues a Nexus at the natural expansion when saturated (22 workers, 1 active Nexus).
     * Expansion position is hardcoded — Phase 3+ will use SC2MapAnalysis sidecar.
     */
    public void checkExpansion(GameStateTick tick) {
        long nexusCount = tick.buildings().stream()
            .filter(b -> b.type() == BuildingType.NEXUS).count();
        if (nexusCount >= 2) return;
        if (tick.workers().size() < PROBE_CAP) return;
        if (!tick.budget().spendMinerals(NEXUS_COST)) return;
        tick.workers().stream().findFirst().ifPresent(probe -> {
            intentQueue.add(new BuildIntent(probe.tag(), BuildingType.NEXUS, NATURAL_EXPANSION_POS));
            log.infof("[FLOW-ECONOMICS] Expanding to natural at %s", NATURAL_EXPANSION_POS);
        });
    }
}
```

- [ ] **Run tests — expect pass**

```bash
mvn test -Dtest=EconomicsDecisionServiceTest -q
```

Expected: All tests in `EconomicsDecisionServiceTest` pass.

> **Note on ResourceType:** If `ResourceType` doesn't exist in the domain, replace `ResourceType.VESPENE_GEYSER` in the test helper with whatever enum or constructor `Resource` uses. Check `Resource.java` in `domain/` before running.

- [ ] **Run all tests**

```bash
mvn test -q
```

Expected: All tests pass (count increases by number of new `EconomicsDecisionServiceTest` tests).

- [ ] **Commit**

```bash
git add src/test/java/org/acme/starcraft/plugin/EconomicsDecisionServiceTest.java \
        src/main/java/org/acme/starcraft/plugin/flow/EconomicsDecisionService.java
git commit -m "feat(economics): TDD EconomicsDecisionService — supply, probes, gas, expansion"
```

---

## Task 6: FlowEconomicsTask — CaseHub Plugin Shim

**Files:**
- Create: `src/main/java/org/acme/starcraft/plugin/FlowEconomicsTask.java`

- [ ] **Create `FlowEconomicsTask.java`**

```java
package org.acme.starcraft.plugin;

import io.casehub.annotation.CaseType;
import io.casehub.core.CaseFile;
import io.smallrye.reactive.messaging.MutinyEmitter;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.acme.starcraft.agent.ResourceBudget;
import org.acme.starcraft.agent.StarCraftCaseFile;
import org.acme.starcraft.agent.plugin.EconomicsTask;
import org.acme.starcraft.domain.*;
import org.eclipse.microprofile.reactive.messaging.Channel;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Set;

/**
 * CaseHub plugin shim for the Flow-backed economics implementation.
 *
 * <p>Each CaseHub tick, this class extracts game state from the CaseFile into a
 * {@link GameStateTick} and emits it on the {@code economics-ticks} in-memory channel.
 * The long-lived {@link org.acme.starcraft.plugin.flow.EconomicsFlow} instance processes
 * the tick asynchronously (one-tick lag — see ADR-0001).
 *
 * <p>The budget in the tick is a <em>snapshot copy</em> — independent of the CaseHub
 * shared ResourceBudget, which may already be partially consumed by other plugins
 * before the flow processes it.
 */
@ApplicationScoped
@CaseType("starcraft-game")
public class FlowEconomicsTask implements EconomicsTask {

    private static final Logger log = Logger.getLogger(FlowEconomicsTask.class);

    @Inject
    @Channel("economics-ticks")
    MutinyEmitter<GameStateTick> emitter;

    @Override public String getId()   { return "economics.flow"; }
    @Override public String getName() { return "Flow Economics"; }
    @Override public Set<String> entryCriteria() { return Set.of(StarCraftCaseFile.READY); }
    @Override public Set<String> producedKeys()  { return Set.of(); }

    @Override
    @SuppressWarnings("unchecked")
    public void execute(CaseFile caseFile) {
        List<Unit>     workers   = (List<Unit>)     caseFile.get(StarCraftCaseFile.WORKERS,      List.class).orElse(List.of());
        List<Building> buildings = (List<Building>) caseFile.get(StarCraftCaseFile.MY_BUILDINGS, List.class).orElse(List.of());
        List<Resource> geysers   = (List<Resource>) caseFile.get(StarCraftCaseFile.GEYSERS,      List.class).orElse(List.of());
        int supplyUsed = caseFile.get(StarCraftCaseFile.SUPPLY_USED, Integer.class).orElse(0);
        int supplyCap  = caseFile.get(StarCraftCaseFile.SUPPLY_CAP,  Integer.class).orElse(0);
        int minerals   = caseFile.get(StarCraftCaseFile.MINERALS,    Integer.class).orElse(0);
        int vespene    = caseFile.get(StarCraftCaseFile.VESPENE,     Integer.class).orElse(0);
        String strategy = caseFile.get(StarCraftCaseFile.STRATEGY, String.class).orElse("MACRO");
        ResourceBudget shared = caseFile.get(StarCraftCaseFile.RESOURCE_BUDGET, ResourceBudget.class)
            .orElse(new ResourceBudget(0, 0));

        // Snapshot copy — flow processes this one tick later; shared budget already consumed
        ResourceBudget snapshot = new ResourceBudget(shared.minerals(), shared.vespene());

        boolean gasReady = buildings.stream().anyMatch(b -> b.type() == BuildingType.GATEWAY);

        GameStateTick tick = new GameStateTick(minerals, vespene, supplyUsed, supplyCap,
            workers, buildings, geysers, snapshot, strategy, gasReady);

        emitter.sendAndForget(tick);
        log.debugf("[FLOW-ECONOMICS] Tick emitted: workers=%d supply=%d/%d gasReady=%b",
            workers.size(), supplyUsed, supplyCap, gasReady);
    }
}
```

- [ ] **Verify compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS.

> **Note on `MutinyEmitter`:** If `io.smallrye.mutiny.infrastructure.Infrastructure` isn't needed, remove it. The `@Channel` + `MutinyEmitter<T>` pattern is standard Quarkus Reactive Messaging. If the API differs in 3.34.x, use `@Channel("economics-ticks") Emitter<GameStateTick> emitter` (non-Mutiny variant).

- [ ] **Run all tests**

```bash
mvn test -q
```

Expected: All tests pass (no channel consumer yet — emitted messages are dropped, which is fine).

- [ ] **Commit**

```bash
git add src/main/java/org/acme/starcraft/plugin/FlowEconomicsTask.java
git commit -m "feat(plugin): FlowEconomicsTask — CaseHub shim emitting GameStateTick on economics-ticks channel"
```

---

## Task 7: EconomicsFlow — The Workflow

**Files:**
- Create: `src/main/java/org/acme/starcraft/plugin/flow/EconomicsFlow.java`

> **API verification step:** Quarkus Flow 0.7.1 may have API differences from the examples in this plan. Before writing the full flow, verify the exact `listen` + channel integration by checking the quarkus-flow 0.7.1 test suite or docs at `https://github.com/quarkiverse/quarkus-flow`. The key questions: (1) does `listen(name, toOne(type))` receive from a SmallRye in-memory channel, or does it require a separate event mechanism? (2) what is the correct loop-back syntax?

- [ ] **Create `EconomicsFlow.java`**

```java
package org.acme.starcraft.plugin.flow;

import io.quarkiverse.flow.Flow;
import io.serverlessworkflow.api.types.Workflow;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.acme.starcraft.domain.GameStateTick;

import static io.serverlessworkflow.fluent.func.dsl.FuncDSL.*;
import static io.serverlessworkflow.fluent.func.spec.FuncWorkflowBuilder.workflow;

/**
 * Quarkus Flow workflow managing Protoss economics across the game lifetime.
 *
 * <p>Started once per game (via {@link EconomicsLifecycle}), this workflow loops
 * indefinitely: listening for {@link GameStateTick} events from the
 * {@code economics-ticks} channel and invoking four decision functions per tick.
 *
 * <p>Each function is a side-effecting call into {@link EconomicsDecisionService},
 * which writes to {@link org.acme.starcraft.sc2.IntentQueue}.
 *
 * <p>Phase state (SATURATION vs EXPANDING) is tracked in the workflow context
 * via {@code exportAs} on the phase-transition step. The Mermaid diagram of
 * this workflow is visible in the Quarkus Dev UI.
 *
 * <p>One-tick lag is a design decision — see ADR-0001.
 */
@ApplicationScoped
public class EconomicsFlow extends Flow {

    @Inject
    EconomicsDecisionService decisions;

    @Override
    public Workflow descriptor() {
        return workflow("starcraft-economics")
            .tasks(
                // Wait for the next GameStateTick from FlowEconomicsTask
                listen("tick", toOne("org.acme.starcraft.economics.tick")),

                // Supply management — always runs, regardless of phase
                function("supplyCheck", decisions::checkSupply, GameStateTick.class),

                // Probe training — always runs (capped by PROBE_CAP in the service)
                function("probeCheck", decisions::checkProbes, GameStateTick.class),

                // Gas transition — builds assimilator when gateway exists
                function("gasCheck", decisions::checkGas, GameStateTick.class),

                // Expansion — queues 2nd nexus when saturated
                function("expansionCheck", decisions::checkExpansion, GameStateTick.class),

                // Always loop back to listen for next tick
                switchWhenOrElse(
                    (GameStateTick t) -> false,
                    "tick",
                    "tick",
                    GameStateTick.class
                )
            )
            .build();
    }
}
```

- [ ] **Verify compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS. If FuncDSL import issues arise, check the quarkus-flow 0.7.1 API — the static import package may differ from the plan.

- [ ] **Run existing tests**

```bash
mvn test -q
```

Expected: All tests still pass. The flow is not yet started (no `EconomicsLifecycle` yet); it's just registered as a CDI bean.

- [ ] **Commit**

```bash
git add src/main/java/org/acme/starcraft/plugin/flow/EconomicsFlow.java
git commit -m "feat(flow): EconomicsFlow — stateful multi-tick economics workflow"
```

---

## Task 8: EconomicsLifecycle + In-Memory Channel Config

**Files:**
- Create: `src/main/java/org/acme/starcraft/plugin/flow/EconomicsLifecycle.java`
- Modify: `src/main/resources/application.properties`

- [ ] **Create `EconomicsLifecycle.java`**

```java
package org.acme.starcraft.plugin.flow;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import org.acme.starcraft.sc2.GameStarted;
import org.acme.starcraft.sc2.GameStopped;
import org.jboss.logging.Logger;

import java.util.Map;

/**
 * Manages the EconomicsFlow instance lifecycle.
 *
 * <p>Starts a long-lived flow instance when the game begins and cancels it when
 * the game ends. One flow instance per game.
 */
@ApplicationScoped
public class EconomicsLifecycle {

    private static final Logger log = Logger.getLogger(EconomicsLifecycle.class);

    @Inject
    EconomicsFlow flow;

    void onGameStart(@Observes GameStarted event) {
        log.info("[FLOW-ECONOMICS] Starting economics workflow");
        flow.instance(Map.of(
            "phase", "SATURATION",
            "nexusCount", 1
        )).start();
        // fire-and-forget: the flow runs indefinitely, listening for tick events
    }

    void onGameStop(@Observes GameStopped event) {
        log.info("[FLOW-ECONOMICS] Game stopped — flow will idle (no more tick events)");
        // Phase 2: cancel the running workflow instance explicitly via the Flow API
        // For now, the flow will block on listen() until the application shuts down
    }
}
```

- [ ] **Add in-memory channel config to `application.properties`**

Append to the file:

```properties
# SmallRye in-memory channel for economics tick events (FlowEconomicsTask → EconomicsFlow)
mp.messaging.outgoing.economics-ticks.connector=smallrye-in-memory
mp.messaging.incoming.economics-ticks.connector=smallrye-in-memory
# Disable in test profile — flow not started in unit tests
%test.mp.messaging.outgoing.economics-ticks.enabled=false
%test.mp.messaging.incoming.economics-ticks.enabled=false
```

- [ ] **Verify compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Run all tests**

```bash
mvn test -q
```

Expected: All tests pass. The flow is not started in tests (scheduler disabled, no `GameStarted` event fired).

- [ ] **Commit**

```bash
git add src/main/java/org/acme/starcraft/plugin/flow/EconomicsLifecycle.java \
        src/main/resources/application.properties
git commit -m "feat(lifecycle): EconomicsLifecycle starts/stops flow on GameStarted/GameStopped"
```

---

## Task 9: Integration Test — EconomicsFlowTest

**Files:**
- Create: `src/test/java/org/acme/starcraft/plugin/EconomicsFlowTest.java`

This test verifies the full pipeline: emit `GameStarteTick` → flow processes → `IntentQueue` contains intents. It does NOT go through `FlowEconomicsTask.execute()` — it emits ticks directly to test the flow in isolation.

> **Note:** If the `listen` mechanism in Task 7 requires Kafka rather than the in-memory connector, this test may need adjustment. Validate that the in-memory connector delivers events to the flow before writing the full test.

- [ ] **Create `EconomicsFlowTest.java`**

```java
package org.acme.starcraft.plugin;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.acme.starcraft.agent.ResourceBudget;
import org.acme.starcraft.domain.*;
import org.acme.starcraft.plugin.flow.EconomicsDecisionService;
import org.acme.starcraft.plugin.flow.EconomicsFlow;
import org.acme.starcraft.sc2.IntentQueue;
import org.acme.starcraft.sc2.intent.BuildIntent;
import org.acme.starcraft.sc2.intent.TrainIntent;
import org.eclipse.microprofile.reactive.messaging.Channel;
import io.smallrye.reactive.messaging.MutinyEmitter;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import java.util.stream.IntStream;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration test for the full Flow economics pipeline.
 *
 * <p>Requires @QuarkusTest — Quarkus Flow needs CDI context to initialise
 * the workflow engine (same constraint as Drools tests needing @QuarkusTest).
 *
 * <p>Emits GameStateTick events directly to the economics-ticks channel and
 * asserts IntentQueue contents after the flow processes them.
 * Drains IntentQueue in @BeforeEach / @AfterEach to prevent state bleed.
 */
@QuarkusTest
class EconomicsFlowTest {

    @Inject EconomicsFlow flow;
    @Inject IntentQueue intentQueue;

    @Inject
    @Channel("economics-ticks")
    MutinyEmitter<GameStateTick> emitter;

    @BeforeEach
    @AfterEach
    void drainQueue() {
        intentQueue.drainAll();
    }

    @Test
    void flowProcessesSupplyCheckAndQueuesPylon() throws Exception {
        // Start flow instance
        flow.instance(Map.of("phase", "SATURATION", "nexusCount", 1)).start();

        // supply=13/15 → headroom=2, below threshold → pylon queued
        GameStateTick tick = tick(300, 13, 15, workers(6), List.of(nexus()), List.of());
        emitter.sendAndAwait(tick);

        // Allow async processing
        Thread.sleep(200);

        assertThat(intentQueue.pending())
            .anyMatch(i -> i instanceof BuildIntent bi && bi.buildingType() == BuildingType.PYLON);
    }

    @Test
    void flowProcessesProbeCheckAndQueuesProbe() throws Exception {
        flow.instance(Map.of("phase", "SATURATION", "nexusCount", 1)).start();

        // supply headroom=9 (well above threshold), workers=6 < 22 → probe queued
        GameStateTick tick = tick(200, 6, 15, workers(6), List.of(nexus()), List.of());
        emitter.sendAndAwait(tick);
        Thread.sleep(200);

        assertThat(intentQueue.pending())
            .anyMatch(i -> i instanceof TrainIntent ti && ti.unitType() == UnitType.PROBE);
    }

    @Test
    void flowBuildsAssimilatorWhenGatewayExists() throws Exception {
        flow.instance(Map.of("phase", "SATURATION", "nexusCount", 1)).start();

        Resource geyser = new Resource("g-0", new Point2d(20, 20), 2250);
        GameStateTick tick = tick(200, 6, 15, workers(6),
            List.of(nexus(), completeGateway()), List.of(geyser));
        emitter.sendAndAwait(tick);
        Thread.sleep(200);

        assertThat(intentQueue.pending())
            .anyMatch(i -> i instanceof BuildIntent bi && bi.buildingType() == BuildingType.ASSIMILATOR);
    }

    @Test
    void flowSignalsExpansionWhenSaturated() throws Exception {
        flow.instance(Map.of("phase", "SATURATION", "nexusCount", 1)).start();

        GameStateTick tick = tick(500, 22, 50, workers(22), List.of(nexus()), List.of());
        emitter.sendAndAwait(tick);
        Thread.sleep(200);

        assertThat(intentQueue.pending())
            .anyMatch(i -> i instanceof BuildIntent bi && bi.buildingType() == BuildingType.NEXUS);
    }

    // ─── Helpers ─────────────────────────────────────────────────────────────

    private GameStateTick tick(int minerals, int supplyUsed, int supplyCap,
                               List<Unit> workers, List<Building> buildings,
                               List<Resource> geysers) {
        boolean gasReady = buildings.stream().anyMatch(b -> b.type() == BuildingType.GATEWAY);
        return new GameStateTick(minerals, 0, supplyUsed, supplyCap,
            workers, buildings, geysers,
            new ResourceBudget(minerals, 0), "MACRO", gasReady);
    }

    private List<Unit> workers(int count) {
        return IntStream.range(0, count)
            .mapToObj(i -> new Unit("probe-" + i, UnitType.PROBE, new Point2d(9, 9), 45, 45))
            .toList();
    }

    private Building nexus() {
        return new Building("n-0", BuildingType.NEXUS, new Point2d(8, 8), 1500, 1500, true);
    }

    private Building completeGateway() {
        return new Building("gw-0", BuildingType.GATEWAY, new Point2d(17, 18), 550, 550, true);
    }
}
```

- [ ] **Run integration tests**

```bash
mvn test -Dtest=EconomicsFlowTest -q
```

Expected: All four tests pass. If timing is flaky, increase the `Thread.sleep` duration. If the flow's `listen` doesn't receive from the in-memory channel (no intents produced), the channel integration mechanism needs investigation — see the note in Task 7.

- [ ] **Run all tests**

```bash
mvn test -q
```

Expected: All tests pass.

- [ ] **Commit**

```bash
git add src/test/java/org/acme/starcraft/plugin/EconomicsFlowTest.java
git commit -m "test(flow): EconomicsFlowTest — full pipeline integration for economics workflow"
```

---

## Task 10: Full Test Suite Verification

- [ ] **Run all tests**

```bash
mvn test -q
```

Expected: All tests pass. Count should be at least 107 (baseline) + EconomicsDecisionServiceTest count + EconomicsFlowTest count.

- [ ] **Verify mock mode starts cleanly**

```bash
mvn quarkus:dev &
sleep 8
curl -s http://localhost:8080/q/health | grep -o '"status":"[^"]*"'
kill %1
```

Expected: `"status":"UP"` — the Flow extension initialises and the economics channel is wired.

- [ ] **Final commit (if any loose files)**

```bash
git status --short
```

If clean: done. If any staged changes remain, commit them.

---

## Known Limitations and Future Work

| Item | Note |
|---|---|
| Flow instance cancellation | `EconomicsLifecycle.onGameStop()` logs but doesn't cancel the instance — Phase 2 once the quarkus-flow cancellation API is validated |
| Natural expansion position | `NATURAL_EXPANSION_POS = new Point2d(32, 32)` is hardcoded — Phase 3+ uses SC2MapAnalysis sidecar |
| One-tick lag budget | The flow's budget snapshot is independent of CaseHub's shared budget — both plugins may attempt the same spend in adjacent ticks; SC2 rejects unaffordable commands |
| Drools ASSIMILATOR rule | The DRL rule still fires and writes "ASSIMILATOR" to `buildDecisions` — the dispatch branch was removed from `DroolsStrategyTask`, so the decision is silently ignored. The DRL rule can be removed in a future Drools cleanup |
| Multi-base probeTarget | `EconomicsContext.probeTarget()` is correct (22 × nexusCount) but `checkProbes` uses hardcoded `PROBE_CAP=22` — update when multi-base tracking is added |
