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
