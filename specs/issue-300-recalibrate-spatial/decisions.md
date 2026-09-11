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
