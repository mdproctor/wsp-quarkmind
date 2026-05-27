# IEM10 gameEvents → TimedIntent Stream — Design Spec

**Issue:** #150
**Date:** 2026-05-27
**Branch:** issue-150-iem10-timed-intent-stream

---

## Problem

`IEM10JsonSimulatedGame` reads only `trackerEvents` from SC2EGSet JSON replays. The
`gameEvents` array — which contains `Cmd` events with `abilLink`, `abilCmdIndex`, `loop`,
and `userid.userId` — is ignored. Without parsing `gameEvents`, there is no way to extract
training intents from IEM10 replays, blocking:

- Multi-game `ReplayValidationHarness` runs across all 30 IEM10 games
- Statistical coverage of training patterns across PvT, PvZ, PvP matchups
- Cross-validation of calibration results between scelight binary and SC2EGSet JSON parsers

---

## Known Gotchas (Garden)

Two gotchas are directly in the critical path — both must be handled correctly or extraction
silently returns zero results:

**GE-20260522-f0d1ec — userId ≠ playerID - 1**
`userid.userId` in `gameEvents` matches `ToonPlayerDescMap.userID`, not `playerID - 1`.
Fix: read both `playerID` (for tracker events) and `userID` (for game events) from
`ToonPlayerDescMap` at construction time.

**GE-20260522-eccbde — abilLink IDs are patch-specific**
IEM10 (2016 build 39948) uses different abilLink constants than AI Arena (2023+ build 67188).
Confirmed: Nexus/Probe = 167 in IEM10 vs 175 in AI Arena. Gateway, Robotics, Stargate
equivalents are unknown and must be discovered empirically before implementation.

---

## Design

### 1. `IEM10JsonSimulatedGame` — Minimal additions

Two additions only — the class remains a tick-based game state simulator, not an extractor:

**`watchedUserId` field** — extracted from `ToonPlayerDescMap` alongside the existing
`watchedPlayerId`. Used for `gameEvents` filtering; exposed via package-private accessor
`watchedUserId()`.

**`gameEvents` field** — `List<JsonNode>` parsed from the JSON root in the constructor,
stored alongside `trackerEvents`. Exposed via package-private accessor `gameEvents()`.

No other changes to `IEM10JsonSimulatedGame`. Tick model, reset, `isComplete`, and
`snapshot()` are untouched.

### 2. `IEM10CommandExtractor` — New class, `io.quarkmind.sc2.mock`

Static extraction class. Mirrors `ReplayCommandExtractor` in structure.

```java
// General form — any player by userId (all species supported)
static List<TimedIntent> extract(IEM10JsonSimulatedGame game, int userId)

// Convenience — watched (Protoss) player
static List<TimedIntent> extract(IEM10JsonSimulatedGame game) {
    return extract(game, game.watchedUserId());
}
```

**Selection tracking:** a single pass over `gameEvents` maintains `List<String> currentSelection`
(the "j-" tagged buildings/units currently selected). On a `SSelectionDeltaEvent` for the
target userId, the selection is updated from `unitTagIndex`/`unitTagRecycle` pairs using the
same `makeTag()` format as tracker events. On a `Cmd` event for the target userId, the first
element of `currentSelection` is the building tag for the resulting `TrainIntent`.

If `currentSelection` is empty when a `Cmd` fires, the command is dropped — same behaviour
as `AbilityMapping`.

**Species coverage:** the ability dispatch table covers all three races (Protoss, Zerg, Terran)
using IEM10 2016 patch constants. The Protoss convenience overload is the primary entry point,
but Zerg and Terran intents can be extracted by passing the appropriate userId.

**Ability constants:** private statics inside `IEM10CommandExtractor`, documented with the
IEM10 2016 patch version. Not shared with `AbilityMapping` — the 2016 values are different
from 2023+ AI Arena values, and sharing would obscure the patch-version dependency.

Known constant: `NEXUS_2016 = 167`. All others discovered via `IEM10AbilityDiscoveryTest`
(see Tests section below) before mapping code is written.

### 3. `ReplayValidationHarness` — Generalized

A new general overload is added. All harness logic moves here:

```java
static DivergenceReport run(SimulatedGame groundTruth, List<TimedIntent> intents, int tickLimit)
```

The existing binary-replay entry point becomes a one-line delegate:

```java
static DivergenceReport run(Path replayPath, int playerId, int tickLimit) {
    var game    = new ReplaySimulatedGame(replayPath, playerId);
    var intents = ReplayCommandExtractor.extract(replayPath, playerId).intents();
    return run(game, intents, tickLimit);
}
```

`SimulatedGame` is the correct abstraction — both `ReplaySimulatedGame` and
`IEM10JsonSimulatedGame` already extend it and provide `reset()`, `tick()`, `isComplete()`,
and `snapshot()`. No new interface is needed.

All harness internals (building sync, supply/vespene sync, probe count sync, intent
application, divergence snapshot collection) operate on `SimulatedGame` and `GameState`
only — no source-specific logic.

Existing callers of `run(Path, int, int)` are unaffected.

---

## Tests

### `IEM10AbilityDiscoveryTest` — prerequisite, run first

`@Tag("diagnostic")`, excluded from default surefire run.

Iterates all 30 IEM10 games. For each game and each player, dumps unique
`(abilLink, abilCmdIndex, count)` tuples sorted by count descending. Output cross-referenced
against expected unit-born counts from tracker events to identify each ability.

Run once before implementing the ability table. Results are hard-coded as constants into
`IEM10CommandExtractor`. The test is retained in the repo as documentation of how the
constants were derived.

### `IEM10CommandExtractorTest` — unit test (no Quarkus, fast)

- Constructs first IEM10 game; calls `extract(game)`; asserts non-empty, all intents are
  `TrainIntent`, all loop values positive
- Asserts non-trivial count (guards against abilLink/userId bugs that silently return zero)
- For a known PvZ game: extracts Protoss intents (default overload) AND Zerg intents
  (explicit userId overload); asserts both are non-empty

### `IEM10MultiGameValidationTest` — multi-game report

`@Tag("report")`, excluded from default surefire run. Mirrors `ReplayValidationReportTest`.

Iterates all 30 IEM10 games via `IEM10JsonSimulatedGame.enumerate()`. For each game:

```java
ReplayValidationHarness.run(game, IEM10CommandExtractor.extract(game), Integer.MAX_VALUE)
```

Prints aggregate divergence stats (mean unit count divergence, max divergence, per-matchup
breakdown for PvT / PvZ / PvP). Run via `mvn test -Preport`.

### `ReplayValidationHarnessTest` — regression

Existing test must pass unchanged. The binary-replay path now delegates to the general form
but behaviour is identical.

---

## Implementation Order

1. Run `IEM10AbilityDiscoveryTest` → record 2016 constants
2. Extend `IEM10JsonSimulatedGame` (add `watchedUserId`, `gameEvents`)
3. Implement `IEM10CommandExtractor` (ability table + selection tracking)
4. Write `IEM10CommandExtractorTest` — must pass before proceeding
5. Generalize `ReplayValidationHarness` — verify existing test still passes
6. Write `IEM10MultiGameValidationTest` — run and record baseline stats

---

## Out of Scope

- Movement order extraction from IEM10 `gameEvents` (only training intents)
- Terran/Zerg ground truth tracking in `IEM10JsonSimulatedGame` (Protoss-only tick model)
- Creating an `IEM10ValidationHarness` separate class — consolidation via generalised harness
  covers this
