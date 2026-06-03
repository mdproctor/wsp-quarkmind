# Adaptive Plugin Selection (Layer 5) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire CaseHub binding conditions so TacticsTask only activates when enemies are visible (NEAREST_THREAT gate) and StrategyTask only activates after Scouting has run (ENEMY_ARMY_SIZE ordering dependency).

**Architecture:** Extract tick execution into a package-private `GameTickExecutor` that returns `AgentOrchestrator.TickResult(CaseFile, TickTimings)`. Tests call `orchestrator.gameTick()` then `orchestrator.getLastTickResult()` to inspect what the platform scheduled. Criteria changes are validated by calling `canActivate(CaseFile)` on the plugin beans directly — this is the default method on `TaskDefinition` that evaluates `entryCriteria()`.

**Tech Stack:** Java 21, Quarkus, casehub-core (`CaseFile`, `TaskDefinition`), `InMemoryCaseFileRepository` (unit tests), `@QuarkusTest` (integration tests), AssertJ.

**Spec:** `docs/superpowers/specs/2026-06-03-adaptive-plugin-selection-design.md`

---

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `src/main/java/io/quarkmind/agent/GameTickExecutor.java` | **Create** | Package-private; runs one tick and returns TickResult |
| `src/main/java/io/quarkmind/agent/AgentOrchestrator.java` | **Modify** | Add TickResult nested record; delegate gameTick() to executor; expose getLastTickResult() |
| `src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java` | **Modify** | entryCriteria → {READY, STRATEGY, NEAREST_THREAT} |
| `src/main/java/io/quarkmind/plugin/BasicTacticsTask.java` | **Modify** | entryCriteria → {READY, STRATEGY, NEAREST_THREAT} (interface parity) |
| `src/main/java/io/quarkmind/plugin/DroolsStrategyTask.java` | **Modify** | entryCriteria → {READY, ENEMY_ARMY_SIZE}; add C2 stub read |
| `src/main/java/io/quarkmind/plugin/BasicStrategyTask.java` | **Modify** | entryCriteria → {READY, ENEMY_ARMY_SIZE}; add C2 stub read (interface parity) |
| `src/test/java/io/quarkmind/plugin/BasicTacticsTaskTest.java` | **Modify** | Add 2 canActivate() criterion tests |
| `src/test/java/io/quarkmind/plugin/BasicStrategyTaskTest.java` | **Modify** | Add 2 canActivate() criterion tests |
| `src/test/java/io/quarkmind/sc2/mock/AdaptivePluginSelectionIT.java` | **Create** | @QuarkusTest: 4 integration tests |

---

## Task 1: Extract GameTickExecutor and expose TickResult

**Files:**
- Create: `src/main/java/io/quarkmind/agent/GameTickExecutor.java`
- Modify: `src/main/java/io/quarkmind/agent/AgentOrchestrator.java`
- Create (skeleton): `src/test/java/io/quarkmind/sc2/mock/AdaptivePluginSelectionIT.java`

- [ ] **Step 1.1: Write the failing test (compile error = red)**

Create `src/test/java/io/quarkmind/sc2/mock/AdaptivePluginSelectionIT.java`:

```java
package io.quarkmind.sc2.mock;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import io.quarkmind.agent.AgentOrchestrator;
import io.quarkmind.agent.QuarkMindCaseFile;
import io.quarkmind.sc2.IntentQueue;
import io.quarkmind.sc2.ScenarioRunner;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class AdaptivePluginSelectionIT {

    @Inject AgentOrchestrator orchestrator;
    @Inject SimulatedGame     simulatedGame;
    @Inject IntentQueue       intentQueue;
    @Inject ScenarioRunner    scenarioRunner;

    @BeforeEach
    void setUp() {
        simulatedGame.reset();
        orchestrator.startGame();
        intentQueue.drainAll();
    }

    @Test
    void tickResultReturnsValidCaseFile() {
        orchestrator.gameTick();
        AgentOrchestrator.TickResult result = orchestrator.getLastTickResult(); // compile error

        assertThat(result).isNotNull();
        assertThat(result.solveSucceeded()).isTrue();
        assertThat(result.caseFile().contains(QuarkMindCaseFile.READY)).isTrue();
    }
}
```

