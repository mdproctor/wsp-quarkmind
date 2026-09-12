# Decisions — #290 Synchronized Replay-Commentary Model

## D1: Sync scope — replay-only

**Choice:** Synchronize commentary with the tick loop in replay mode only. Live game modes (sc2, emulated, mock) keep fire-and-forget async dispatch.
**Alternatives:**
- All modes — live games can't pause for LLM without gameplay consequences
**Rationale:** Replays are immutable — pausing costs nothing. Live games cannot wait for LLM responses without affecting gameplay.
**Trade-offs:** Commentary in live modes continues to lag behind game state at high tick rates. Acceptable because live mode runs at real-time pace.
**Sources:** `AgentOrchestrator.java:112` (@Scheduled gameTick), `InlineCommentaryDispatcher.java:66` (executeAsync), `ReplayEngine.java:37` (@IfBuildProfile("replay"))
**Exploration:** quick
**Status:** captured

## D2: Commentary patterns to sync

**Choice:** Both reactive and narrative patterns sync by default, runtime-configurable via `quarkmind.replay.sync.mode` with values FULL (default), REACTIVE_ONLY, NONE.
**Alternatives:**
- Reactive-only — narrative is backward-looking and doesn't need precise frame alignment, but pausing for it gives a "chapter marker" experience
- None — current fire-and-forget behavior
**Rationale:** Full sync as default gives the best viewer experience. At high speed, narrative pauses can be significant (~3-5s every ~30-45 game-seconds) so REACTIVE_ONLY is available as a runtime override.
**Trade-offs:** FULL sync at 8x speed means ~50-90% overhead from narrative pauses. Users who want speed over commentary fidelity can switch to REACTIVE_ONLY at runtime.
**Sources:** Commentary design spec §Pattern A/B, `CommentaryTriggerBuilder.java:28` (110-frame cooldown), `CommentaryAccumulator.java:40` (672-frame minimum floor)
**Exploration:** quick
**Status:** captured

## D3: Observation resolution — auto-scale with speed

**Choice:** At speed N, call `engine.tick()` N times per scheduler fire but run the pipeline only once on the final observed state. Resolution auto-scales: `resolution = max(1, speed)`. Override via `quarkmind.replay.observation.resolution` (auto | 1 | 2 | 4 | 8).
**Alternatives:**
- Fixed resolution (always observe every loop) — full fidelity but pipeline can't keep up at >2x (100-200ms × 8 = 800-1600ms exceeds 500ms tick budget)
- Independent resolution config only — more flexible but auto-scale is the right default for nearly all cases
**Rationale:** The full pipeline (Drools rules, CaseHub signal+settle) costs ~100-200ms. Running it once per scheduler fire regardless of speed keeps CPU load constant. Moment detection cooldowns (110+ frames) dwarf the sampling gap (8 frames at 8x = 0.36 game-seconds), so no moments are missed.
**Trade-offs:** At 8x, moment detection sees deltas across 8 frames instead of 1. A very short-lived event (< 8 frames) could theoretically be missed, but SC2 moments are all longer-lived than that.
**Sources:** `GameTickExecutor.java:67-169` (pipeline execution), `ReplayEngine.java:138-140` (tick advances one loop), video frame rate analogy from user
**Exploration:** quick
**Status:** captured

## D4: Timeout behavior — hard timeout with skip

**Choice:** Wait up to 15 seconds for the LLM response. On timeout or failure, abandon the commentary for that moment, log a warning, and resume the replay.
**Alternatives:**
- Retry then skip — more resilient to transient errors but doubles worst-case pause (30s)
**Rationale:** Commentary is informational — a missed line is far less harmful than a 30-second freeze. The trust scoring infrastructure already records latency, so chronically slow models get deprioritised by the routing policy.
**Trade-offs:** No retry means a transient network blip causes a missed commentary. Acceptable for a dev tool.
**Sources:** Trust routing policy in commentary design spec §Trust Dimensions, `LlmWorkerLatencyRecorder` (latency trust scoring)
**Exploration:** quick
**Status:** captured

