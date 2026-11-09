# Worker Income Direction Analysis — Design Spec

**Issue:** #591
**Branch:** issue-572-worker-activity-modelling
**Date:** 2026-11-09

## Problem

EmulatedGame consistently under-produces income. At 7-min: minerals 19.3% below GT (rate), 34% below GT (balance). Gas 45% below GT (balance). The deficit grows with time (2.6% at 1-min → 19.3% at 7-min on rate). The per-race efficiency factors (0.92/0.95/1.0) from #566 were reverted as pre-sub-tick calibration. Re-running the sweep shows all factors optimal at 1.0 — but the sweep only goes to 1.0. MAPE monotonically decreases as factor increases, proving the model needs factors ABOVE 1.0.

## Root Cause Analysis

Worker count MAPE is only 2.5% at 7-min — the right number of workers exist. The deficit is per-worker income rate. Each worker produces 5 minerals per round trip, so the rate depends entirely on round trips per minute. The model's round trip is ~112 loops (5.0s) at 6.6u close patch distance. If real SC2's effective round trip is ~95 loops (5% shorter walk, faster docking, less contention), each worker produces ~17% more income — matching the observed deficit.

The growing deficit (worse at 7-min than 1-min) is consistent with saturation under-modeling: as worker counts grow, real SC2's saturation curve is steeper than the model's (20-25% real vs 10.2% modeled from 16→24 workers). But this makes the model OVER-produce at saturation, not under-produce. The dominant effect must be per-cycle overhead — something in the cycle time that compounds with worker count.

## Design

### Phase 1: Calibration playbooks against real SC2

Write targeted playbooks that isolate per-worker income in controlled scenarios:

1. **mineral-only-12w.playbook** — 12 workers on 1 base, no buildings, mine for 5 minutes. Isolates the base per-worker mineral rate with no saturation (1.5 workers/patch).

2. **mineral-saturated-24w.playbook** — Train 12 more workers, total 24 on 1 base (3 per patch). Measures the saturation curve at high density.

3. **gas-only-3w.playbook** — 3 workers on 1 geyser, mine for 5 minutes. Isolates per-worker gas rate.

For each playbook:
- Run against real SC2 via `PlaybookSC2RecorderTest` → record per-tick CalibrationBaseline JSON
- Run against EmulatedGame via `DualRunCalibrationTest` → tick-by-tick comparison with early exit on first divergence
- Compare mineral/gas income per tick to find exact divergence point

### Phase 2: Extended oracle efficiency factor sweep

Extend `MiningEfficiencyCalibrationTest` to sweep 0.85-1.30 (currently 0.85-1.0). Optimize for **`mineralsCollectionRate` MAPE** (rate), not `mineralsCurrent` (balance). Also sweep gas factors separately.

This provides the macro-level correction factor validated across 118 replays, while the playbooks provide the micro-level cycle time diagnosis.

### Phase 3: Apply per-race factors

Add `miningEfficiencyFactor()` method to each `RaceModel` implementation. Default 1.0. The factor multiplies worker cargo at deposit:

```java
// In WorkerSimulation, when worker arrives at base:
income += w.cargo * raceModel.miningEfficiencyFactor();
```

Factor values come from the Phase 2 oracle sweep (rate-optimized) or from the Phase 1 playbook calibration, whichever gives better precision.

Similarly for gas: `gasEfficiencyFactor()` on RaceModel.

Configure via existing Quarkus config pattern:
```properties
starcraft.race.protoss.mining-efficiency=1.08  # example
starcraft.race.zerg.mining-efficiency=1.12     # example
starcraft.race.terran.mining-efficiency=1.05   # example
```

### Phase 4: Validation

1. Run `StandaloneEmulatedGameBaselineTest` — measure rate MAPE improvement
2. Run `DivergenceRegressionTest` — no regression in unit/building accuracy
3. Run `SC2PlaybookCalibrationTest` — existing playbooks still pass
4. Update `docs/sc2-physics-constants.md` — new factor values, updated baseline

## Files changed

| File | Change |
|------|--------|
| New: `src/test/resources/playbooks/mineral-only-12w.playbook` | Controlled mineral-only scenario |
| New: `src/test/resources/playbooks/mineral-saturated-24w.playbook` | Saturation scenario |
| New: `src/test/resources/playbooks/gas-only-3w.playbook` | Gas isolation scenario |
| Modify: `MiningEfficiencyCalibrationTest.java` | Extend sweep to 1.30, add rate MAPE optimization |
| Modify: `RaceModel.java` (interface) | Add `miningEfficiencyFactor()`, `gasEfficiencyFactor()` |
| Modify: `ProtossRaceModel.java` | Implement factors |
| Modify: `TerranRaceModel.java` | Implement factors |
| Modify: `ZergRaceModel.java` | Implement factors |
| Modify: `WorkerSimulation.java` | Apply factor at cargo deposit |
| Modify: `docs/sc2-physics-constants.md` | Updated factors and baseline |

## Phasing

Phases 1 (playbooks) and 2 (oracle sweep) can be done in either order. Phase 2 is faster (no SC2 required) and gives immediate MAPE improvement. Phase 1 gives deeper understanding of why the factor is needed.

**Recommended execution order:** Phase 2 first (quick win), then Phase 1 (deeper calibration), then Phase 3 (apply), then Phase 4 (validate). If SC2 is available during the session, Phase 1 and 2 can be done in parallel.

## Success criteria

- mineralsCollectionRate MAPE at 7-min drops below 15% (from 22.1%)
- vespeneCollectionRate MAPE at 7-min drops below 25% (from 29.1%)
- All existing tests pass — no regression
- Per-race factors documented in physics constants with calibration source

## References

- `MiningEfficiencyCalibrationTest.java` — existing sweep, needs extension above 1.0
- `PlaybookSC2RecorderTest.java` — records real SC2 baselines
- `DualRunCalibrationTest.java` — tick-by-tick comparison infrastructure
- `SC2PlaybookCalibrationTest.java` — existing economy playbooks (9 total)
- `WorkerSimulation.java:249-340` — mineral income pipeline
- `RaceModel.java` — race-specific constant interface
- `docs/sc2-physics-constants.md` §6 — current efficiency factors (reverted, not applied)
- `docs/sc2-physics-constants.md` §14 — per-cycle overhead analysis
- Issue #566 — original efficiency factor calibration (pre-sub-tick)
- Issue #583, #584, #586 — sub-tick fixes that changed cycle timing
