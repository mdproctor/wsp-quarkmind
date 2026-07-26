# InMemoryCaseFile Null Fix (#175) + casehub-ledger Integration (#156)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix `InMemoryCaseFile` null handling in casehub-poc and add casehub-ledger audit recording for plugin decisions in quarkmind.

**Architecture:** Two sequential issues on one branch. #175 is a one-file bug fix in casehub-poc with a 4-site null-invariant fix and a rebuild. #156 adds 5 new classes in `io.quarkmind.agent`, wires transition-detection event firing into 4 plugins/services, and an integration test in `@QuarkusTest`.

**Tech Stack:** Java 21, Quarkus 3.34, casehub-ledger 0.2-SNAPSHOT, CDI `@ObservesAsync`, `Event<T>`, `Instance<T>`, quarkmind mock profile

---

## Part 1: #175 — InMemoryCaseFile null fix (casehub-poc)

### Task 1: Write failing tests and fix `InMemoryCaseFile`

**Files:**
- Modify: `/Users/mdproctor/claude/casehub-poc/casehub-persistence-memory/src/main/java/io/casehub/persistence/memory/InMemoryCaseFile.java` (lines 52-60, 76-81, 105-109, 181)
- Create: `/Users/mdproctor/claude/casehub-poc/casehub-persistence-memory/src/test/java/io/casehub/persistence/memory/InMemoryCaseFileNullTest.java`

- [ ] **Step 1.1: Write the failing test class**

```java
// InMemoryCaseFileNullTest.java
package io.casehub.persistence.memory;

import org.junit.jupiter.api.Test;

import java.util.HashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class InMemoryCaseFileNullTest {

    private InMemoryCaseFile make() {
        return new InMemoryCaseFileRepository().create("test", Map.of(), null);
    }

    @Test
    void put_null_value_is_noop() {
        var cf = make();
        cf.put("key", null);
        assertThat(cf.contains("key")).isFalse();
        assertThat(cf.get("key", String.class)).isEmpty();
    }

    @Test
    void put_value_then_put_null_leaves_original() {
        var cf = make();
        cf.put("key", "hello");
        cf.put("key", null);
        assertThat(cf.contains("key")).isTrue();
        assertThat(cf.get("key", String.class)).hasValue("hello");
    }

    @Test
    void put_then_put_different_value_updates() {
        var cf = make();
        cf.put("key", "first");
        cf.put("key", "second");
        assertThat(cf.get("key", String.class)).hasValue("second");
    }

    @Test
    void constructor_with_null_value_in_initial_state_filters_entry() {
        var initial = new HashMap<String, Object>();
        initial.put("present", "value");
        initial.put("absent", null);
        var cf = new InMemoryCaseFileRepository().create("test", initial, null);
        assertThat(cf.contains("present")).isTrue();
        assertThat(cf.contains("absent")).isFalse();
        assertThat(cf.get("absent", String.class)).isEmpty();
    }

    @Test
    void fireEvent_first_write_delivers_empty_previous_to_listener() {
        var cf = make();
        var box = new Object[]{null};
        cf.onChange("k", evt -> box[0] = evt.previousValue());
        cf.put("k", "v");
        assertThat(box[0]).isInstanceOf(java.util.Optional.class);
        assertThat(((java.util.Optional<?>) box[0])).isEmpty();
    }
}
```

- [ ] **Step 1.2: Run tests — expect failures**

```bash
mvn test -Dtest=InMemoryCaseFileNullTest -pl casehub-persistence-memory -q -f /Users/mdproctor/claude/casehub-poc/pom.xml
```

Expected: failures (put_null_value_is_noop NPEs; constructor test fails if null bypasses)

- [ ] **Step 1.3: Fix `put()` — add null guard at the top**

In `InMemoryCaseFile.java`, method `put(String key, Object value)` (around line 105):

```java
@Override public void put(String key, Object value) {
    if (value == null) return;                          // ADD THIS LINE
    AtomicReference<ItemEntry> ref = store.computeIfAbsent(key, k -> new AtomicReference<>());
    ItemEntry previous = ref.getAndSet(new ItemEntry(value, writeVersion.incrementAndGet()));
    fireEvent(key, value, previous);
}
```

- [ ] **Step 1.4: Fix the constructor — filter null values from `initialState`**

Constructor body (around line 56):

