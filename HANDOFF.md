# HANDOFF — quarkmind

## Last Session

Debugged and fixed the engine→plugin dispatch gap (#296). Two root causes found and fixed:

1. **JQ trigger prefix mismatch** — All 8 JQ trigger expressions in `QuarkMindCaseHub` used `.working["game.frame"]` but the engine's `JQExpressionEngine.evaluate()` runs JQ against `context.layer("working").asJsonNode()` — the working layer data directly, no `"working"` wrapper. Changed all triggers to `.[\"game.frame\"]` etc.

2. **Engine settlement tracker unwired** — `SignalSettlementTracker.recordCompletion()` and `QuiescenceTracker.onWorkerCompleted()` both have zero callers in the engine. Worker completion is never signalled, so `signalAndAwait` always times out when any binding fires. Bypassed by executing tick plugins inline in `signalAndAwaitSync` via `TickOrchestratorWorker.executeInline()`, then signalling the engine with enriched data (fire-and-forget) for advisory/commentary bindings. Removed tick-decision binding/worker from CaseDefinition.

Also fixed: `CommentaryTriggerBuilder` reading `ARMY` (List\<Unit\>) as Integer (masked by null ctx), and `CbrCase.withOutcome` API migration (upstream `Double` → `Confidence` type change).

Verified end-to-end: Playwright screenshot shows **ZERG_ROACH_RUSH (66%)** with counter recommendations in the Pattern tab during live replay at 9:33 game time. `WorkbenchPipelineIT` (3 tests) and `WorkbenchRenderTest` (5 tests including full-pipeline screenshot) all pass.

## Remaining Work

**Strategy panel empty in live demo** — `StrategySelectionPublished` CDI event fires (test proves it), but in live replay the event fires before the WebSocket client connects (28s gap). Fix: either re-fire on every tick or cache latest strategy event and push to new WebSocket clients on connect (WorkbenchBroadcaster already caches `latestStrategy` — check if `pushSnapshot` sends it).

**Moment detection crash** — `summarisation.moment-detection` plugin throws `UnsupportedOperationException` during inline execution (`MutableMapCaseContext` likely missing a method the plugin needs). This breaks the plugin chain after strategy but before economics. Investigate which `CaseContext` method is unsupported and fix.

**Coaching/Commentary panels** — require `ChatModel` (LLM) bean. Advisory/commentary workers dispatch via engine bindings but fail with "Bridge type mismatch: expected java.util.Map but received null" in replay mode without LLM configured. These panels populate only with an LLM provider (e.g. quarkus-langchain4j-anthropic).

**Engine bugs to file upstream** — (1) `SignalSettlementTracker.recordCompletion()` never called; (2) `QuiescenceTracker.onWorkerCompleted()` never called; (3) dev mode HTTP routing broken (all REST endpoints return 404 in mock profile, works in replay profile).

## References

- `quarkmind-sc2/.../QuarkMindCaseHub.java` — inline tick execution, fixed JQ triggers
- `quarkmind-sc2/.../TickOrchestratorWorker.java` — new `executeInline()` public method
- `quarkmind-sc2/.../WorkbenchPipelineIT.java` — 3 tests (pattern, strategy, WebSocket)
- `quarkmind-sc2/.../WorkbenchRenderTest.java` — Playwright tests including full-pipeline screenshot
- `quarkmind-sc2/.../CommentaryTriggerBuilder.java` — ARMY List→int fix
