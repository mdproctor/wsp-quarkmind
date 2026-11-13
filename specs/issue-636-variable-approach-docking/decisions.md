## D1: Contention model architecture

**Choice:** Patch state machine with wait-then-reroute
**Alternatives:**
- Contention delay function — simpler but structurally a compensation formula, not physics simulation
- Full worker decision loop — most physical but O(workers × patches) per loop, over-engineered for diminishing returns
**Rationale:** Models actual SC2 physics: workers wait at occupied patches, then reroute after calibrated threshold. Approach docking is conditional (0 loops for idle patch, 11 for just-freed). Naturally reduces collision tail since workers wait at patch instead of colliding en route.
**Trade-offs:** Two parameters to calibrate (conditional approach duration + re-routing threshold). Re-routing logic adds complexity to stepOneMineralLoop.
**Sources:** WorkerSimulation.java, MineralPatch.java, docs/sc2-physics-constants.md §3, issue #634 (uniform collision model), issue #650 (collision tail analysis)
**Exploration:** quick
**Status:** captured