```java
InMemoryCaseFile(String caseType, Map<String, Object> initialState,
                 PropagationContext propagationContext) {
    this.caseType = caseType;
    this.propagationContext = propagationContext;
    if (initialState != null) {
        initialState.forEach((k, v) -> {
            if (v != null) {                            // ADD NULL CHECK
                store.put(k, new AtomicReference<>(new ItemEntry(v, writeVersion.incrementAndGet())));
            }
        });
    }
}
```

- [ ] **Step 1.5: Run tests — expect all pass**

```bash
mvn test -Dtest=InMemoryCaseFileNullTest -pl casehub-persistence-memory -q -f /Users/mdproctor/claude/casehub-poc/pom.xml
```

Expected: 5 tests pass

- [ ] **Step 1.6: Run the full persistence-memory test suite to check for regressions**

```bash
mvn test -pl casehub-persistence-memory -q -f /Users/mdproctor/claude/casehub-poc/pom.xml
```

Expected: all tests pass

- [ ] **Step 1.7: Commit in casehub-poc**

```bash
git -C /Users/mdproctor/claude/casehub-poc add casehub-persistence-memory/src/
git -C /Users/mdproctor/claude/casehub-poc commit -m "fix(#175): InMemoryCaseFile null guard in put() and constructor initialState"
```

---

### Task 2: Install the fixed snapshot

- [ ] **Step 2.1: Install casehub-poc to local Maven repo**

```bash
mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub-poc/pom.xml
```

Expected: `BUILD SUCCESS`

- [ ] **Step 2.2: Verify quarkmind still builds**

```bash
mvn compile -q -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml
```

Expected: `BUILD SUCCESS`

---

## Part 2: #156 — casehub-ledger integration (quarkmind)

### Task 3: Add dependencies and configuration

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/quarkmind/pom.xml`
- Modify: `/Users/mdproctor/claude/casehub/quarkmind/src/main/resources/application.properties`

- [ ] **Step 3.1: Add three ledger dependencies to `pom.xml`**

Locate the existing casehub dependencies block (after `casehub-persistence-memory`). Add:

```xml
<!-- casehub-ledger: OutcomeRecorder SPI + DefaultOutcomeRecorder + in-memory SPIs -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ledger-api</artifactId>
    <version>0.2-SNAPSHOT</version>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ledger</artifactId>
    <version>0.2-SNAPSHOT</version>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ledger-memory</artifactId>
    <version>0.2-SNAPSHOT</version>
</dependency>
```

Artifacts are already at `0.2-SNAPSHOT` in the local Maven repo. Reinstall from
`/Users/mdproctor/claude/casehub/ledger/` via `mvn install -DskipTests` if stale.

- [ ] **Step 3.2: Add global and per-profile config to `application.properties`**

Append to the end of `src/main/resources/application.properties`:

```properties
# --- casehub-ledger (Layer 4) ---
# DefaultOutcomeRecorder throws ISE if this is absent and OutcomeRecord.attestorId is null
casehub.ledger.outcome.default-attestor-id=quarkmind:game-engine@v1

# %test profile: in-memory ledger, no JPA datasource needed
%test.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.memory.InMemoryLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryLedgerMerkleFrontierRepository,\
  io.casehub.ledger.memory.InMemoryActorTrustScoreRepository,\
  io.casehub.ledger.memory.InMemoryKeyRotationRepository,\
  io.casehub.ledger.memory.InMemoryAgentSigner,\
  io.casehub.ledger.memory.InMemoryActorIdentityBindingRepository,\
  io.casehub.ledger.memory.InMemoryReactiveLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryReactiveKeyRotationRepository
%test.quarkus.index-dependency.casehub-ledger-memory.group-id=io.casehub
%test.quarkus.index-dependency.casehub-ledger-memory.artifact-id=casehub-ledger-memory
%test.casehub.ledger.hash-chain.enabled=false

# %mock profile: same 8 alternatives (default dev profile, no SC2 needed)
%mock.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.memory.InMemoryLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryLedgerMerkleFrontierRepository,\
  io.casehub.ledger.memory.InMemoryActorTrustScoreRepository,\
  io.casehub.ledger.memory.InMemoryKeyRotationRepository,\
  io.casehub.ledger.memory.InMemoryAgentSigner,\
  io.casehub.ledger.memory.InMemoryActorIdentityBindingRepository,\
  io.casehub.ledger.memory.InMemoryReactiveLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryReactiveKeyRotationRepository
