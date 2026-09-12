# Synchronized Replay-Commentary Model

**Issue:** casehubio/quarkmind#290
**Date:** 2026-09-12
**Decisions:** D1–D7 in `decisions.md`

## 1. Overview

When running replays at >1x speed, commentary lags behind game state because the LLM takes wall-clock time (1–5 seconds) to generate. At 4x speed, a reactive commentary response arrives 8–20 game-frames after the triggering moment — the battle may be over by the time the narrator reacts.

This spec adds a synchronized model where the replay engine pauses at commentary-worthy moments, waits for the LLM response, then resumes. Commentary always appears at the right game frame regardless of replay speed.

**What changes:**
- Speed multiplier wired into the tick loop (currently dead code)
- Observation resolution auto-scales with speed
- `InlineCommentaryDispatcher` extended for narrative commentary (currently only reactive)
- `GameTickExecutor` gains replay-sync mode: blocks on LLM response before returning
- New configuration properties for sync mode, timeout, and resolution

**What doesn't change:**
- Live game modes (sc2, emulated, mock) — fire-and-forget async dispatch unchanged
- Commentary pipeline (triggers, accumulator, channel broker, workbench surfacing)
- Trust scoring infrastructure
- ReplayEngine's observe-only contract (intents drained but not applied)

**Scope constraint:** Replay mode only (D1). The `%replay` profile enables sync; all other profiles default to `NONE` (current behavior).

## 2. Speed Multiplier Wiring

The `speedMultiplier` field in `AgentOrchestrator` (line 47) has a getter, setter, and QA endpoint (`/qa/replay/speed`) but is never read by `gameTick()`. Speed control is dead code.

### Change to AgentOrchestrator.gameTick()

Pass the speed multiplier to `GameTickExecutor`:

```java
@Scheduled(every = "${starcraft.tick.interval:500ms}",
           concurrentExecution = Scheduled.ConcurrentExecution.SKIP)
public void gameTick() {
    if (schedulerPaused) return;
    if (engine.isConnected()) {
        engineWasConnected = true;
        lastTickResult.set(tickExecutor.execute(speedMultiplier));
    } else if (engineWasConnected) {
        engineWasConnected = false;
        fireGameStoppedOnce(engine.lastOutcome());
    }
}
```

### Change to GameTickExecutor.execute()

Accept speed parameter. Advance the engine by `speed` game loops, observe only the final state:

```java
TickResult execute(int speed) {
    long t0 = System.currentTimeMillis();
    for (int i = 0; i < speed - 1; i++) {
        engine.tick();
    }
    engine.tick();
    var gameState = engine.observe();
    // ... rest of pipeline unchanged
}
```

The default `speed = 1` preserves current behavior. At speed N, `engine.tick()` is called N times (cheap for ReplayEngine — pointer advancement through pre-parsed tracker events) but the full pipeline (Drools, CaseHub signal+settle) runs once on the final observed state.

### Backward compatibility

The existing zero-arg `execute()` delegates to `execute(1)`:

```java
TickResult execute() { return execute(1); }
```

This preserves the call site in tests (`GameTickExecutorMigrationTest`, `GameLoopBenchmarkTest`).

## 3. Observation Resolution

At speed N, the pipeline sees deltas across N game loops instead of 1. This is the "6fps video" model — the agent observes at a lower framerate but still comprehends the game.

**Why this is safe:** Moment detection cooldowns (110+ frames for reactive, 672+ frames for narrative) dwarf the sampling gap. At 8x speed, 8 frames = 0.36 game-seconds. No moment type has a lifecycle shorter than that.

**Auto-scale formula:** `resolution = max(1, speed)`. One pipeline execution per scheduler fire regardless of speed.

**Override:** `quarkmind.replay.observation.resolution` accepts `auto` (default), or a fixed integer (1, 2, 4, 8). When set to a fixed value, that many ticks are processed per scheduler fire regardless of speed. This lets a user run at 4x but observe every 2nd loop (2 pipeline executions per fire) for higher fidelity at the cost of more CPU.

Implementation: when resolution is explicitly set and differs from speed, the loop becomes:

```java
int resolution = resolveResolution(speed);
int totalTicks = speed;
int observed = 0;
for (int i = 0; i < totalTicks; i++) {
    engine.tick();
    if ((i + 1) % resolution == 0 || i == totalTicks - 1) {
        var gameState = engine.observe();
        // run pipeline on this observation
        observed++;
    }
}
```

For the default `auto` case, this simplifies to the speed-loop-then-observe pattern in §2 (resolution = speed, so only the final tick is observed).

