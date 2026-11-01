## D1: Divergence diagnosis tool architecture

**Choice:** Two-phase scanner + tracer
**Alternatives:**
- Extended DivergenceReport — single pass with inline diagnostics, but bloats TickSnapshot for all consumers
- Single combined test — less code but mixes triage and root-cause responsibilities
**Rationale:** Mirrors the natural diagnostic workflow: scan broadly to find problematic replays, then deep-trace the worst offenders. Scanner stays fast (just counts), tracer captures production context (buildings, reactor state, queue state, rejection reasons). Neither modifies existing harness code.
**Trade-offs:** Two test classes instead of one; deep-tracing requires a second run on selected replays
**Sources:** HellionDeficitDiagnosticTest, ReactorStateAtIntentTest (existing bespoke patterns), TrackerDrivenAccuracyBaselineTest (aggregate approach), ReplayValidationHarness (tick loop orchestrator)
**Exploration:** quick
**Status:** captured
**Depends on:** —

## D2: Diagnostic context capture method

**Choice:** Intent execution listener on EmulatedGame
**Alternatives:**
- Custom tick loop — copy harness loop with inline probe points, full control but duplicates loop logic and drifts from real harness
- Post-hoc log parsing — no code changes but fragile and incomplete, not all rejection reasons are logged
**Rationale:** Lightweight callback on EmulatedGame that fires when a TrainIntent is processed or rejected (with outcome + reason). Tracer registers the listener and correlates outcomes with ground-truth counts. Gives causal tracing without modifying the harness loop or duplicating it.
**Trade-offs:** Small addition to EmulatedGame (listener field + fire points), but the listener is optional (null-safe) so no impact on existing consumers
**Sources:** EmulatedGame.trackerTrainDiagnostics() (existing per-type counters), HellionDeficitDiagnosticTest (manual building inspection pattern), ReactorStateAtIntentTest (custom tick loop pattern)
**Exploration:** quick
**Status:** captured
**Depends on:** D1 (tracer uses this listener)

## D3: SC2EGSet ingestion scope

**Choice:** Download + Python ingest
**Alternatives:**
- Download + Java race parameterisation — also fix IEM10JsonSimulatedGame to support all player perspectives for EmulatedGame validation, but adds Java scope
- Download only — defer pipeline work, but then dataset sits unused
**Rationale:** Python pipeline (sc2egset_extractor.py) already handles both player perspectives and supports the SC2EGSet nested ZIP format. No Java changes needed for classifier training data expansion. Java race parameterisation is a natural follow-up when we want to validate emulator accuracy against SC2EGSet replays.
**Trade-offs:** SC2EGSet replays can't be used for Java-side EmulatedGame accuracy validation until IEM10JsonSimulatedGame is parameterised (follow-up work)
**Sources:** quarkmind-classifier/src/sc2egset_extractor.py, IEM10JsonSimulatedGame.enumerate(), replay-index.md
**Exploration:** quick
**Status:** captured
**Depends on:** —
