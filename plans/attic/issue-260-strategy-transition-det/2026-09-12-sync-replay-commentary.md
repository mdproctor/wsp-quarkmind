# Synchronized Replay-Commentary Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #290 — Synchronized replay-commentary model (replay pauses for LLM)
**Issue group:** #290

**Goal:** Make replay commentary synchronous — the game loop pauses at commentary-worthy moments, waits for the LLM response, then resumes — so commentary is never stale regardless of replay speed.

**Architecture:** Extend `InlineCommentaryDispatcher` with a `CommentaryType`-aware API and a synchronous `executeWithTimeout()` method. Wire the unused `speedMultiplier` into `GameTickExecutor` as a multi-tick loop. Add config properties for sync mode (`full`/`reactive-only`/`none`) defaulting to `full` in `%replay`. Move narrative commentary from the broken `caseHub.signal()` path to the inline dispatcher.

**Tech Stack:** Java 21 (virtual threads), Quarkus CDI, Mockito, AssertJ

## Global Constraints

- Replay-only sync — live game modes (`sc2`, `emulated`, `mock`) must not be affected (D1)
- `ConcurrentExecution.SKIP` on `gameTick()` prevents overlapping ticks — do not add manual locking
- `gameActive` must never be written by `gameTick()` — protocol PP-20260612-afe621
- CDI observers on game lifecycle must remain `@Observes` (synchronous) — protocol PP-20260610-88dbbd
- No `@QuarkusTest` for tests that can be plain JUnit

---

## Batch 1: InlineCommentaryDispatcher — narrative support + sync API

### Task 1: Extend InlineCommentaryDispatcher with CommentaryType and sync

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/commentary/InlineCommentaryDispatcher.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/commentary/InlineCommentaryDispatcherTest.java` (new)

**Interfaces:**
- Consumes: `CommentaryWorkerFactory.buildReactiveSystemPrompt(AgentDescriptor)`, `CommentaryWorkerFactory.buildReactiveUserMessage(Map)`, `CommentaryWorkerFactory.buildNarrativeSystemPrompt(AgentDescriptor)`, `CommentaryWorkerFactory.buildNarrativeUserMessage(Map)`, `CommentaryType` enum, `CommentaryCompleted` record, `QuarkMindAgentRegistrar.descriptors()`, `QuarkMindCaseFile.COMMENTARY_TRIGGER`, `QuarkMindCaseFile.COMMENTARY_NARRATIVE_TRIGGER`
- Produces: `InlineCommentaryDispatcher.executeAsync(Map<String, Object>, CommentaryType)`, `InlineCommentaryDispatcher.executeWithTimeout(Map<String, Object>, CommentaryType, int)`, `InlineCommentaryDispatcher.isAvailable()`, `InlineCommentaryDispatcher.executeAsync(Map<String, Object>)` (backward compat)

- [ ] **Step 1: Write the failing test — narrative dispatch uses narrative prompts**

Create test file `quarkmind-sc2/src/test/java/io/quarkmind/plugin/commentary/InlineCommentaryDispatcherTest.java`:

```java
package io.quarkmind.plugin.commentary;

