# Design Journal — epic-phase-6

## Phase 6 — Replay-Accurate Forward Simulation
**Date:** 2026-05-15  
**Issue:** #137

### What was built

Replay validation infrastructure that feeds a human player's actual commands from `.SC2Replay` GAME_EVENTS into `EmulatedGame` and compares the resulting economic state tick-by-tick against `ReplaySimulatedGame` (ground truth from tracker events).

### Key design decisions

**`GameEventStream` split into thin reader + extraction pipeline.** The original `parse(Path) → List<UnitOrder>` monolith was replaced with `events(Path) → List<Event>` (MPQ reader only). All extraction logic moved to `AbilityMapping` + `ReplayCommandExtractor`. This enables independent testing of each concern and makes the extraction strategy swappable.

**`AbilityMapping` is stateful, not a lookup table.** Owns selection state per-player (`SelectionDeltaEvent` → `onSelection()`; `CmdEvent` → `process() → List<ReplayCommand>`). Returning `List` (not `Optional`) was the right call — movement commands fan out one `UnitOrder` per selected unit. Ability IDs discovered empirically via `AbilityDiscoveryTest` scanning real replays.

**BuildIntent extraction abandoned; building injection chosen instead.** Bot SC2 replays use `abilLink=42` (Smart command) for building placement — identical to the move command, impossible to distinguish from game events. Solution: inject buildings from `ReplaySimulatedGame.snapshot().myBuildings()` (tracker events, unambiguous) into `EmulatedGame` each tick using replay tags. `TrainIntent`s then resolve directly since they carry the same tracker-event tags. Two focused public methods added to `EmulatedGame`: `injectReplayBuilding(Building)` (updates supply) and `markReplayBuildingComplete(String)` (updates supply).

**Unit count within 2 of ground truth over 3 minutes.** Exact match requires a saturation-aware mining model (#141) — the flat rate causes EmulatedGame to accumulate excess minerals, occasionally training 1 tick early. `maxUnitDelta ≤ 2` is sufficient to confirm ability mapping is working correctly.

**Dual test surface.** `ReplayValidationTest` (normal surefire, regression guard) + `ReplayValidationReportTest` (`@Tag("report")`, `mvn test -Preport`, human-readable divergence dump). Both share the same `ReplayValidationHarness` and `DivergenceReport`.

### Open gaps (filed as issues)

- #140 — Terran `.SC2Replay` files needed for `AbilityMapping` Terran ability IDs
- #141 — Saturation-aware mining model for exact unit count match
- #138 — Terran/Zerg `EmulatedGame` mechanics (separate from ability mapping)
