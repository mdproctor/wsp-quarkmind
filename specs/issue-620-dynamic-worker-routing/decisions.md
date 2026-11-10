# Decisions — #620 Dynamic Worker Routing

## D1: Design goal — match SC2 routing exactly

**Choice:** Match SC2's distance-first patch routing, then measure per-tick improvement
**Alternatives:**
- Per-tick delivery match (optimise for metric regardless of mechanism) — risks overfitting to 12w baseline
- Both (match then verify) — over-scopes for a focused change
**Rationale:** The dual-run shows 0.7% physics error — the problem is routing, not physics. Matching SC2's mechanism directly addresses root cause.
**Trade-offs:** If SC2's routing is more complex than "nearest available", we'll discover it from the measurement delta
**Sources:** docs/sc2-physics-constants.md §14, cal-worker-cadence-12w.json, WorkerSimulation.java:536-572
**Exploration:** quick
**Status:** captured

## D2: Routing rule — nearest truly-available patch

**Choice:** Distance-first among patches with no active miner AND expired lock
**Alternatives:**
- Nearest patch regardless of lock (worker queues) — SC2 workers do queue, but primarily at available patches
- Need more data — deferred; measure impact first, refine if needed
**Rationale:** SC2 workers visibly route to unoccupied patches. The cadence data shows workers spread across patches with low cycle variance (σ=0.56), consistent with distance-first to available patches rather than queuing.
**Trade-offs:** May miss SC2 queuing behavior at saturation; addressed by D4
**Sources:** cal-worker-cadence-12w.json analysis (682 deliveries, cycles 4-6 ticks), WorkerSimulation.java:536-572
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D3: Scope — routing change only, measure, iterate

**Choice:** Change assignToNearestPatch() only, measure cadence impact, file follow-ups
**Alternatives:**
- Full cadence package (routing + stagger + collision + patch positions) — larger scope, harder to attribute
- Routing + initial stagger — two changes but stagger is independent
**Rationale:** The routing change is small (~10 lines) and directly testable. Measuring before adding more changes tells us how much each factor contributes.
**Trade-offs:** May not reach 90%+ match from routing alone; follow-ups needed
**Sources:** CLAUDE.md tick-alignment priority, memory: tick-alignment-priority
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D4: Saturation fallback — wait at nearest patch

**Choice:** When all patches occupied, worker walks to nearest patch and waits for lock
**Alternatives:**
- Fall back to load-balanced — different from SC2, prevents starvation of far patches
- Hybrid (nearest if close, load-balance if far) — unnecessary complexity before measurement
**Rationale:** Matches SC2 behavior — workers cluster at close patches under saturation, creating natural contention. May help fix the +6.7% saturation curve error.
**Trade-offs:** Far patches may be under-utilised at saturation; SC2 has this same behavior
**Sources:** docs/sc2-physics-constants.md §6 saturation curve, WorkerSimulation.java:280-288
**Exploration:** quick
**Depends on:** D2
**Status:** captured

## D5: Call sites — all use distance-first

**Choice:** Apply distance-first routing to all 7 call sites of assignToNearestPatch()
**Alternatives:**
- Post-deposit only — minimal change but creates inconsistent routing between call sites
**Rationale:** SC2 routes distance-first in all cases. Consistent behavior prevents subtle bugs from different routing at different call sites.
**Trade-offs:** None significant — all sites should use the same routing logic
**Sources:** WorkerSimulation.java (lines 351, 362, 620, 639, 675, 768, 799)
**Exploration:** quick
**Depends on:** D2
**Status:** captured