- [ ] **Step 1.2: Confirm compile failure**

```bash
mvn compile -q
```

Expected: `error: cannot find symbol … getLastTickResult()`

- [ ] **Step 1.3: Create `GameTickExecutor.java`**

Create `src/main/java/io/quarkmind/agent/GameTickExecutor.java`:

```java
package io.quarkmind.agent;

import io.casehub.coordination.CaseEngine;
import io.casehub.core.CaseFile;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import io.quarkmind.sc2.SC2Engine;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.util.Map;

@ApplicationScoped
class GameTickExecutor {

    private static final Logger log = Logger.getLogger(GameTickExecutor.class);

    @Inject SC2Engine          engine;
    @Inject GameStateTranslator translator;
    @Inject CaseEngine          caseEngine;

    AgentOrchestrator.TickResult execute() {
        long t0 = System.currentTimeMillis();
        engine.tick();
        var gameState = engine.observe();
        long t1 = System.currentTimeMillis();

        Map<String, Object> caseData = translator.toMap(gameState);
        CaseFile caseFile = null;
        try {
            caseFile = caseEngine.createAndSolve("starcraft-game", caseData, Duration.ofSeconds(5));
        } catch (Exception e) {
            log.errorf("CaseEngine decision cycle failed at frame %d: %s",
                       gameState.gameFrame(), e.getMessage());
        }
        long t2 = System.currentTimeMillis();

        engine.dispatch();
        long t3 = System.currentTimeMillis();

        var timings = new AgentOrchestrator.TickTimings(t1 - t0, t2 - t1, t3 - t2);
        log.debugf("Tick %d — physics=%dms plugins=%dms dispatch=%dms total=%dms | minerals=%d supply=%d/%d",
            gameState.gameFrame(), timings.physicsMs(), timings.pluginsMs(),
            timings.dispatchMs(), timings.totalMs(),
            gameState.minerals(), gameState.supplyUsed(), gameState.supply());
        return new AgentOrchestrator.TickResult(caseFile, timings);
    }
}
```

- [ ] **Step 1.4: Update `AgentOrchestrator.java`**

Apply these changes to `AgentOrchestrator.java`:

**a) Add `TickResult` public nested record** (insert after the existing `TickTimings` record at line 33):

```java
public record TickResult(CaseFile caseFile, AgentOrchestrator.TickTimings timings) {
    public boolean solveSucceeded() { return caseFile != null; }
}
```

**b) Add `CaseFile` import** at the top of the file:

```java
import io.casehub.core.CaseFile;
```

**c) Replace the three injected fields** (`SC2Engine engine`, `GameStateTranslator translator`, `CaseEngine caseEngine`) with two fields:

```java
@Inject SC2Engine          engine;       // keep — needed for startGame/stopGame
@Inject GameTickExecutor   tickExecutor; // new
```

Remove `@Inject GameStateTranslator translator;` and `@Inject CaseEngine caseEngine;`

**d) Replace `AtomicReference<TickTimings> lastTickTimings`** with:

```java
private final AtomicReference<TickResult> lastTickResult = new AtomicReference<>();
```

**e) Replace `gameTick()` body** with delegation:

```java
@Scheduled(every = "${starcraft.tick.interval:500ms}", concurrentExecution = Scheduled.ConcurrentExecution.SKIP)
public void gameTick() {
    if (schedulerPaused) return;
    if (!engine.isConnected()) return;
    lastTickResult.set(tickExecutor.execute());
}
```

**f) Replace `getLastTickTimings()`** and add `getLastTickResult()`:

```java
public TickResult getLastTickResult() { return lastTickResult.get(); }

/** Backward compatibility for GameLoopBenchmarkTest. */
public TickTimings getLastTickTimings() {
    TickResult r = lastTickResult.get();
    return r != null ? r.timings() : null;
}
```

