# HANDOFF — quarkmind

## Last Session

Closed 3 issues from the #583/#584/#585 queue extension. Major finding: tick-quantised mining phase was hiding physics constant errors — a third instance of the pattern from #578. Real SC2 API measurement confirmed the mining animation is ~5 loops (0.22s), not ~1 second. Filed 4 new issues (#586-#589) for remaining accuracy gaps.

### What was done

- **#583 — Mining animation calibration (closed).** Three fixes:

  1. **Sub-tick mining phase completion.** `tickMineralWorker` consumed entire ticks when mining wasn't complete — durations 5-15 all produced identical output. Fix: check if mining completes within remaining loops, chain to return trip within the same tick. Same pattern as #578 movement sub-tick fix.

  2. **Mining duration recalibrated (5 → 12 loops).** Oracle MAPE sweep across 118 replays × 13 durations found 12 loops optimal. At mining=5 with sub-tick fix, income was 65.6% too high at 1-min. Duration=12: 11.4% at 1-min, 23.1% at 7-min.

  3. **Real SC2 measurement via API.** `MiningAnimationMeasurementTest` connects to SC2, tracks Probe position every game loop for 3000 frames. Mining animation median: 5 loops (0.22s), mean: 6.1 loops. The 7-loop gap between measured (5) and calibrated (12) is unmodeled per-cycle overhead: deposit time at town hall (~2-3 loops), approach/docking delay, path contention.

- **Physics constants externalized to Quarkus config.** `WorkerPhysicsConfig` record + CDI producer. Override via `-Dstarcraft.worker.mining-duration-loops=15` — no recompile for calibration sweeps.

- **docs/sc2-physics-constants.md** — consolidated reference for all worker economy constants with sources, validation status, and calibration tests.

- **#584 — Gas worker sub-tick transitions (closed).** Rewrote `tickGasWorker` with same `loopsRemaining` pattern. Impact: vespeneCollectionRate 38.6%→37.1%, mineralsCollectionRate 23.1%→22.3%. Modest because gas harvest duration (31.696 loops) dominates the cycle.

- **#585 — SC2Mapster scaling audit (closed).** Acceleration/deceleration were the only SC2Mapster-sourced constants needing ×1.4 (already fixed in #578). All other time-dimension constants come from Liquipedia, replay calibration, or simulation units.

- **Blog written:** "The Constant That Couldn't Be Wrong" — tick-quantisation hiding physics errors, sub-tick chaining making the physics visible, SC2 measurement confirming calibrated values absorb unmodeled overhead.

---

## Current Accuracy State

### Standalone EmulatedGameBaselineTest — 7-min checkpoint

| Field | Previous Session | This Session | Status |
|---|---|---|---|
| mineralsCollectionRate | 25.0% | **22.3%** | Improved — sub-tick mining + recalibration |
| vespeneCollectionRate | 38.0% | **37.1%** | Improved — gas sub-tick transitions |
| mineralsCurrent | 74.5% | **67.6%** | Improved — rate fix cascading |
| vespeneCurrent | 77.4% | 77.4% | Unchanged |
| foodMade | 3.1% | 3.1% | At floor |
| foodUsed | 3.7% | 3.7% | At floor |
| workersActiveCount | 2.5% | 2.5% | At floor |
| mineralsUsedCurrentArmy | 7.1% | 7.1% | At floor |
| mineralsUsedCurrentEconomy | 5.8% | 5.8% | At floor |
| mineralsUsedCurrentTechnology | 17.0% | 17.0% | Near floor |
| vespeneUsedCurrentArmy | 7.4% | 7.4% | At floor |
| vespeneUsedCurrentTechnology | 42.0% | 42.0% | Structural — InProgress/Current timing |

### DivergenceRegressionTest — all green

100% units, buildings, upgrades. All matchup deltas within thresholds.

---

## Remaining Queue

```
- [ ] #586 — Gas harvest duration recalibration ← next
- [ ] #587 — Deposit time modelling
- [ ] #588 — Mineral balance cumulative drift
- [ ] #589 — Vespene balance cumulative drift
```

Plus standing issues (at or near structural floor):
- #578 — Rate fields (22.3% MAPE, partially resolved)
- #579 — Tech spending timing (42%, structural — InProgress/Current boundary)
- #580 — foodUsed (3.7%, at floor)
- #581 — Army spending (7%, at floor)
- #582 — Economy spending (5.8%, at floor)

---

## Critical Insights for Next Session

### 1. Gas harvest duration likely needs recalibration (#586)

The sub-tick fix changed the gas cycle structure just as it did for mining. The 31.696-loop constant from Liquipedia may need a sweep like we did for mining (which moved from 44.576 → 5 → 12). Gas sub-tick fix only improved MAPE by 1.5% — there may be more to recover.

### 2. Deposit time modelling would let mining approach physical accuracy (#587)

The SC2 measurement showed 2-3 loops of stationary time at the Nexus position — deposit is NOT instant. Adding explicit deposit delay would let us reduce mining duration toward the measured ~5 loops while preserving MAPE. This is the cleanest path to a physically accurate model.

### 3. Cumulative drift fields (#588, #589) improve automatically with rate fixes

mineralsCurrent (67.6%) and vespeneCurrent (77.4%) are cumulative — per-tick rate errors compound. Fixing rate fields reduces these automatically. But spending deduction timing (when minerals are subtracted for purchases) could also contribute.

### 4. Physics constants are now configurable without recompile

`WorkerPhysicsConfig` CDI bean reads from Quarkus config properties:
```
-Dstarcraft.worker.mining-duration-loops=15
-Dstarcraft.worker.gas-harvest-duration-loops=35
-Dstarcraft.worker.acceleration=4.0
```

### 5. MiningDurationCalibrationTest is the oracle sweep tool

`mvn test -pl quarkmind-sc2 -Dtest=MiningDurationCalibrationTest -Preport` runs both unit-test saturation curves and oracle MAPE sweeps. Use for any future mining/gas calibration.

---

## Key Code Locations

| What | Where |
|------|-------|
| WorkerPhysicsConfig (configurable constants) | `quarkmind-sc2/.../emulated/WorkerPhysicsConfig.java` |
| WorkerPhysicsConfigProducer (CDI bean) | `quarkmind-sc2/.../emulated/WorkerPhysicsConfigProducer.java` |
| SimulatedWorker (sub-tick mining + gas) | `quarkmind-sc2/.../emulated/SimulatedWorker.java` |
| WorkerSimulation (sub-tick tickMineralWorker + tickGasWorker) | `quarkmind-sc2/.../emulated/WorkerSimulation.java` |
| MiningDurationCalibrationTest (oracle sweep) | `quarkmind-sc2/.../emulated/MiningDurationCalibrationTest.java` |
| MiningAnimationMeasurementTest (real SC2 via API) | `quarkmind-sc2/.../real/MiningAnimationMeasurementTest.java` |
| SC2 physics constants reference | `docs/sc2-physics-constants.md` |
| Economy structural floor | `docs/benchmarks/economy-structural-floor.md` |
| Standalone baseline | `docs/benchmarks/standalone-emulated-game-baseline.md` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name: `-Dtest=ClassName`. Not related to Epic #534.