**Scope note:** The multi-observation loop (resolution < speed) is deferred to a follow-up. For #290, only the `auto` path is implemented — `resolution = speed`, one observation per scheduler fire. The config property is declared and parsed but only `auto` and values equal to speed are honored; other values are clamped to `auto` with a log warning.

## 4. Synchronized Commentary — GameTickExecutor Changes

### Sync mode detection

A new config property determines sync behavior:

```java
@ConfigProperty(name = "quarkmind.replay.sync.mode", defaultValue = "none")
String replaySyncMode;
```

Values: `full`, `reactive-only`, `none`. Profile-specific defaults in `application.properties`:

```properties
%replay.quarkmind.replay.sync.mode=full
```

All other profiles inherit the global default `none`.

### Sync timeout

```java
@ConfigProperty(name = "quarkmind.replay.sync.timeout-seconds", defaultValue = "15")
int replaySyncTimeoutSeconds;
```

### Execute with sync

When sync mode is not `none`, `GameTickExecutor.execute()` calls commentary dispatchers synchronously and blocks until the LLM responds (or timeout):

```java
// Reactive commentary — sync or async based on mode
Map<String, Object> reactiveTriggers = Map.of();
if (ctx != null) {
    reactiveTriggers = commentaryTriggerBuilder.build(ctx, gameState.gameFrame());
    if (!reactiveTriggers.isEmpty() && inlineCommentaryDispatcher.isAvailable()) {
        if (syncReactive()) {
            inlineCommentaryDispatcher.executeWithTimeout(
                reactiveTriggers, CommentaryType.REACTIVE, replaySyncTimeoutSeconds);
        } else {
            inlineCommentaryDispatcher.executeAsync(reactiveTriggers, CommentaryType.REACTIVE);
        }
    }
}

// Narrative commentary — sync or async based on mode
if (!narrativeTriggers.isEmpty() && inlineCommentaryDispatcher.isAvailable()) {
    if (syncNarrative()) {
        inlineCommentaryDispatcher.executeWithTimeout(
            narrativeTriggers, CommentaryType.NARRATIVE, replaySyncTimeoutSeconds);
    } else {
        inlineCommentaryDispatcher.executeAsync(narrativeTriggers, CommentaryType.NARRATIVE);
    }
}
```

Helper methods:
```java
private boolean syncReactive() {
    return "full".equals(replaySyncMode) || "reactive-only".equals(replaySyncMode);
}
private boolean syncNarrative() {
    return "full".equals(replaySyncMode);
}
```

**Blocking semantics:** `executeWithTimeout()` runs the LLM call on the current thread (the Quarkus scheduler thread). Since `@Scheduled` uses `ConcurrentExecution.SKIP`, no new tick fires while the current tick is blocked. The 500ms tick interval becomes a minimum — a tick that takes 3 seconds (LLM wait) simply delays the next tick by 3 seconds. This is exactly the behavior we want for replay sync.

### Narrative dispatch path change

Currently, narrative triggers go through `caseHub.signal()` (GameTickExecutor lines 127–133), which dispatches engine workers. But the settlement tracker never signals worker completion (noted in the comment at line 116–117), so narrative workers dispatched via `caseHub.signal()` never execute.

This spec moves narrative commentary to `InlineCommentaryDispatcher` (D7), which calls `ChatModel` directly — the same proven path as reactive commentary. The `caseHub.signal()` path for narrative is removed.

## 5. InlineCommentaryDispatcher Changes

### Extended API

The dispatcher gains:
1. A `CommentaryType` parameter to select reactive vs narrative prompts
2. A synchronous `executeWithTimeout()` method for replay sync
3. A narrative-capable `AgentDescriptor` lookup