Remove unused imports: `java.time.Duration`, `java.util.Map` (they moved to `GameTickExecutor`). Keep `java.util.concurrent.atomic.AtomicReference`.

- [ ] **Step 1.5: Compile**

```bash
mvn compile -q
```

Expected: clean compile.

- [ ] **Step 1.6: Run the new integration test**

```bash
mvn test -Dtest=AdaptivePluginSelectionIT -q
```

Expected: `tickResultReturnsValidCaseFile` PASSES. All existing tests still pass.

- [ ] **Step 1.7: Run the full suite to confirm no regressions**

```bash
mvn test -q
```

Expected: all tests pass (including `GameLoopBenchmarkTest` which calls `getLastTickTimings()`).

- [ ] **Step 1.8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/agent/GameTickExecutor.java \
  src/main/java/io/quarkmind/agent/AgentOrchestrator.java \
  src/test/java/io/quarkmind/sc2/mock/AdaptivePluginSelectionIT.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "refactor(#157): extract GameTickExecutor; expose TickResult from AgentOrchestrator"
```

---

## Task 2: TacticsTask binding criteria — unit tests then implementation

**Files:**
- Modify: `src/test/java/io/quarkmind/plugin/BasicTacticsTaskTest.java`
- Modify: `src/main/java/io/quarkmind/plugin/BasicTacticsTask.java`
- Modify: `src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java`

- [ ] **Step 2.1: Write failing tests in `BasicTacticsTaskTest.java`**

Add after the last `@Test` method (before the `// --- Helpers ---` section):

```java
// --- Entry criteria ---

@Test
void tacticsTaskRequiresNearestThreatToActivate() {
    var cf = new InMemoryCaseFileRepository()
        .create("starcraft-game", Map.of(), PropagationContext.createRoot());
    cf.put(QuarkMindCaseFile.READY, Boolean.TRUE);
    cf.put(QuarkMindCaseFile.STRATEGY, "DEFEND");
    // NEAREST_THREAT absent → canActivate must return false
    assertThat(task.canActivate(cf)).isFalse();
}

@Test
void tacticsTaskActivatesWhenAllCriteriaPresent() {
    var cf = new InMemoryCaseFileRepository()
        .create("starcraft-game", Map.of(), PropagationContext.createRoot());
    cf.put(QuarkMindCaseFile.READY, Boolean.TRUE);
    cf.put(QuarkMindCaseFile.STRATEGY, "DEFEND");
    cf.put(QuarkMindCaseFile.NEAREST_THREAT, new Point2d(50, 50));
    assertThat(task.canActivate(cf)).isTrue();
}
```

`InMemoryCaseFileRepository` and `PropagationContext` are already imported in this file.

- [ ] **Step 2.2: Run to confirm first test fails**

```bash
mvn test -Dtest=BasicTacticsTaskTest#tacticsTaskRequiresNearestThreatToActivate -q
```

Expected: FAIL — `expected: false but was: true` (current criteria is `{READY}`; READY is present so `canActivate()` returns true).

- [ ] **Step 2.3: Update `BasicTacticsTask.entryCriteria()`**

In `BasicTacticsTask.java`, replace line 49:

```java
// before:
@Override public Set<String> entryCriteria() { return Set.of(QuarkMindCaseFile.READY); }

// after:
@Override public Set<String> entryCriteria() {
    return Set.of(QuarkMindCaseFile.READY,
                  QuarkMindCaseFile.STRATEGY,
                  QuarkMindCaseFile.NEAREST_THREAT);
}
```

- [ ] **Step 2.4: Update `DroolsTacticsTask.entryCriteria()`**

In `DroolsTacticsTask.java`, replace line 105:

```java
// before:
@Override public Set<String> entryCriteria() { return Set.of(QuarkMindCaseFile.READY); }

// after:
@Override public Set<String> entryCriteria() {
    return Set.of(QuarkMindCaseFile.READY,
                  QuarkMindCaseFile.STRATEGY,
                  QuarkMindCaseFile.NEAREST_THREAT);
}
```