import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import io.casehub.eidos.api.AgentDescriptor;
import io.quarkmind.agent.QuarkMindCaseFile;
import jakarta.enterprise.event.Event;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class InlineCommentaryDispatcherTest {

    private ChatModel chatModel;
    private Event<CommentaryCompleted> completedEvent;
    private AgentDescriptor reactiveDescriptor;
    private AgentDescriptor narrativeDescriptor;
    private InlineCommentaryDispatcher dispatcher;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() {
        chatModel = mock(ChatModel.class);
        completedEvent = mock(Event.class);
        reactiveDescriptor = stubDescriptor("commentator-energetic", "commentary-reactive");
        narrativeDescriptor = stubDescriptor("narrator-dramatic", "commentary-narrative");

        ChatResponse response = mock(ChatResponse.class);
        when(response.aiMessage()).thenReturn(new AiMessage("test commentary"));
        when(chatModel.chat(any(ChatRequest.class))).thenReturn(response);

        dispatcher = new InlineCommentaryDispatcher(
            chatModel, reactiveDescriptor, narrativeDescriptor, completedEvent);
    }

    @Test
    void execute_narrativeType_usesNarrativePrompts() {
        Map<String, Object> triggers = Map.of(
            QuarkMindCaseFile.COMMENTARY_NARRATIVE_TRIGGER,
            Map.of("gameFrame", 1000L, "moments", "test"));

        dispatcher.execute(triggers, CommentaryType.NARRATIVE);

        ArgumentCaptor<CommentaryCompleted> captor = ArgumentCaptor.forClass(CommentaryCompleted.class);
        verify(completedEvent).fire(captor.capture());
        assertThat(captor.getValue().commentaryType()).isEqualTo(CommentaryType.NARRATIVE);
        assertThat(captor.getValue().capability()).isEqualTo("commentary-narrative");
        assertThat(captor.getValue().workerId()).isEqualTo("narrator-dramatic");
    }

    @Test
    void execute_reactiveType_usesReactivePrompts() {
        Map<String, Object> triggers = Map.of(
            QuarkMindCaseFile.COMMENTARY_TRIGGER,
            Map.of("gameFrame", 500L, "momentTypes", "FIRST_CONTACT"));

        dispatcher.execute(triggers, CommentaryType.REACTIVE);

        ArgumentCaptor<CommentaryCompleted> captor = ArgumentCaptor.forClass(CommentaryCompleted.class);
        verify(completedEvent).fire(captor.capture());
        assertThat(captor.getValue().commentaryType()).isEqualTo(CommentaryType.REACTIVE);
        assertThat(captor.getValue().capability()).isEqualTo("commentary-reactive");
    }

    @Test
    void executeWithTimeout_completesWithinTimeout() {
        Map<String, Object> triggers = Map.of(
            QuarkMindCaseFile.COMMENTARY_TRIGGER,
            Map.of("gameFrame", 200L, "momentTypes", "BATTLE_ENDED"));

        dispatcher.executeWithTimeout(triggers, CommentaryType.REACTIVE, 5);

        verify(completedEvent).fire(any(CommentaryCompleted.class));
    }

    @Test
    void executeWithTimeout_timeoutSkips() {
        when(chatModel.chat(any(ChatRequest.class))).thenAnswer(inv -> {
            Thread.sleep(5000);
            return null;
        });

        Map<String, Object> triggers = Map.of(
            QuarkMindCaseFile.COMMENTARY_TRIGGER,
            Map.of("gameFrame", 300L, "momentTypes", "FIRST_CONTACT"));

        dispatcher.executeWithTimeout(triggers, CommentaryType.REACTIVE, 1);

        verify(completedEvent, never()).fire(any());
    }

    @Test
    void executeAsync_backwardCompat_singleArg() {
        Map<String, Object> triggers = Map.of(
            QuarkMindCaseFile.COMMENTARY_TRIGGER,
            Map.of("gameFrame", 100L, "momentTypes", "ARMY_SHIFT"));

        dispatcher.executeAsync(triggers);

        // Wait briefly for virtual thread to complete
        try { Thread.sleep(500); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        verify(completedEvent).fire(any(CommentaryCompleted.class));
    }

    private static AgentDescriptor stubDescriptor(String agentId, String capabilityName) {
        var cap = mock(io.casehub.eidos.api.Capability.class);
        when(cap.name()).thenReturn(capabilityName);
        var desc = mock(AgentDescriptor.class);
        when(desc.agentId()).thenReturn(agentId);
        when(desc.capabilities()).thenReturn(java.util.List.of(cap));
        when(desc.disposition()).thenReturn(null);
        return desc;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=InlineCommentaryDispatcherTest -q`
Expected: FAIL — constructor `InlineCommentaryDispatcher(ChatModel, AgentDescriptor, AgentDescriptor, Event)` does not exist yet

- [ ] **Step 3: Implement InlineCommentaryDispatcher changes**

Rewrite `InlineCommentaryDispatcher.java` to:
1. Add a `narrativeDescriptor` field
2. Add a test-friendly constructor: `InlineCommentaryDispatcher(ChatModel, AgentDescriptor, AgentDescriptor, Event<CommentaryCompleted>)`
3. Refactor `execute(Map, CommentaryType)` to select descriptor and prompts based on type
4. Add `executeWithTimeout(Map, CommentaryType, int)` using `CompletableFuture.runAsync` + virtual thread + `.get(timeout, SECONDS)`
5. Add `executeAsync(Map, CommentaryType)` overload
6. Keep backward-compat `executeAsync(Map)` single-arg delegating to `executeAsync(triggers, CommentaryType.REACTIVE)`
7. Update `extractGameFrame` to use the correct trigger key based on `CommentaryType`

The CDI constructor continues to resolve descriptors from the registrar. The test constructor bypasses CDI for plain JUnit testing.

```java
package io.quarkmind.plugin.commentary;

import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import io.casehub.eidos.api.AgentDescriptor;
import io.quarkmind.agent.QuarkMindCaseFile;
import io.quarkmind.plugin.advisory.QuarkMindAgentRegistrar;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.Map;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;

@ApplicationScoped
public class InlineCommentaryDispatcher {

    private static final Logger log = Logger.getLogger(InlineCommentaryDispatcher.class);

    private final ChatModel chatModel;
    private final AgentDescriptor reactiveDescriptor;
    private final AgentDescriptor narrativeDescriptor;
    private final Event<CommentaryCompleted> completedEvent;
    private final boolean available;

    @Inject
    InlineCommentaryDispatcher(Instance<ChatModel> chatModelInstance,
                               Instance<QuarkMindAgentRegistrar> registrarInstance,
                               Event<CommentaryCompleted> completedEvent) {
        this.completedEvent = completedEvent;
        if (!chatModelInstance.isResolvable() || !registrarInstance.isResolvable()) {
            this.chatModel = null;
            this.reactiveDescriptor = null;
            this.narrativeDescriptor = null;
            this.available = false;
            return;
        }
        this.chatModel = chatModelInstance.get();
        var descriptors = registrarInstance.get().descriptors();
        this.reactiveDescriptor = findDescriptor(descriptors, "commentary-reactive");
        this.narrativeDescriptor = findDescriptor(descriptors, "commentary-narrative");
        this.available = reactiveDescriptor != null;
        if (available) {
            log.infof("[INLINE-COMMENTARY] Ready — reactive: %s, narrative: %s",
                reactiveDescriptor.agentId(),
                narrativeDescriptor != null ? narrativeDescriptor.agentId() : "none");
        }
    }

    InlineCommentaryDispatcher(ChatModel chatModel,
                               AgentDescriptor reactiveDescriptor,
                               AgentDescriptor narrativeDescriptor,
                               Event<CommentaryCompleted> completedEvent) {
        this.chatModel = chatModel;
        this.reactiveDescriptor = reactiveDescriptor;
        this.narrativeDescriptor = narrativeDescriptor;
        this.completedEvent = completedEvent;
        this.available = reactiveDescriptor != null;
    }

    public boolean isAvailable() { return available; }

    public void executeAsync(Map<String, Object> triggers) {
        executeAsync(triggers, CommentaryType.REACTIVE);
    }

    public void executeAsync(Map<String, Object> triggers, CommentaryType type) {
        if (!available) return;
        Thread.startVirtualThread(() -> {
            try {
                execute(triggers, type);
            } catch (Exception e) {
                log.warnf(e, "[INLINE-COMMENTARY] %s failed: %s", type, e.getMessage());
            }
        });
    }

    public void executeWithTimeout(Map<String, Object> triggers,
                                    CommentaryType type, int timeoutSeconds) {
        if (!available) return;
        try {
            CompletableFuture<Void> future = CompletableFuture.runAsync(
                () -> execute(triggers, type), Thread::startVirtualThread);
            future.get(timeoutSeconds, TimeUnit.SECONDS);
        } catch (TimeoutException e) {
            log.warnf("[INLINE-COMMENTARY] %s timed out after %ds — skipping",
                type, timeoutSeconds);
        } catch (Exception e) {
            log.warnf(e, "[INLINE-COMMENTARY] %s sync failed: %s", type, e.getMessage());
        }
    }

    void execute(Map<String, Object> triggers, CommentaryType type) {
        long startNanos = System.nanoTime();
        AgentDescriptor descriptor = type == CommentaryType.REACTIVE
            ? reactiveDescriptor : narrativeDescriptor;
        if (descriptor == null) return;

        String systemPrompt = type == CommentaryType.REACTIVE
            ? CommentaryWorkerFactory.buildReactiveSystemPrompt(descriptor)
            : CommentaryWorkerFactory.buildNarrativeSystemPrompt(descriptor);
        String userMessage = type == CommentaryType.REACTIVE
            ? CommentaryWorkerFactory.buildReactiveUserMessage(triggers)
            : CommentaryWorkerFactory.buildNarrativeUserMessage(triggers);

        ChatResponse response = chatModel.chat(ChatRequest.builder()
            .messages(new SystemMessage(systemPrompt), new UserMessage(userMessage))
            .build());

        String text = response.aiMessage().text();
        long latencyMs = (System.nanoTime() - startNanos) / 1_000_000;
        long gameFrame = extractGameFrame(triggers, type);

        String capability = type == CommentaryType.REACTIVE
            ? "commentary-reactive" : "commentary-narrative";
        completedEvent.fire(new CommentaryCompleted(
            descriptor.agentId(), capability,
            gameFrame, text != null ? text : "", type, latencyMs));

        log.infof("[INLINE-COMMENTARY] %s completed in %dms at frame %d: %s",
            type, latencyMs, gameFrame,
            text != null ? text.substring(0, Math.min(80, text.length())) : "");
    }

    private static long extractGameFrame(Map<String, Object> triggers, CommentaryType type) {
        String key = type == CommentaryType.REACTIVE
            ? QuarkMindCaseFile.COMMENTARY_TRIGGER
            : QuarkMindCaseFile.COMMENTARY_NARRATIVE_TRIGGER;
        Object trigger = triggers.get(key);
        if (trigger instanceof Map<?, ?> m) {
            Object frame = m.get("gameFrame");
            if (frame instanceof Number n) return n.longValue();
        }
        return 0L;
    }

    private static AgentDescriptor findDescriptor(
            java.util.List<AgentDescriptor> descriptors, String capabilityName) {
        return descriptors.stream()
            .filter(d -> d.capabilities().stream()
                .anyMatch(c -> c.name().equals(capabilityName)))
            .findFirst().orElse(null);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=InlineCommentaryDispatcherTest -q`
Expected: all 5 tests PASS

- [ ] **Step 5: Run full module tests to verify no regressions**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: PASS — existing tests unaffected (backward-compat `executeAsync(Map)` preserved)

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/plugin/commentary/InlineCommentaryDispatcher.java
git add quarkmind-sc2/src/test/java/io/quarkmind/plugin/commentary/InlineCommentaryDispatcherTest.java
git commit -m "feat: extend InlineCommentaryDispatcher with narrative + sync API

Add CommentaryType-aware execute, executeAsync, and executeWithTimeout.
Narrative uses buildNarrativeSystemPrompt/UserMessage from CommentaryWorkerFactory.
executeWithTimeout runs LLM on virtual thread with CompletableFuture.get(timeout).
Backward-compat executeAsync(Map) preserved.

Refs #290"
```

---

## Batch 2: Speed multiplier + sync mode in GameTickExecutor

### Task 2: Wire speed multiplier and add sync config to GameTickExecutor

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/GameTickExecutor.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/AgentOrchestrator.java`
- Modify: `quarkmind-sc2/src/main/resources/application.properties`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/agent/GameTickExecutorMigrationTest.java` (extend)

**Interfaces:**
- Consumes: `InlineCommentaryDispatcher.executeAsync(Map, CommentaryType)`, `InlineCommentaryDispatcher.executeWithTimeout(Map, CommentaryType, int)` from Task 1
- Produces: `GameTickExecutor.execute(int speed)` (pipeline with speed loop + sync), `GameTickExecutor.execute()` (backward-compat), `GameTickExecutor.setSyncMode(String)` (runtime override), `GameTickExecutor.getSyncMode()`, `GameTickExecutor.getSyncTimeoutSeconds()`

- [ ] **Step 1: Write the failing test — speed multiplier calls tick N times**

Add to `GameTickExecutorMigrationTest.java`:

```java
@Test
void execute_speedMultiplier_callsTickNTimes() {
    GameState state = stubGameState(42L, 200, 100);
    when(engine.observe()).thenReturn(state);
    when(caseHub.signalAndAwaitSync(any(), any(), any())).thenReturn(mock(CaseContext.class));

    executor.execute(4);

    verify(engine, times(4)).tick();
    verify(engine, times(1)).observe();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=GameTickExecutorMigrationTest#execute_speedMultiplier_callsTickNTimes -q`
Expected: FAIL — method `execute(int)` does not exist

- [ ] **Step 3: Write the failing test — sync mode blocks on commentary**

Add to `GameTickExecutorMigrationTest.java`:

1. Add field to the test class:
```java
private io.quarkmind.plugin.commentary.InlineCommentaryDispatcher inlineCommentaryDispatcher;
```

2. Add to `setUp()` after existing field assignments:
```java
inlineCommentaryDispatcher = mock(io.quarkmind.plugin.commentary.InlineCommentaryDispatcher.class);
when(inlineCommentaryDispatcher.isAvailable()).thenReturn(true);
executor.inlineCommentaryDispatcher = inlineCommentaryDispatcher;
executor.replaySyncMode = "none";
executor.replaySyncTimeoutSeconds = 15;
```

3. Add new tests:
@Test
void execute_syncModeFull_callsExecuteWithTimeout() {
    executor.replaySyncMode = "full";
    executor.replaySyncTimeoutSeconds = 15;
    GameState state = stubGameState(500L, 200, 100);
    when(engine.observe()).thenReturn(state);
    CaseContext ctx = mock(CaseContext.class);
    when(caseHub.signalAndAwaitSync(any(), any(), any())).thenReturn(ctx);

    Map<String, Object> reactiveMap = Map.of(
        QuarkMindCaseFile.COMMENTARY_TRIGGER,
        Map.of("gameFrame", 500L, "momentTypes", "FIRST_CONTACT"));
    when(commentaryTriggerBuilder.build(any(CaseContext.class), anyLong())).thenReturn(reactiveMap);

    executor.execute(1);

    verify(inlineCommentaryDispatcher).executeWithTimeout(
        eq(reactiveMap), eq(io.quarkmind.plugin.commentary.CommentaryType.REACTIVE), eq(15));
    verify(inlineCommentaryDispatcher, never()).executeAsync(any(), any());
}

@Test
void execute_syncModeNone_callsExecuteAsync() {
    executor.replaySyncMode = "none";
    GameState state = stubGameState(500L, 200, 100);
    when(engine.observe()).thenReturn(state);
    CaseContext ctx = mock(CaseContext.class);
    when(caseHub.signalAndAwaitSync(any(), any(), any())).thenReturn(ctx);

    Map<String, Object> reactiveMap = Map.of(
        QuarkMindCaseFile.COMMENTARY_TRIGGER,
        Map.of("gameFrame", 500L, "momentTypes", "FIRST_CONTACT"));
    when(commentaryTriggerBuilder.build(any(CaseContext.class), anyLong())).thenReturn(reactiveMap);

    executor.execute(1);

    verify(inlineCommentaryDispatcher).executeAsync(eq(reactiveMap), eq(io.quarkmind.plugin.commentary.CommentaryType.REACTIVE));
    verify(inlineCommentaryDispatcher, never()).executeWithTimeout(any(), any(), anyInt());
}

@Test
void execute_syncModeFull_narrativeUsesInlineDispatcher() {
    executor.replaySyncMode = "full";
    executor.replaySyncTimeoutSeconds = 15;
    GameState state = stubGameState(2000L, 300, 100);
    when(engine.observe()).thenReturn(state);
    when(caseHub.signalAndAwaitSync(any(), any(), any())).thenReturn(mock(CaseContext.class));

    Map<String, Object> narrativeMap = Map.of(
        QuarkMindCaseFile.COMMENTARY_NARRATIVE_TRIGGER,
        Map.of("gameFrame", 2000L, "moments", "accumulated"));
    when(commentaryAccumulator.tick(anyLong())).thenReturn(narrativeMap);

    executor.execute(1);

    verify(inlineCommentaryDispatcher).executeWithTimeout(
        eq(narrativeMap), eq(io.quarkmind.plugin.commentary.CommentaryType.NARRATIVE), eq(15));
    verify(caseHub, never()).signal(any(), eq(narrativeMap));
}

@Test
void execute_syncModeReactiveOnly_narrativeIsAsync() {
    executor.replaySyncMode = "reactive-only";
    executor.replaySyncTimeoutSeconds = 15;
    GameState state = stubGameState(2000L, 300, 100);
    when(engine.observe()).thenReturn(state);
    when(caseHub.signalAndAwaitSync(any(), any(), any())).thenReturn(mock(CaseContext.class));

    Map<String, Object> narrativeMap = Map.of(
        QuarkMindCaseFile.COMMENTARY_NARRATIVE_TRIGGER,
        Map.of("gameFrame", 2000L, "moments", "accumulated"));
    when(commentaryAccumulator.tick(anyLong())).thenReturn(narrativeMap);

    Map<String, Object> reactiveMap = Map.of(
        QuarkMindCaseFile.COMMENTARY_TRIGGER,
        Map.of("gameFrame", 2000L, "momentTypes", "BATTLE_ENDED"));
    when(commentaryTriggerBuilder.build(any(CaseContext.class), anyLong())).thenReturn(reactiveMap);

    executor.execute(1);

    // Reactive is sync
    verify(inlineCommentaryDispatcher).executeWithTimeout(
        eq(reactiveMap), eq(io.quarkmind.plugin.commentary.CommentaryType.REACTIVE), eq(15));
    // Narrative is async
    verify(inlineCommentaryDispatcher).executeAsync(
        eq(narrativeMap), eq(io.quarkmind.plugin.commentary.CommentaryType.NARRATIVE));
}
```

- [ ] **Step 4: Implement GameTickExecutor changes**

Modify `GameTickExecutor.java`:

1. Add config fields:
```java
@ConfigProperty(name = "quarkmind.replay.sync.mode", defaultValue = "none")
String replaySyncMode;

@ConfigProperty(name = "quarkmind.replay.sync.timeout-seconds", defaultValue = "15")
int replaySyncTimeoutSeconds;
```

2. Add `execute(int speed)`:
```java
AgentOrchestrator.TickResult execute(int speed) {
    long t0 = System.currentTimeMillis();
    for (int i = 0; i < speed - 1; i++) {
        engine.tick();
    }
    engine.tick();
    var gameState = engine.observe();
    timelineSampler.tick(gameState);
    long t1 = System.currentTimeMillis();

    Map<String, Object> caseData = translator.toMap(gameState);
    caseData = new HashMap<>(caseData);
    caseData.put(QuarkMindCaseFile.GAME_MODE, gameMode);
    try { pluginDispatchBroker.recordTick(caseData); } catch (Exception | Error e) { log.debugf("Broker recordTick skipped: %s", e.getMessage()); }
    long t1b = System.currentTimeMillis();

    CaseContext ctx = null;
    try {
        ctx = caseHub.signalAndAwaitSync(gameSession.id(), caseData, TICK_TIMEOUT);
    } catch (Exception | Error e) {
        log.warnf("Engine signal+settle failed at frame %d: %s",
                   gameState.gameFrame(), e.getMessage());
    }
    long t2 = System.currentTimeMillis();

    narrativeContextHolder.updateCbr(ctx);
    summarisationLifecycle.tick(gameState.gameFrame());
    Map<String, Object> narrativeTriggers = commentaryAccumulator.tick(gameState.gameFrame());

    boolean isCoachMode = "coach".equals(gameMode);

    if (!isCoachMode) {
        milestoneOutcomeRecorder.evaluateMilestones(gameState);
        advisoryMilestoneOutcomeRecorder.evaluateMilestones(gameState);
    }

    if (ctx != null && !isCoachMode) {
        deferredAdvisoryEvaluator.evaluate(ctx, gameState.gameFrame());
    }

    if (isCoachMode) {
        coachingComplianceEvaluator.evaluate(gameState, gameState.gameFrame());
    }

    // Reactive commentary — sync or async based on mode
    if (ctx != null) {
        Map<String, Object> reactiveTriggers = commentaryTriggerBuilder.build(ctx, gameState.gameFrame());
        if (!reactiveTriggers.isEmpty() && inlineCommentaryDispatcher.isAvailable()) {
            if (syncReactive()) {
                inlineCommentaryDispatcher.executeWithTimeout(
                    reactiveTriggers, CommentaryType.REACTIVE, replaySyncTimeoutSeconds);
            } else {
                inlineCommentaryDispatcher.executeAsync(reactiveTriggers, CommentaryType.REACTIVE);
            }
        }
    }

    // Narrative commentary — inline dispatch (replaces caseHub.signal path)
    if (!narrativeTriggers.isEmpty() && inlineCommentaryDispatcher.isAvailable()) {
        if (syncNarrative()) {
            inlineCommentaryDispatcher.executeWithTimeout(
                narrativeTriggers, CommentaryType.NARRATIVE, replaySyncTimeoutSeconds);
        } else {
            inlineCommentaryDispatcher.executeAsync(narrativeTriggers, CommentaryType.NARRATIVE);
        }
    }

    // Advisory trigger: fire-and-forget signal — only in AI mode
    if (ctx != null && !isCoachMode) {
        Map<String, Object> triggers = AdvisoryTriggerBuilder.buildTriggers(ctx, gameState.gameFrame());
        if (!triggers.isEmpty()) {
            try {
                caseHub.signal(gameSession.id(), triggers);
            } catch (Exception ex) {
                log.warnf("Advisory signal failed at frame %d: %s", gameState.gameFrame(), ex.getMessage());
            }
        }
    }

    // Coaching trigger: fire-and-forget signal — only in coach mode
    if (ctx != null && isCoachMode) {
        Map<String, Object> coachingTriggers = coachingTriggerBuilder.build(ctx, gameState.gameFrame());
        if (!coachingTriggers.isEmpty()) {
            try {
                caseHub.signal(gameSession.id(), coachingTriggers);
            } catch (Exception ex) {
                log.warnf("Coaching trigger failed at frame %d: %s", gameState.gameFrame(), ex.getMessage());
            }
        }
    }

    engine.dispatch();
    long t3 = System.currentTimeMillis();

    var timings = new AgentOrchestrator.TickTimings(t1 - t0, t2 - t1b, t3 - t2, t1b - t1);
    log.debugf("Tick %d — physics=%dms broker=%dms plugins=%dms dispatch=%dms total=%dms | minerals=%d supply=%d/%d",
               gameState.gameFrame(), timings.physicsMs(), timings.brokerMs(),
               timings.pluginsMs(), timings.dispatchMs(), timings.totalMs(),
               gameState.minerals(), gameState.supplyUsed(), gameState.supply());
    return new AgentOrchestrator.TickResult(ctx, timings);
}

AgentOrchestrator.TickResult execute() { return execute(1); }

private boolean syncReactive() {
    return "full".equals(replaySyncMode) || "reactive-only".equals(replaySyncMode);
}

private boolean syncNarrative() {
    return "full".equals(replaySyncMode);
}
```

3. Update `AgentOrchestrator.gameTick()` to pass speed:

Replace `lastTickResult.set(tickExecutor.execute())` with `lastTickResult.set(tickExecutor.execute(speedMultiplier))`.

4. Add config to `application.properties`:

```properties
# Replay sync mode (default: none; %replay overrides to full)
quarkmind.replay.sync.mode=none
%replay.quarkmind.replay.sync.mode=full
quarkmind.replay.sync.timeout-seconds=15
quarkmind.replay.observation.resolution=auto
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=GameTickExecutorMigrationTest -q`
Expected: all existing + 5 new tests PASS

- [ ] **Step 6: Run full module tests**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/agent/GameTickExecutor.java
git add quarkmind-sc2/src/main/java/io/quarkmind/agent/AgentOrchestrator.java
git add quarkmind-sc2/src/main/resources/application.properties
git add quarkmind-sc2/src/test/java/io/quarkmind/agent/GameTickExecutorMigrationTest.java
git commit -m "feat: wire speed multiplier + replay sync mode in GameTickExecutor

Speed loop: execute(int speed) calls engine.tick() N times, observes once.
AgentOrchestrator.gameTick() passes speedMultiplier to execute().
Sync mode: FULL blocks on reactive+narrative, REACTIVE_ONLY blocks reactive
only, NONE preserves fire-and-forget (default for non-replay profiles).
Narrative commentary moved from caseHub.signal() to InlineCommentaryDispatcher.
Config: quarkmind.replay.sync.mode (none|full|reactive-only),
        quarkmind.replay.sync.timeout-seconds (15).

Refs #290"
```

---

## Batch 3: QA endpoint + runtime override

### Task 3: Add sync mode QA endpoint and runtime override

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/qa/ReplayControlsResource.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/GameTickExecutor.java` (add setter/getter for runtime override)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/agent/GameTickExecutorMigrationTest.java` (extend)

**Interfaces:**
- Consumes: `GameTickExecutor.getSyncMode()`, `GameTickExecutor.setSyncMode(String)`, `GameTickExecutor.getSyncTimeoutSeconds()`
- Produces: `GET /qa/replay/sync` → `ReplaySyncResponse(String mode, int timeoutSeconds)`, `POST /qa/replay/sync?mode=<value>`

- [ ] **Step 1: Write the failing test — runtime sync mode override**

Add to `GameTickExecutorMigrationTest.java`:

```java
@Test
void setSyncMode_overridesConfigProperty() {
    executor.replaySyncMode = "none";
    executor.setSyncMode("full");
    assertThat(executor.getSyncMode()).isEqualTo("full");

    // Verify behavior uses the overridden mode
    GameState state = stubGameState(500L, 200, 100);
    when(engine.observe()).thenReturn(state);
    CaseContext ctx = mock(CaseContext.class);
    when(caseHub.signalAndAwaitSync(any(), any(), any())).thenReturn(ctx);

    Map<String, Object> reactiveMap = Map.of(
        QuarkMindCaseFile.COMMENTARY_TRIGGER,
        Map.of("gameFrame", 500L, "momentTypes", "FIRST_CONTACT"));
    when(commentaryTriggerBuilder.build(any(CaseContext.class), anyLong())).thenReturn(reactiveMap);

    executor.execute(1);

    verify(inlineCommentaryDispatcher).executeWithTimeout(
        eq(reactiveMap), eq(io.quarkmind.plugin.commentary.CommentaryType.REACTIVE), eq(15));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=GameTickExecutorMigrationTest#setSyncMode_overridesConfigProperty -q`
Expected: FAIL — `setSyncMode` does not exist

- [ ] **Step 3: Implement runtime override in GameTickExecutor**

Add to `GameTickExecutor.java`:

```java
private volatile String syncModeOverride;

public void setSyncMode(String mode) {
    this.syncModeOverride = mode;
}

public String getSyncMode() {
    String override = syncModeOverride;
    return override != null ? override : replaySyncMode;
}

public int getSyncTimeoutSeconds() {
    return replaySyncTimeoutSeconds;
}
```

Update `syncReactive()` and `syncNarrative()` to use `getSyncMode()`:

```java
private boolean syncReactive() {
    String mode = getSyncMode();
    return "full".equals(mode) || "reactive-only".equals(mode);
}
private boolean syncNarrative() {
    return "full".equals(getSyncMode());
}
```

- [ ] **Step 4: Implement QA endpoint**

Add to `ReplayControlsResource.java`:

```java
@Inject GameTickExecutor tickExecutor;

@GET @Path("/sync") @Produces(MediaType.APPLICATION_JSON)
public Response syncStatus() {
    return Response.ok(new ReplaySyncResponse(
        tickExecutor.getSyncMode(), tickExecutor.getSyncTimeoutSeconds()
    )).build();
}

@POST @Path("/sync")
public Response setSyncMode(@QueryParam("mode") String mode) {
    if (mode == null || (!mode.equals("full") && !mode.equals("reactive-only") && !mode.equals("none"))) {
        return Response.status(Response.Status.BAD_REQUEST)
            .entity("mode must be full, reactive-only, or none").build();
    }
    tickExecutor.setSyncMode(mode);
    return Response.noContent().build();
}

record ReplaySyncResponse(String mode, int timeoutSeconds) {}
```

- [ ] **Step 5: Run tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=GameTickExecutorMigrationTest -q`
Expected: PASS

- [ ] **Step 6: Run full module tests**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/agent/GameTickExecutor.java
git add quarkmind-sc2/src/main/java/io/quarkmind/qa/ReplayControlsResource.java
git add quarkmind-sc2/src/test/java/io/quarkmind/agent/GameTickExecutorMigrationTest.java
git commit -m "feat: add /qa/replay/sync endpoint for runtime sync mode control

GET /qa/replay/sync returns current mode + timeout.
POST /qa/replay/sync?mode=full|reactive-only|none overrides config.
GameTickExecutor.setSyncMode() uses volatile override field.

Refs #290"
```

---

## References

- `specs/issue-290-sync-replay-commentary/2026-09-12-sync-replay-commentary-design.md` — design spec
- `AgentOrchestrator.java:112` — @Scheduled with ConcurrentExecution.SKIP
- `GameTickExecutor.java:67-169` — tick pipeline
- `InlineCommentaryDispatcher.java:28-106` — existing reactive-only inline dispatch
- `CommentaryWorkerFactory.java:190-349` — reactive + narrative prompt builders
- `ReplayControlsResource.java:18-116` — existing QA endpoints
- `GameTickExecutorMigrationTest.java:49-310` — existing test patterns
- `QuarkMindCaseFile.java:44-45` — COMMENTARY_TRIGGER + COMMENTARY_NARRATIVE_TRIGGER keys
- Protocol PP-20260612-afe621 — gameTick must not write gameActive
- Protocol PP-20260610-88dbbd — game lifecycle observer synchrony
- GitHub #290 — Synchronized replay-commentary model
