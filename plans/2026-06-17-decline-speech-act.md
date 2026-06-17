# DECLINE Speech Act Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire the qhorus DECLINE speech act in QuarkMind's game-loop dispatch so that plugins outside their activation scope formally declare it — closing QG2 (no silent no-op).

**Architecture:** A new `PluginDispatchBroker` CDI bean evaluates each registered plugin's activation state before `createAndSolve()` runs. On state transitions (first tick or activation change), it emits COMMAND+DONE or COMMAND+DECLINE on the `quarkmind-plugin-dispatch` qhorus channel using a collect-then-apply pattern to keep in-memory state consistent with DB commits. A new `MapCaseContext` provides a `CaseContext` backed by the tick's `caseData` map, safe to use pre-engine without touching the database.

**Tech Stack:** Java 21, Quarkus CDI (`@ApplicationScoped`, `@Inject`, `@Transactional`, `@Observes`, `@PostConstruct`), casehub-qhorus (`MessageService`, `ChannelService`, `MessageDispatch`, `DispatchResult`), casehub-core (`TaskDefinitionRegistry`), JUnit 5 + AssertJ

**Spec:** `docs/superpowers/specs/2026-06-17-decline-speech-act-design.md`

**Issue:** #199

---

## File Map

| Status | File | Purpose |
|---|---|---|
| CREATE | `src/main/java/io/quarkmind/agent/MapCaseContext.java` | `CaseContext` backed by `Map<String, Object>`; used for pre-engine activation evaluation |
| CREATE | `src/main/java/io/quarkmind/agent/PluginDispatchBroker.java` | CDI bean: owns `quarkmind-plugin-dispatch` channel, emits COMMAND+DONE/DECLINE on activation transitions |
| CREATE | `src/test/java/io/quarkmind/agent/MapCaseContextTest.java` | Unit tests for `MapCaseContext` |
| CREATE | `src/test/java/io/quarkmind/agent/PluginDispatchBrokerTest.java` | Unit tests for `PluginDispatchBroker` via package-private constructor |
| MODIFY | `src/main/java/io/quarkmind/agent/AgentOrchestrator.java` | Add `brokerMs` field to `TickTimings` record; update `totalMs()` |
| MODIFY | `src/main/java/io/quarkmind/agent/GameTickExecutor.java` | Inject `PluginDispatchBroker`; add `recordTick()` call; add `t1b` timestamp; update `TickTimings` constructor call (3 → 4 args); update log format |
| MODIFY | `src/test/java/io/quarkmind/agent/GameLoopBenchmarkTest.java` | Add `brokerMs[]` array; read `t.brokerMs()`; update `formatReport()` signature and format string |
| MODIFY | `src/test/java/io/quarkmind/sc2/mock/AdaptivePluginSelectionIT.java` | Add `@Inject PluginDispatchBroker`, `@Inject MessageService`; capture cursor in `@BeforeEach`; add `firstTickEmitsCorrectDeclineSignals()` test |
| MODIFY | `ARC42STORIES.MD` | Mark QG2 resolved in C2 known limitations; update L2/L3 gap table row |

**Compile dependency note:** `AgentOrchestrator.TickTimings` (3-arg constructor → 4-arg) and `GameTickExecutor` (constructor call site) must be updated in the same step — one breaks the other.

---

## Task 1: MapCaseContext

**Files:**
- Create: `src/main/java/io/quarkmind/agent/MapCaseContext.java`
- Create: `src/test/java/io/quarkmind/agent/MapCaseContextTest.java`

- [ ] **Step 1: Write the failing tests**

```java
// src/test/java/io/quarkmind/agent/MapCaseContextTest.java
package io.quarkmind.agent;

import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

class MapCaseContextTest {

    @Test
    void containsReturnsTrueForPresentKey() {
        var ctx = new MapCaseContext(Map.of("READY", Boolean.TRUE));
        assertThat(ctx.contains("READY")).isTrue();
    }

    @Test
    void containsReturnsFalseForAbsentKey() {
        var ctx = new MapCaseContext(Map.of());
        assertThat(ctx.contains("READY")).isFalse();
    }

    @Test
    void getReturnsValueForPresentKey() {
        var ctx = new MapCaseContext(Map.of("MINERALS", 150));
        assertThat(ctx.get("MINERALS")).isEqualTo(150);
    }

    @Test
    void getReturnsNullForAbsentKey() {
        var ctx = new MapCaseContext(Map.of());
        assertThat(ctx.get("MINERALS")).isNull();
    }

    @Test
    void getOrDefaultReturnsValueWhenPresent() {
        var ctx = new MapCaseContext(Map.of("MINERALS", 200));
        assertThat(ctx.<Integer>getOrDefault("MINERALS", 0)).isEqualTo(200);
    }

    @Test
    void getOrDefaultReturnsDefaultWhenAbsent() {
        var ctx = new MapCaseContext(Map.of());
        assertThat(ctx.<Integer>getOrDefault("MINERALS", 0)).isEqualTo(0);
    }

    @Test
    void getListReturnsListWhenPresent() {
        var list = List.of("unit1", "unit2");
        var ctx = new MapCaseContext(Map.of("WORKERS", list));
        assertThat(ctx.getList("WORKERS", String.class)).isEqualTo(list);
    }

    @Test
    void getListReturnsEmptyWhenAbsent() {
        var ctx = new MapCaseContext(Map.of());
        assertThat(ctx.getList("WORKERS", String.class)).isEmpty();
    }

    @Test
    void sizeAndIsEmpty() {
        assertThat(new MapCaseContext(Map.of()).isEmpty()).isTrue();
        assertThat(new MapCaseContext(Map.of("K", "V")).size()).isEqualTo(1);
    }

    @Test
    void getKeysReturnsAllKeys() {
        var ctx = new MapCaseContext(Map.of("A", 1, "B", 2));
        assertThat(ctx.getKeys()).containsExactlyInAnyOrder("A", "B");
    }

    @Test
    void setThrowsUnsupportedOperationException() {
        var ctx = new MapCaseContext(Map.of());
        assertThatThrownBy(() -> ctx.set("K", "V"))
            .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void removeThrowsUnsupportedOperationException() {
        var ctx = new MapCaseContext(Map.of());
        assertThatThrownBy(() -> ctx.remove("K"))
            .isInstanceOf(UnsupportedOperationException.class);
    }
}
```

