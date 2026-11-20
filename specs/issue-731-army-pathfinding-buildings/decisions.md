## D1: Obstacle registration timing

**Choice:** Register buildings as obstacles on placement start (incomplete building)
**Alternatives:**
- On completion only — simpler, matches current enforceWall isComplete() check, but allows army to path through buildings under construction
**Rationale:** SC2 blocks pathing from the moment construction starts. The building tiles are occupied immediately. Matching this produces correct army routing throughout the build lifecycle.
**Trade-offs:** enforceWall() currently only checks `isComplete()` for building entry — this means the obstacle layer will block A* routing before enforceWall would have rejected the move. This is correct (the obstacle prevents the path from being planned through the building at all), and enforceWall remains as a safety net for edge cases.
**Sources:** SC2 game mechanics (buildings block pathing immediately), EmulatedGame.java:handleBuild() (placement lifecycle), EmulatedGame.java:enforceWall() (current isComplete check)
**Exploration:** quick
**Status:** captured

## D2: Repath scope on building change

**Choice:** Invalidate all army unit paths when a building is placed or destroyed
**Alternatives:**
- Intersection-filtered — only invalidate units whose cached path crosses the new/removed footprint. More precise but adds path-geometry intersection code for a scenario that happens infrequently.
**Rationale:** Army units are few (~20-50). One A* recompute per unit on a 64x64 grid is sub-millisecond. Building events are rare (~1 per 30s). The cost of invalidating all paths is negligible; the complexity of path-footprint intersection testing is not justified.
**Trade-offs:** A brief spike of A* recomputes on building placement. At 50 units on a 64x64 grid, this is well under 1ms total — invisible at game-loop granularity.
**Sources:** PathfindingMovement.java:invalidatePath() (existing per-unit invalidation), AStarPathfinder.java (8-directional A* on 64x64 grid)
**Exploration:** quick
**Status:** captured

## D3: Registration architecture

**Choice:** Lifecycle hooks directly in EmulatedGame building methods
**Alternatives:**
- Extracted BuildingObstacleManager — cleaner separation but adds a class for ~20 lines of logic. No event/listener pattern exists for buildings, so adding one is YAGNI.
- TerrainGrid self-registers from PlayerState — couples domain to simulation state. Per-call scanning is expensive. Worst option.
**Rationale:** The building lifecycle methods in EmulatedGame (handleBuild, handleBuildingBorn, handleBuildingDied, etc.) are already the right integration points. The obstacle API is a one-liner per call site. No new abstractions needed.
**Trade-offs:** Slightly increases EmulatedGame's responsibility, but the registration calls are trivially small and directly coupled to the building lifecycle they serve.
**Sources:** EmulatedGame.java:handleBuild() (line 764), EmulatedGame.java:handleBuildingBorn() (line 1806), EmulatedGame.java:handleBuildingDied() (line 1823), TerrainGrid.java:markRectangularObstacle() (line 75)
**Exploration:** quick
**Status:** captured
