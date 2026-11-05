# Decisions — #523 Standalone Building Lifecycle

## D1: Initial building source

**Choice:** Tracker UnitBorn events at loop 0
**Alternatives:**
- Race model seed via reset() — simpler but creates a split between initial and subsequent buildings
- Both (conditional) — tracker for harness, race model for standalone — more complex
**Rationale:** Consistent with how all other buildings will work. The harness uses resetForHarness() (no race model seed) and gets all buildings from tracker data. True standalone mode (training) can still use race model seed via reset().
**Trade-offs:** Requires emitting BuildingBornIntent from UnitBorn events for buildings, adding a new code path. Loop-0 intents must be explicitly ordered: BuildingBornIntents before TrainIntents at the same loop to guarantee the initial town hall exists before worker trains resolve. Use a secondary sort key (intent type priority) rather than relying on SC2 tracker event ordering.
**Sources:** EmulatedGame.java:152 (resetForHarness), ReplayValidationHarness.java:80 (reset call), TrackerIntentExtractor.java:82 (UnitBorn skip)
**Exploration:** quick
**Status:** revised (added explicit loop-0 ordering requirement per R2-02)

## D2: Building construction timing

**Choice:** Pinned completion from UnitDone tracker events
**Alternatives:**
- Computed from SC2Data.buildTimeInLoops() — tests physics but timing errors cascade into supply/tech/spending
- Computed with assertion — compute but assert match with UnitDone timing
**Rationale:** Consistent with TrainIntent's pinnedCompletionLoop pattern. Tests economy/supply accuracy rather than build-time calibration. The pure-computed alternative is rejected because timing errors cascade into supply/tech/spending — a wrong build time shifts everything downstream. The computed-with-assertion variant is compatible with pinning: compute the build time, assert it matches UnitDone timing (within tolerance), log a diagnostic on mismatch, always use the pinned value. This provides runtime validation of SC2Data.buildTimeInLoops() constants against actual replays. The assertion can be added during implementation as a diagnostic — it doesn't change the completion timing decision.
**Trade-offs:** Doesn't test EmulatedGame's build-time calculation accuracy at runtime (unless the assertion diagnostic is added). Requires tracking UnitDone events in TrackerIntentExtractor to capture completion loops.
**Sources:** EmulatedGame.java:606-619 (startTraining with pinnedCompletionLoop), TrackerIntentExtractor.java:133 (UnitDone no-op)
**Exploration:** quick
**Status:** revised (rationale clarified: correctly distinguished computed-with-assertion from pure-computed per R2-05)

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

## D5: Mid-game building intent source

**Choice:** Emit BuildIntent from UnitInit tracker events for mid-game buildings
**Alternatives:**
- Dedicated BuildingStartedIntent — new intent type for construction start; cleaner semantics but adds a third building intent type when BuildIntent already means "start building construction"
- Reuse BuildingBornIntent with construction flag — overloads born semantics; a building being "born" (exists at game start) vs "starting construction" are different lifecycle events with different cost/timing behavior
- Extend MorphIntent for Zerg buildings — Zerg buildings morph from Drones, so MorphIntent could be extended; but non-Zerg races don't morph workers into buildings, and this conflates unit-to-building morph with building-to-building morph (Lair, Hive)
**Rationale:** BuildIntent already represents "start building construction." Adding replayTag and completionLoop fields naturally extends it for tracker-driven builds. No new type needed — the existing intent semantics match exactly. UnitInit is the correct tracker event because it fires at construction start; UnitBorn fires only for instant buildings (initial town hall, creep tumors, nydus worms).
**Trade-offs:** BuildIntent gains two new fields that are null/-1 for non-tracker builds. Existing call sites updated with factory method.
**Sources:** TrackerIntentExtractor.java:108-120 (UnitInit building skip), EmulatedGame.java:693-724 (handleBuild)
**Exploration:** quick (surfaced by R2-12)
**Status:** captured

## D6: Building cost deduction strategy

**Choice:** Always deduct mineral and gas costs from PlayerState when processing BuildIntent
**Alternatives:**
- Skip deduction for tracker-driven builds — the player already spent the resources, and the mineral GT sync overwrites the balance. Simpler but creates a silent dependency on the GT sync lifecycle: when mineral GT sync is later removed (#516), building costs silently disappear from the balance.
- Conditional deduction — deduct only when mineral GT sync is inactive. Requires awareness of GT sync state inside handleBuild, coupling building logic to harness state.
**Rationale:** Consistent with handleTrain which always deducts costs for tracker-driven trains (EmulatedGame.java:486-490). While the mineral GT sync is active, the deduction is immediately overwritten — no harm. When the mineral GT sync is later removed (#516), the deduction is essential. The "skip deduction" approach creates a hidden cross-issue dependency.
**Trade-offs:** Mineral balance may briefly go negative before GT sync overwrites it. This already happens for tracker-driven TrainIntents and is handled correctly (minerals recover through mining income).
**Sources:** EmulatedGame.java:486-490 (handleTrain tracker-driven cost deduction), EmulatedGame.java:693-724 (handleBuild current mineral-only deduction), EconomyTracker.java:52-62 (recordBuildSpending 2-arg vs 3-arg)
**Exploration:** quick (surfaced by R2-08/R2-15)
**Status:** captured

## D7: trackerDrivenBuildings registration from building intents

**Choice:** Explicitly add building tags to trackerDrivenBuildings when processing BuildingBornIntent and tracker-driven BuildIntent
**Alternatives:**
- Rely on handleTrain to populate — the first tracker-driven TrainIntent at a building adds its tag to the set (EmulatedGame.java:521). Works in practice because all trains in harness mode are tracker-driven, but relies on train ordering rather than building lifecycle.
**Rationale:** Building intents should self-register as tracker-driven — the building lifecycle owns this state, not the training path. Explicit registration eliminates the reliance on train ordering and is cleaner for code readers.
**Trade-offs:** Minor redundancy — handleTrain continues to populate the set independently. The set grows slightly earlier per building, which is harmless.
**Sources:** EmulatedGame.java:1584 (trackerDrivenBuildings field), EmulatedGame.java:521 (handleTrain population), EmulatedGame.java:685 (drainBuildingQueues supply bypass)
**Exploration:** quick (surfaced by R2-10)
**Status:** captured
