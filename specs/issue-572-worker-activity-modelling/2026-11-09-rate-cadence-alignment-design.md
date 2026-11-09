# Rate Measurement Cadence Alignment — Design Spec

**Issue:** #590
**Branch:** issue-572-worker-activity-modelling
**Date:** 2026-11-09

## Problem

EconomyTracker recalculates mineral/vespene rates every 7 ticks (154 game loops / 6.88s). SC2's PlayerStatsEvents fire every 160 game loops (7.14s). The 6-loop mismatch means EM and GT measure rates over different windows, creating irreducible error at the rate comparison boundary. Rate errors compound in balance fields — mineralsCollectionRate MAPE grows from 11.4% at 1-min to 22.8% at 7-min. For training-quality simulation, every source of per-window error matters because compound drift makes the simulation unreliable over full-length games.

## Design

### Phase 1: Diagnostic — measure cadence contribution

Write `RateCadenceDiagnosticTest` (`@Tag("diagnostic")`) that:

1. **Confirms PlayerStatsEvent cadence.** Iterate oracle replays, extract all PlayerStatsEvent loops for the watched player, compute inter-event deltas. Report mean, median, min, max. Expected: ~160 loops (±1-2 for rounding). This confirms the constant empirically rather than trusting the hardcoded value in other tests.

2. **Measures cadence-only MAPE contribution.** Run the standalone baseline at two rate windows:
   - Current: 154 loops (7 ticks × 22 loops/tick)
   - Aligned: 160 loops (matching SC2 cadence)
   
   Compare mineralsCollectionRate and vespeneCollectionRate MAPE at each checkpoint (1, 3, 5, 7 min). The delta between the two runs isolates the cadence contribution from the income deficit (#591).

3. **Reports per-checkpoint MAPE delta.** Print a table showing current vs aligned MAPE and the improvement (or lack thereof) at each checkpoint.

### Phase 2: Implementation — loop-based rate windowing

Change EconomyTracker from tick-counting to loop-counting for rate recalculation.

**Current mechanism (tick-based):**
```
ticksSinceRateUpdate++
if (ticksSinceRateUpdate >= RATE_INTERVAL_TICKS) → recalculate
```

**New mechanism (loop-based):**
```
loopsSinceRateUpdate += LOOPS_PER_TICK  (22)
if (loopsSinceRateUpdate >= rateIntervalLoops) → recalculate
```

The rate formula changes from dividing by tick count to dividing by elapsed loops, then scaling:
```
perLoopMineral = (grossMinerals - prevGrossMinerals) / loopsSinceRateUpdate
mineralRate = (int) Math.max(0, perLoopMineral * LOOPS_PER_MINUTE)
```

where `LOOPS_PER_MINUTE = 60.0 * GAME_LOOPS_PER_SECOND = 1344.0`.

**Config property:** `starcraft.economy.rate-interval-loops` (default: 160). Injected into EconomyTracker via a new config class or through EmulatedGame's existing config wiring.

**Config location:** New `EconomyConfig` class following the `WorkerPhysicsConfig` pattern — `@ConfigMapping(prefix = "starcraft.economy")` with a single property `rateIntervalLoops()` defaulting to 160. EconomyTracker receives the value at construction time (it's already package-private and instantiated by EmulatedGame).

### Phase 3: Validation

1. Run `StandaloneEmulatedGameBaselineTest` — compare rate MAPE before and after.
2. Run `DivergenceRegressionTest` — ensure no regression in unit/building/upgrade accuracy.
3. Update `docs/sc2-physics-constants.md` — add rate interval to economy constants section, update baseline numbers if improved.
4. Run `EconomyTrackerTest` — existing unit tests must pass with the new loop-based calculation.

## Files changed

| File | Change |
|------|--------|
| `EconomyTracker.java` | Replace tick-based rate window with loop-based; accept rateIntervalLoops at construction |
| `EmulatedGame.java` | Pass rateIntervalLoops from config to EconomyTracker |
| New: `EconomyConfig.java` | `@ConfigMapping(prefix = "starcraft.economy")` with `rateIntervalLoops` |
| New: `RateCadenceDiagnosticTest.java` | `@Tag("diagnostic")` — cadence measurement + MAPE comparison |
| `EconomyTrackerTest.java` | Update to construct EconomyTracker with loop-based interval |
| `docs/sc2-physics-constants.md` | Add rate interval constant, update baseline if improved |

## What this does NOT fix

The 34% income deficit (#591) is the dominant contributor to rate MAPE. This fix eliminates the cadence mismatch component so that remaining rate error is attributable purely to income physics accuracy. After this fix, rate MAPE improvements require fixing actual income production (#591, #592).

## Success criteria

- RateCadenceDiagnosticTest confirms PlayerStatsEvent cadence is 160 loops (±2)
- Rate MAPE at 7-min improves by the measured cadence contribution (expected 1-3%)
- All existing tests pass — no regression
- EconomyTracker rate window configurable for future calibration sweeps

## References

- `EconomyTracker.java:43` — RATE_INTERVAL_TICKS=7 (current implementation)
- `EmulatedGameFeatureExtractor.java:9` — PLAYER_STATS_INTERVAL=160 (SC2 cadence constant)
- `OracleAccuracyBaselineTest.java:40` — STATS_INTERVAL=160
- `WorkerPhysicsConfig.java` — existing config pattern for physics constants
- `docs/sc2-physics-constants.md` §14 — cumulative drift mechanics
- Issue #591 — income deficit (dominant MAPE contributor)
- Issue #592 — overhead absorption limit