```java
@ApplicationScoped
public class InlineCommentaryDispatcher {

    private final ChatModel chatModel;
    private final AgentDescriptor reactiveDescriptor;
    private final AgentDescriptor narrativeDescriptor;
    private final Event<CommentaryCompleted> completedEvent;
    private final boolean available;

    @Inject
    InlineCommentaryDispatcher(Instance<ChatModel> chatModelInstance,
                               Instance<QuarkMindAgentRegistrar> registrarInstance,
                               Event<CommentaryCompleted> completedEvent) {
        // ... existing ChatModel + registrar resolution ...
        this.reactiveDescriptor = findDescriptor(descriptors, "commentary-reactive");
        this.narrativeDescriptor = findDescriptor(descriptors, "commentary-narrative");
        this.available = reactiveDescriptor != null;
        // narrative descriptor may be null — narrative-only availability not required
    }

    /** Fire-and-forget async dispatch (live game modes). */
    public void executeAsync(Map<String, Object> triggers, CommentaryType type) {
        Thread.startVirtualThread(() -> {
            try {
                execute(triggers, type);
            } catch (Exception e) {
                log.warnf(e, "[INLINE-COMMENTARY] %s failed: %s", type, e.getMessage());
            }
        });
    }

    /** Synchronous dispatch with timeout (replay sync mode). */
    public void executeWithTimeout(Map<String, Object> triggers,
                                    CommentaryType type, int timeoutSeconds) {
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

        completedEvent.fire(new CommentaryCompleted(
            descriptor.agentId(), type == CommentaryType.REACTIVE
                ? "commentary-reactive" : "commentary-narrative",
            gameFrame, text != null ? text : "", type, latencyMs));
    }
}
```

### Backward compatibility

The existing `executeAsync(Map<String, Object>)` single-arg method delegates to `executeAsync(triggers, CommentaryType.REACTIVE)` to preserve any callers.

### Why CompletableFuture.runAsync + virtual thread

`executeWithTimeout()` uses `CompletableFuture.runAsync(() -> execute(...), Thread::startVirtualThread)` so the LLM call runs on a virtual thread (not the scheduler thread), while `.get(timeout, SECONDS)` blocks the scheduler thread with a bounded wait. This keeps the LLM I/O on a virtual thread (platform-thread-friendly) while the scheduler thread simply waits for the result.

## 6. Configuration Summary

### New properties

| Property | Default | `%replay` override | Description |
|---|---|---|---|
| `quarkmind.replay.sync.mode` | `none` | `full` | Sync mode: `full`, `reactive-only`, `none` |
| `quarkmind.replay.sync.timeout-seconds` | `15` | — | Max wait for LLM response before skip |
| `quarkmind.replay.observation.resolution` | `auto` | — | Observation resolution: `auto` or fixed int |

### Existing properties (unchanged)

| Property | Value | Used by |
|---|---|---|
| `starcraft.tick.interval` | `500ms` | `@Scheduled` interval |
| `quarkmind.commentary.reactive.cooldown-frames` | `110` | `CommentaryTriggerBuilder` |
| `quarkmind.commentary.narrative.window-frames` | `1000` | `CommentaryAccumulator` |

### QA endpoint additions

`ReplayControlsResource` gains a new endpoint to query and change sync mode at runtime:

```
GET  /qa/replay/sync       → { "mode": "full", "timeoutSeconds": 15 }
POST /qa/replay/sync?mode=reactive-only
```

Runtime changes override the config property for the current session. This lets the viewer switch between full sync and reactive-only without restarting the server.

For runtime sync mode changes, `GameTickExecutor` reads the mode from a volatile field that both the config property and the QA endpoint write to.

## 7. Data Flow — Sync Mode

```
Scheduler fires (500ms)
  │
  ▼
AgentOrchestrator.gameTick()
  │ speed = getSpeedMultiplier()
  ▼
GameTickExecutor.execute(speed)
  │
  ├─ engine.tick() × (speed - 1)     ← advance without observing
  ├─ engine.tick()                     ← final tick
  ├─ engine.observe() → gameState     ← snapshot accumulated state
  │
  ├─ translator.toMap() → caseData
  ├─ caseHub.signalAndAwaitSync()     ← plugins run
  ├─ summarisationLifecycle.tick()    ← L2→L3→L4
  ├─ commentaryAccumulator.tick()     ← narrative window check
  │
  ├─ commentaryTriggerBuilder.build() ← reactive trigger check
  │   └─ if triggers:
  │       ├─ sync mode?  → executeWithTimeout(triggers, REACTIVE, 15s)
  │       │                 ├─ LLM call on virtual thread (1-5s)
  │       │                 ├─ .get(15, SECONDS) blocks scheduler
  │       │                 ├─ CommentaryCompleted fires
  │       │                 └─ returns (or timeout → skip + warn)
  │       └─ async mode? → executeAsync(triggers, REACTIVE)
  │
  ├─ if narrativeTriggers:
  │   ├─ sync mode?  → executeWithTimeout(triggers, NARRATIVE, 15s)
  │   └─ async mode? → executeAsync(triggers, NARRATIVE)
  │
  ├─ advisory/coaching triggers       ← unchanged
  ├─ engine.dispatch()                ← drain intents
  └─ return TickResult
```