%mock.quarkus.index-dependency.casehub-ledger-memory.group-id=io.casehub
%mock.quarkus.index-dependency.casehub-ledger-memory.artifact-id=casehub-ledger-memory
%mock.casehub.ledger.hash-chain.enabled=false

# %emulated profile: physics simulation, no SC2 needed
%emulated.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.memory.InMemoryLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryLedgerMerkleFrontierRepository,\
  io.casehub.ledger.memory.InMemoryActorTrustScoreRepository,\
  io.casehub.ledger.memory.InMemoryKeyRotationRepository,\
  io.casehub.ledger.memory.InMemoryAgentSigner,\
  io.casehub.ledger.memory.InMemoryActorIdentityBindingRepository,\
  io.casehub.ledger.memory.InMemoryReactiveLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryReactiveKeyRotationRepository
%emulated.quarkus.index-dependency.casehub-ledger-memory.group-id=io.casehub
%emulated.quarkus.index-dependency.casehub-ledger-memory.artifact-id=casehub-ledger-memory
%emulated.casehub.ledger.hash-chain.enabled=false

# %replay profile: replay mode, no SC2 datasource
%replay.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.memory.InMemoryLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryLedgerMerkleFrontierRepository,\
  io.casehub.ledger.memory.InMemoryActorTrustScoreRepository,\
  io.casehub.ledger.memory.InMemoryKeyRotationRepository,\
  io.casehub.ledger.memory.InMemoryAgentSigner,\
  io.casehub.ledger.memory.InMemoryActorIdentityBindingRepository,\
  io.casehub.ledger.memory.InMemoryReactiveLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryReactiveKeyRotationRepository
%replay.quarkus.index-dependency.casehub-ledger-memory.group-id=io.casehub
%replay.quarkus.index-dependency.casehub-ledger-memory.artifact-id=casehub-ledger-memory
%replay.casehub.ledger.hash-chain.enabled=false
```

Note on reactive alternatives: `casehub.ledger.reactive.enabled` defaults to `false` at build
time — the reactive beans are not augmented and never resolved. The reactive memory alternatives
are listed defensively for future profiles that enable reactive.

- [ ] **Step 3.3: Verify compile**

```bash
mvn compile -q -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml
```

Expected: `BUILD SUCCESS`

---

### Task 4: New agent types — `GameSession`, `PluginDecisionEvent`, `QuarkMindCapabilityTag`

**Files:**
- Create: `src/main/java/io/quarkmind/agent/GameSession.java`
- Create: `src/main/java/io/quarkmind/agent/PluginDecisionEvent.java`
- Create: `src/main/java/io/quarkmind/agent/QuarkMindCapabilityTag.java`

No tests needed — plain records and a CDI session bean; tested indirectly through plugin integration tests.

- [ ] **Step 4.1: Create `GameSession.java`**

```java
package io.quarkmind.agent;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.UUID;

/**
 * Holds the current game session UUID for ledger subjectId scoping.
 * Reset in AgentOrchestrator.startGame() only — NOT in stopGame() (async
 * CDI observers dispatched during final ticks must attribute to the correct session).
 */
@ApplicationScoped
public class GameSession {
    private volatile UUID id = UUID.randomUUID();

    public UUID id() { return id; }

    // public — required for CDI proxy access in @QuarkusTest; package-private is unreachable
    // through @ApplicationScoped proxies even from the same package.
    public void reset() { id = UUID.randomUUID(); }
}
```

- [ ] **Step 4.2: Create `PluginDecisionEvent.java`**

```java
package io.quarkmind.agent;

import io.casehub.ledger.api.model.AttestationVerdict;
import java.util.UUID;

/**
 * CDI event fired by plugins on state transitions.
 * Consumed by PluginOutcomeAuditor (@ObservesAsync) to write a ledger OutcomeRecord.
 * Only fired on transitions — not every tick.
 */
public record PluginDecisionEvent(
        String actorId,             // plugin.getId()
        String capabilityTag,       // QuarkMindCapabilityTag constant
        AttestationVerdict verdict, // SOUND (normal) or FLAGGED (plugin error)
        UUID gameSessionId,
        int gameFrame               // logged by observer for tracing; 0 when unavailable
) {}
```

- [ ] **Step 4.3: Create `QuarkMindCapabilityTag.java`**

```java
package io.quarkmind.agent;