- [ ] **Step 2: Run tests — expect compile failure (MapCaseContext doesn't exist yet)**

```
mvn test -Dtest=MapCaseContextTest -q 2>&1 | head -5
```

Expected: compile error `cannot find symbol: class MapCaseContext`

- [ ] **Step 3: Implement MapCaseContext**

```java
// src/main/java/io/quarkmind/agent/MapCaseContext.java
package io.quarkmind.agent;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.context.CaseContext;
import io.casehub.api.context.ReadablePanel;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.function.Function;

/**
 * CaseContext backed by a Map<String, Object> snapshot.
 *
 * <p>Used for pre-engine activation evaluation in PluginDispatchBroker — wraps the
 * immutable caseData map produced by GameStateTranslator without touching the database.
 * Also simplifies unit tests: replaces the CaseFileContext + InMemoryCaseFileRepository
 * construction pattern with new MapCaseContext(Map.of(...)).
 *
 * <p>Only read operations are implemented; write operations throw UnsupportedOperationException.
 * activateIf() predicates must use only contains() and get() from this context (or CDI singletons).
 */
public final class MapCaseContext implements CaseContext {

    private final Map<String, Object> data;

    public MapCaseContext(Map<String, Object> data) {
        this.data = Map.copyOf(data);   // immutable snapshot
    }

    @Override public boolean contains(String key) { return data.containsKey(key); }
    @Override public Object  get(String key)       { return data.get(key); }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getAs(String key, Class<T> type) {
        Object v = data.get(key);
        return type.isInstance(v) ? (T) v : null;
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getOrDefault(String key, T defaultValue) {
        Object v = data.get(key);
        if (v == null) return defaultValue;
        try { return (T) v; } catch (ClassCastException e) { return defaultValue; }
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> List<T> getList(String key, Class<T> elementType) {
        Object v = data.get(key);
        if (v instanceof List<?> list) return (List<T>) list;
        return List.of();
    }

    @Override public String  getString(String key) { return getAs(key, String.class); }
    @Override public Integer getInt(String key)    { return getAs(key, Integer.class); }
    @Override public Long    getLong(String key)   { return getAs(key, Long.class); }
    @Override public Double  getDouble(String key) { return getAs(key, Double.class); }
    @Override public Boolean getBoolean(String key){ return getAs(key, Boolean.class); }

    @Override public Set<String>        getKeys()  { return Set.copyOf(data.keySet()); }
    @Override public Map<String, Object> getData() { return Map.copyOf(data); }
    @Override public boolean isEmpty()             { return data.isEmpty(); }
    @Override public int     size()                { return data.size(); }
    @Override public long    getVersion()          { return 0L; }

    @Override
    public Map<String, Object> getAll(String... keys) {
        Map<String, Object> result = new LinkedHashMap<>();
        for (String k : keys) { Object v = data.get(k); if (v != null) result.put(k, v); }
        return result;
    }

    // ── Write operations — not supported on a snapshot ───────────────────────

    @Override public CaseContext set(String key, Object value)              { throw new UnsupportedOperationException("MapCaseContext is read-only"); }
    @Override public CaseContext setAll(Map<String, Object> values)         { throw new UnsupportedOperationException("MapCaseContext is read-only"); }
    @Override public CaseContext remove(String key)                          { throw new UnsupportedOperationException("MapCaseContext is read-only"); }
    @Override public CaseContext clear()                                     { throw new UnsupportedOperationException("MapCaseContext is read-only"); }
    @Override public CaseContext update(String key, Function<Object,Object> fn) { throw new UnsupportedOperationException("MapCaseContext is read-only"); }
    @Override public boolean compareAndSet(String key, Object expected, Object newValue) { throw new UnsupportedOperationException("MapCaseContext is read-only"); }
    @Override public Object putIfAbsent(String key, Object value)           { throw new UnsupportedOperationException("MapCaseContext is read-only"); }
    @Override public Object computeIfAbsent(String key, Function<String, Object> f) { throw new UnsupportedOperationException("MapCaseContext is read-only"); }

    // ── Unsupported operations ────────────────────────────────────────────────

    @Override public ReadablePanel panel(String name)                        { throw new UnsupportedOperationException(); }
    @Override public Object getPath(String path)                             { throw new UnsupportedOperationException(); }
    @Override public String getPathAsString(String path)                     { throw new UnsupportedOperationException(); }
    @Override public CaseContext setPath(String path, Object value)         { throw new UnsupportedOperationException(); }
    @Override public Optional<JsonNode> applyAndDiff(String path, Object v) { throw new UnsupportedOperationException(); }
    @Override public JsonNode asJsonNode()                                   { throw new UnsupportedOperationException(); }
    @Override public CaseContext merge(CaseContext other)                    { throw new UnsupportedOperationException(); }
    @Override public CaseContext snapshot()                                  { throw new UnsupportedOperationException(); }
    @Override public JsonNode diff(CaseContext other)                        { throw new UnsupportedOperationException(); }
    @Override public void applyDiff(JsonNode diff)                           { throw new UnsupportedOperationException(); }
}
```

- [ ] **Step 4: Run tests — expect all pass**

```
mvn test -Dtest=MapCaseContextTest -q
```

Expected: `BUILD SUCCESS` with 12 tests passing.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/agent/MapCaseContext.java \
  src/test/java/io/quarkmind/agent/MapCaseContextTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(agent): add MapCaseContext — CaseContext backed by Map snapshot

Refs #199"
```

---

## Task 2: PluginDispatchBrokerTest (write first)

**Files:**
- Create: `src/test/java/io/quarkmind/agent/PluginDispatchBrokerTest.java`

This test will NOT compile until `PluginDispatchBroker` exists. Write it first so the implementation is guided by the test's expectations.

- [ ] **Step 1: Write the test file**

```java
// src/test/java/io/quarkmind/agent/PluginDispatchBrokerTest.java
package io.quarkmind.agent;

import io.casehub.core.CaseFile;
import io.casehub.core.TaskDefinition;
import io.casehub.core.TaskDefinitionRegistry;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.message.MessageService;
import io.quarkmind.sc2.GameStarted;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.atomic.AtomicLong;
import java.util.function.Predicate;

import static org.assertj.core.api.Assertions.*;

class PluginDispatchBrokerTest {

    // ── Stubs ─────────────────────────────────────────────────────────────────

    private static final UUID CHANNEL_ID = UUID.randomUUID();

    /** Stub plugin implementing both poc and new engine interfaces. */
    static class StubPlugin implements TaskDefinition, io.quarkmind.agent.TaskDefinition {
        private final String id;
        private final Set<String> requiresKeys;
        private boolean activateResult;

        StubPlugin(String id, Set<String> requiresKeys, boolean activateResult) {
            this.id = id;
            this.requiresKeys = requiresKeys;
            this.activateResult = activateResult;
        }

        void setActivateResult(boolean v) { this.activateResult = v; }

        // io.quarkmind.agent.TaskDefinition
        @Override public String getId()   { return id; }
        @Override public String getName() { return id; }
        @Override public Set<String> requires() { return requiresKeys; }
        @Override public Predicate<io.casehub.api.context.CaseContext> activateIf() {
            return ctx -> activateResult;
        }
        @Override public void execute(io.casehub.api.context.CaseContext ctx) {}
        @Override public Set<String> produces() { return Set.of(); }

        // io.casehub.core.TaskDefinition
        @Override public Set<String> entryCriteria() { return requiresKeys; }
        @Override public Set<String> producedKeys()  { return Set.of(); }
        @Override public boolean canActivate(CaseFile caseFile) { return activateResult; }
        @Override public void execute(CaseFile caseFile) {}
    }

    /** Stub MessageService that captures dispatches and returns sequential message IDs. */
    private final List<MessageDispatch>  captured  = new ArrayList<>();
    private final AtomicLong             idCounter = new AtomicLong(1L);
    private boolean                      throwOnNextDispatch = false;

    private final MessageService stubMessageService = new MessageService() {
        @Override
        public DispatchResult dispatch(MessageDispatch d) {
            if (throwOnNextDispatch) {
                throwOnNextDispatch = false;
                throw new RuntimeException("simulated DB error");
            }
            captured.add(d);
            long id = idCounter.getAndIncrement();
            return new DispatchResult(id, d.channelId(), d.sender(), d.type(),
                d.correlationId(), d.inReplyTo(), null, d.target(),
                null, null, null, 0, null);
        }
    };

    // ── Helpers ───────────────────────────────────────────────────────────────

    private TaskDefinitionRegistry registryWith(TaskDefinition... plugins) {
        var registry = new TaskDefinitionRegistry();
        for (TaskDefinition p : plugins) {
            registry.register(p, Set.of("starcraft-game"));
        }
        return registry;
    }

    private PluginDispatchBroker brokerWith(TaskDefinition... plugins) {
        return new PluginDispatchBroker(registryWith(plugins), stubMessageService, CHANNEL_ID);
    }

    @BeforeEach
    void setUp() {
        captured.clear();
        throwOnNextDispatch = false;
    }

    // ── Tests ─────────────────────────────────────────────────────────────────

    @Test
    void firstTickActivatingPluginEmitsCommandPlusDone() {
        var plugin = new StubPlugin("scouting", Set.of("READY"), true);
        var broker = brokerWith(plugin);

        broker.recordTick(Map.of("READY", true));

        assertThat(captured).hasSize(2);
        assertThat(captured.get(0).type()).isEqualTo(MessageType.COMMAND);
        assertThat(captured.get(0).target()).isEqualTo("plugin:scouting");
        assertThat(captured.get(1).type()).isEqualTo(MessageType.DONE);
        assertThat(captured.get(1).sender()).isEqualTo("plugin:scouting");
        assertThat(captured.get(1).inReplyTo()).isEqualTo(1L); // ID of the COMMAND
    }

    @Test
    void firstTickDecliningPluginEmitsCommandPlusDecline() {
        var plugin = new StubPlugin("strategy.early", Set.of("READY"), false);
        var broker = brokerWith(plugin);

        broker.recordTick(Map.of("READY", true));

        assertThat(captured).hasSize(2);
        assertThat(captured.get(0).type()).isEqualTo(MessageType.COMMAND);
        assertThat(captured.get(1).type()).isEqualTo(MessageType.DECLINE);
        assertThat(captured.get(1).sender()).isEqualTo("plugin:strategy.early");
    }

    @Test
    void repeatedTickSameStateSendsNoSignal() {
        var plugin = new StubPlugin("scouting", Set.of("READY"), true);
        var broker = brokerWith(plugin);

        broker.recordTick(Map.of("READY", true));
        captured.clear();

        broker.recordTick(Map.of("READY", true)); // same state

        assertThat(captured).isEmpty();
    }

    @Test
    void stateChangeActiveToInactiveEmitsCommandPlusDecline() {
        var plugin = new StubPlugin("tactics", Set.of("READY"), true);
        var broker = brokerWith(plugin);
        broker.recordTick(Map.of("READY", true)); // first tick: DONE
        captured.clear();

        plugin.setActivateResult(false);
        broker.recordTick(Map.of("READY", true)); // transition to DECLINE

        assertThat(captured).hasSize(2);
        assertThat(captured.get(0).type()).isEqualTo(MessageType.COMMAND);
        assertThat(captured.get(1).type()).isEqualTo(MessageType.DECLINE);
    }

    @Test
    void stateChangeInactiveToActiveEmitsCommandPlusDone() {
        var plugin = new StubPlugin("tactics", Set.of("READY"), false);
        var broker = brokerWith(plugin);
        broker.recordTick(Map.of("READY", true)); // first tick: DECLINE
        captured.clear();

        plugin.setActivateResult(true);
        broker.recordTick(Map.of("READY", true)); // transition to DONE

        assertThat(captured).hasSize(2);
        assertThat(captured.get(1).type()).isEqualTo(MessageType.DONE);
    }

    @Test
    void outOfScopePluginSendsNoSignal() {
        // Plugin requires STRATEGY, but caseData only has READY — out of scope
        var plugin = new StubPlugin("tactics", Set.of("READY", "STRATEGY"), true);
        var broker = brokerWith(plugin);

        broker.recordTick(Map.of("READY", true)); // STRATEGY absent — not in scope

        assertThat(captured).isEmpty();
    }

    @Test
    void scopeReentryAfterAbsenceTreatedAsFirstSeen() {
        var plugin = new StubPlugin("tactics", Set.of("READY", "STRATEGY"), true);
        var broker = brokerWith(plugin);

        broker.recordTick(Map.of("READY", true));       // out of scope — no signal
        broker.recordTick(Map.of("READY", true, "STRATEGY", "ATTACK")); // re-enters scope

        // Should emit for the scope re-entry as if first-seen
        assertThat(captured).hasSize(2);
        assertThat(captured.get(1).type()).isEqualTo(MessageType.DONE);
    }

    @Test
    void gameStartedClearsState_nextTickReestablishesBaseline() {
        var plugin = new StubPlugin("scouting", Set.of("READY"), true);
        var broker = brokerWith(plugin);

        broker.recordTick(Map.of("READY", true)); // first tick: DONE, state recorded
        captured.clear();

        broker.onGameStarted(new GameStarted()); // clears priorActivation

        broker.recordTick(Map.of("READY", true)); // treated as first-seen again

        assertThat(captured).hasSize(2);  // COMMAND + DONE re-emitted
    }

    @Test
    void lastDispatchedIdAdvancesAfterSuccessfulDispatch() {
        var plugin = new StubPlugin("scouting", Set.of("READY"), true);
        var broker = brokerWith(plugin);

        assertThat(broker.lastDispatchedId()).isEqualTo(0L);

        broker.recordTick(Map.of("READY", true));

        // Last dispatched = DONE message ID (2 — COMMAND gets 1, DONE gets 2)
        assertThat(broker.lastDispatchedId()).isEqualTo(2L);
    }

    @Test
    void rollbackSimulation_priorActivationUnchangedOnDispatchFailure() {
        var plugin = new StubPlugin("scouting", Set.of("READY"), true);
        var broker = brokerWith(plugin);

        // Make the COMMAND dispatch succeed but DONE dispatch throw
        // (second dispatch in sendCommitmentSignal throws)
        // We can't easily control which dispatch throws, so let's throw on the first dispatch
        throwOnNextDispatch = true;

        // recordTick should propagate the exception
        assertThatThrownBy(() -> broker.recordTick(Map.of("READY", true)))
            .isInstanceOf(RuntimeException.class)
            .hasMessage("simulated DB error");

        // priorActivation must be unchanged — plugin not recorded as seen
        // So the next call should re-emit (not be suppressed as "no change")
        assertThat(captured).isEmpty();
        broker.recordTick(Map.of("READY", true)); // retry — should emit again
        assertThat(captured).hasSize(2);
    }

    @Test
    void correlationIdPairsCommandWithReply() {
        var plugin = new StubPlugin("scouting", Set.of("READY"), true);
        var broker = brokerWith(plugin);

        broker.recordTick(Map.of("READY", true));

        assertThat(captured).hasSize(2);
        // Same correlationId on COMMAND and DONE
        assertThat(captured.get(0).correlationId())
            .isEqualTo(captured.get(1).correlationId())
            .isNotNull();
    }

    @Test
    void commandTargetContainsColonToBypassObligorTrustCheck() {
        var plugin = new StubPlugin("scouting", Set.of("READY"), true);
        var broker = brokerWith(plugin);

        broker.recordTick(Map.of("READY", true));

        assertThat(captured.get(0).target()).contains(":");
        assertThat(captured.get(0).target()).isEqualTo("plugin:scouting");
    }
}
```

- [ ] **Step 2: Verify the test file compiles (will fail — PluginDispatchBroker missing)**

```
mvn test-compile -q 2>&1 | grep "cannot find symbol" | head -5
```

Expected: `cannot find symbol: class PluginDispatchBroker`

---

## Task 3: PluginDispatchBroker implementation

**Files:**
- Create: `src/main/java/io/quarkmind/agent/PluginDispatchBroker.java`

- [ ] **Step 1: Create PluginDispatchBroker**

```java
// src/main/java/io/quarkmind/agent/PluginDispatchBroker.java
package io.quarkmind.agent;

import io.casehub.core.TaskDefinitionRegistry;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.ChannelCreateRequest;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.message.MessageService;
import io.quarkmind.sc2.GameStarted;
import io.quarkus.narayana.jta.QuarkusTransaction;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.jboss.logging.Logger;

import java.util.HashSet;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Owns the quarkmind-plugin-dispatch qhorus channel.
 *
 * <p>Emits COMMAND+DONE or COMMAND+DECLINE on plugin activation transitions.
 * Called by GameTickExecutor before createAndSolve() — pre-engine, on the tick snapshot.
 *
 * <p>Signal frequency: transitions only. First tick after GameStarted establishes baseline.
 * Signals only fire when a plugin's activation state changes (or is first observed).
 *
 * <p>Phase 1 semantic compromise: DONE is dispatched before the plugin executes.
 * Phase 2 SequenceWorker (engine#484) will emit DONE post-execution.
 *
 * <p>Refs #199
 */
@ApplicationScoped
public class PluginDispatchBroker {

    public static final String CHANNEL_NAME = "quarkmind-plugin-dispatch";

    private static final Logger log = Logger.getLogger(PluginDispatchBroker.class);

    private final TaskDefinitionRegistry registry;
    private final MessageService         messageService;
    private final ChannelService         channelService;

    private final ConcurrentHashMap<String, Boolean> priorActivation = new ConcurrentHashMap<>();
    private volatile long lastDispatchedId = 0L;

    private UUID channelId;

    // ── Constructors ──────────────────────────────────────────────────────────

    @Inject
    public PluginDispatchBroker(TaskDefinitionRegistry registry,
                                 MessageService messageService,
                                 ChannelService channelService) {
        this.registry       = registry;
        this.messageService = messageService;
        this.channelService = channelService;
    }

    /** Package-private — unit tests; bypasses @PostConstruct channel setup. channelId must be non-null. */
    PluginDispatchBroker(TaskDefinitionRegistry registry, MessageService messageService, UUID channelId) {
        this.registry       = registry;
        this.messageService = messageService;
        this.channelService = null;   // only used in @PostConstruct
        this.channelId      = channelId;
    }

    // ── Lifecycle ─────────────────────────────────────────────────────────────

    @PostConstruct
    void init() {
        // GE-20260529-88b7b6: @Transactional on @PostConstruct not intercepted by Arc;
        // ChannelService.create() is not idempotent — findByName() first.
        channelId = QuarkusTransaction.requiringNew().call(() ->
            channelService.findByName(CHANNEL_NAME)
                .map(c -> c.id)
                .orElseGet(() -> channelService.create(
                    new ChannelCreateRequest(
                        CHANNEL_NAME,
                        "Plugin activation commitment dispatch",
                        ChannelSemantic.APPEND,
                        null, null, null, null, null,
                        Set.of(MessageType.COMMAND, MessageType.DONE, MessageType.DECLINE),
                        null, null, null, null, null
                    )
                ).id)
        );
        log.infof("[DISPATCH-BROKER] Channel ready: %s", channelId);
    }

    void onGameStarted(@Observes GameStarted event) {
        // Clear so first tick re-establishes baseline for the new game.
        // lastDispatchedId is NOT reset — it is a monotonic DB cursor.
        priorActivation.clear();
        log.debugf("[DISPATCH-BROKER] State cleared for new game");
    }

    // ── Public API ────────────────────────────────────────────────────────────

    public UUID channelId()        { return channelId; }
    public long lastDispatchedId() { return lastDispatchedId; }

    // ── Core dispatch ─────────────────────────────────────────────────────────

    /**
     * Evaluates each registered plugin's activation against caseData and emits
     * COMMAND+DONE or COMMAND+DECLINE for any that changed since the last tick.
     *
     * <p>Called from GameTickExecutor before createAndSolve(). All dispatches
     * share one transaction; collect-then-apply ensures priorActivation and
     * lastDispatchedId only advance after the transaction commits.
     */
    @Transactional
    public void recordTick(Map<String, Object> caseData) {
        var evalCtx    = new MapCaseContext(caseData);
        var toRemove   = new HashSet<String>();
        var toUpdate   = new LinkedHashMap<String, Boolean>();
        Long lastReplyId = null;

        for (var td : registry.getForCaseType("starcraft-game")) {
            if (!(td instanceof io.quarkmind.agent.TaskDefinition qmTd)) continue;

            boolean inScope = qmTd.requires().stream().allMatch(caseData::containsKey);
            if (!inScope) {
                toRemove.add(qmTd.getId());
                continue;
            }

            boolean nowActive = qmTd.activateIf().test(evalCtx);
            Boolean wasActive = priorActivation.get(qmTd.getId());

            if (wasActive == null || wasActive != nowActive) {
                lastReplyId = sendCommitmentSignal(qmTd.getId(), nowActive);
                toUpdate.put(qmTd.getId(), nowActive);
                log.debugf("[DISPATCH-BROKER] %s → %s", qmTd.getId(),
                    nowActive ? "DONE" : "DECLINE");
            }
        }

        // Apply in-memory state only after all dispatches succeeded.
        // If sendCommitmentSignal() throws → transaction rolls back → these lines
        // never execute → priorActivation unchanged → next tick re-detects and re-emits.
        toRemove.forEach(priorActivation::remove);
        toUpdate.forEach(priorActivation::put);
        if (lastReplyId != null) lastDispatchedId = lastReplyId;
    }

    // ── Private ───────────────────────────────────────────────────────────────

    /**
     * Dispatches COMMAND then DONE or DECLINE on the plugin-dispatch channel.
     * Returns the DONE/DECLINE message ID (for lastDispatchedId cursor update in caller).
     */
    private Long sendCommitmentSignal(String pluginId, boolean activating) {
        String correlationId = UUID.randomUUID().toString();

        DispatchResult commandResult = messageService.dispatch(
            MessageDispatch.builder()
                .channelId(channelId)
                .sender("agent.orchestrator")
                .type(MessageType.COMMAND)
                .correlationId(correlationId)
                .content(pluginId)
                .target("plugin:" + pluginId)   // ":" bypasses obligor trust check
                .actorType(ActorType.SYSTEM)
                .build()
        );

        DispatchResult replyResult = messageService.dispatch(
            MessageDispatch.builder()
                .channelId(channelId)
                .sender("plugin:" + pluginId)
                .type(activating ? MessageType.DONE : MessageType.DECLINE)
                .correlationId(correlationId)
                .inReplyTo(commandResult.messageId())
                .actorType(ActorType.SYSTEM)
                .build()
        );

        return replyResult.messageId();
    }
}
```

- [ ] **Step 2: Run unit tests**

```
mvn test -Dtest=MapCaseContextTest,PluginDispatchBrokerTest -q
```

Expected: `BUILD SUCCESS` — all tests pass.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/agent/PluginDispatchBroker.java \
  src/test/java/io/quarkmind/agent/PluginDispatchBrokerTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(agent): PluginDispatchBroker — DECLINE/DONE on plugin activation transitions

Emits COMMAND+DONE or COMMAND+DECLINE on quarkmind-plugin-dispatch channel
when a plugin's activation state changes. Collect-then-apply pattern keeps
priorActivation and lastDispatchedId consistent with committed DB state.

Refs #199"
```

---

## Task 4: TickTimings + GameTickExecutor (must be one commit — constructor site changes)

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/AgentOrchestrator.java`
- Modify: `src/main/java/io/quarkmind/agent/GameTickExecutor.java`

- [ ] **Step 1: Update `TickTimings` record in `AgentOrchestrator`**

Locate this in `AgentOrchestrator.java` (around line 29):
```java
// BEFORE:
public record TickTimings(long physicsMs, long pluginsMs, long dispatchMs) {
    public long totalMs() { return physicsMs + pluginsMs + dispatchMs; }
}
```

Replace with:
```java
// AFTER:
public record TickTimings(long physicsMs, long pluginsMs, long dispatchMs, long brokerMs) {
    public long totalMs() { return physicsMs + pluginsMs + dispatchMs + brokerMs; }
}
```

- [ ] **Step 2: Rewrite `GameTickExecutor`**

Replace the entire file content:

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

    @Inject SC2Engine            engine;
    @Inject GameStateTranslator  translator;
    @Inject CaseEngine           caseEngine;
    @Inject PluginDispatchBroker pluginDispatchBroker;

    AgentOrchestrator.TickResult execute() {
        long t0 = System.currentTimeMillis();
        engine.tick();
        var gameState = engine.observe();
        long t1 = System.currentTimeMillis();        // physics end: engine.tick + observe

        Map<String, Object> caseData = translator.toMap(gameState);
        pluginDispatchBroker.recordTick(caseData);   // commitment signals before engine
        long t1b = System.currentTimeMillis();       // broker end: toMap + recordTick

        CaseFile caseFile = null;
        try {
            caseFile = caseEngine.createAndSolve("starcraft-game", caseData, Duration.ofSeconds(5));
        } catch (Exception e) {
            log.errorf("CaseEngine decision cycle failed at frame %d: %s",
                       gameState.gameFrame(), e.getMessage());
        }
        long t2 = System.currentTimeMillis();        // plugins end: createAndSolve

        // dispatch() reads IntentQueue (plugin-populated), not CaseFile — safe even on failed solve
        engine.dispatch();
        long t3 = System.currentTimeMillis();        // dispatch end

        var timings = new AgentOrchestrator.TickTimings(t1 - t0, t2 - t1b, t3 - t2, t1b - t1);
        log.debugf("Tick %d — physics=%dms broker=%dms plugins=%dms dispatch=%dms total=%dms | minerals=%d supply=%d/%d",
            gameState.gameFrame(), timings.physicsMs(), timings.brokerMs(),
            timings.pluginsMs(), timings.dispatchMs(), timings.totalMs(),
            gameState.minerals(), gameState.supplyUsed(), gameState.supply());
        return new AgentOrchestrator.TickResult(caseFile, timings);
    }
}
```

- [ ] **Step 3: Compile — both files must compile together**

```
mvn compile -q
```

Expected: `BUILD SUCCESS`. If it fails, the TickTimings constructor call or GameTickExecutor imports are wrong.

- [ ] **Step 4: Run full test suite (non-benchmark) to check nothing regressed**

```
mvn test -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/agent/AgentOrchestrator.java \
  src/main/java/io/quarkmind/agent/GameTickExecutor.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(agent): wire PluginDispatchBroker into game tick; add brokerMs timing

- GameTickExecutor: inject broker, call recordTick() before createAndSolve(),
  measure as brokerMs (t1b-t1) — toMap + commitment dispatch overhead
- AgentOrchestrator.TickTimings: add brokerMs field (3 args -> 4 args);
  totalMs() includes brokerMs

Refs #199"
```

---

## Task 5: GameLoopBenchmarkTest

**Files:**
- Modify: `src/test/java/io/quarkmind/agent/GameLoopBenchmarkTest.java`

- [ ] **Step 1: Add `brokerMs` to measurement loop**

In `gameLoopSmokeTimings()`, add the `brokerMs` array alongside the others and read `t.brokerMs()`:

```java
// BEFORE:
long[] physicsMs  = new long[MEASURE_TICKS];
long[] pluginsMs  = new long[MEASURE_TICKS];
long[] dispatchMs = new long[MEASURE_TICKS];
long[] totalMs    = new long[MEASURE_TICKS];

for (int i = 0; i < MEASURE_TICKS; i++) {
    orchestrator.gameTick();
    AgentOrchestrator.TickTimings t = orchestrator.getLastTickTimings();
    physicsMs [i] = t.physicsMs();
    pluginsMs [i] = t.pluginsMs();
    dispatchMs[i] = t.dispatchMs();
    totalMs   [i] = t.totalMs();
}

String report = formatReport(physicsMs, pluginsMs, dispatchMs, totalMs);

// AFTER:
long[] physicsMs  = new long[MEASURE_TICKS];
long[] brokerMs   = new long[MEASURE_TICKS];
long[] pluginsMs  = new long[MEASURE_TICKS];
long[] dispatchMs = new long[MEASURE_TICKS];
long[] totalMs    = new long[MEASURE_TICKS];

for (int i = 0; i < MEASURE_TICKS; i++) {
    orchestrator.gameTick();
    AgentOrchestrator.TickTimings t = orchestrator.getLastTickTimings();
    physicsMs [i] = t.physicsMs();
    brokerMs  [i] = t.brokerMs();
    pluginsMs [i] = t.pluginsMs();
    dispatchMs[i] = t.dispatchMs();
    totalMs   [i] = t.totalMs();
}

String report = formatReport(physicsMs, brokerMs, pluginsMs, dispatchMs, totalMs);
```

- [ ] **Step 2: Update `formatReport()` signature and format string**

Replace the entire `formatReport()` method:

```java
private static String formatReport(long[] physics, long[] broker,
                                    long[] plugins, long[] dispatch, long[] total) {
    String ts = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm"));
    return String.format("""
            QuarkMind Game Loop Benchmark
            ──────────────────────────────────────────────────────
            Date:    %s
            Profile: %%test (MockEngine)   Warmup: %d   Samples: %d
            ──────────────────────────────────────────────────────
            Phase               mean     p95      max
            engine.tick()      %4dms   %4dms   %4dms
            engine.observe()   (included in physics above)
            commit signals     %4dms   %4dms   %4dms
            caseEngine plugins %4dms   %4dms   %4dms
            engine.dispatch()  %4dms   %4dms   %4dms
            ────────────────────────────────────────
            Total gameTick()   %4dms   %4dms   %4dms
            ──────────────────────────────────────────────────────
            Raw total samples (ms): %s
            ──────────────────────────────────────────────────────
            Paste this table into docs/benchmarks/YYYY-MM-DD-<context>.md to track over time.
            Run with: mvn test -Pbenchmark
            """,
        ts,
        WARMUP_TICKS, MEASURE_TICKS,
        mean(physics),   p95(physics),   max(physics),
        mean(broker),    p95(broker),    max(broker),
        mean(plugins),   p95(plugins),   max(plugins),
        mean(dispatch),  p95(dispatch),  max(dispatch),
        mean(total),     p95(total),     max(total),
        Arrays.toString(total));
}
```

- [ ] **Step 3: Compile the test**

```
mvn test-compile -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/test/java/io/quarkmind/agent/GameLoopBenchmarkTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "test(benchmark): add brokerMs tracking to GameLoopBenchmarkTest

Add brokerMs[] array, t.brokerMs() read in measurement loop, and
'commit signals' row in formatReport() between physics and plugins.
brokerMs covers toMap() + recordTick() overhead per tick.

Refs #199"
```

---

## Task 6: AdaptivePluginSelectionIT extension

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/mock/AdaptivePluginSelectionIT.java`

- [ ] **Step 1: Add new field declarations after existing @Inject fields**

At the top of the class (after `@Inject ScoutingIntelBroker broker;`), add:

```java
@Inject PluginDispatchBroker dispatchBroker;
@Inject MessageService        messageService;
private long afterId;
```

- [ ] **Step 2: Add cursor capture to `@BeforeEach setUp()`**

After `broker.clearLatest();`, add:

```java
afterId = dispatchBroker.lastDispatchedId();   // cursor: ignore messages from prior tests
```

- [ ] **Step 3: Add the new test method**

Add this test method to the class:

```java
@Test
void firstTickEmitsCorrectDeclineSignals() {
    orchestrator.gameTick();

    // pollAfter(channelId, afterId, limit): returns messages with id > afterId, excl. EVENTs.
    // afterId == 0 on first test → pass null (get all); afterId > 0 → get only delta.
    List<io.casehub.qhorus.runtime.message.Message> delta = messageService.pollAfter(
        dispatchBroker.channelId(),
        afterId > 0 ? afterId : null,
        20);

    List<io.casehub.qhorus.runtime.message.Message> commands =
        delta.stream().filter(m -> m.messageType == io.casehub.qhorus.api.message.MessageType.COMMAND).toList();
    List<io.casehub.qhorus.runtime.message.Message> dones =
        delta.stream().filter(m -> m.messageType == io.casehub.qhorus.api.message.MessageType.DONE).toList();
    List<io.casehub.qhorus.runtime.message.Message> declines =
        delta.stream().filter(m -> m.messageType == io.casehub.qhorus.api.message.MessageType.DECLINE).toList();

    // First-tick counts from spec first-tick state table:
    // - scouting.drools-cep: DONE (READY present, activateIf ctx.contains(READY) → true)
    // - economics.flow: DONE (READY present, activateIf default ctx→true)
    // - tactics.drools: OUT OF SCOPE (STRATEGY absent from caseData)
    // - strategy.drools: OUT OF SCOPE (ENEMY_ARMY_SIZE absent from caseData)
    // - strategy.early-pressure: DECLINE (READY present, isSelected → false)
    // - strategy.economic-expansion: DECLINE (READY present, isSelected → false)
    assertThat(commands).hasSize(4);
    assertThat(dones).hasSize(2);
    assertThat(declines).hasSize(2);

    assertThat(dones.stream().map(m -> m.sender).toList())
        .containsExactlyInAnyOrder("plugin:scouting.drools-cep", "plugin:economics.flow");

    assertThat(declines.stream().map(m -> m.sender).toList())
        .containsExactlyInAnyOrder("plugin:strategy.early-pressure", "plugin:strategy.economic-expansion");
}
```

- [ ] **Step 4: Add missing imports to the file**

Ensure these are present at the top of the file:
```java
import io.quarkmind.agent.PluginDispatchBroker;
import io.casehub.qhorus.runtime.message.MessageService;
import java.util.List;
```

- [ ] **Step 5: Run the integration test**

```
mvn test -Dtest=AdaptivePluginSelectionIT -q
```

Expected: `BUILD SUCCESS` — all tests including `firstTickEmitsCorrectDeclineSignals` pass.

If the test fails with wrong counts, check: (a) `StrategySelector` reset correctly on `startGame()`, (b) `priorActivation` cleared on `GameStarted`, (c) `translator.toMap()` keys.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/test/java/io/quarkmind/sc2/mock/AdaptivePluginSelectionIT.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "test(it): verify DECLINE signals in AdaptivePluginSelectionIT

Add firstTickEmitsCorrectDeclineSignals(): asserts 4 COMANDs, 2 DONEs
(scouting.drools-cep, economics.flow), 2 DECLINEs
(strategy.early-pressure, strategy.economic-expansion) on first tick.
Cursor isolation via dispatchBroker.lastDispatchedId() in @BeforeEach.

Refs #199"
```

---

## Task 7: ARC42STORIES.MD — mark QG2 resolved

**Files:**
- Modify: `ARC42STORIES.MD`

- [ ] **Step 1: Update C2 known limitations entry**

Find this line in `ARC42STORIES.MD` (around the C2 chapter section):
```
- Quality Goal 2 (Formal DECLINE) not closed: DECLINE speech act is platform-defined in casehub-qhorus but not wired in QuarkMind game-loop dispatch (#199)
```

Replace with:
```
- Quality Goal 2 (Formal DECLINE) closed (#199): `PluginDispatchBroker` emits COMMAND+DECLINE on `quarkmind-plugin-dispatch` when a plugin's `activateIf()` gate returns false (requires() keys present); COMMAND+DONE when active. Phase 1: DONE is pre-execution; Phase 2 `SequenceWorker` (engine#484) corrects this.
```

- [ ] **Step 2: Update L2/L3 gap table row**

Find this line (around line 834):
```
| Formal out-of-scope signal | Silent no-op or throws | Silent no-op or throws | Typed inter-plugin channels (L3); DECLINE speech act is platform-defined, not yet wired in QuarkMind's game-loop dispatch (#199) |
```

Replace `not yet wired` with `wired via PluginDispatchBroker (#199)`:
```
| Formal out-of-scope signal | Silent no-op or throws | Silent no-op or throws | Typed inter-plugin channels (L3); DECLINE speech act wired via `PluginDispatchBroker` on `quarkmind-plugin-dispatch` channel (#199) |
```

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "docs(arc42): mark QG2 (Formal DECLINE) resolved — Closes #199

C2 known limitations updated: PluginDispatchBroker wires DECLINE on
quarkmind-plugin-dispatch channel; Phase 1 DONE semantic noted.
L2/L3 gap table row updated.

Closes #199"
```

---

## Task 8: Final verification

- [ ] **Step 1: Run full test suite**

```
mvn test -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 2: Push branch**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind push -u origin issue-200-docs-and-decline
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| `MapCaseContext` with `contains()`, `get()`, `getOrDefault()`, `getList()`, write UOE | Task 1 |
| `MapCaseContextTest` — unit tests | Task 1 |
| `PluginDispatchBroker.CHANNEL_NAME` constant | Task 3 |
| `@Inject` + package-private test constructors | Task 3 |
| `@PostConstruct` channel setup with `QuarkusTransaction.requiringNew()`, `findByName()` first | Task 3 |
| State fields: `priorActivation = new ConcurrentHashMap<>()`, `lastDispatchedId = 0L` | Task 3 |
| `channelId()` and `lastDispatchedId()` accessors | Task 3 |
| `@Observes GameStarted` → `priorActivation.clear()` (not `lastDispatchedId`) | Task 3 |
| `@Transactional recordTick()` — collect-then-apply with `toRemove`, `toUpdate`, `lastReplyId` | Task 3 |
| `activateIf().test(evalCtx)` not `testActivation()` after `inScope` guard | Task 3 |
| `sendCommitmentSignal()` returns `Long`; COMMAND with `"plugin:"` target; DONE/DECLINE with `inReplyTo` | Task 3 |
| `PluginDispatchBrokerTest` — all cases including rollback simulation | Task 2+3 |
| `TickTimings` add `brokerMs`, update `totalMs()` | Task 4 |
| `GameTickExecutor` — inject broker, `recordTick()` before engine, `t1b` timing, 4-arg TickTimings constructor | Task 4 |
| `GameLoopBenchmarkTest` — `brokerMs[]`, loop read, `formatReport()` signature, "commit signals" row | Task 5 |
| `AdaptivePluginSelectionIT` — cursor in `@BeforeEach`, `firstTickEmitsCorrectDeclineSignals()` | Task 6 |
| `ARC42STORIES.MD` — C2 known limitations + gap table row | Task 7 |

All spec requirements covered. No gaps found.

**Placeholder scan:** No TBDs, no "similar to Task N" references, all code blocks are complete.

**Type consistency:** `PluginDispatchBroker` created in Task 3, used in Task 4 (`@Inject PluginDispatchBroker`) and Task 6 (`@Inject PluginDispatchBroker`) — name consistent throughout.