When both reactive and narrative trigger on the same tick in FULL sync mode, both block sequentially — the tick waits for reactive, then waits for narrative. Worst case: ~10 seconds (2 × 5s). This is rare (requires a moment to both trigger reactive and fill the narrative window simultaneously) and the 15-second per-call timeout caps the total at 30 seconds.

## 8. Testing Strategy

### Unit tests (plain JUnit)

**InlineCommentaryDispatcherTest** — extend existing tests:
- `executeWithTimeout_completesWithinTimeout` — mock ChatModel returns in 100ms, verify CommentaryCompleted fires
- `executeWithTimeout_timeoutSkips` — mock ChatModel blocks for 20s, verify timeout at 1s (test config), no CommentaryCompleted, warning logged
- `executeAsync_narrativeType` — verify narrative prompts used when `CommentaryType.NARRATIVE` passed
- `executeAsync_reactiveType` — verify reactive prompts used (backward compat)

**GameTickExecutorMigrationTest** — extend:
- `execute_speedMultiplier_callsTickNTimes` — mock SC2Engine, call `execute(4)`, verify `engine.tick()` called 4 times, `engine.observe()` called once
- `execute_syncMode_blocksOnCommentary` — set sync mode to `full`, verify `executeWithTimeout` called instead of `executeAsync`
- `execute_syncModeNone_asyncDispatch` — set sync mode to `none`, verify `executeAsync` called (current behavior)

**ReplayControlsResourceTest** — extend:
- `syncEndpoint_getMode` — GET `/qa/replay/sync` returns current mode
- `syncEndpoint_setMode` — POST `/qa/replay/sync?mode=reactive-only` changes mode

### Integration tests (@QuarkusTest)

**SynchronizedReplayCommentaryIT** — new:
- Boot with `%test` profile + replay sync config overrides
- Inject `AgentOrchestrator`, `InlineCommentaryDispatcher` (mock ChatModel)
- Call `gameTick()` with mock engine returning commentary-triggering game state
- Verify tick blocks until mock ChatModel responds
- Verify `CommentaryCompleted` CDI event fires with correct game frame

### Replay visual tests (@Tag("browser"))

Existing `ReplayVisualizerIT` extended:
- Start replay at 4x speed with sync mode `full`
- Verify commentary messages appear synchronized with game state (commentary frame <= current visible frame)

## 9. Protocols Checked

| Protocol | Status |
|---|---|
| `game-lifecycle-observer-synchrony` | Not affected — sync blocking is in the tick pipeline, not in CDI observers. `CommentaryCompleted` fires synchronously from the virtual thread, observable by `@Observes` on the scheduler thread after `.get()` returns. |
| `gametick-must-not-write-gameactive` | Not affected — no changes to `gameActive` handling. The speed loop only calls `engine.tick()`, not `gameActive`. |

## 10. Scope Boundaries

**In scope:**
- Speed multiplier wiring (D5)
- Replay sync mode with `full`, `reactive-only`, `none` (D2, D6)
- InlineCommentaryDispatcher extended for narrative (D7)
- Hard timeout with skip (D4)
- Observation resolution config property declared (D3) — `auto` path only
- QA endpoint for runtime sync mode control
- Unit + integration tests

**Out of scope (follow-up):**
- Multi-observation resolution (resolution < speed) — declared but clamped to `auto` in #290
- Frontend pause indicator ("generating commentary...") — the canvas freezes naturally when no ticks fire; a visual indicator is polish
- Scrubbing / frame stepping — requires the dedicated replay coordinator (Approach C), deferred
- Narrative quality improvements at high speed — the narrative prompt is unchanged; summarisation quality at lower observation resolution may need tuning

## References

- `AgentOrchestrator.java` — scheduler, speed multiplier, ConcurrentExecution.SKIP
- `GameTickExecutor.java` — tick pipeline, commentary trigger points
- `InlineCommentaryDispatcher.java` — existing reactive inline dispatch
- `CommentaryWorkerFactory.java` — reactive + narrative prompt builders
- `ReplayEngine.java` — replay engine contract, tick/observe/seekTo
- `ReplayControlsResource.java` — QA endpoints for replay control
- `CommentaryCompleted.java` — CDI event record
- Commentary design spec (2026-07-06, #181) — dual-pattern architecture, trust dimensions
- Workbench blocks-ui spec (2026-08-26, #289) — commentary surfacing, defers #290
- Protocol `game-lifecycle-observer-synchrony` (PP-20260610-88dbbd)
- Protocol `gametick-must-not-write-gameactive` (PP-20260612-afe621)
