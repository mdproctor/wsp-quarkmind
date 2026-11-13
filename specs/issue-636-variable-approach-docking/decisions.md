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

## D3: Re-routing trigger and behavior

**Choice:** Wait-then-scan with travel-time comparison
**Alternatives:**
- Fixed threshold, always re-route — simpler but causes workers to walk to far patches when current patch is about to free up
- Queue-position-based — re-route if queue > 1; adds queue tracking infrastructure without clear SC2 correspondence
**Rationale:** Workers only switch when it's genuinely faster (travel time to idle patch < estimated remaining wait). Prevents pathological ping-ponging. Wait threshold is the single calibration parameter; the comparison itself is a cheap distance calculation.
**Trade-offs:** Requires estimating "remaining wait time" at the current patch — depends on knowing how long the active miner has left. Estimation may be imprecise, but the alternative (no comparison) is worse.
**Sources:** WorkerSimulation.java (stepOneMineralLoop busy-wait pattern), MineralPatch.java (lockEndLoop timing)
**Exploration:** quick
**Depends on:** D1 (contention model architecture)
**Status:** captured

## D4: Calibration approach

**Choice:** Diagnostic test analyzing #622 worker snapshots, fall back to new SC2 dual-run if insufficient
**Alternatives:**
- Sweep-based calibration — sweep wait threshold against dual-run baselines; risks compensation-constant territory
**Rationale:** #622 per-worker snapshot data (position + orders + buffs at every tick) already exists. Re-routing events should be detectable as workers changing nearest mineral patch between consecutive delivery cycles. Free data before new SC2 runs. If per-tick resolution is insufficient for re-routing detection, fall back to a dedicated SC2 dual-run with contention instrumentation.
**Trade-offs:** Per-tick position data may lack the per-loop resolution needed to measure exact wait-before-reroute timing. Fallback to SC2 dual-run adds a recording step.
**Sources:** Issue #622 (per-worker SC2 baseline), WorkerSnapshotEmTest.java, DualRunCalibrationTest.java
**Exploration:** quick
**Depends on:** D3 (re-routing trigger — defines what we're calibrating)
**Status:** captured
