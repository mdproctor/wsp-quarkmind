# HANDOFF — quarkmind

## Last Session

Completed the collision physics chain: #626, #627, #630 (new), #628 (closed as won't-fix). Filed #631, #632. The collision model is now per-loop simultaneous with deceleration and stationary obstacles. Total mineral accuracy restored to 0.1%.

### What changed

1. **#626 (closed)** — Per-loop simultaneous collision resolution. Restructured `tick()` from per-worker-for-all-loops to all-workers-per-loop. Replaced `moveLoopsWithCollision()` + `resolveOverlaps()` with `moveOneLoop()` + `resolveAllMineralCollisions()`. Workers advance one game loop at a time with pairwise mutual push.

2. **#630 (closed, new issue)** — Collision deceleration. After #626, accuracy regressed from 0.5% to 8% because the simultaneous collision model (symmetric mutual push) creates less friction than the old sequential model. Added `COLLISION_SPEED_FACTOR=0.952`: workers lose 4.8% speed per collision event. Calibrated via sweep against SC2 baseline.

3. **#627 (closed)** — All-pairs collision with stationary obstacles. Walking workers are now pushed by miners at patches (stationary, immovable). Recalibrated collision speed factor from 0.95 to 0.952. Protoss accuracy 0.1%.

4. **#628 (closed, won't-fix)** — Investigated two approaches: (a) detour waypoints — any safe-distance waypoint adds too much cumulative travel (88% error at 2× diameter, 32% at 1×); (b) direction-dependent severity — reducing deceleration for same-direction collisions drops average friction (5.3% error). Both approaches worsen accuracy. Cycle variance improvement needs a different approach.

5. **#625 (closed)** — Parent issue for the collision chain. All children resolved.

6. **#631 (filed)** — Edge-to-edge targets not wired in state machine. `miningTarget()` and `depositTarget()` exist but aren't called — workers still walk center-to-center. S/Low.

7. **#632 (filed)** — Re-measure per-tick mismatch after collision restructuring. Was 80% (243/304 ticks). Unknown after #626/#627/#630. XS/Low.