- [ ] **Step 2.5: Run tactics unit tests**

```bash
mvn test -Dtest=BasicTacticsTaskTest -q
```

Expected: all tests PASS, including both new criterion tests.

- [ ] **Step 2.6: Run full suite**

```bash
mvn test -q
```

Expected: all tests pass.

- [ ] **Step 2.7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/test/java/io/quarkmind/plugin/BasicTacticsTaskTest.java \
  src/main/java/io/quarkmind/plugin/BasicTacticsTask.java \
  src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#157): TacticsTask entryCriteria requires STRATEGY + NEAREST_THREAT"
```

---

## Task 3: StrategyTask ordering dependency — unit tests then implementation

**Files:**
- Modify: `src/test/java/io/quarkmind/plugin/BasicStrategyTaskTest.java`
- Modify: `src/main/java/io/quarkmind/plugin/BasicStrategyTask.java`
- Modify: `src/main/java/io/quarkmind/plugin/DroolsStrategyTask.java`

- [ ] **Step 3.1: Write failing tests in `BasicStrategyTaskTest.java`**

Add after the last `@Test` method (before the `// --- Helpers ---` section):

```java
// --- Entry criteria ---

@Test
void strategyTaskRequiresEnemyArmySizeToActivate() {
    var cf = new InMemoryCaseFileRepository()
        .create("starcraft-game", Map.of(), PropagationContext.createRoot());
    cf.put(QuarkMindCaseFile.READY, Boolean.TRUE);
    // ENEMY_ARMY_SIZE absent → canActivate must return false
    assertThat(task.canActivate(cf)).isFalse();
}

@Test
void strategyTaskActivatesWhenEnemyArmySizePresent() {
    var cf = new InMemoryCaseFileRepository()
        .create("starcraft-game", Map.of(), PropagationContext.createRoot());
    cf.put(QuarkMindCaseFile.READY, Boolean.TRUE);
    cf.put(QuarkMindCaseFile.ENEMY_ARMY_SIZE, 0); // scouting wrote 0 (no enemies visible)
    assertThat(task.canActivate(cf)).isTrue();
}
```

`InMemoryCaseFileRepository` and `PropagationContext` are already imported in this file.

- [ ] **Step 3.2: Run to confirm first test fails**

```bash
mvn test -Dtest=BasicStrategyTaskTest#strategyTaskRequiresEnemyArmySizeToActivate -q
```

Expected: FAIL — `expected: false but was: true`

- [ ] **Step 3.3: Update `BasicStrategyTask.entryCriteria()` and add stub read**

In `BasicStrategyTask.java`:

a) Replace line 67:

```java
// before:
@Override public Set<String> entryCriteria() { return Set.of(QuarkMindCaseFile.READY); }

// after:
@Override public Set<String> entryCriteria() {
    return Set.of(QuarkMindCaseFile.READY, QuarkMindCaseFile.ENEMY_ARMY_SIZE);
}
```

b) Add stub read as the first line inside `execute()`, before the existing `workers` read (after line 72 `public void execute(CaseFile caseFile) {`):

```java
// C2 stub (tracked as #169): ENEMY_ARMY_SIZE establishes scouting → strategy ordering.
// When C2 lands: replace enemies DataStore feed with ENEMY_POSTURE + ENEMY_BUILD_ORDER.
int enemyCount = caseFile.get(QuarkMindCaseFile.ENEMY_ARMY_SIZE, Integer.class).orElse(0);
```

c) Update the debug log (line 89) to include `enemyCount`:

```java
log.debugf("[STRATEGY] %s | stalkers=%d | enemies(raw)=%d | enemies(scouted)=%d | %s",
    strategy,
    army.stream().filter(u -> u.type() == UnitType.STALKER).count(),
    enemies.size(), enemyCount, budget);
```

- [ ] **Step 3.4: Update `DroolsStrategyTask.entryCriteria()` and add stub read**

In `DroolsStrategyTask.java`:

a) Replace line 56:

```java
// before:
@Override public Set<String> entryCriteria() { return Set.of(QuarkMindCaseFile.READY); }

// after:
@Override public Set<String> entryCriteria() {
    return Set.of(QuarkMindCaseFile.READY, QuarkMindCaseFile.ENEMY_ARMY_SIZE);
}
```

b) Add stub read as the first line inside `execute()`, before the existing `workers` read (after line 60 `public void execute(CaseFile caseFile) {`):

```java
// C2 stub (tracked as #169): ENEMY_ARMY_SIZE establishes scouting → strategy ordering.
// When C2 lands: replace enemies DataStore feed with ENEMY_POSTURE + ENEMY_BUILD_ORDER.
int enemyCount = caseFile.get(QuarkMindCaseFile.ENEMY_ARMY_SIZE, Integer.class).orElse(0);
```

c) Update the debug log (line 83) to include `enemyCount`:

```java
log.debugf("[DROOLS-STRATEGY] %s | stalkers=%d | enemies(scouted)=%d | builds=%s | %s",
    strategy,
    army.stream().filter(u -> u.type() == UnitType.STALKER).count(),
    enemyCount, data.getBuildDecisions(), budget);
```

- [ ] **Step 3.5: Run strategy unit tests**

```bash
mvn test -Dtest=BasicStrategyTaskTest -q
```

Expected: all tests PASS, including both new criterion tests.

- [ ] **Step 3.6: Run full suite**

```bash
mvn test -q
```

Expected: all tests pass.

- [ ] **Step 3.7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/test/java/io/quarkmind/plugin/BasicStrategyTaskTest.java \
  src/main/java/io/quarkmind/plugin/BasicStrategyTask.java \
  src/main/java/io/quarkmind/plugin/DroolsStrategyTask.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#157): StrategyTask entryCriteria requires ENEMY_ARMY_SIZE; C2 stub read added"
```

---

## Task 4: Complete integration tests

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/mock/AdaptivePluginSelectionIT.java`

- [ ] **Step 4.1: Add remaining tests to `AdaptivePluginSelectionIT.java`**

Add imports at the top of the file:

```java
import io.casehub.annotation.CaseType;
import io.casehub.coordination.PropagationContext;
import io.casehub.persistence.memory.InMemoryCaseFileRepository;
import io.quarkmind.agent.plugin.StrategyTask;
import io.quarkmind.agent.plugin.TacticsTask;
import io.quarkmind.sc2.intent.AttackIntent;
import io.quarkmind.sc2.intent.BlinkIntent;
import java.util.Map;
```

Add injections after the existing `@Inject ScenarioRunner scenarioRunner;`:

```java
@Inject @CaseType("starcraft-game") TacticsTask tacticsTask;
@Inject @CaseType("starcraft-game") StrategyTask strategyTask;
```

Add three new tests after `tickResultReturnsValidCaseFile`:

