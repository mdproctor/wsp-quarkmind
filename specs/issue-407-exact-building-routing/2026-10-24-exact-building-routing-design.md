# Exact Replay Reconstruction via Game Event Building Routing

**Issue:** casehubio/quarkmind#407
**Branch:** issue-407-exact-building-routing
**Date:** 2026-10-24

## Problem

TrackerIntentExtractor creates TrainIntents from tracker UnitBorn events with opaque
`"r-{tagIndex}-{tagRecycle}"` tags. EmulatedGame cannot match these to real buildings,
so it auto-resolves via `shortestQueue()` — selecting the production building with the
shortest training queue. In reality, the player selected a specific building via hotkey.
This causes ~16 units at the 5-min checkpoint to route to the wrong building and
complete 1-2 ticks late.

The same issue applies to ResearchIntents (`"r-tracker"` tag) and AbilityIntents
(`"r-NEXUS"` tag for chrono boost).

## Scope

- **In scope:** TrainIntent, ResearchIntent, and AbilityIntent building routing in
  TrackerIntentExtractor for oracle replays (100% game event coverage)
- **Out of scope:** Non-oracle replays (~43% CmdEvent coverage) — these fall back to
  existing `"r-"` auto-resolution with zero behavior change

## Architecture

### Pre-indexed CmdEvent map (D1)

Pre-process all game events in a single pass to build a `TreeMap<Integer, String>`
mapping `CmdEvent.loop → selected building tag`. When TrackerIntentExtractor processes
tracker events, look up the back-calculated command loop in the pre-built index.

Two passes over the data: one for game events (building index), one for tracker events
(intent extraction). Both operate on the same in-memory `Replay` object.

### Selection tracking (D2)

A private `buildBuildingIndex(Event[] gameEvents, int userId)` method in
TrackerIntentExtractor:

1. Creates a `SelectionState` and iterates game events in loop order
2. For each `SelectionDeltaEvent` with matching userId: applies the remove mask
   (None/ZeroIndices/OneIndices/Mask variants), then adds new unit tags via
   `GameEventStream.decodeTag()`
3. For each `CmdEvent` with matching userId: records `(cmd.getLoop(), selection.first())`
   in the TreeMap — only when selection is non-empty
4. Returns the populated TreeMap

The remove-mask handling replicates `AbilityMapping.onSelection()` lines 403-428.
Tag decoding uses `GameEventStream.decodeTag(int)`.

### Index key structure (D3)

`TreeMap<Integer, String>` keyed by CmdEvent loop only. Selection state is set by
SelectionDeltaEvents which precede all CmdEvents at that loop. Same-loop CmdEvents
share the same selection, so loop-only key is sufficient.

## Data Flow

```
Game Events (SelectionDeltaEvent + CmdEvent)
    │
    ▼
buildBuildingIndex()  ──→  TreeMap<Integer, String>
    │                       key: CmdEvent loop
    │                       val: selection.first() decoded tag
    │
    ▼
extractFromReplay() reads tracker events:
    │
    ├─ UnitBorn  → commandLoop = completionLoop - trainTime
    │              → lookup index within ±5 loops
    │              → if found: use real tag  │  else: "r-{tagIdx}-{tagRecycle}"
    │
    ├─ Upgrade   → commandLoop = completionLoop - upgradeTime
    │              → same lookup  │  else: "r-tracker"
    │
    └─ ChronoBoost → loop from CmdEvent directly
                     → lookup index at cmd.getLoop()
                     → if found: use real Nexus tag  │  else: "r-NEXUS"
```

Non-oracle replays: `buildBuildingIndex` returns an empty or sparse map. Every lookup
misses. All intents fall back to existing `"r-"` auto-resolution. Zero behavior change.

## Lookup Logic

For each tracker event needing building routing:

1. Compute `commandLoop` (back-calculated or direct)
2. `Map.Entry<Integer, String> entry = index.floorEntry(commandLoop + 5)`
3. If `entry != null && commandLoop - entry.getKey() <= 5`: use `entry.getValue()`
4. Else: fall back to existing `"r-"` tag

`floorEntry(commandLoop + 5)` finds the nearest CmdEvent at or before
`commandLoop + 5`. The second check ensures it's not more than 5 loops before
`commandLoop`. This gives the ±5 window in O(log n).

## API Changes

None. `TrackerIntentExtractor.extract(Path, int)` signature is unchanged. The building
index is built internally from the same `Replay` object. All 26 callers (test files)
are unaffected.

## Protocol Compliance

- `PP-20260528-d9f967` (replay tag prefix): All tags retain the `"r-"` prefix —
  `GameEventStream.decodeTag()` produces `"r-{index}-{recycle}"` format
- `PP-20260528-612dee` (extractor separation): Building index logic stays in
  TrackerIntentExtractor (a dedicated extractor class), not SimulatedGame

## Testing

- **Unit test:** Extend `TrackerIntentExtractorIntentTest` — verify oracle replay
  intents use real decoded building tags (not auto-resolution tags) when game events
  are present
- **Accuracy validation:** Run `TrackerDrivenAccuracyBaselineTest` before and after —
  expect the 16-unit deficit at 5-min checkpoint to shrink
- **Regression:** `mvn test -pl quarkmind-sc2` — all existing tests pass

## Trade-offs

- **Two passes over data:** Acceptable — game event arrays are already in memory
- **~40 lines of duplicated SelectionDelta processing:** The alternative is coupling
  to the 900-line AbilityMapping class for 40 lines of logic. Duplication is the
  better trade-off here.
- **±5 loop window:** Empirically derived from the issue. May miss some correlations
  if CmdEvent and tracker event timing diverge by more than 5 loops. Fallback to
  auto-resolution is safe.

## References

- `TrackerIntentExtractor.java` — primary modification target
- `AbilityMapping.java:391-465` — SelectionDelta processing pattern to replicate
- `AbilityMapping.java:779-783` — trainIntent() building tag from selection
- `GameEventStream.java:34-36` — decodeTag() for `"r-"` format
- `SelectionState.java` — tag tracking data structure
- `EmulatedGame.java:361-468` — handleTrain() auto-resolution logic
- `TrackerEventFeatureExtractor.java:130-232` — extractWithCommands() interleaved
  processing pattern (reference, not reused)
- `PP-20260528-d9f967` — replay tag prefix protocol
- `PP-20260528-612dee` — extractor separation protocol
