# HANDOFF — quarkmind

## Last Session

Extended #296 to fix all four workbench tabs end-to-end, from server-side pipeline through WebSocket to browser rendering. Added inline LLM commentary via Vertex AI, replay reset with full state cleanup, and upstream API compatibility fixes.

### Fixes delivered

1. **Moment detection crash** — `MomentDetectionTask.fireRules()` returned `List.of()` (immutable) when `pendingIntel` was empty. `updateBattleFSM` tried `moments.add()` on BATTLE_ENDED transition → `UnsupportedOperationException`. Changed to `new ArrayList<>()`.

2. **Strategy tab race condition** — WebSocket snapshot arrived before tab element rendered. Added `applyWorkbenchSnapshot()` function + polling to apply cached data when lazily-created tab elements become available.

3. **Commentary tab empty** — Engine's `caseHub.signal()` worker dispatch broken (settlement tracker never signals completion). Created `InlineCommentaryDispatcher` — calls ChatModel directly on a virtual thread when commentary triggers fire. Pipeline: moments → trigger → inline Claude CLI via Vertex AI → `CommentaryCompleted` CDI event → WebSocket.

4. **WorkbenchSocket onOpen crash** — `CommentaryChannelBroker.init()` JPA in `@PostConstruct` threw before `broadcaster.addSession()` ran. Reordered: addSession first, commentary history in try-catch.

5. **Commentary duplication on reconnect** — `sendCommentaryHistory()` replayed all past messages from qhorus channel. Added session-scoped filtering via `CommentaryChannelBroker.sessionStart()`.

6. **Replay reset** — `/qa/replay/reset` endpoint: stop → seek(0) → start. Fires `GameStarted` CDI event resetting all stateful beans (cooldowns, first-contact flags, scouting state). UI Reset button clears all workbench tab state.

7. **Stale JS path** — `visualizer.html` loaded workbench-blocks.js from `visualizer-lib/` (stale checked-in copy) instead of `/blocks/` (Quinoa build output). Fixed path, removed stale copy.

8. **Upstream API compat** — `LevelEvent` 4th arg (tenancyId), Flyway V50→V52 migration rename, `TickOrchestratorWorker` catches `Error` not just `Exception`, `signalAndAwaitSync` returns enriched CaseContext even when `signal()` fails.

9. **Creep rendering** — Units hidden under creep. Fixed renderOrder: ground(0) < creep(2) < units/buildings(3) < fog(5). Creep opacity 0.7→0.45.

### Test coverage added

- `WorkbenchPipelineIT` — 9 tests: pattern, strategy, late-connect, commentary CDI→WS, coaching CDI→WS, moment detection chain, LLM commentary end-to-end, reset state cleanup
- `WorkbenchRenderTest` — 8 Playwright tests: strategy data after pipeline, coaching status text, commentary status text
- `MomentDetectionBattleTest` — regression test for immutable list bug

## Critical Discovery: #298

**Enemy unit positions are static in replay data.** The replay engine extracts positions from tracker birth events but never updates them. All spatial features (posture, timing, threat, map control, coaching) were calibrated on stale position data. Pattern classification (types/counts) is unaffected.

**Action:** Fix replay position tracking (#298), then re-run full calibration suite. Filed #299 for replay smoke tests that would catch this on first run.

## Remaining Work

- **#298** — Fix replay engine to apply position updates from SC2 tracker event snapshots
- **#299** — Replay smoke tests: position delta, pipeline health checks, WebSocket delivery
- **Commentary auto-apply timing** — `applyWorkbenchSnapshot` poll works via inline `<script>` in HTML but the `setInterval` at end of `visualizer.js` doesn't execute (async init() blocks). Inline script is a workaround, not a proper fix.
- **Coaching tab** — Only fires in coach mode (`gameMode=coach`). Shows "No coaching advice yet" in AI/replay mode. By design, but could show mode-specific guidance.
- **Upstream casehub-blocks API** — `EventStreamBus`/`LevelEvent` moved from `casehub-blocks` to `casehub-blocks-summarisation-api`. `Confidence` moved to `casehub-neocortex-cognitive-api`. `Commitment` constructor changed. Test compilation has remaining errors (`ScoredCbrCase`, `SC2CbrRetentionObserverTest`). Main sources compile.

## References

- `InlineCommentaryDispatcher.java` — bypasses broken engine worker dispatch
- `ReplayControlsResource.java` — `/qa/replay/reset` endpoint
- `CommentaryChannelBroker.java` — session-scoped commentary history
- `WorkbenchPipelineIT.java` — 9 pipeline integration tests
- `WorkbenchRenderTest.java` — 8 Playwright visual tests
- `visualizer.js` — `applyWorkbenchSnapshot`, creep/unit renderOrder, reset button
