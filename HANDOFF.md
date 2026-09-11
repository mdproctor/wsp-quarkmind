# HANDOFF — quarkmind

## Last Session

Completed #300 (spatial recalibration). Fixed ENEMY_POSTURE stickiness — posture now caches the last DRL classification instead of reverting to UNKNOWN after the 3-minute unit buffer eviction. Removed dead `EnemyPostureClassifiedEvent` (zero consumers). Created `SpatialCalibrationTest` measuring posture UNKNOWN rate, army-near-base events, and posture transitions across 59 replays. Baseline: 0% UNKNOWN, 36-82 army events/replay, thresholds confirmed correct. Updated protocol and ARC42.

Filed epic #301 with 4 follow-up issues (#302-#305) for cascade verification, ALL_IN calibration, and MACRO→ALL_IN pivot detection.

## What's Next

Epic #301 — Spatial intelligence post-recalibration follow-ups:

| # | Title | Scale | Complexity | Blocked by |
|---|-------|-------|------------|------------|
| #302 | Verify TacticalPosture cascade with timing/rush replays | S | Low | — |
| #304 | ALL_IN replay calibration — posture persistence e2e | S | Low | — |
| #303 | GamePhaseSummariser spatial sensitivity audit | M | Med | #302 |
| #305 | Detect MACRO→ALL_IN pivot (expansion sacrifice) | M | High | — |

#302 and #304 are independent quick wins — start with either.

## References

- Spec: `docs/specs/issue-300-recalibrate-spatial/2026-09-11-spatial-recalibration-design.md`
- Decisions: `docs/specs/issue-300-recalibrate-spatial/decisions.md`
- Landing commit: `467829a`
- Epic: https://github.com/casehubio/quarkmind/issues/301
