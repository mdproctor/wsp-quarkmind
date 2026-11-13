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

## D2: Conditional approach docking behavior

**Choice:** Binary — 0 loops for idle patch, full 11 loops for transitioning
**Alternatives:**
- Proportional to wait time — approach scales inversely with wait duration; more nuanced but unobservable from data, harder to calibrate
- Always full approach — keep 11 loops for all cases; simpler but doesn't capture the uncontested→contested timing delta
**Rationale:** Directly matches SC2 observation data: uncontested cycles are exactly 66 loops (walk+approach+mine), contested are 88 loops (+22 docking overhead). The 11-loop approach is the docking animation that plays when a worker aligns with the patch after waiting. Binary model — no formula, no compensation.
**Trade-offs:** Assumes SC2's approach timing is purely binary (idle vs not-idle). If SC2 has intermediate states, this won't capture them — but observation data shows clean 66/88 split.
**Sources:** docs/sc2-physics-constants.md §3 (SC2 observed cycle phases table), #631 per-worker cadence data (1366 transitions)
**Exploration:** quick
**Depends on:** D1 (contention model architecture)
**Status:** captured
