# HANDOFF — quarkmind

## Last Session

Worker mining contention fix (#596) — diagnosed and fixed assignment-based contention that was wasting 15.5% of worker-ticks. Root cause: `assignToNearestPatch` counted only active miners (workers physically mining), not assigned workers (including in-transit), so all workers clustered on the same patch.

### What was done

- **Diagnostic test (`MiningCycleDiagnosticTest`, @Tag("diagnostic")):**
  - `singleWorkerCycleTrace` — per-tick state transition trace
  - `singleWorkerPerLoopTrace` — per-loop trace for first 3 cycles
  - `twelveWorkerContentionAnalysis` — waiting worker-tick measurement
  - `rateComparison_1v8v12workers` — income scaling across worker counts

- **Contention fix (2 changes to `WorkerSimulation`):**
  1. `assignToNearestPatch` now counts assigned workers (excluding self) instead of active miners — workers spread evenly across patches at assignment time
  2. Wake-up pass after tick processing — workers waiting at a patch freed during the current tick start mining immediately without a 22-loop gap
  3. `countAssignedWorkers(patch, exclude)` helper added for assignment counting
  4. Package-private accessors `worker(tag)` and `allWorkers()` added for diagnostic tests

- **Test assertion updated:**
  - `sixteenWorkers_showsSaturationPenalty` — changed from strict "< 2x" assertion to verifying per-worker rate decreases (16w ≤ 8w rate); old assertion relied on artificial contention bug

### Impact

| Metric | Before | After | Direction |
|--------|--------|-------|-----------|
| 12-worker rate at 6.0u | 52.6/min/worker | 64.1/min/worker | Fixed |
| Worker-tick contention | 15.5% wasted | 0% | Fixed |
| `mineralsCollectionRate` MAPE (standalone) | 22.1% | 19.0% | Improved |
| `mineralsCurrent` MAPE (standalone) | 67.5% | 132.2% | Regressed |
| Tracker-driven / DivergenceRegressionTest | Pass | Pass | Unchanged |
| SC2 dual-run comparison (12w baseline) | 3275 vs 3460 (under) | 3890 vs 3460 (over) | See below |

**mineralsCurrent regression explanation:** The old code had two compensating errors — artificial contention (too slow) and synthetic 6.0u patches (too close). They partially canceled. Removing contention reveals the distance mismatch. The standalone EM now overproduces minerals because workers mine at perfect efficiency at artificially short distances. The tracker-driven path is unaffected because GT economy sync overrides the worker simulation.

---

## Current Accuracy State

### What works — contention-free worker distribution

Workers now spread evenly across patches via assignment counting. At any worker count (1, 8, 12, 16), per-worker rate is consistent — zero artificial contention.

### What works — per-worker mineral physics

SC2 playbook calibration with 12 workers (no buildings, pure mining) on TorchesAIE_v4:

| Race | SC2 per-worker rate | EM at real distances | Delta |
|---|---|---|---|
| Protoss | 57.9 minerals/min | ~57.8 (weighted 4×6.6u + 4×7.6u) | **<1%** |
| Terran | 57.2 minerals/min | ~57.8 | **~1%** |
| Zerg | 57.2 minerals/min | ~57.8 | **~1%** |

### What doesn't work — standalone economy divergence

The standalone EmulatedGame (no GT sync) overproduces minerals because:
1. **Synthetic patch distance (6.0u)** — real SC2 patches are 6.6-7.6u. Shorter distance = faster mining.
2. **Perfect worker AI** — EM distributes workers optimally; SC2's worker AI has realistic inefficiency.

Both are separate from the contention fix and are #599's scope.

### What's not yet measured

| Category | Status | Blocker |
|---|---|---|
| Gas per-worker rate | No data (0 vespene in baselines) | #597 — Assimilator dispatch + worker assignment |
| MULE income per calldown | No data | #598 — morph + calldown dispatch |
| Expansion transfer delay | No data | #598 — rally commands or debug workers |
| 24w saturation (3/patch) | No data (supply-capped at 15) | Needs re-recording with fixed adapter |

### Standalone EmulatedGameBaselineTest — 7-min checkpoint (oracle replays)

| Field | MAPE | Status |
|---|---|---|
| mineralsCollectionRate | 19.0% | Improved (was 22.1%) |
| vespeneCollectionRate | 28.5% | Gas physics not yet SC2-calibrated |
| mineralsCurrent | 132.2% | Regressed — compensating error removed |
| vespeneCurrent | 65.2% | Cumulative drift |
| foodMade/Used | 3.1%/3.7% | At floor |
| workersActiveCount | 2.5% | At floor |

---

## Issue Map

### Open issues by priority

| # | Title | Scale | Complexity | Blocked by | Status |
|---|---|---|---|---|---|
| **596** | EM implementation efficiency gap | M | Med | — | **Contention fix landed — close pending real-distance validation** |
| **599** | Saturation contention model | L | High | — | Next — realistic worker AI inefficiency + real distances |
| **597** | SC2 adapter gas dispatch | S | Med | — | Fix committed, untested |
| **598** | SC2 adapter MULE/expansion | M | High | — | Morph/calldown implemented, untested |
| 592 | Overhead absorption limit | S | Med | — | May be resolved by #599 |
| 593 | SC2 rate algorithm RE | M | High | — | Deprioritised — playbook calibration more direct |
| 594 | Sub-tick rate tracking | S | Med | — | Deprioritised |
| 595 | Gas rate regression | S | Med | #597 | Needs gas data first |

### Closed this session
| # | Title | Resolution |
|---|---|---|
| 596 | EM implementation efficiency gap | Contention fix — assignment-based spreading + wake-up pass |

### Closed last session
| # | Title | Resolution |
|---|---|---|
| 591 | Worker income direction analysis | Factors reverted to 1.0, physics proven correct |

---

## What to do next

### Priority 1 — #599: Realistic mining efficiency

The contention fix makes the EM too efficient. SC2 workers have inherent inefficiency:
- Workers don't always pick the optimal patch
- Mining animation has variable timing
- Workers idle briefly between trips

Options (in order of simplicity):
1. **Real patch distances** — use actual map resource positions instead of synthetic 6.0u. Longest lever — real distances (6.6-7.6u) would drop the rate from 64.1 to ~57.8/min, matching SC2.
2. **Mining efficiency factor** — per-race factor on mining income (Protoss=0.92, Zerg=0.95 from `MiningEfficiencyCalibrationTest`). Already calibrated but currently set to 1.0.
3. **Patch contention at saturation** — model SC2's 3rd-worker-per-patch penalty for 24+ workers on 8 patches.

### Priority 2 — #597: Gas calibration

Same as last session — verify Assimilator dispatch fix works, add explicit worker-to-gas assignment.

### Priority 3 — Close #596

Once real-distance validation confirms the contention fix holds at real distances, close #596.

---

## Key Code Locations

| What | Where |
|------|-------|
| WorkerSimulation (contention fix + wake-up pass) | `quarkmind-sc2/.../emulated/WorkerSimulation.java` |
| MiningCycleDiagnosticTest (4 diagnostic tests) | `quarkmind-sc2/.../emulated/MiningCycleDiagnosticTest.java` |
| PlaybookSC2Adapter (debug build, geyser targeting, morph, calldown) | `quarkmind-sc2/.../real/PlaybookSC2Adapter.java` |
| DualRunCalibrationTest (SC2 comparison + self-baselines) | `quarkmind-sc2/.../playbook/DualRunCalibrationTest.java` |
| SC2 calibration baselines (8 JSON files) | `quarkmind-sc2/src/test/resources/calibration/cal-*.json` |
| Calibration playbooks (10 YAML files) | `quarkmind-sc2/src/test/resources/playbooks/cal-*.yaml` |
| MineralPatch.PATCH_DISTANCE (6.0f — synthetic) | `quarkmind-sc2/.../emulated/MineralPatch.java:11` |
| EmulatedGame.tick() (efficiency factor wiring) | `quarkmind-sc2/.../emulated/EmulatedGame.java` |
| SC2 physics constants reference | `docs/sc2-physics-constants.md` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name: `-Dtest=ClassName`. Not related to Epic #534.
- **Terran/Zerg unit type mapping** — SCVs and Drones show as `UNKNOWN` in SC2 baselines. `ObservationTranslator` doesn't map all ocraft unit types. Mineral data is correct (unit types don't affect mineral counts).
