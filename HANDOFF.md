# HANDOFF — quarkmind

## Orientation

**A* tile-center snap diagnosed and fixed (#766). Obstacle marking gap discovered.**

The +6-19% over-production on real TorchesAIE terrain was caused by A* `findPath()` snapping target positions to tile centers via integer truncation. The half-integer coordinate offset (-116.5, -151.5) between the 64×64 flat grid and 160×208 real grid shifted edge-point targets across tile boundaries, producing different walk distances. Fix: replace the final A* waypoint with the actual target position. MULE improved from +1.4% to -0.1%.

**Secondary finding:** `markCircularObstacle()` with radius 0.3 produces zero marked tiles on TorchesAIE because all mineral patch x-coordinates are integers (0.5 from any tile center in x, exceeding the 0.3 radius). The obstacle-aware A* from #707 is effectively a no-op for real SC2 maps.

**Start with:** Fix the obstacle radius so that actual path deviation emerges from the simulation instead of relying on approach animation as a flat overhead. This would also resolve the high-saturation pinned test gap.

---

## Key data (post-#766)

| Scenario | Before #766 | After #766 | Note |
|----------|------------|------------|------|
| MULE isolation | +1.4% | **-0.1%** | Improved — walks to actual edge points |
| Pinned single-base 36w | -5.0% | **-6.4%** | Slightly worse — intermediate waypoint detour at high saturation |
| Pinned two-base | -5.2% | **-8.3%** | Same mechanism as single-base |
| All tests (273) | Pass | Pass | No regressions |

**Obstacle tile diagnostic:** `AStarPathSnapDiagnosticTest` confirmed zero tiles marked for all 8 nearby patches at radius 0.3 on both grid sizes. The obstacle-aware A* is routing straight lines.

---

## What's ready

- **Tile-center snap fix (#766):** `setTargetWithPath()` replaces final smoothed waypoint with actual target
- **Diagnostic test:** `AStarPathSnapDiagnosticTest` — proves zero obstacle tiles and 4% walk distance discrepancy between grids
- **Worker distribution fix (#762):** `onWorkerProducedAtBase()`, `autoTransferEnabled` flag
- **Per-base tracking:** `CalibrationSnapshot.workersPerBase`, per-base columns in `pinnedCalibrate()`
- **13 re-recorded SC2 baselines (#764):** All include per-base `workersPerBase`
- **TorchesAIE pathing grid:** 160×208, all mineral-line tiles verified walkable
- **Wait-time absorption finding:** `tryStartMining()` floor makes absorption a no-op

---

## Investigation findings (this session)

1. **Root cause identified:** A* `findPath()` truncates `(int) to.x(), (int) to.y()` and returns tile-center waypoints `(x+0.5, y+0.5)`. Workers walk to tile centers instead of actual edge-point targets. The half-integer offset (base at 124.5, 159.5 → offset -116.5, -151.5) flips fractional tile positions, changing which tiles edge points fall in. Example: patch (125,166) has 2-tile gap on real terrain vs 3-tile gap on flat → 33% shorter walk on real terrain.

2. **Zero obstacle tiles:** All TorchesAIE mineral patches have integer x, .5 y coordinates. For `markCircularObstacle(125.0, 165.5, 0.3)`: tile (124,165) center is (124.5,165.5), distance = 0.5 > radius 0.3. No tile center is close enough to any patch center. The radius would need to be ≥ 0.5 for any tile to be marked.

3. **Fix v1 (shipped):** Replace final waypoint with actual target. Works well for most scenarios (MULE improved). Creates a small detour for diagonal walks because the FIRST waypoint in a 2+ element path is still a tile center. At high saturation (pinned 36w), this detour compounds.

4. **Fix v2 (rejected):** Skip all waypoints and walk directly to target when path ≤ 2 elements. Too aggressive — removes ALL path overhead, causing +3.6% MULE over-production.

---

## Next steps (priority order)

1. **Fix obstacle radius** — increase from 0.3 to ≥0.5 so tiles get marked for integer-x positions. With marked tiles, A* provides real path deviation (workers route around mineral patches). This would naturally resolve both the intermediate-waypoint detour issue AND the approach animation dependency. Needs careful calibration — #733 showed 2×1 rectangular footprint caused +4.1% MULE regression.

2. **Re-evaluate approach animation** — with actual obstacle tiles marked, the approach animation (20 loops) may need adjustment. Currently it absorbs ALL path deviation compensation because no tiles are marked.

3. **Re-evaluate pinned saturation curve** — with real path deviation from obstacles, the high-saturation gap (-6.4%) may improve as workers take physically correct paths around the mineral line.

4. **Wait-time absorption** — needs a different approach than `max(0, ...)`. Either threshold genuine waits or accept approach animation as fixed physics.

---

## Notes

- **Branch:** `issue-761-intent-rejection-diagnostic` (stacked: 756→757→758→761→762→763→764→766)
- **All 273 tests pass** — including DualRunCalibrationTest (46), SC2PlaybookCalibrationTest (17), WorkerSimulationTest (47)
- **SC2 is NOT running** — killed after recordings
- **Issue #766 open** — tracks the tile-center snap fix and obstacle radius follow-up
