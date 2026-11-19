# Decisions — #707 Obstacle-Aware Worker Pathfinding

## D1: How workers get obstacle-aware paths

**Choice:** Connect workers to existing A* pathfinding (Approach A)
**Alternatives:**
- Steering-based local avoidance — simpler but gets stuck in concave mineral-line formations, produces last-second swerves, doesn't plan ahead
- Precomputed path templates per patch — zero per-tick cost but rigid, doesn't adapt to building changes, is precomputation not simulation
**Rationale:** Reuses proven A* + PathfindingMovement infrastructure. Path geometry emerges from physics simulation (obstacle grid + pathfinder), not formulas or constants. Performance is a non-issue (A* on 64x64 grid is microseconds, computed once per route not per loop). Naturally produces the outbound/return path asymmetry that SC2 exhibits.
**Trade-offs:** 1-tile grid resolution may be too coarse for precise mineral-line navigation — addressable with finer grid or real SC2 pathing grid.
**Sources:** WorkerSimulation.java (moveOneLoop straight-line movement), PathfindingMovement.java (existing army A* wrapper), AStarPathfinder.java, TerrainGrid.java, MineralPatch.computeApproachDocking() (compensation stack to remove)
**Exploration:** quick
**Status:** captured
