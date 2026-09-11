## D1: ENEMY_POSTURE stickiness fix approach

**Choice:** Last-known posture persistence — cache the last non-UNKNOWN posture in `DroolsScoutingTask` and use it as fallback when the DRL produces no posture decisions (empty buffer after eviction)
**Alternatives:**
- Evict tags alongside events — remove tags from `seenUnitTags` during eviction so re-sighted units re-enter buffer. Corrupts pattern classification timestamps (old units get new timestamps, look like fresh production).
- Separate active-presence tracker — parallel buffer in `ScoutingSessionManager` without tag dedup. Over-engineered — the bug is in conclusion interpretation, not in the buffer.
**Rationale:** The 3-minute eviction window controls evidence freshness, not conclusion freshness. ENEMY_POSTURE is a build-order classification — once the DRL evaluates evidence and produces ALL_IN or MACRO, that conclusion should persist until contradicted by new evidence (e.g., expansion spotted → MACRO). UNKNOWN should mean "never classified," not "evidence expired."
**Trade-offs:** MACRO → ALL_IN pivot detection remains impossible (expansion events are permanent). This was already a limitation before #300 and is unrelated to the stickiness fix.
**Sources:** `DroolsScoutingTask.java:249-251`, `ScoutingSessionManager.java:50-105`, protocol `strategy-attack-under-unknown-posture.md`
**Exploration:** deep-analysis
**Status:** captured

## D2: Military posture vocabulary placement

**Choice:** Rely on existing TacticalPosture cascade — fix inputs (D1 + timing threshold calibration), let improvements cascade through `ENEMY_POSTURE → MomentDetectionTask → GamePhaseSummariser → TacticalPosture`. No new vocabulary in scouting. Verify cascade accuracy with calibration tests.
**Alternatives:**
- Add AGGRESSIVE/DEFENSIVE to ENEMY_POSTURE — expand scouting posture vocabulary using army-near-base events. Creates a second source of truth for military posture alongside TacticalPosture, conflating build-order classification with army-movement classification.
**Rationale:** The architecture already separates economic posture (scouting layer: ALL_IN/MACRO) from military posture (summarisation layer: EARLY_AGGRESSION/DEFENSIVE_HOLD/MID_SKIRMISH/EARLY_MACRO). The TacticalPosture pipeline is the heavily-consumed system (strategy DRL, dominance DRL, CBR, commentary, advisory triggers). Duplicating military posture into scouting would violate layer separation with no consumer benefit.
**Trade-offs:** Requires verifying the cascade works correctly with real movement data — if TacticalPosture quality doesn't improve automatically, a follow-up may be needed on GamePhaseSummariser's spatial sensitivity.
**Depends on:** D1 (posture persistence — TacticalPosture receives ENEMY_POSTURE as input via MomentDetectionTask)
**Sources:** `TacticalPosture.java`, `GamePhaseSummariser.java`, `DominanceWeightAdjustment.drl`, `StarCraftStrategy.drl`, `QuarkMindCapabilityTag.java`
**Exploration:** deep-analysis
**Status:** captured

## D3: EnemyPostureClassifiedEvent cleanup

**Choice:** Remove dead event — delete `EnemyPostureClassifiedEvent`, remove the `@Inject Event<>` field and `fire()` call from `DroolsScoutingTask`
**Alternatives:**
- Keep for future wiring — ARC42STORIES.MD mentions a `StrategyTrustObserver` consumer, but that class doesn't exist. Strategy routing works via `ScoutingIntelBroker` + `TacticalPosture` instead.
**Rationale:** Dead code is noise. Zero `@Observes` consumers exist. If a future design needs a posture event, it can be re-added with the correct semantics and consumers.
**Trade-offs:** None — no consumer code is affected.
**Sources:** `EnemyPostureClassifiedEvent.java`, `DroolsScoutingTask.java:74,277`, `ARC42STORIES.MD:836`
**Exploration:** quick
**Status:** captured

## D4: Calibration methodology

**Choice:** Replay-driven SpatialCalibrationTest — a single new calibration test that replays AI Arena/IEM10 datasets and measures three metrics per replay: (1) posture UNKNOWN rate after first classification (should be near zero with D1 fix), (2) timing attack detection rate when army crosses NEAR_BASE_DISTANCE, (3) TacticalPosture transition count and stability. Run under `@Tag("benchmark")` alongside existing calibration tests.
**Alternatives:**
- Per-feature separate tests — individual calibration tests for posture, timing, and TacticalPosture. More granular but runs the same replays three times for what is fundamentally one measurement pass.
**Rationale:** One replay pass collecting all spatial metrics is more efficient and shows the correlation between features (does posture persistence improve TacticalPosture quality?). The existing calibration tests (scouting, map control, pattern classification) already pass and are unaffected — this test covers the new spatial dimensions that #298 enables.
**Trade-offs:** Single test is less granular on failure — if it fails, you need to inspect which metric broke. Acceptable for calibration tests where the output is a printed report, not a binary pass/fail.
**Depends on:** D1 (posture persistence — the UNKNOWN rate metric only makes sense after the fix)
**Sources:** `ScoutingCalibrationTest.java`, `MapControlCalibrationTest.java`, `PatternClassificationCalibrationTest.java`
**Exploration:** quick
**Status:** captured