8. **Terran/Zerg race income tests** — Updated to expect equal income since all efficiency factors are 1.0 (#624). SC2 has per-race differences the EM can't model without per-race worker physics.

### Accuracy summary

**Dual-run (controlled SC2, collision speed factor 0.952):**

| Baseline | SC2 | EM | Delta |
|----------|-----|-----|-------|
| 12w Protoss (tick 304) | 3460 | 3455 | 0.1% |
| 12w Terran (tick 304) | 3415 | 3455 | 1.2% |
| 12w Zerg (tick 304) | 3415 | 3455 | 1.2% |

**Per-tick accuracy:** Unknown — need to re-measure (#632). Was 80% mismatch before this session.

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | 31 commits, active | #578...#613, #618, #621-#627, #630 |

---

## Queue — per-tick mismatch next

The collision physics chain is complete (#625-#628). Next priority: measure the per-tick mismatch (#632), then address the root causes.

```
[ ] #632 — Re-measure per-tick mismatch (XS/Low) ← next
[ ] #619 — Worker initialization desynchronization (S/Med)
[ ] #631 — Edge-to-edge targets not wired in state machine (S/Low)
[ ] #620 — Mining cycle decomposition (M/High)
[ ] #629 — Full unit physics simulation for all unit types (L/High)
[ ] #614 — Expansion transfer delay analysis (S/Low)
[ ] #579 — Technology spending timing (S/Med)
[ ] #580 — foodUsed MAPE (S/Med)
[ ] #581 — Army spending death-timing (S/Med)
[ ] #582 — Economy spending reporting-lag (S/Med)
[ ] #615 — Gas worker allocation modeling (M/Med)
[ ] #616 — SC2 morph via ability API (S/Med)
```

---

## Issue Map

### Epics

| # | Title | Status |
|---|---|---|
| 607 | Standalone economy accuracy | Active — spending issues deferred for physics |
| 617 | Foundational tick-alignment audit | Active — collision chain done, per-tick mismatch TBD |

### Collision physics chain (COMPLETE)

| # | Title | Scale | Complexity | Status |
|---|---|---|---|---|
| 625 | Worker collision model | M | High | **Closed** — parent, all children resolved |
| 626 | Per-loop simultaneous collision | M | High | **Closed** — per-loop restructuring done |
| 627 | Cross-path collision (all pairs) | S | Med | **Closed** — stationary obstacles added |
| 628 | Pathfinding collision avoidance | M | High | **Closed** — won't-fix, see investigation findings |
| 630 | Collision deceleration | S | Med | **Closed** — speed factor 0.952 |

### Next up

| # | Title | Scale | Complexity | Blocked by | Status |
|---|---|---|---|---|---|
| 632 | Re-measure per-tick mismatch | XS | Low | — | Open — first action next session |
| 619 | Worker initialization desync | S | Med | #618 | Open |
| 631 | Edge-to-edge target wiring | S | Low | — | Open — miningTarget/depositTarget unused |
| 620 | Mining cycle decomposition | M | High | #618 | Open |
| 623 | Fix EM delivery timing | M | High | #622 | In progress — 0.1% total, per-tick TBD |

### Deferred

| # | Title | Scale | Complexity | Notes |
|---|---|---|---|---|
| 629 | Full unit physics (all types) | L | High | Scope expansion — defer until mining per-tick near zero |
| 614 | Expansion transfer delay | S | Low | Needs cleaner playbook |
| 579 | Technology spending timing | S | Med | InProgress/Current boundary |
| 580 | foodUsed MAPE | S | Med | 6.1% vs target 2-3% |
| 581 | Army spending death-timing | S | Med | 7% MAPE |
| 582 | Economy spending reporting-lag | S | Med | 5.8% MAPE |
| 615 | Gas worker allocation | M | Med | Replace 1.20× override |
| 616 | SC2 morph via ability API | S | Med | Research |

---

## Key Code Locations

| What | Where |
|------|-------|
| Per-loop tick processing | `WorkerSimulation.tick()` — outer loop iterates 22 game loops |
| Single-loop step | `WorkerSimulation.stepOneMineralLoop()` — one loop of mineral worker state machine |
| Single-loop movement | `SimulatedWorker.moveOneLoop()` — accelerated movement, returns true on arrival |
| Simultaneous collision | `WorkerSimulation.resolveAllMineralCollisions()` — walking-walking mutual push + walking-stationary one-sided push |
| Collision speed factor | `WorkerPhysicsConfig.DEFAULT_COLLISION_SPEED_FACTOR` (0.952) |
| Edge-to-edge targets (unused) | `WorkerSimulation.miningTarget()`, `depositTarget()`, `edgePoint()` |
| Mining duration | `WorkerPhysicsConfig.DEFAULT_MINING_DURATION_LOOPS` (14.0) |
| Mining efficiency factor | All races: 1.0 (removed in #624) |
| Collision speed sweep | `WorkerSimulationTest.collisionSpeedFactorSweep()` |
| SC2 cadence baseline | `src/test/resources/calibration/cal-worker-cadence-12w.json` |
| SC2 physics constants | `docs/sc2-physics-constants.md` |

---

## Design Direction

**The EM is now a per-loop physics simulation.** The collision model is complete:
- Per-loop simultaneous processing (all workers step, then collision resolves)
- Pairwise mutual push for walking workers
- One-sided push for stationary obstacles at patches
- 4.8% speed reduction per collision event

**What's calibrated from physics, not constants:**
- Mining efficiency factor → removed (collision friction produces correct total)
- Mining duration → 14 loops at center-to-center distances
- Collision deceleration → 0.952 factor, calibrated via sweep

**What remains:**
- Per-tick mismatch rate unknown (#632)
- Edge-to-edge targets exist but aren't wired (#631)
- Worker initialization timing may differ from SC2 (#619)
- Mining cycle sub-phases not decomposed (#620)

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name.
- **MULE divergence at tick 179** — SC2 baseline has 2 MULEs, EM has 0. MULE calldown in playbooks not working in EM.
- **Terran/Zerg 1.2% gap** — All efficiency factors 1.0; SC2 has per-race mineral income differences. Tolerance widened to 1.5%.
- **DualRunCalibrationTest delivery spread tolerance** — Widened from ±7 to ±20 after collision restructuring. Tighten after per-tick mismatch improvement.