public final class QuarkMindCapabilityTag {
    public static final String STRATEGY  = "starcraft.strategy";
    public static final String ECONOMICS = "starcraft.economics";
    public static final String TACTICS   = "starcraft.tactics";
    public static final String SCOUTING  = "starcraft.scouting";

    private QuarkMindCapabilityTag() {}
}
```

- [ ] **Step 4.4: Verify compile**

```bash
mvn compile -q -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml
```

Expected: `BUILD SUCCESS`

---

### Task 5: `PluginOutcomeAuditor` and `LedgerLifecycleAdapter`

**Files:**
- Create: `src/main/java/io/quarkmind/agent/PluginOutcomeAuditor.java`
- Create: `src/main/java/io/quarkmind/agent/LedgerLifecycleAdapter.java`

- [ ] **Step 5.1: Create `PluginOutcomeAuditor.java`**

```java
package io.quarkmind.agent;

import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.OutcomeRecorder;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

/**
 * Async CDI observer that writes a LedgerEntry for every plugin decision transition.
 * Lives in agent/ (cross-cutting infrastructure, not a game plugin).
 *
 * OutcomeRecorder @DefaultBean is from casehub-ledger runtime; delegates to
 * InMemoryLedgerEntryRepository in mock/emulated/test/replay profiles.
 */
@ApplicationScoped
public class PluginOutcomeAuditor {

    private static final Logger log = Logger.getLogger(PluginOutcomeAuditor.class);

    @Inject
    OutcomeRecorder outcomeRecorder;

    public void onDecision(@ObservesAsync PluginDecisionEvent e) {
        outcomeRecorder.record(OutcomeRecord.of(
                e.actorId(),
                e.gameSessionId(),
                e.capabilityTag(),
                e.verdict(),
                0.7   // game-level decision scope per OutcomeRecord Javadoc
        ));
        log.debugf("Ledger: actor=%s capability=%s verdict=%s frame=%d",
                e.actorId(), e.capabilityTag(), e.verdict(), e.gameFrame());
    }
}
```

- [ ] **Step 5.2: Create `LedgerLifecycleAdapter.java`**

```java
package io.quarkmind.agent;

import io.casehub.ledger.memory.InMemoryLedgerEntryRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;
import io.quarkmind.sc2.GameStopped;

/**
 * Bridges GameStopped to InMemoryLedgerEntryRepository.clear().
 *
 * clear() is not on the LedgerEntryRepository SPI, so the concrete type is
 * injected via @Any Instance<> — isUnsatisfied() guards the call so this is
 * a no-op when the JPA implementation is active (%sc2 profile).
 *
 * Follows the EconomicsLifecycle pattern (plugin/flow/EconomicsLifecycle.java).
 */
@ApplicationScoped
public class LedgerLifecycleAdapter {

    private static final Logger log = Logger.getLogger(LedgerLifecycleAdapter.class);

    @Inject @Any
    Instance<InMemoryLedgerEntryRepository> memoryLedger;

    void onGameStop(@Observes GameStopped event) {
        if (!memoryLedger.isUnsatisfied()) {
            memoryLedger.get().clear();
            log.debug("In-memory ledger cleared at game stop");
        }
    }
}
```

- [ ] **Step 5.3: Verify compile**

```bash
mvn compile -q -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml
```

Expected: `BUILD SUCCESS`

---

### Task 6: Wire `AgentOrchestrator.startGame()` → `GameSession.reset()`

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/AgentOrchestrator.java`

- [ ] **Step 6.1: Inject `GameSession` and call `reset()` in `startGame()`**

Add field injection (after existing `@Inject Event<GameStopped> gameStoppedEvent;`):

```java
@Inject GameSession gameSession;
```

In `startGame()`, add `gameSession.reset()` as the first statement:

```java
public void startGame() {
    gameSession.reset();            // ADD: new game session UUID before any events fire
    engine.connect();
    engine.joinGame();
    gameStartedEvent.fire(new GameStarted());
    log.info("Game started");
}
```

- [ ] **Step 6.2: Verify compile**

```bash
mvn compile -q -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml
```

Expected: `BUILD SUCCESS`

---

