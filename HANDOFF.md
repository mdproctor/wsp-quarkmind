# HANDOFF — quarkmind

## Last Session

Closed 4 issues (#586, #587, #588, #589). Gas harvest recalibrated 31.696 → 22.0 loops via oracle MAPE sweep. Deposit time modelling investigated and found that lumped constants (mining=12, gas=22) beat split phases — explicit deposit=2 shifted the MAPE minimum upward. Closed cumulative drift issues as rate-dependent. Filed 3 new issues (#590-592) for remaining accuracy vectors.

### What was done

- **#586 — Gas harvest duration recalibration (closed).** Oracle MAPE sweep across 118 replays found 22.0 loops optimal, down from Liquipedia's 31.696. Same pattern as mining (#583): sub-tick fix changed cycle structure, exposing need for recalibration. vespeneCollectionRate 37.1% → 26.7%, vespeneCurrent 77.4% → 65.4%.

- **#587 — Deposit time investigation (closed).** Added DEPOSITING_MINERALS/DEPOSITING_GAS worker states with sub-tick chaining. Oracle sweep with deposit=2 (SC2 API measurement) showed no improvement — mineralsCollectionRate 22.8% → 23.1% at optimal mining=15. The lumped constant captures overhead interactions (deposit + docking + contention) more accurately than split phases. Default set to 0; infrastructure available via `-Dstarcraft.worker.deposit-duration-loops=2`.

- **#588/#589 — Cumulative drift (closed).** mineralsCurrent (67.5%) and vespeneCurrent (65.4%) are rate integrals — no separate fix exists. Balance accuracy improves automatically with rate accuracy.

- **New issues filed:**
  - **#590** — Rate measurement cadence alignment (structural floor from ~6-loop sampling mismatch)
  - **#591** — Worker income direction analysis (EM consistently under-produces)
  - **#592** — Overhead absorption limit (lumped vs explicit phases investigation)

- **Blog written:** "When One Number Beats Three" — lumped constants, gas recalibration, deposit investigation, what this means for simulation work.

- **docs/sc2-physics-constants.md** fully revised — gas harvest=22, deposit investigation findings, cumulative drift numbers updated, overhead gap table updated with #587 status, GasHarvestCalibrationTest added.

---

## Current Accuracy State

### Standalone EmulatedGameBaselineTest — 7-min checkpoint

| Field | Previous Session | This Session | Status |
|---|---|---|---|
| mineralsCollectionRate | 22.3% | **22.8%** | Near structural floor |
| vespeneCollectionRate | 37.1% | **26.7%** | Improved — gas recalibration |
| mineralsCurrent | 67.6% | **67.5%** | Rate-dependent |
| vespeneCurrent | 77.4% | **65.4%** | Improved — gas rate fix cascade |
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
- [ ] #590 — Rate measurement cadence alignment ← next
- [ ] #591 — Worker income direction analysis
- [ ] #592 — Overhead absorption limit investigation
```

Plus standing issues (at or near structural floor):
- #578 — Rate fields (22.8% MAPE, near floor)
- #579 — Tech spending timing (42%, structural — InProgress/Current boundary)
- #580 — foodUsed (3.7%, at floor)
- #581 — Army spending (7%, at floor)
- #582 — Economy spending (5.8%, at floor)

---

## Critical Insights for Next Session

### 1. Rate cadence mismatch is the likely structural floor (#590)

SC2 emits PlayerStatsEvents every ~160 game loops. EconomyTracker samples at RATE_INTERVAL_TICKS=7 (154 loops). The ~6-loop offset creates irreducible error. Aligning the intervals or interpolating between GT sample points could reveal the true physics accuracy floor.

### 2. EM consistently under-produces income (#591)

At 7-min: mineral GT=89788 vs EM=59425 (34% deficit), gas GT=52228 vs EM=28844 (45% deficit). The deficit grows with time. Possible causes: worker speed underestimate, mineral patch distance overestimate, residual idle time at state transitions.

### 3. Lumped constants are at their accuracy ceiling (#592)

The deposit investigation proved that the current model architecture (fixed-duration phases + lumped overhead) can't improve further by splitting constants. Next-level accuracy requires either modeling worker collision (spatial contention) or accepting the current floor. Joint 3D sweep (deposit × mining × gas) might find a slightly better triple.

### 4. Deposit infrastructure is in place but dormant

`DEPOSITING_MINERALS`/`DEPOSITING_GAS` states, configurable via Quarkus config. Default 0.0. Activates with `-Dstarcraft.worker.deposit-duration-loops=2`. Useful if worker collision is ever modeled.

---

## Key Code Locations

| What | Where |
|------|-------|
| WorkerPhysicsConfig (all constants + deposit) | `quarkmind-sc2/.../emulated/WorkerPhysicsConfig.java` |
| WorkerTask (DEPOSITING states) | `quarkmind-sc2/.../emulated/WorkerTask.java` |
| SimulatedWorker (deposit field + setters) | `quarkmind-sc2/.../emulated/SimulatedWorker.java` |
| WorkerSimulation (deposit fast path + sub-tick) | `quarkmind-sc2/.../emulated/WorkerSimulation.java` |
| GasHarvestCalibrationTest (oracle sweep) | `quarkmind-sc2/.../emulated/GasHarvestCalibrationTest.java` |
| SC2 physics constants reference | `docs/sc2-physics-constants.md` |
| Standalone baseline | `docs/benchmarks/standalone-emulated-game-baseline.md` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name: `-Dtest=ClassName`. Not related to Epic #534.