## D5: Speed multiplier wiring — in scope

**Choice:** Wire the existing `speedMultiplier` field in `AgentOrchestrator` into the tick loop as part of this issue.
**Alternatives:**
- Separate issue — cleaner separation of concerns but prevents end-to-end testing of sync model
**Rationale:** Speed control is a prerequisite for the sync model to be useful. Without working speed control, there's no >1x replay to synchronize against. The change is small (a loop in GameTickExecutor) and tightly coupled.
**Trade-offs:** Slightly larger scope for #290. Acceptable given the dependency.
**Sources:** `AgentOrchestrator.java:47,62-63` (speedMultiplier field, getter/setter — never read by gameTick), `ReplayControlsResource.java:108-115` (/qa/replay/speed endpoint)
**Exploration:** quick
**Status:** captured

## D6: Synchronization mechanism — synchronous blocking in GameTickExecutor

**Choice:** Approach A — `InlineCommentaryDispatcher` exposes a synchronous path returning `CommentaryCompleted` directly. In replay sync mode, `GameTickExecutor.execute()` calls it synchronously, blocking the scheduler thread until the LLM responds (with D4 timeout). `ConcurrentExecution.SKIP` prevents overlapping ticks.
**Alternatives:**
- Event-driven pause/resume — set `schedulerPaused` flag on trigger, clear on CDI event. Race condition between trigger and next tick; timeout requires a separate scheduled task; reasoning split across two code paths.
- Dedicated replay tick loop — own thread with full timing control. Over-engineered for a dev tool; duplicates orchestrator lifecycle.
**Rationale:** Minimal new code. Leverages existing `ConcurrentExecution.SKIP` guarantee — if a tick takes 3 seconds, the next tick simply doesn't fire. No race conditions, no new threading model. Sync decision is local to the tick execution.
**Trade-offs:** Blocks a Quarkus scheduler thread during LLM wait. Acceptable because `gameTick()` is the only scheduled task. Would need revisiting if other scheduled tasks are added.
**Sources:** `AgentOrchestrator.java:112` (ConcurrentExecution.SKIP), `InlineCommentaryDispatcher.java:77` (existing synchronous execute method)
**Exploration:** quick
**Status:** captured

## D7: Narrative dispatch path — extend InlineCommentaryDispatcher

**Choice:** Extend `InlineCommentaryDispatcher` to handle both reactive and narrative commentary. Add a second execution path that uses `CommentaryWorkerFactory.buildNarrativeSystemPrompt()` instead of the reactive prompt. Same class, same ChatModel wiring, two prompt builders.
**Alternatives:**
- Separate `InlineNarrativeDispatcher` — duplicates ChatModel wiring, CDI event firing, and synchronous/async paths for no benefit
**Rationale:** `InlineCommentaryDispatcher` already has the ChatModel, CDI `Event<CommentaryCompleted>` firing, and the synchronous path. Narrative is the same operation with a different prompt. The current narrative path via `caseHub.signal()` is broken (settlement tracker never signals completion — workers never execute). Moving narrative to the inline dispatcher fixes this and enables sync mode.
**Trade-offs:** InlineCommentaryDispatcher grows in responsibility (both commentary types). Acceptable — both are "call ChatModel with a prompt, fire CommentaryCompleted."
**Sources:** `GameTickExecutor.java:116-117` (comment: engine worker dispatch is broken), `GameTickExecutor.java:127-133` (narrative via caseHub.signal), `InlineCommentaryDispatcher.java:28-106` (existing reactive-only inline dispatcher), `CommentaryWorkerFactory` (has both reactive and narrative prompt builders)
**Exploration:** quick
**Depends on:** D6 (synchronization mechanism)
**Status:** captured
