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

## Changes made (this session, 2026-11-24)

1. **Tile-center waypoint snap fix** (`WorkerSimulation.setTargetWithPath`, line 128): Replace final smoothed A* waypoint with actual target position. Workers walk to intended edge-point destinations instead of nearest tile center. Single-line change: `smoothed.set(smoothed.size() - 1, target)`.

2. **Obstacle radius 0.3→0.5** (`WorkerPhysicsConfig.DEFAULT_MINERAL_PATCH_OBSTACLE_RADIUS`): TorchesAIE patches have integer x-coordinates (0.5 from tile centers). At 0.3, zero tiles marked → obstacle-aware A* was a no-op. At 0.5, each patch marks 2 tiles.

3. **enforceWall decoupling** (`EmulatedGame.enforceWall`, line 331): Changed from `!isWalkable()` (terrain+obstacles) to `heightAt()==WALL` (terrain only). Removed the `isWalkable` skip for building footprint checks (lines 342-343, 351-352 deleted). Mineral patch obstacle tiles now only affect worker A* routing, not army combat movement. Building collision still enforced via `insideFootprint()` for all buildings.

4. **Diagnostic test** (`AStarPathSnapDiagnosticTest`): `@Tag("diagnostic")`, run with `-Pdiagnostic`. Proves zero obstacle tiles at radius 0.3 and 4% walk distance discrepancy between grids from tile-center snapping.

---

## Approaches tried and rejected

These are dead ends — don't re-explore:

1. **Fix v2 (direct walk for short paths):** Skipped all A* waypoints when smoothed path ≤ 2 elements, walking directly from edge point to edge point. Too aggressive — removed ALL walk overhead. MULE jumped to +3.6% (over-production). The intermediate tile-center waypoint in multi-tile A* paths provides a small beneficial path deviation that approximates SC2's curved worker paths.

2. **Approach animation = 23:** Tried increasing approach from 20 to 23 to compensate for shorter walks after the waypoint fix. Caused MULE isolation to crash to -10.9% because the increased approach only affects regular workers, not MULEs (MULEs use `MULE_MINING_OVERHEAD_LOOPS`). Regular worker mining and MULE mining constants must stay independently calibrated.

3. **Approach animation = 22:** Only shifted dual-run divergence by 2 ticks — not enough. The approach animation is a physical constant (measured at 20 loops from SC2 observation data, #679), not a compensation lever.

---

## Architectural understanding (key for next session)

### How the obstacle layer interacts with the two movement systems

- **Worker pathfinding:** Uses `AStarPathfinder.findPath()` which calls `terrainGrid.isWalkable()` — checks BOTH terrain height AND obstacle layer. Workers route around mineral patch obstacles. The final waypoint is replaced with the actual target so workers stop at edge points, not tile centers.

- **Army movement enforcement:** Uses `enforceWall()` which now checks `terrainGrid.heightAt() == WALL` — checks ONLY terrain height, NOT obstacle layer. Army units pass through mineral patch obstacle tiles freely. Buildings are caught by `insideFootprint()` checks for all buildings (no skip).

- **Why this matters:** If you change `enforceWall()` back to use `isWalkable()`, the kiting test will fail (Zealot blocked by mineral patch obstacle tile at (8,14) from default patch at (8.90,14.64)).

### Why zero obstacle tiles at radius 0.3

All TorchesAIE mineral patches have integer x-coordinates (25.0, 53.0, 125.0, etc.) and .5 y-coordinates (165.5, 159.5, etc.). For `markCircularObstacle(125.0, 165.5, 0.3)`:
- Nearest tile center in x: 124.5 or 125.5 — both 0.5 away, exceeding radius 0.3
- `dx² + dy² = 0.25 + 0.0 = 0.25 > 0.09 (0.3²)` → no tile marked

At radius 0.5: `dx² + dy² = 0.25 ≤ 0.25 (0.5²)` → boundary case, included by `<=`.

### Why the offset creates different walk distances (the root cause)

DualRunCalibrationTest offsets coordinates by `(8f - basePosition[0], 8f - basePosition[1])` = `(-116.5, -151.5)`. This is a HALF-INTEGER offset. It flips x coordinates from integer (tile boundary) to .5 (tile center) and y from .5 to .0 (tile boundary). When A* truncates `(int) target.x()`, the same edge-point target lands in different tiles on the two grids, producing different tile-center destinations and therefore different walk distances.

### The intermediate waypoint effect

After the waypoint fix, the smoothed A* path is typically `[intermediate_tile_center, actual_target]`. The worker walks: position → intermediate tile center → actual target. For horizontal walks this adds zero detour. For diagonal walks it adds ~4% detour. This small detour provides beneficial walk overhead that partially compensates for the EM's straight-line movement (SC2 workers curve). At high saturation (36w pinned test), this detour compounds, contributing to the -9.3% gap.

---

## Next steps (priority order)

1. **High-saturation contention model** — pinned single-base at -9.3% (36w on 8 patches). The EM's collision/contention model at 4.5 workers/patch produces less income than SC2. This gap predates #766 but widened. Needs investigation of SC2 high-saturation worker behavior — record a 36w SC2 baseline with per-worker cadence instrumentation.

2. **Approach animation re-evaluation** — the 12w is at +1.0% (slight over-production). The approach animation (20 loops) may need adjustment now that obstacle tiles are marked and providing real path deviation. But this should be calibrated from SC2 observation, not adjusted as compensation.

3. **Wait-time absorption** — `tryStartMining()` floor of `APPROACH_ANIMATION_LOOPS` makes absorption a no-op. Needs threshold-based approach (distinguish brief shuffle-order waits from genuine contention waits).

---

## Notes

- **Branch:** `issue-761-intent-rejection-diagnostic` (stacked: 756→757→758→761→762→763→764→766)
- **All 287 tests pass** — DualRunCalibrationTest (46), SC2PlaybookCalibrationTest (17), WorkerSimulationTest (47), EmulatedGameTest (128)
- **SC2 is NOT running** — killed after recordings in previous session
- **Issue #766 open** — tracks all three changes
- **Worktree cleanup:** `/Users/mdproctor/claude/casehub/quarkmind/.claude/worktrees/agent-ace0ecd96715c77a7` can be removed — its baseline commit was cherry-picked into the main branch