### Task 7: Wire `DroolsStrategyTask` — strategy transition detection

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/DroolsStrategyTask.java`

- [ ] **Step 7.1: Add fields and imports to `DroolsStrategyTask`**

Add imports (after existing imports):
```java
import io.casehub.ledger.api.model.AttestationVerdict;
import io.quarkmind.agent.GameSession;
import io.quarkmind.agent.PluginDecisionEvent;
import io.quarkmind.agent.QuarkMindCapabilityTag;
import jakarta.enterprise.event.Event;
import java.util.Objects;
```

Add fields (after existing `private final IntentQueue intentQueue;`):
```java
@Inject Event<PluginDecisionEvent> decisionEvents;
@Inject GameSession gameSession;
private volatile String prevStrategy = null;
```

- [ ] **Step 7.2: Fire transition event at end of `execute()`**

The last lines of `execute()` currently are:
```java
String strategy = data.getStrategyDecisions().stream().findFirst().orElse("MACRO");
caseFile.put(QuarkMindCaseFile.STRATEGY, strategy);
log.debugf(...)
```

Add transition detection after `caseFile.put(...)`:
```java
String strategy = data.getStrategyDecisions().stream().findFirst().orElse("MACRO");
caseFile.put(QuarkMindCaseFile.STRATEGY, strategy);
// existing log.debugf(...)

if (!Objects.equals(strategy, prevStrategy)) {
    prevStrategy = strategy;
    int frame = caseFile.get(QuarkMindCaseFile.GAME_FRAME, Long.class)
            .map(Long::intValue).orElse(0);
    decisionEvents.fire(new PluginDecisionEvent(
            getId(), QuarkMindCapabilityTag.STRATEGY,
            AttestationVerdict.SOUND, gameSession.id(), frame));
}
```

- [ ] **Step 7.3: Verify compile + run strategy tests**

```bash
mvn test -Dtest=DroolsStrategyTaskTest,DroolsStrategyTaskStaticTest -q -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml
```

Expected: all pass

---

### Task 8: Wire `DroolsTacticsTask` — enemy presence transition detection

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java`

- [ ] **Step 8.1: Add fields and imports**

Add imports:
```java
import io.casehub.ledger.api.model.AttestationVerdict;
import io.quarkmind.agent.GameSession;
import io.quarkmind.agent.PluginDecisionEvent;
import io.quarkmind.agent.QuarkMindCapabilityTag;
import jakarta.enterprise.event.Event;
import java.util.Objects;
```

Add fields (after existing injected fields):
```java
@Inject Event<PluginDecisionEvent> decisionEvents;
@Inject GameSession gameSession;
private volatile String prevThreatState = null;
```

- [ ] **Step 8.2: Fire transition event in `execute()`**

In `execute()`, after reading `enemies` (currently around line 127), before `if (army.isEmpty()) return;`:

```java
List<Unit> enemies = (List<Unit>) caseFile.get(QuarkMindCaseFile.ENEMY_UNITS, List.class).orElse(List.of());
// ... (other reads stay as-is)

// Transition: detect when enemies appear or disappear
String threatState = enemies.isEmpty() ? "none" : "present";
if (!Objects.equals(threatState, prevThreatState)) {
    prevThreatState = threatState;
    int frame = caseFile.get(QuarkMindCaseFile.GAME_FRAME, Long.class)
            .map(Long::intValue).orElse(0);
    decisionEvents.fire(new PluginDecisionEvent(
            getId(), QuarkMindCapabilityTag.TACTICS,
            AttestationVerdict.SOUND, gameSession.id(), frame));
}

if (army.isEmpty()) return;
// ... rest of execute() unchanged
```

- [ ] **Step 8.3: Verify compile + run tactics tests**

```bash
mvn test -Dtest=DroolsTacticsTaskTest,DroolsTacticsTaskIT -q -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml
```

Expected: all pass

---

### Task 9: Wire `DroolsScoutingTask` — enemy set transition detection

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java`

- [ ] **Step 9.1: Add fields and imports**

Add imports:
```java
import io.casehub.ledger.api.model.AttestationVerdict;
import io.quarkmind.agent.GameSession;
import io.quarkmind.agent.PluginDecisionEvent;
import io.quarkmind.agent.QuarkMindCapabilityTag;
import jakarta.enterprise.event.Event;
import java.util.stream.Collectors;
```

Add fields (after existing injected fields):
```java
@Inject Event<PluginDecisionEvent> decisionEvents;
@Inject GameSession gameSession;
private volatile int prevEnemyHash = 0;
```

- [ ] **Step 9.2: Fire transition event in `execute()`**

In `execute()`, after reading `enemies` (line 85) and `frame` (line 88), add transition check:

```java
List<Unit> enemies = (List<Unit>) caseFile.get(QuarkMindCaseFile.ENEMY_UNITS, List.class).orElse(List.of());
// ... (other reads stay as-is)
long frame = caseFile.get(QuarkMindCaseFile.GAME_FRAME, Long.class).orElse(0L);

