# HANDOFF — quarkmind

## Last Session

Completed #296 (workbench tabs). All four tabs show live data with TDD coverage. Discovered #298 (static enemy positions in replay) and filed #299 (smoke tests). Advanced .plan to #298.

## Next: #298 — Replay engine static positions

**Root cause:** `ReplayEngine` extracts enemy unit positions from tracker birth events but never applies position updates from subsequent tracker snapshots. All enemy units stay at their creation position.

**Evidence:**
```
Frame 42: 16 enemies — DRONE at (127.0, 162.0)
Frame 62: 20 enemies — same DRONE still at (127.0, 162.0)
NO units changed position in 20 frames
```

**What to fix:**
1. SC2 replay tracker data contains periodic `NNet.Replay.Tracker.SUnitPositionsEvent` entries with updated positions for all units. The replay parser needs to apply these.
2. Check `ReplayEngine.java` and the replay parser (`s2protocol` or Scelight-based) for how unit state is tracked per loop.
3. After fix: enemy units should move on-screen during replay playback.

**After fix — recalibrate (#299 smoke tests first):**
- Write smoke test: assert enemy positions change across 10 frames
- Write smoke tests for each pipeline stage (pattern, strategy, moments, commentary, WebSocket)
- Re-run calibration suite: `PatternClassificationCalibrationTest`, `ScoutingCalibrationTest`, `MapControlCalibrationTest`
- Review posture/timing/threat results with real movement data

**What's NOT broken:** Pattern classification by type/count (70% accuracy target valid). Only spatial features are affected.

## Branch state

Branch `issue-296-replay-workbench-cascade-empty` has uncommitted upstream compat fixes (CLAUDE.md auto-update). The branch will need a new name for #298 work, or continue on the same branch if scope overlaps.

**Test compilation:** Main sources compile. Test sources have remaining `ScoredCbrCase` and `SC2CbrRetentionObserverTest` errors from upstream API changes — fix those before running full test suite.

## .plan

```
[x] #296 — Workbench tabs (done)
[ ] #298 — Replay engine static positions ← active
[ ] #299 — Replay smoke tests
```

## References

- `quarkmind-sc2/.../replay/ReplayEngine.java` — replay loop, unit state
- `quarkmind-sc2/.../replay/ReplayCommandExtractor.java` — replay event parsing
- Issue #298: https://github.com/casehubio/quarkmind/issues/298
- Issue #299: https://github.com/casehubio/quarkmind/issues/299
