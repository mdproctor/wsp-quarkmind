# Variable Approach Docking — Contention-Based Mining Delay & Re-Routing

**Issue:** #636 (re-scoped post-#634)
**Branch:** issue-636-variable-approach-docking
**Date:** 2026-11-13
**Parent:** #617 (Foundational tick-alignment audit)

## Problem

The EM's mining simulation uses a fixed 11-loop approach/docking delay regardless of patch state. In SC2, workers arriving at an idle patch start mining immediately, while workers arriving at an occupied patch wait and may re-route to an idle patch. The EM's fixed timing produces:

- **Per-tick cadence mismatch: 75.3%** (229/304 ticks differ in 12w scenario)
- **Cycle variance σ=0.90** vs SC2's σ=0.58 (EM too variable — collision stacking tail, not insufficient contention)
- **12w mineral over-production: +7.2%** (straight-line walks faster than SC2 path-dependent routing)

The original issue description targeted increasing variance (σ≈0.5→σ≈3+). This was based on pre-#634 numbers — the measurement bug inflated SC2 σ to 2.54. Corrected SC2 σ=0.58 shows mining cycles are extremely uniform. The EM already has MORE variance than SC2. This design re-scopes the issue to reduce cadence mismatch through contention-based approach timing and dynamic patch re-routing.

## Design

### Patch state awareness

`MineralPatch` gains an `isIdle(double currentLoop)` helper that derives state from existing fields:

| State | Condition | Worker behavior on arrival |
|-------|-----------|---------------------------|
| IDLE | `miners.isEmpty() && currentLoop >= lockEndLoop` | Start mining immediately (0 approach loops) |
| ACTIVE | `!miners.isEmpty()` | Wait, increment per-worker wait counter |
| COOLDOWN | `miners.isEmpty() && currentLoop < lockEndLoop` | Wait for lock to expire |

No new enum — state is derived from `miners` list and `lockEndLoop`. This avoids adding state management that could diverge from the existing fields.

### Conditional approach docking

When a worker begins mining at a patch, the approach docking duration depends on how the worker got access:

- **Patch was IDLE when worker arrived** → `phaseEndLoop = currentLoop + MINING_DURATION_LOOPS` (0 approach loops — worker starts immediately)
- **Patch transitioned to available while worker was waiting** → `phaseEndLoop = currentLoop + approachDockingLoops + MINING_DURATION_LOOPS` (full 11-loop docking animation)

This matches SC2 observation data: uncontested cycles are exactly 66 loops (walk 33 + approach 11 + mine 22), contested cycles are 88 loops (+22 docking overhead). The 11-loop approach represents the docking animation that plays when a worker aligns with the patch after contention. Workers arriving at an empty patch skip this animation.

Implementation in `tryStartMining()`:
```java
private void tryStartMining(SimulatedWorker w, double currentLoop) {
    if (w.assignedPatch.miners.size() < MineralPatch.MAX_MINERS
        && currentLoop >= w.assignedPatch.lockEndLoop) {
        boolean wasWaiting = w.waitingSinceLoop > 0;
        w.assignedPatch.miners.add(w);
        double approach = wasWaiting ? w.assignedPatch.approachDockingLoops : 0;
        w.phaseEndLoop = currentLoop + approach + SimulatedWorker.MINING_DURATION_LOOPS;
        w.waitingSinceLoop = 0;
    }
}
```

### Wait-then-reroute

New per-worker field: `double waitingSinceLoop` (default 0, meaning not waiting).