// Transition: detect when enemy set changes (tag-only hash — position excluded because Unit
// is a record and position changes every tick, making record hashCode() useless here)
int enemyHash = enemies.stream()
        .map(Unit::tag)
        .sorted()
        .collect(Collectors.joining())
        .hashCode();
if (enemyHash != prevEnemyHash) {
    prevEnemyHash = enemyHash;
    decisionEvents.fire(new PluginDecisionEvent(
            getId(), QuarkMindCapabilityTag.SCOUTING,
            AttestationVerdict.SOUND, gameSession.id(), (int) frame));
}

// Detect game restart (mock loop resets frame to 0) — existing code below
if (frame < lastFrame) { ... }
```

- [ ] **Step 9.3: Verify compile + run scouting tests**

```bash
mvn test -Dtest=DroolsScoutingTaskTest,DroolsScoutingTaskIT,DroolsScoutingRulesTest -q -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml
```

Expected: all pass

---

### Task 10: Wire `EconomicsDecisionService` — build type transition detection

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/flow/EconomicsDecisionService.java`

- [ ] **Step 10.1: Add fields, imports, and `maybeFireBuildEvent` helper**

Add imports:
```java
import io.casehub.ledger.api.model.AttestationVerdict;
import io.quarkmind.agent.GameSession;
import io.quarkmind.agent.PluginDecisionEvent;
import io.quarkmind.agent.QuarkMindCapabilityTag;
import jakarta.enterprise.event.Event;
```

Add fields (after existing `private final IntentQueue intentQueue;`):
```java
@Inject Event<PluginDecisionEvent> decisionEvents;
@Inject GameSession gameSession;
private volatile String prevBuildType = null;
```

Add private helper method (at the bottom of the class, before closing `}`):

```java
private void maybeFireBuildEvent(String buildType) {
    if (!buildType.equals(prevBuildType)) {
        prevBuildType = buildType;
        decisionEvents.fire(new PluginDecisionEvent(
                "economics.flow", QuarkMindCapabilityTag.ECONOMICS,
                AttestationVerdict.SOUND, gameSession.id(), 0));
    }
}
```

Note: `gameFrame = 0` — `EconomicsDecisionService` operates on `GameStateTick` which has no
frame field. Zero is correct here; frame is used for logging only.

- [ ] **Step 10.2: Add `maybeFireBuildEvent()` call in `checkSupply()` after `intentQueue.add()`**

Inside the `tick.workers().stream().findFirst().ifPresent(probe -> { ... })` lambda:

```java
intentQueue.add(new BuildIntent(probe.tag(), BuildingType.PYLON, pos));
log.debugf("[FLOW-ECONOMICS] Pylon at %s (supply %d/%d)", pos, tick.supplyUsed(), tick.supplyCap());
maybeFireBuildEvent("PYLON");    // ADD
```

- [ ] **Step 10.3: Add `maybeFireBuildEvent()` call in `checkProbes()` after `intentQueue.add()`**

Inside the `tick.buildings().stream()...findFirst().ifPresent(nexus -> { ... })` lambda:

```java
intentQueue.add(new TrainIntent(nexus.tag(), UnitType.PROBE));
log.debugf("[FLOW-ECONOMICS] Probe (workers=%d/%d)", tick.workers().size(), PROBE_CAP);
maybeFireBuildEvent("PROBE");    // ADD
```

- [ ] **Step 10.4: Add `maybeFireBuildEvent()` call in `checkGas()` after `intentQueue.add()`**

Inside the inner `tick.workers().stream().findFirst().ifPresent(probe -> { ... })` lambda:

```java
intentQueue.add(new BuildIntent(probe.tag(), BuildingType.ASSIMILATOR, geyser.position()));
log.debugf("[FLOW-ECONOMICS] Assimilator at %s", geyser.position());
maybeFireBuildEvent("ASSIMILATOR");    // ADD
```

- [ ] **Step 10.5: Add `maybeFireBuildEvent()` call in `checkExpansion()` after `intentQueue.add()`**