```java
@Test
void tacticsSkippedWhenNoEnemiesVisible() {
    // Default reset state: no enemies → scouting never writes NEAREST_THREAT
    orchestrator.gameTick();
    AgentOrchestrator.TickResult result = orchestrator.getLastTickResult();

    assertThat(result.solveSucceeded()).isTrue();
    // Gate key absent: scouting didn't write NEAREST_THREAT (no enemies)
    assertThat(result.caseFile().contains(QuarkMindCaseFile.NEAREST_THREAT)).isFalse();
    // Platform did not activate tactics (canActivate returned false)
    assertThat(tacticsTask.canActivate(result.caseFile())).isFalse();
    // No tactical intents dispatched
    assertThat(intentQueue.drainAll())
        .noneMatch(i -> i instanceof AttackIntent || i instanceof BlinkIntent);
}

@Test
void scoutingRunsBeforeStrategyInOrderedChain() {
    // After a tick: scouting wrote ENEMY_ARMY_SIZE (ordering dep), strategy ran and wrote STRATEGY
    orchestrator.gameTick();
    AgentOrchestrator.TickResult result = orchestrator.getLastTickResult();

    assertThat(result.solveSucceeded()).isTrue();
    assertThat(result.caseFile().contains(QuarkMindCaseFile.ENEMY_ARMY_SIZE)).isTrue();
    assertThat(result.caseFile().contains(QuarkMindCaseFile.STRATEGY)).isTrue();

    // Cross-check: strategy cannot activate without ENEMY_ARMY_SIZE
    var readyOnly = new InMemoryCaseFileRepository()
        .create("starcraft-game", Map.of(), PropagationContext.createRoot());
    readyOnly.put(QuarkMindCaseFile.READY, Boolean.TRUE);
    assertThat(strategyTask.canActivate(readyOnly)).isFalse();
}

@Test
void tacticsGateMetWhenEnemyPresent() {
    // Enemies visible → scouting writes NEAREST_THREAT → tactics gate is met
    scenarioRunner.run("spawn-enemy-attack");
    orchestrator.gameTick();
    AgentOrchestrator.TickResult result = orchestrator.getLastTickResult();

    assertThat(result.solveSucceeded()).isTrue();
    assertThat(result.caseFile().contains(QuarkMindCaseFile.NEAREST_THREAT)).isTrue();
    assertThat(result.caseFile().contains(QuarkMindCaseFile.STRATEGY)).isTrue();
    // Platform activated tactics
    assertThat(tacticsTask.canActivate(result.caseFile())).isTrue();
}
```

- [ ] **Step 4.2: Run integration tests**

```bash
mvn test -Dtest=AdaptivePluginSelectionIT -q
```

Expected: all 4 tests PASS.

- [ ] **Step 4.3: Run full suite**

```bash
mvn test -q
```

Expected: all tests pass with no regressions.

- [ ] **Step 4.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/test/java/io/quarkmind/sc2/mock/AdaptivePluginSelectionIT.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "test(#157): AdaptivePluginSelectionIT — tactics gate and ordering chain verified"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|-----------------|------|
| `GameTickExecutor` (new, package-private) | Task 1 |
| `AgentOrchestrator.TickResult` public nested record | Task 1 |
| `AgentOrchestrator.getLastTickResult()` | Task 1 |
| `getLastTickTimings()` backward compatibility | Task 1 |
| `DroolsTacticsTask.entryCriteria()` → `{READY, STRATEGY, NEAREST_THREAT}` | Task 2 |
| `BasicTacticsTask.entryCriteria()` → `{READY, STRATEGY, NEAREST_THREAT}` | Task 2 |
| `DroolsStrategyTask.entryCriteria()` → `{READY, ENEMY_ARMY_SIZE}` + C2 stub | Task 3 |
| `BasicStrategyTask.entryCriteria()` → `{READY, ENEMY_ARMY_SIZE}` + C2 stub | Task 3 |
| `AdaptivePluginSelectionIT` — 4 tests | Tasks 1 + 4 |
| `FullMockPipelineIT` unchanged | ✅ gameTick() still void, call sites unchanged |
| `GameLoopBenchmarkTest` unchanged | ✅ getLastTickTimings() forward preserved |
| C2 comment references #169 | Task 3 |

**Placeholder scan:** No TBD, TODO, or vague steps. All code is complete.

**Type consistency:**
- `AgentOrchestrator.TickResult` — used in Task 1 (declaration), Task 1 (test), Task 4 (test). Consistent.
- `AgentOrchestrator.TickTimings` — declared in AgentOrchestrator (Task 1 changes keep it there). `GameTickExecutor` references `AgentOrchestrator.TickTimings` directly. Consistent.
- `canActivate(CaseFile)` — called on `TacticsTask` and `StrategyTask` in Tasks 2, 3, 4. Default method inherited from `TaskDefinition`. Consistent.
- `intentQueue.drainAll()` — integration tests. `intentQueue.pending()` — unit tests. Both exist on `IntentQueue`. Consistent.
- `InMemoryCaseFileRepository().create("starcraft-game", Map.of(), PropagationContext.createRoot())` — matches existing pattern in `BasicStrategyTaskTest` and `BasicTacticsTaskTest`. Consistent.
