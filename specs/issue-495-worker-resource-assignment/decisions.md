# Decisions — #495 Worker-to-Resource Assignment Tracking

## D1: Accuracy scope

**Choice:** Full event-driven tracking — gas assignment, expansion transfers, worker death per-base, new worker routing
**Alternatives:**
- Gas assignment accuracy only — lower complexity, but doesn't model expansion transfers or per-base death
- Replay harness accuracy only — safer but doesn't improve standalone emulated mode
**Rationale:** The full state machine models real SC2 worker behavior across all modes (emulated, replay, tracker-driven), giving the highest fidelity improvement
**Trade-offs:** Higher implementation complexity; more events to handle; more edge cases in worker transfer logic
**Sources:** EmulatedGame.java:122-203 (current tick economy flow), issue #495 acceptance criteria
**Exploration:** quick
**Status:** captured

## D2: State location

**Choice:** New `WorkerAssignment` class — standalone package-private class in the emulated package, composed into EmulatedGame alongside EconomyTracker
**Alternatives:**
- Extend PlayerState — more cohesive but increases PlayerState's scope beyond flat container
- Extend EconomyTracker — economy-focused but EconomyTracker has no unit/building awareness
**Rationale:** Keeps PlayerState flat and simple, keeps EconomyTracker focused on rate accumulation. WorkerAssignment is a new concern (assignment tracking) that doesn't fit cleanly into either existing class.
**Trade-offs:** One more class to manage; EmulatedGame needs to coordinate calls between PlayerState, EconomyTracker, and WorkerAssignment
**Sources:** PlayerState.java (flat container, 95 lines), EconomyTracker.java (accumulator, 127 lines)
**Exploration:** quick
**Status:** captured

## D3: Integration mode

**Choice:** Replace `countWorkersPerBase()` entirely — WorkerAssignment becomes the single source of truth for all modes
**Alternatives:**
- Coexist as replay-only — keep countWorkersPerBase for standalone mode, WorkerAssignment only for tracker events. Safer but maintains two parallel paths.
**Rationale:** Single source of truth eliminates dual-path complexity. Standalone mode feeds WorkerAssignment events from EmulatedGame (building complete, unit born, unit died). Replay mode feeds tracker events. Same model, different event sources.
**Trade-offs:** Changes the standalone mode's worker distribution from position-based to event-driven, which could introduce regressions if events are missed. Must verify with existing playbook calibration tests.
**Sources:** EmulatedGame.java:1282-1306 (countWorkersPerBase), EmulatedGame.java:1258-1262 (setMiningProbesPerBase override)
**Exploration:** quick
**Status:** captured