Inside the `tick.workers().stream().findFirst().ifPresent(probe -> { ... })` lambda:

```java
intentQueue.add(new BuildIntent(probe.tag(), BuildingType.NEXUS, NATURAL_EXPANSION_POS));
log.infof("[FLOW-ECONOMICS] Expanding to natural at %s", NATURAL_EXPANSION_POS);
maybeFireBuildEvent("NEXUS");    // ADD
```

- [ ] **Step 10.6: Verify compile + run economics tests**

```bash
mvn test -Dtest=EconomicsFlowTest,BasicEconomicsTaskTest -q -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml
```

Expected: all pass

---

### Task 11: `LedgerAuditIT` integration test

**Files:**
- Create: `src/test/java/io/quarkmind/plugin/LedgerAuditIT.java`

- [ ] **Step 11.1: Write the failing integration test**

```java
package io.quarkmind.plugin;

import io.casehub.annotation.CaseType;
import io.casehub.coordination.PropagationContext;
import io.casehub.core.CaseFile;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.persistence.memory.InMemoryCaseFileRepository;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import io.quarkmind.agent.GameSession;
import io.quarkmind.agent.plugin.StrategyTask;
import io.quarkmind.domain.*;
import io.quarkmind.sc2.IntentQueue;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Verifies that plugin decisions are written to the in-memory ledger via PluginOutcomeAuditor.
 *
 * SINGLE TEST METHOD BY DESIGN: DroolsStrategyTask.prevStrategy persists across @Test methods
 * on the same @ApplicationScoped CDI bean. The first call (prevStrategy=null) always fires a
 * transition. A second method using the same CaseFile state will NOT fire — event never comes,
 * assertion fails. Future tests must produce a different strategy output, or add a test-only
 * clearPrevState() method to DroolsStrategyTask.
 */
@QuarkusTest
class LedgerAuditIT {

    @Inject @CaseType("starcraft-game") StrategyTask strategyTask;
    @Inject LedgerEntryRepository ledgerRepo;
    @Inject GameSession gameSession;
    @Inject IntentQueue intentQueue;

    @BeforeEach
    void setup() {
        gameSession.reset();
        intentQueue.drainAll();
    }

    @AfterEach
    void cleanup() {
        intentQueue.drainAll();
    }

    @Test
    void strategyTransitionWritesLedgerEntry() throws InterruptedException {
        // State: enough minerals + workers + nexus to let Drools fire a strategy rule
        // caseFile() signature matches DroolsStrategyTaskTest helper pattern:
        // (minerals, gas, workers, buildings, enemyPosture, timingAttack)
        CaseFile cf = caseFile(200, 0, workers(4), List.of(nexus()), "UNKNOWN", false);

        strategyTask.execute(cf);

        Thread.sleep(300); // wait for @ObservesAsync to complete

        List<LedgerEntry> entries = ledgerRepo.findBySubjectId(gameSession.id());
        assertThat(entries)
                .as("Expected at least one ledger entry from DroolsStrategyTask")
                .isNotEmpty();
        assertThat(entries)
                .anyMatch(e -> "strategy.drools".equals(e.actorId()));
    }

    // --- Helpers (mirrors DroolsStrategyTaskTest pattern) ---

    private CaseFile caseFile(int minerals, int gas,
                               List<Unit> workers, List<Building> buildings,
                               String enemyPosture, boolean timingAttack) {
        var repo = new InMemoryCaseFileRepository();
        var cf = repo.create("starcraft-game", Map.of(), PropagationContext.createRoot());
        cf.put(io.quarkmind.agent.QuarkMindCaseFile.READY, Boolean.TRUE);
        cf.put(io.quarkmind.agent.QuarkMindCaseFile.MINERALS, minerals);
        cf.put(io.quarkmind.agent.QuarkMindCaseFile.VESPENE, gas);
        cf.put(io.quarkmind.agent.QuarkMindCaseFile.WORKERS, workers);
        cf.put(io.quarkmind.agent.QuarkMindCaseFile.MY_BUILDINGS, buildings);
        cf.put(io.quarkmind.agent.QuarkMindCaseFile.ARMY, List.of());
        cf.put(io.quarkmind.agent.QuarkMindCaseFile.GEYSERS, List.of());
        cf.put(io.quarkmind.agent.QuarkMindCaseFile.ENEMY_UNITS, List.of());
        cf.put(io.quarkmind.agent.QuarkMindCaseFile.ENEMY_POSTURE, enemyPosture);
        cf.put(io.quarkmind.agent.QuarkMindCaseFile.TIMING_ATTACK_INCOMING, timingAttack);
        cf.put(io.quarkmind.agent.QuarkMindCaseFile.ENEMY_ARMY_SIZE, 0);
        cf.put(io.quarkmind.agent.QuarkMindCaseFile.RESOURCE_BUDGET,
                new io.quarkmind.agent.ResourceBudget(minerals, gas));
        cf.put(io.quarkmind.agent.QuarkMindCaseFile.GAME_FRAME, 0L);
        return cf;
    }

    private List<Unit> workers(int count) {
        return java.util.stream.IntStream.range(0, count)
                .mapToObj(i -> new Unit("probe-" + i, UnitType.PROBE,
                        new Point2d(8 + i, 8), 40, 40, 20, 20, 0, 0))
                .toList();
    }

    private Building nexus() {
        return new Building("nexus-0", BuildingType.NEXUS, new Point2d(8, 8), true, 1000, 1000, 1000, 1000);
    }
}
```

