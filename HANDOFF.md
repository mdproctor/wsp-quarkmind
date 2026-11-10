# HANDOFF — quarkmind

## Last Session

Closed #621, #622. Made significant progress on #623/#625. Filed #625-#629. The foundational insight: the EM needs **complete per-loop physics simulation** — not constant tuning — to match SC2 tick-for-tick.

### The physics gap

SC2 simulates every unit every game loop (22.4 loops/second): acceleration, collision, pathfinding. The EM processes workers sequentially with no collision. This creates two specific gaps:

1. **Walk distance** — EM workers walk center-to-center (Nexus center → patch center = ~7u). SC2 workers walk edge-to-edge (~3.5u). Fixed: edge-to-edge targets via `edgePoint()`.

2. **Collision** — EM workers pass through each other. SC2 workers push apart and repath. Without collision, EM cycle variance is 4-6 ticks per worker; SC2 is 2-17 ticks. Added initial opposing-path lateral dodge, but need per-loop simultaneous resolution + all-pairs collision + steering avoidance for full match.

### What changed this session

1. **#621 (closed)** — Per-worker orders/buffs extraction from SC2 API. `WorkerAction` enum, `WorkerSnapshot`/`WorkerCadenceSnapshot` records, `ObservationTranslator.toWorkerSnapshot()`, `EmulatedGame.workerSnapshots()`. 20 tests.

2. **#622 (closed)** — SC2 12w delivery cadence baseline recorded (12 workers × 305 ticks). Both cadence and calibration baselines now from same game run. Key finding: SC2 delivery cadence varies ~0.3% between runs.

3. **#623/#625 (in progress)** — Three physics changes applied:
   - Edge-to-edge walk distances (3.56u instead of 7.08u)
   - Worker collision (lateral dodge on opposing paths)
   - Mining efficiency factor removed (1.006→1.0 for all races)
   - Mining duration recalibrated (12→14 loops)
   - **Result: +5 minerals final gap without compensation, 243/304 mismatch ticks**

4. **#624 (done within #623)** — Factor removed as part of the physics fix.

5. **#625-#629 filed** — Complete physics simulation path.

### Accuracy summary

**Dual-run (controlled SC2, no efficiency factor):**

| Baseline | SC2 | EM | Delta |
|----------|-----|-----|-------|
| 12w Protoss (tick 304) | 3460 | 3465 | +0.1% |

**Per-tick accuracy:** 243/304 mismatch ticks (80%), ±30 mineral oscillation, +5 final gap. Target: 0% mismatch — requires complete collision physics (#625-#629).

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | 28 commits, active | #578...#613, #618, #621, #622, #623, #625 |

---

## Queue — physics simulation first

The path to 0% mismatch is complete per-loop physics simulation. Worker collision proves the approach; full unit physics completes it.

```
[ ] #626 — Per-loop simultaneous collision resolution (M/High) ← next
[ ] #627 — Cross-path collision at mineral line (S/Med) — blocked by #626
[ ] #628 — Pathfinding-based collision avoidance (M/High) — blocked by #626, #627
[ ] #629 — Full unit physics simulation for all unit types (L/High) — blocked by #625
[ ] #619 — Worker initialization desynchronization (S/Med)
[ ] #620 — Mining cycle decomposition (M/High)
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
| 617 | Foundational tick-alignment audit | Active — top priority |

### Physics simulation chain (#625→#629)

| # | Title | Scale | Complexity | Blocked by | Status |
|---|---|---|---|---|---|
| 625 | Worker collision model | M | High | — | In progress — lateral dodge implemented, needs per-loop resolution |
| 626 | Per-loop simultaneous collision | M | High | #625 | Open — restructure tick loop to all-workers-per-loop |
| 627 | Cross-path collision (all pairs) | S | Med | #626 | Open — remove opposing-task filter |
| 628 | Pathfinding collision avoidance | M | High | #626, #627 | Open — steering waypoints instead of lateral dodge |
| 629 | Full unit physics for all types | L | High | #625-#628 | Open — PhysicsEngine for workers + army + buildings |

### Other foundational

| # | Title | Scale | Complexity | Blocked by | Status |
|---|---|---|---|---|---|
| 623 | Fix EM delivery timing | M | High | #622 | Partially done — edge-to-edge + factor removal done, 0% mismatch needs #625-#628 |
| 624 | Remove compensation factor | S | Low | #623 | Done — factor set to 1.0 |
| 619 | Worker initialization desync | S | Med | #618 | Open |
| 620 | Mining cycle decomposition | M | High | #618 | Open |

### Deferred

| # | Title | Scale | Complexity | Notes |
|---|---|---|---|---|
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
| Worker collision | `SimulatedWorker.moveLoopsWithCollision()`, `resolveOverlaps()` |
| Edge-to-edge targets | `WorkerSimulation.edgePoint()`, `returnTarget()`, `miningTarget()`, `depositTarget()` |
| Edge-to-edge toggle | `WorkerSimulation.useEdgeToEdge` (currently true) |
| Mining duration | `WorkerPhysicsConfig.DEFAULT_MINING_DURATION_LOOPS` (14.0) |
| Mining efficiency factor | `ProtossRaceModel.miningEfficiencyFactor()` (1.0 — all races) |
| Worker cadence snapshot | `ObservationTranslator.toWorkerSnapshot()`, `EmulatedGame.workerSnapshots()` |
| Cadence comparison diagnostic | `WorkerCadenceComparisonTest.compareDeliveryCadence()` |
| Edge calibration sweep | `EdgeToEdgeCalibrationTest.sweepMiningDurationAtEdgeDistances()` |
| Delivery cadence diagnostic | `DualRunCalibrationTest.deliveryCadenceDiagnostic()` |
| SC2 cadence baseline | `src/test/resources/calibration/cal-worker-cadence-12w.json` |
| SC2 physics constants | `docs/sc2-physics-constants.md` |

---

## Design Direction

**The EM needs to be a per-loop physics simulation, not a per-tick constant-driven model.**

Every compensation constant (mining efficiency factor, lumped mining duration, docking overhead) exists because the EM lacks physics that SC2 has. The fix path is to add the missing physics, not tune constants:

- Mining efficiency factor → removed (edge-to-edge walk + collision produces correct total)
- Lumped mining=12 → recalibrated to 14 at correct walk distances
- Per-tick mismatch → requires complete collision (#626-#628)
- Full simulation → PhysicsEngine for all unit types (#629)

**Design rules (from CLAUDE.md):**
- No compensation constants — fix root cause
- No formulas — physics simulation only
- Calibrate from SC2 dual-run, not oracle replays
- Per-tick deltas must be exact multiples of 5

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name.
- **MULE divergence at tick 179** — SC2 baseline has 2 MULEs, EM has 0. MULE calldown in playbooks not working in EM.
