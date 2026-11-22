# HANDOFF — quarkmind

## Orientation for Next Session

**#731 and #732 landed** this session. Army pathfinding now routes around buildings and vespene geysers via the obstacle layer on `TerrainGrid`. The architectural shift: from reactive `enforceWall` rejection loops to proactive A* routing around obstacles.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**. §15 has the army building/resource obstacle entries.

**Key changes landed this session:**

1. **Army building obstacles (#731):** `registerBuildingObstacle()`/`clearBuildingObstacle()` wired into all 8 building lifecycle methods. Buildings register from placement start (incomplete buildings block pathing, matching SC2). `invalidateAllPaths()` added to `MovementStrategy`. `enforceWall()` `isComplete()` guard dropped — both complete and incomplete buildings block entry. `clearAllObstacles()` added to `resetInternal()`.

2. **Resource obstacles (#732):** Vespene geysers registered as 3×3 rectangular obstacles (`GEYSER_OBSTACLE_HALF_EXTENT = 1.5f`) in `setGeyserResources()`. Mineral patches already registered (radius 0.3) by `WorkerSimulation` on the shared grid.

3. **Physics doc revised:** §15 updated with army building obstacles (#731) and resource obstacles (#732).

**Key design decisions:**
- Register on placement start, not completion — matches SC2 behavior
- Invalidate all army paths on building change — simple, correct at current unit counts
- `enforceWall` stays as safety net — almost never triggers now
- Geyser footprint assumed 3×3 — not empirically verified (#742)

**Accuracy state unchanged** — building/resource obstacles affect army pathfinding only. Worker economy paths use the same shared `TerrainGrid` but mineral patch obstacles (radius 0.3) are unchanged.

---

## Completed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #731 | Army pathfinding: register buildings as obstacles | Done — 2 commits after squash (invalidateAllPaths + obstacle wiring) |
| #732 | Army pathfinding: mineral patches and geysers as obstacles | Done — 1 commit (geyser registration) |

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #742 | Verify geyser obstacle footprint against SC2 data editor | XS/Low | 3×3 assumed, not verified |
| #743 | Optimise enforceWall: skip building iteration when obstacle layer active | S/Low | O(units × buildings) per tick, almost never triggers |

**Blog entries:** `2026-11-22-mdp01-pathfinder-that-couldnt-see-buildings.md` — reactive vs proactive pathfinding, the obstacle layer architecture, with SVG diagram

**Published:** project + workspace pushed to origin

---

## Queue (open items)

| # | Issue | Scale | What |
|---|-------|-------|------|
| 1 | #730 | S/Low | Blog SC2/EM column swap audit |
| 2 | #738 | XS/Low | EVOLUTION_CHAMBER footprint verification |
| 3 | #739 | XS/Low | Remove dead APPROACH_DOCKING_LOOPS config |
| 4 | #740 | XS/Low | buildingFootprint() shared mutable int[] |
| 5 | #742 | XS/Low | Verify geyser obstacle footprint |
| 6 | #743 | S/Low | Optimise enforceWall building iteration |

`work continue` to pick up the next item. All remaining items are XS-S/Low — a session of cleanup work.
