# Decisions — #523 Standalone Building Lifecycle

## D1: Initial building source

**Choice:** Tracker UnitBorn events at loop 0
**Alternatives:**
- Race model seed via reset() — simpler but creates a split between initial and subsequent buildings
- Both (conditional) — tracker for harness, race model for standalone — more complex
**Rationale:** Consistent with how all other buildings will work. The harness uses resetForHarness() (no race model seed) and gets all buildings from tracker data. True standalone mode (training) can still use race model seed via reset().
**Trade-offs:** Requires emitting BuildingBornIntent from UnitBorn events for buildings, adding a new code path.
**Sources:** EmulatedGame.java:152 (resetForHarness), ReplayValidationHarness.java:80 (reset call), TrackerIntentExtractor.java:82 (UnitBorn skip)
**Exploration:** quick
**Status:** captured

## D2: Building construction timing

**Choice:** Pinned completion from UnitDone tracker events
**Alternatives:**
- Computed from SC2Data.buildTimeInLoops() — tests physics but timing errors cascade into supply/tech/spending
- Computed with assertion — compute but assert match with UnitDone timing
**Rationale:** Consistent with TrainIntent's pinnedCompletionLoop pattern. Tests economy/supply accuracy rather than build-time calibration. Avoids build-time errors cascading into all downstream tracking.
**Trade-offs:** Doesn't test EmulatedGame's build-time calculation accuracy. Requires tracking UnitDone events in TrackerIntentExtractor to capture completion loops.
**Sources:** EmulatedGame.java:606-619 (startTraining with pinnedCompletionLoop), TrackerIntentExtractor.java:133 (UnitDone no-op)
**Exploration:** quick
**Status:** captured

## D3: Building tag correspondence

**Choice:** Pass tracker tags through building intents
**Alternatives:**
- Rely on auto-resolve — keep bldg-N tags, TrainIntents auto-resolve by building type. Simpler but less reliable for multiple buildings of same type.
**Rationale:** Direct tag match is more reliable than auto-resolve fallback, especially with multiple buildings of the same type. Mirrors how TrainIntent already carries replayUnitTag.
**Trade-offs:** Requires new fields on BuildIntent (replayTag) and new BuildingBornIntent. handleBuild must use the provided tag instead of generating bldg-N.
**Sources:** EmulatedGame.java:703 (tag generation), EmulatedGame.java:385-401 (auto-resolve fallback), TrackerIntentExtractor.java:242 (r-tagIndex-tagRecycle format)
**Exploration:** quick
**Status:** captured

## D4: Intent type design

**Choice:** Dedicated building intent types — BuildingBornIntent, extended BuildIntent, BuildingDiedIntent
**Alternatives:**
- Reuse existing types — overload BuildIntent/UnitDiedIntent. Fewer types but muddied semantics.
- Combined lifecycle intent — single BuildingLifecycleIntent with action enum. Minimal types but god-intent anti-pattern.
**Rationale:** Clean separation. Each lifecycle event maps to a distinct intent. BuildingBornIntent is semantically different from BuildIntent (no construction, no cost). Follows existing pattern where each event type has its own intent.
**Trade-offs:** Adds 2 new intent types (BuildingBornIntent, BuildingDiedIntent) to the sealed Intent hierarchy (currently 12 types). Requires updating switch expressions over Intent.
**Sources:** BuildIntent.java:6 (current record), EmulatedGame.java:332-346 (switch over Intent), MorphIntent.java (building morph already supported)
**Exploration:** quick
**Status:** captured
