# HANDOFF — quarkmind

## Orientation

**A* grid discrepancy fully diagnosed and fixed (#766). Three changes landed, all tests green.**

The +6-19% over-production on real TorchesAIE terrain had two root causes, both fixed:
1. **Tile-center waypoint snapping:** `setTargetWithPath()` replaced final A* waypoint with actual target position
2. **Zero obstacle tiles:** Obstacle radius increased from 0.3 to 0.5 (integer-x patch positions need ≥0.5 to mark tiles). Required decoupling `enforceWall()` from the obstacle layer so mineral obstacles don't block army movement.

12w dual-run improved from +4.8% (pre-fix, real terrain) to **+1.0%**. MULE from +1.4% to **-0.7%**. All 287 tests pass.

---

## Key data (post-#766, all three fixes)

| Scenario | Original | After fix | Note |
|----------|---------|-----------|------|
| 12w dual-run (real terrain) | +4.8% | **+1.0%** | Primary calibration target |
| MULE isolation | +1.4% | **-0.7%** | Improved |
| Pinned single-base 36w | -5.0% | **-9.3%** | Worse at high saturation — contention model gap |
| Pinned two-base | -5.2% | **-8.2%** | Similar |
| All tests | 273 pass | 287 pass | No regressions |

---

## Changes made (this session)

1. **Tile-center waypoint snap fix** (`WorkerSimulation.setTargetWithPath`): Replace final smoothed A* waypoint with actual target position. Workers walk to intended edge-point destinations instead of nearest tile center.

2. **Obstacle radius 0.3→0.5** (`WorkerPhysicsConfig.DEFAULT_MINERAL_PATCH_OBSTACLE_RADIUS`): TorchesAIE patches have integer x-coordinates (0.5 from tile centers). At 0.3, zero tiles marked → obstacle-aware A* was a no-op. At 0.5, each patch marks 2 tiles.

3. **enforceWall decoupling** (`EmulatedGame.enforceWall`): Changed from `!isWalkable()` (terrain+obstacles) to `heightAt()==WALL` (terrain only). Mineral patch obstacle tiles now only affect worker A* routing, not army combat movement. Building collision still enforced via `insideFootprint()` for all buildings.

4. **Diagnostic test** (`AStarPathSnapDiagnosticTest`): Proves zero obstacle tiles at radius 0.3 and 4% walk distance discrepancy between grids from tile-center snapping.

---

## Next steps (priority order)

1. **Re-evaluate approach animation** — with actual obstacle tiles now marked (radius 0.5), the 20-loop approach animation may need adjustment. The 12w is at +1.0% (slight over-production), suggesting the approach animation could increase by 1-2 loops. But this should be calibrated from SC2 observation, not adjusted as compensation.

2. **High-saturation contention model** — pinned single-base at -9.3% (36w on 8 patches). The EM's collision/contention model at 4.5 workers/patch produces less income than SC2. This gap predates #766 but widened slightly. Needs investigation of SC2 high-saturation worker behavior.

3. **Wait-time absorption** — `tryStartMining()` floor of `APPROACH_ANIMATION_LOOPS` makes absorption a no-op. Needs threshold-based approach (distinguish brief shuffle-order waits from genuine contention waits).

---

## Notes

- **Branch:** `issue-761-intent-rejection-diagnostic` (stacked: 756→757→758→761→762→763→764→766)
- **All 287 tests pass**
- **SC2 is NOT running**
- **Issue #766 open** — tracks all three changes