- [ ] **Step 11.2: Run the test — expect failure (CDI wiring not yet confirmed)**

```bash
mvn test -Dtest=LedgerAuditIT -q -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml
```

Expected: CDI boot completes, test assertion passes or fails with a meaningful error.

If CDI boot fails with `UnsatisfiedResolutionException`: a required ledger SPI is missing from
`selected-alternatives`. Check the error message for the missing class and add it.

If CDI boot fails with `ClassSelector resolution failed` (GE-20260519-e13b01): add a
package-private stub in `src/test/java/io/quarkmind/plugin/` for the unsatisfied interface:

```java
@ApplicationScoped
class InMemoryLedgerStub implements <MissingInterface> {
    @Override public /* required methods */ { /* return empty/null-safe defaults */ }
}
```

If the test assertion fails (entries empty): verify `Thread.sleep(300)` is sufficient and
`@ObservesAsync` thread pool is processing. Increase to 500ms if needed.

- [ ] **Step 11.3: Run the test — confirm pass**

```bash
mvn test -Dtest=LedgerAuditIT -q -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml
```

Expected: 1 test passes

- [ ] **Step 11.4: Run the full test suite to check for regressions**

```bash
mvn test -q -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml
```

Expected: all pre-existing tests pass; `LedgerAuditIT` passes

- [ ] **Step 11.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  pom.xml \
  src/main/resources/application.properties \
  src/main/java/io/quarkmind/agent/GameSession.java \
  src/main/java/io/quarkmind/agent/PluginDecisionEvent.java \
  src/main/java/io/quarkmind/agent/QuarkMindCapabilityTag.java \
  src/main/java/io/quarkmind/agent/PluginOutcomeAuditor.java \
  src/main/java/io/quarkmind/agent/LedgerLifecycleAdapter.java \
  src/main/java/io/quarkmind/agent/AgentOrchestrator.java \
  src/main/java/io/quarkmind/plugin/DroolsStrategyTask.java \
  src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java \
  src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java \
  src/main/java/io/quarkmind/plugin/flow/EconomicsDecisionService.java \
  src/test/java/io/quarkmind/plugin/LedgerAuditIT.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#156): Layer 4 casehub-ledger integration — transition-level audit recording for all plugins Refs #175"
```

---

### Task 12: ARC42STORIES.MD update + CLAUDE.md sync

**Files:**
- Modify: `ARC42STORIES.MD` (project root)

- [ ] **Step 12.1: Update Layer 4 row in ARC42STORIES.MD**

Find the Layer 4 row (issue `#156`). Update the status column from `🔲 pending (#156)` to `✅`.
Remove the `(#156)` forward reference from the row.

- [ ] **Step 12.2: Commit ARC42STORIES.MD**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "docs(#156): mark Layer 4 complete in ARC42STORIES.MD Closes #156"
```

- [ ] **Step 12.3: Invoke `update-claude-md`**

Remove the stale sentence "Until migration is complete, new layer completions still require a
LAYER-LOG entry" from `CLAUDE.md`. Issue #166 (migration to ARC42STORIES.MD) is closed —
LAYER-LOG.md is retired. Update the reference to "update ARC42STORIES.MD" instead.

Invoke the `update-claude-md` skill to handle this.
