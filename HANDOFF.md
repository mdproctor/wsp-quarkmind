# HANDOFF — quarkmind

## Last Session

Worker mining contention fix (#596) and saturation model (#599). Fixed two independent issues: artificial contention from worker clustering, and missing patch occupation contention for 3+ workers per patch.

### What was done

- **#596 — Contention fix (CLOSED):**
  1. `assignToNearestPatch` counts assigned workers (excluding self) instead of active miners
  2. Wake-up pass after tick processing — freed patches start waiting workers immediately
  3. Eliminated 15.5% worker-tick waste from clustering

- **#599 — Saturation model (3 commits):**
  1. `MineralPatch.lockEndLoop` — post-mining docking overhead locks the patch after miner departs
  2. Sub-tick lock resolution — waiting workers consume lock time within the tick, not per-tick
  3. Configurable `PATCH_DOCKING_OVERHEAD_LOOPS` (default 30.0) via `WorkerPhysicsConfig`

- **`MiningCycleDiagnosticTest` (@Tag("diagnostic")):**
  - Single-worker cycle trace (per-tick and per-loop)
  - 12-worker contention analysis
  - Full saturation curve: 1-30 workers with SC2 reference points

### Saturation curve (8 patches, 6.0u, docking=30)

| Workers | W/Patch | Rate/worker/min | Efficiency | SC2 ref |
|---------|---------|----------------|------------|---------|
| 1-8     | ≤1.0    | 64.1           | 100%       | —       |
| 16      | 2.0     | 63.6           | 99.3%      | 57.9    |
| 23      | 2.9     | 52.9           | 82.6%      | 48.6    |
| 24      | 3.0     | 51.8           | 80.7%      | —       |
| 30      | 3.8     | 41.4           | 64.6%      | —       |

SC2 at 23w: 48.6/min = ~81% efficiency → EM at 82.6% — good match.

### Standalone baseline impact (7-min checkpoint, oracle replays)

| Field | Before (old bug) | After contention fix | After docking | Direction |
|-------|-----------------|---------------------|---------------|-----------|
| mineralsCurrent MAPE | 67.5% | 132.2% | 87.9% | Recovering |
| mineralsCollectionRate MAPE | 22.1% | 19.0% | 32.4% | Regressed |

**Root cause of remaining regression:** The EM distributes workers perfectly and mines at maximum efficiency. Real SC2 workers have inherent AI overhead (path-finding delays, momentary idle, suboptimal patch selection). The docking overhead models physical patch contention but not worker AI inefficiency. The collection rate regression (32.4%) is from the docking overhead reducing rate at all worker counts, even when contention isn't the issue.

---

## Issue Map

### Open issues by priority

| # | Title | Scale | Complexity | Blocked by | Status |
|---|---|---|---|---|---|
| **599** | Saturation contention model | L | High | — | Docking model landed — needs calibration sweep for docking value |
| **597** | SC2 adapter gas dispatch | S | Med | — | Fix committed, untested |
| **598** | SC2 adapter MULE/expansion | M | High | — | Morph/calldown implemented, untested |
| 592 | Overhead absorption limit | S | Med | — | May be resolved by #599 |

### Closed this session
| # | Title | Resolution |
|---|---|---|
| 596 | EM implementation efficiency gap | Contention fix — assignment-based spreading + wake-up pass |

---

## What to do next

### Priority 1 — Calibrate docking overhead

The 30-loop default was chosen to match SC2's saturation curve at 6.0u. At real distances (~7u), the curve shape should be validated. A sweep test (`MiningEfficiencyCalibrationTest`-style) across docking values 10-50 against oracle replays would find the MAPE-minimizing value.

### Priority 2 — #597: Gas calibration

1. Verify Assimilator dispatch fix works (run `recordCalGasBaseline3w` once with SC2)
2. If vespene=0 in baselines, add explicit worker-to-gas assignment
3. Extract per-worker gas rate, calibrate gas-harvest-duration-loops

### Priority 3 — Collection rate regression

The mineralsCollectionRate MAPE went from 19.0% to 32.4%. Investigate whether this is from:
- Docking overhead applied at all worker counts (even 1/patch where it's irrelevant)
- Worker distribution being too efficient vs real SC2
- A bug in the lock timing at specific worker/patch ratios

---

## Key Code Locations

| What | Where |
|------|-------|
| WorkerSimulation (contention fix + wake-up + docking lock) | `quarkmind-sc2/.../emulated/WorkerSimulation.java` |
| MineralPatch (lockEndLoop field) | `quarkmind-sc2/.../emulated/MineralPatch.java` |
| SimulatedWorker (PATCH_DOCKING_OVERHEAD_LOOPS) | `quarkmind-sc2/.../emulated/SimulatedWorker.java` |
| WorkerPhysicsConfig (DEFAULT_PATCH_DOCKING_OVERHEAD=30.0) | `quarkmind-sc2/.../emulated/WorkerPhysicsConfig.java` |
| WorkerPhysicsConfigProducer (CDI config wiring) | `quarkmind-sc2/.../emulated/WorkerPhysicsConfigProducer.java` |
| MiningCycleDiagnosticTest (saturation curve) | `quarkmind-sc2/.../emulated/MiningCycleDiagnosticTest.java` |
| DualRunCalibrationTest (SC2 comparison) | `quarkmind-sc2/.../playbook/DualRunCalibrationTest.java` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name: `-Dtest=ClassName`.
- **Terran/Zerg unit type mapping** — SCVs and Drones show as `UNKNOWN` in SC2 baselines. Mineral data is correct.
