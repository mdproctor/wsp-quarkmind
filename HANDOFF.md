# HANDOFF — quarkmind

## Orientation for Next Session

**#734, #736, #733 landed** this session. Terrain defaults, rectangular building footprints, enforceWall conversion. #735 (mineral patch orientation) and #737 (edgePoint rectangle) also closed — deferred work tracked by existing queue items.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Key changes landed this session:**

1. **EmulatedGame default terrain (#734):** Constructor wires `TerrainGrid.emulatedMap()`. All tests get obstacle-aware worker pathfinding. Fog decoupled into `fogEnabled` flag (default false).

2. **Rectangular building footprints (#736):** `SC2Data.buildingFootprint(BuildingType)` returns `int[2]` tile dimensions. `enforceWall()` uses `insideFootprint()` rectangular check. `buildingRadius()` deprecated.

3. **Physics doc revised:** §13 rewritten for rectangular footprints. §14 stale PATH_DEVIATION references removed. §15 updated with terrain/fog/collision details. §19 dead constants struck.

**Key finding:** Converting mineral patch obstacles from circular (0.3 radius) to rectangular (2×1) caused +4.1% MULE regression — the rectangle blocks too many tiles. The 0.3 radius was empirically calibrated. Orientation (#735) might fix this — an oriented 2×1 (long axis tangent to mineral arc) blocks 1 tile in the approach direction, closer to the calibrated circle.

**Accuracy state unchanged** — building footprint change affects army collision (enforceWall), not worker economy. Mineral patches reverted to circular.

---

## Completed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #734 | Wire TerrainGrid into all economy tests | Done — constructor default, fog decoupled |
| #736 | SC2Data.buildingFootprint() tile dimensions | Done — static lookup, enforceWall wired, buildingRadius deprecated |
| #733 | Replace circular approximations | Done — building collision converted; mineral patches deferred |
| #735 | Mineral patch orientation | Closed — deferred pending calibration |
| #737 | edgePoint rectangle | Closed — blocked by #735 |

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #738 | EVOLUTION_CHAMBER footprint verification | XS/Low | May be 3×3 not 2×2 |
| #739 | Remove dead APPROACH_DOCKING_LOOPS config | XS/Low | Config property is live but value immediately overwritten |
| #740 | buildingFootprint() shared mutable int[] | XS/Low | Should use immutable record |

**Blog entries:** `2026-11-20-mdp01-rectangles-not-radii.md` — building collision from circular to rectangular, fog decoupling, mineral patch calibration finding

**Published:** project + workspace pushed to origin

---

## Queue (open items)

| # | Issue | Scale | What |
|---|-------|-------|------|
| 1 | **#731** | M/Med | **ACTIVE** — Army pathfinding around buildings |
| 2 | #732 | M/Med | Army pathfinding around resources |
| 3 | #730 | S/Low | Blog SC2/EM column swap audit |
| 4 | #738 | XS/Low | EVOLUTION_CHAMBER footprint verification |
| 5 | #739 | XS/Low | Remove dead APPROACH_DOCKING_LOOPS config |
| 6 | #740 | XS/Low | buildingFootprint() shared mutable int[] |

**On branch `issue-731-army-pathfinding-buildings`** — `work continue` to begin. M/Med task — brainstorm before implementing.