When a worker arrives at its assigned patch and it is ACTIVE or COOLDOWN:
1. Set `waitingSinceLoop = currentLoop` (first arrival only — don't reset on subsequent loops)
2. Each loop while waiting, check: `currentLoop - waitingSinceLoop > rerouteThresholdLoops`
3. If threshold exceeded, scan for idle patches at the same base:
   - Estimate travel time to each idle patch (distance / worker speed)
   - Estimate remaining wait at current patch (`phaseEndLoop` of active miner - `currentLoop`, or `lockEndLoop - currentLoop` for COOLDOWN)
   - If travel time to nearest idle patch < remaining wait → reassign `assignedPatch`, set new `targetPosition`, clear `waitingSinceLoop`
   - If no better option exists → keep waiting (re-checks each loop — O(patches) per waiting worker is trivial)
4. `waitingSinceLoop` is cleared when mining starts or when the worker is reassigned

The `rerouteThresholdLoops` is a config property (`starcraft.worker.reroute-threshold-loops`) with a default value determined by the calibration phase.

```java
private MineralPatch findBetterPatch(SimulatedWorker w, double currentLoop) {
    BaseEntry base = bases.get(w.baseIndex);
    MineralPatch best = null;
    double bestScore = estimatedWait(w.assignedPatch, currentLoop);
    for (MineralPatch p : base.patches()) {
        if (p == w.assignedPatch || p.isDepleted()) continue;
        if (!p.isIdle(currentLoop)) continue;
        double travelTime = w.position.distanceTo(miningTarget(p, w.baseIndex))
            / (SimulatedWorker.WORKER_SPEED / SC2Data.GAME_LOOPS_PER_SECOND);
        if (travelTime < bestScore) {
            bestScore = travelTime;
            best = p;
        }
    }
    return best;
}
```

### Interaction with existing mechanics

**Collision model:** Workers waiting at a patch are stationary — they don't participate in walking collisions. This should naturally reduce the 7+ tick collision stacking tail (#650) since fewer workers are walking simultaneously.

**Patch docking overhead (22 loops):** Unchanged. The lock after a miner departs still holds for 22 loops. The contention model adds behavior on top of this — waiting workers see the COOLDOWN state and wait for lock expiry.

**Worker assignment (`assignToNearestPatch`):** Initial assignment is unchanged. Re-routing creates a secondary, dynamic reassignment during the mining loop. The `assignedPatch` field is the canonical assignment — re-routing updates it permanently.

**Per-loop shuffle:** The shuffle order (#634) still applies. Workers are processed in random order each loop, so which worker gets to start mining first at a just-freed patch is non-deterministic. This is correct — SC2 also randomizes processing order.

## Calibration phase

### Phase 1: Analyze existing #622 worker snapshot data

Write `ContentionCalibrationDiagnosticTest` (`@Tag("diagnostic")`) that:

1. Loads per-worker SC2 snapshot data from #622 (position + orders + buffs at every tick)
2. Detects re-routing events: workers that change their nearest mineral patch between consecutive delivery cycles (worker delivers at base, then heads toward a different patch than their previous cycle)
3. Measures re-routing frequency: what percentage of cycles involve a patch change?
4. Measures wait-before-reroute timing if observable from position data (worker stationary near a patch for N ticks before moving to another)
5. Reports per-worker cycle breakdown: uncontested (immediate start), contested (waited then mined), re-routed (waited then switched)

### Phase 2: Set initial threshold

From diagnostic findings, set `rerouteThresholdLoops` default:
- If re-routing is frequent (>10% of cycles), threshold is short (~11-22 loops)
- If re-routing is rare (<5% of cycles), threshold is longer (~33-44 loops)
- If snapshot data is insufficient for measurement, fall back to a new SC2 dual-run with explicit contention instrumentation

## Files changed

| File | Change |
|------|--------|
| `SimulatedWorker.java` | Add `waitingSinceLoop` field, `REROUTE_THRESHOLD_LOOPS` static field, `resetWaiting()` method |
| `MineralPatch.java` | Add `isIdle(double currentLoop)` method |
| `WorkerSimulation.java` | Modify `stepOneMineralLoop` — conditional approach on arrival + wait counter + reroute check; modify `tryStartMining` — conditional approach duration based on `wasWaiting`; add `findBetterPatch()` + `estimatedWait()` methods |
| `WorkerPhysicsConfig.java` | Add `rerouteThresholdLoops` config property with default |
| New: `ContentionCalibrationDiagnosticTest.java` | `@Tag("diagnostic")` — analyze #622 snapshots for re-routing events and contention timing |
| `WorkerSimulationTest.java` | Tests: worker arrives at idle patch (0 approach), worker arrives at occupied patch (waits, full approach on transition), worker reroutes to idle patch when wait exceeds threshold, worker stays when no better option, re-routing is permanent reassignment |
| `DualRunCalibrationTest.java` | Verify cadence mismatch improvement post-implementation |
| `docs/sc2-physics-constants.md` | Update §3: conditional approach docking, re-routing threshold, contention model description |

## Acceptance criteria (re-scoped)

- [ ] Approach duration varies per-worker based on patch state (0 for idle, 11 for transitioning)
- [ ] Workers waiting at occupied patches re-route to idle patches when travel time < remaining wait
- [ ] `rerouteThresholdLoops` calibrated from #622 data or SC2 dual-run
- [ ] Per-tick cadence mismatch drops below 65% (from 75.3%)
- [ ] Cycle variance σ ≤ 0.75 (from 0.90, toward SC2's 0.58)
- [ ] 12w dual-run total stays within ±5% at tick 304
- [ ] No regression in `SC2PlaybookCalibrationTest`

## What this does NOT fix

- The 7.2% over-production from straight-line routing (#645) — re-routing may partially help but the structural cause is path-dependent navigation vs straight-line walks
- The collision stacking tail (#650) — may improve indirectly (fewer walking workers = fewer collisions) but not targeted
- Gas harvesting contention — separate mechanic with different physics

## References

- `WorkerSimulation.java:314` — `stepOneMineralLoop` (contention busy-wait pattern)
- `WorkerSimulation.java:451` — `tryStartMining` (fixed approach docking)
- `MineralPatch.java:18-28` — `APPROACH_DISTANCE_SCALE` (disabled), `approachDockingLoops` field
- `SimulatedWorker.java:41` — `APPROACH_DOCKING_LOOPS` constant (11.0)
- `docs/sc2-physics-constants.md` §3 — SC2 observed cycle phases, calibration history
- Issue #634 — uniform collision model, per-loop shuffle, measurement bug fix
- Issue #650 — collision stacking tail analysis (σ gap)
- Issue #622 — per-worker SC2 baseline (position + task at every tick)
- Issue #631 — per-worker cadence data (1366 transitions, 66/88 loop split)
- Issue #620 — mining cycle decomposition (phase infrastructure)
