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

## D2: How obstacles get into the pathfinding grid

**Choice:** Mutable dynamic obstacle layer on TerrainGrid
**Alternatives:**
- Separate ObstacleGrid overlay — more indirection, preserves TerrainGrid immutability, but immutability isn't load-bearing here
**Rationale:** TerrainGrid is already the single walkability authority for A*. Adding markObstacle/clearObstacle keeps it as the oracle with no AStarPathfinder changes. Building/resource lifecycle events already exist and can register footprints.
**Trade-offs:** TerrainGrid in domain package becomes mutable — pragmatic, since it's already long-lived state within EmulatedGame.
**Sources:** TerrainGrid.java (immutable Height[][] grid), EmulatedGame.enforceWall() (building positions available)
**Exploration:** quick
**Status:** captured

## D3: What to do with the compensation stack

**Choice:** Remove entire compensation stack, keep only APPROACH_ANIMATION_LOOPS (20)
**Alternatives:**
- Gradual reduction — keep stack with reduced values, safer but contradicts "no compensation constants" project rule
**Rationale:** Path deviation must emerge from physics (obstacle-aware paths), not constants. Keeping compensation double-counts walk overhead. Clean removal lets us measure the gap and recalibrate only the animation constant if needed. Per project rule: "Never use compensation constants."
**Trade-offs:** Accuracy may temporarily regress until obstacle paths produce sufficient deviation. Expected — recalibration is the next step after the physics change.
**Sources:** MineralPatch.computeApproachDocking(), WorkerSimulation.effectiveApproachDocking(), docs/sc2-physics-constants.md, CLAUDE.md design rules
**Exploration:** quick
**Status:** captured

## D4: How workers integrate with PathfindingMovement

**Choice:** Waypoints from PathfindingMovement, workers keep their own moveOneLoop() physics
**Alternatives:**
- Unify all movement into PathfindingMovement — long-term cleaner but couples pathfinder with worker-specific acceleration physics, compounds scope/risk on an already L/High issue
**Rationale:** Clean separation: PathfindingMovement provides the route (waypoints), worker physics provides the movement execution (acceleration, deceleration, collision). Workers set targetPosition to next waypoint, call moveOneLoop(), pop waypoint on arrival. Minimal code change, no risk to army movement.
**Trade-offs:** Two movement systems remain (army constant-speed, worker accelerating). Unification is a valid follow-up.
**Sources:** PathfindingMovement.advance() (fixed speed parameter), SimulatedWorker.moveOneLoop() (acceleration/deceleration physics)
**Exploration:** quick
**Status:** captured

## D5: Grid resolution for mineral-line obstacles

**Choice:** Keep 1 tile/unit resolution (64x64 emulated map)
**Alternatives:**
- Increase to 2 tiles/unit (128x128) — finer obstacle representation, 4x A* cost (still microseconds), requires coordinate scaling throughout
**Rationale:** At 1 tile/unit, mineral patches are 2x2 cells with 1-2 cell gaps between them. Coarse but functional for gross obstacle avoidance. Real SC2 maps already use higher-resolution pathing grids (200x200+). If emulated map resolution proves insufficient, increase later based on measured path accuracy — per calibration-milestones-are-waypoints protocol.
**Trade-offs:** May need to revisit if paths are too coarse to produce sufficient deviation.
**Sources:** TerrainGrid.emulatedMap() (64x64), MineralPatch.patchPositions() (6.7-7.5u from TH)
**Exploration:** quick
**Status:** captured
