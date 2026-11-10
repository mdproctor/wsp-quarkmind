# Design: Distance-First Worker-to-Patch Routing (#620)

## Problem

The EM matches SC2 mineral totals to 0.7% (dual-run), but 240 of 305 ticks (78.7%) have per-tick delivery mismatches. The root cause is routing priority: the EM selects patches by fewest assigned workers (load-first), while SC2 routes workers to the nearest available patch (distance-first). Different worker-patch pairings produce different walk distances and different delivery ticks.

## Change

Modify `WorkerSimulation.assignToNearestPatch()` from load-first to distance-first selection. Two-phase selection:

1. **Phase 1 — nearest available:** Find the closest patch with `miners.isEmpty()` AND `currentLoop >= lockEndLoop`. Distance is the only criterion.
2. **Phase 2 — saturation fallback:** If all patches are occupied, select the nearest patch regardless. The worker walks there and waits for the lock to expire.

### Current code (load-first, WorkerSimulation.java:536-572)

```java
for (MineralPatch p : base.patches()) {
    if (p.isDepleted()) continue;
    int    assigned = countAssignedWorkers(p, w);
    double d        = w.position.distanceTo(p.position);
    if (assigned < nearestAssigned || (assigned == nearestAssigned && d < nearestDist)) {
        nearestAssigned = assigned;
        nearestDist     = d;
        nearest         = p;
    }
}
```

### Proposed code (distance-first)

```java
MineralPatch nearestAvailable = null;
double       nearestAvailDist = Double.MAX_VALUE;
MineralPatch nearestAny       = null;
double       nearestAnyDist   = Double.MAX_VALUE;

for (MineralPatch p : base.patches()) {
    if (p.isDepleted()) continue;
    double d = w.position.distanceTo(p.position);
    boolean available = p.miners.isEmpty()
                        && currentTickStartLoop >= p.lockEndLoop;
    if (available && d < nearestAvailDist) {
        nearestAvailDist = d;
        nearestAvailable = p;
    }
    if (d < nearestAnyDist) {
        nearestAnyDist = d;
        nearestAny     = p;
    }
}
MineralPatch selected = (nearestAvailable != null) ? nearestAvailable : nearestAny;
```

## Scope

- **In scope:** The `assignToNearestPatch()` selection logic. Applies to all 7 call sites (post-deposit, transfer arrival, builder/scout return, gas rebalance, mineral sync, worker produced).
- **Out of scope:** Initial stagger model, collision model, patch position accuracy, per-loop movement physics. These are follow-up investigations if the routing change doesn't close the gap sufficiently.

## Verification

| Test | What it measures | Before | Target |
|------|-----------------|--------|--------|
| `deliveryCadenceDiagnostic` | Per-tick mismatch count (305 ticks) | 240/305 (78.7%) | Measurable reduction |
| `WorkerCadenceComparisonTest` | Per-worker cycle variance EM vs SC2 | σ mismatch | Closer match |
| `dualRunCalMineralBaseline12wAgainstSC2` | 12w mineral total | +0.7% | ≤1% (no regression) |
| `dualRunCalMineralSaturated16wAgainstSC2` | 16w saturation total | +6.7% | Observe change |
| `SC2PlaybookCalibrationTest` | Playbook economy | Passing | No regression |

## Risks

1. **Saturation curve change:** Distance-first + wait-at-nearest will cluster workers at close patches. This changes the saturation curve — may improve or worsen the +6.7% gap. Measured by the 16w dual-run test.
2. **Far patch starvation:** With distance-first, far patches get fewer workers. In SC2 this is accurate — close patches ARE preferred. But if the EM's patch positions don't match SC2's exactly, the starvation pattern may diverge.

## References

- WorkerSimulation.java:536-572 — current `assignToNearestPatch()`
- cal-worker-cadence-12w.json — SC2 per-worker cadence baseline (682 deliveries, cycles 4-6 ticks, σ=0.56)
- DualRunCalibrationTest.java:84-150 — `deliveryCadenceDiagnostic()`
- docs/sc2-physics-constants.md §14 — mining cycle breakdown, tick-boundary quantisation
- Memory: tick-alignment-priority — compensation constants forbidden
- Memory: feedback-simulation-over-formulas — simulate per-loop, never formulas
