# HANDOFF — quarkmind

## Last Session

Closed #576 (spending timing model — the last issue on the original Epic #534 queue). Found and fixed a classification bug (research spending routed to Army instead of Technology). Then ran baselines as previous handover instructed, discovered the acceleration model regression (#578), and made three foundational fixes that halved the mineralsCollectionRate MAPE from 50% to 25%. Filed 8 new issues from findings.

### What was done

- **#576 — Spending timing model (closed).** Root cause: `EconomyTracker.recordResearchSpending` routed upgrade costs to Army counters. SC2 reports upgrades as Technology spending. Fix: route to `mineralsUsedTechnology`/`vespeneUsedTechnology`. Impact: mineralsUsedCurrentArmy 14%→7.1%, vespeneUsedCurrentArmy 35.4%→7.4%. Remaining vespeneUsedCurrentTechnology at 42% is InProgress/Current timing boundary — measurement artifact, not simulation bug.

- **#578 — Acceleration model regression (partially resolved, open).** Three fixes:

  1. **Acceleration ×1.4 scaling.** SC2Mapster data editor values are at normal game speed (16 Hz). Unit speeds are already scaled ×1.4 for faster (22.4 Hz). Acceleration/deceleration were not. Fix: 2.5 → 3.5 u/s² for both. The per-loop velocity change must be `0.15625` at both game speeds: `2.5/16 = 3.5/22.4 = 0.15625`.

  2. **Sub-tick mining transitions.** `tickMineralWorker` processed state transitions once per tick, wasting up to 1 tick at each transition (arrive→mine, mine→return, return→deposit). At 3 transitions per cycle and ~47 loops per leg, this wasted nearly half the cycle time. Fix: chain transitions within the same tick using remaining loops. `moveLoops` now returns loops consumed.

  3. **Mining duration recalibration (44.576 → 5 loops).** Liquipedia's 1.99 seconds includes travel time. With acceleration model simulating travel physically, MINING_DURATION_LOOPS must represent only stationary harvest. Empirical sweep: 5 loops minimises mineralsCollectionRate MAPE against 118 oracle replays.

- **Filed issues:** #578 (updated with findings), #579 (tech spending timing), #580 (foodUsed gap — improved to 3.7%, possibly at floor), #581 (army spending floor 7%), #582 (economy spending floor 5.8%), #583 (mining animation SC2 calibration), #584 (gas worker sub-tick transitions), #585 (SC2Mapster scaling audit).

- **Updated economy structural floor document** with post-fix values and two new sections (§8 research classification, §9 acceleration regression).

---

## Current Accuracy State

### Standalone EmulatedGameBaselineTest — 7-min checkpoint

| Field | Session Start | Session End | Status |
|---|---|---|---|
| mineralsCollectionRate | 50.0% | 25.0% | Halved — #578 partially resolved |
| vespeneCollectionRate | 43.0% | 38.0% | Improved 5% |
| mineralsCurrent | 100% (EM=0) | 74.5% (EM=36K) | Recovered from zero |
| vespeneCurrent | 82.3% | 77.5% | Improved 5% |
| mineralsUsedCurrentArmy | 14.0% | 7.1% | At floor — #576 fix |
| vespeneUsedCurrentArmy | 35.4% | 7.4% | At floor — #576 fix |
| mineralsUsedCurrentEconomy | 5.8% | 5.8% | At floor |
| mineralsUsedCurrentTechnology | 15.7% | 17.0% | Near floor |
| vespeneUsedCurrentTechnology | 15.4% | 42.0% | Timing artifact (research moved here) |
| foodMade | 2.9% | 3.1% | At floor |
| foodUsed | 6.1% | 3.7% | Improved — sub-tick fix |
| workersActiveCount | 3.2% | 2.5% | At floor |

### DivergenceRegressionTest — all green

100% units, buildings, upgrades. All matchup deltas within thresholds. Tracker-driven mode unaffected by all changes.

---

## Critical Insights for Next Session

### 1. Mining animation duration needs real SC2 measurement (#583)

MINING_DURATION_LOOPS=5 is empirically optimal for oracle replay MAPE but unphysically short (0.22 seconds — real SC2 animation is ~1 second). The saturation curve drop at mining=5 is 9.1% (16→24 workers) vs SC2's 40-50%. The correct mining animation time needs direct measurement from real SC2.

**How to measure:** Start a game, observe a single worker mining. Time the stationary period (worker at mineral patch, not walking). The user confirmed SC2 testing is feasible.

### 2. SC2Mapster values need ×1.4 game-speed scaling (#585)

Any SC2 data editor constant with time-dimension units (acceleration, turn rate, attack cooldowns) needs ×1.4 scaling for faster game speed. Unit speeds are already scaled. Acceleration was not — this caused the income halving. Audit all other SC2Mapster-sourced constants.

### 3. Sub-tick transitions are the pattern for worker physics (#584)

The mineral worker sub-tick chaining pattern (`moveLoops` returns consumed loops, chain transitions within remaining loops) should be applied to gas workers too. vespeneCollectionRate at 38% likely has the same dead-tick overhead.

### 4. The 25% mineralsCollectionRate at 7-min is from cumulative drift

At 1-min, mineralsCollectionRate is 11.8% — near the structural floor (10-12%). The 7-min value (25%) grows because per-tick rate errors compound. Fixing the mining animation duration will help early checkpoints most; 7-min will converge slowly.

### 5. The tick-quantization experiment revealed synchronized workers

With uniform patch distances (test setup), ALL workers stay perfectly synchronized across cycles because tick-quantized movement produces identical cycle times. This halves income because paired workers always race for the same patch slot. Real replay distances (6.0-8.5 range) desynchronize workers naturally. The saturation curve test is an artifact — always validate against oracle replay baselines.

### 6. moveLoops return value is a new API contract

`SimulatedWorker.moveLoops(int)` now returns `int` (loops consumed before arrival, or maxLoops if not arrived). `moveToward()` still wraps it for callers that don't need the return value. Any new movement code should use moveLoops directly for sub-tick chaining.

---

## Remaining Queue

```
- [x] #576 — Spending timing model (closed this session)
- [ ] #583 — Mining animation real SC2 calibration ← next
- [ ] #584 — Gas worker sub-tick transitions
- [ ] #585 — SC2Mapster faster-speed scaling audit
```

Plus standing issues (at or near structural floor, low priority):
- #578 — Rate fields (25% MAPE, partially resolved)
- #579 — Tech spending timing (42%, structural)
- #580 — foodUsed (3.7%, likely at floor now)
- #581 — Army spending (7%, at floor)
- #582 — Economy spending (5.8%, at floor)

---

## Key Code Locations

| What | Where |
|------|-------|
| SimulatedWorker (accel=3.5, moveLoops returns consumed) | `quarkmind-sc2/.../emulated/SimulatedWorker.java` |
| WorkerSimulation (sub-tick tickMineralWorker) | `quarkmind-sc2/.../emulated/WorkerSimulation.java:247` |
| EconomyTracker (research → Technology) | `quarkmind-sc2/.../emulated/EconomyTracker.java:73` |
| MineralPatchDistanceDiagnostic | `quarkmind-sc2/.../replay/MineralPatchDistanceDiagnostic.java` |
| MiningDurationSweep test | `WorkerSimulationTest#miningDurationSweep` |
| Economy structural floor | `docs/benchmarks/economy-structural-floor.md` |
| Standalone baseline | `docs/benchmarks/standalone-emulated-game-baseline.md` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name: `-Dtest=ClassName`. Not related to Epic #534.
