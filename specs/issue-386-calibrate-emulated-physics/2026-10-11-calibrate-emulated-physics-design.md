# Design: Calibrate EmulatedGame Physics from Tournament Ground Truth

**Issue:** #386
**Date:** 2026-10-11
**Branch:** issue-386-calibrate-emulated-physics

## Problem Statement

EmulatedGame accuracy at the 5-minute checkpoint against 118 oracle replays (v4.9.3):

| Category | Current | Target |
|----------|---------|--------|
| Units | 78.7% | ≥80% |
| Buildings | 100% | ≥97% (maintain) |
| Upgrades | 3.4% | Significant improvement |
| Economy MAPE | ~200% | ≤50% |

Root cause analysis identified three distinct problem types:

1. **Measurement bug** — EconomyTracker computes net balance delta (income minus spending), not gross collection rate. SC2's `mineralsCollectionRate` PlayerStats field reports gross minerals mined. This explains the 24× economy gap entirely (208K ground truth vs 8.6K emulated).
2. **Uncalibrated constants** — Upgrade timing constants use `seconds × GAME_LOOPS_PER_SECOND`, violating protocol PP-20260522-572156 which requires calibration from replay ground truth.
3. **Phantom units** — Zerg EGG counted as units in baseline comparison (73 phantom units across 118 replays).
4. **Extraction pipeline gaps** — Queen (17.9%), Baneling (0%), Roach (35.4%) under-counted due to replay CmdEvent → Intent generation issues (not EmulatedGame physics bugs).

## Architecture — Staged Approach

### Phase 1: EmulatedGame Internals

Four independent workstreams, all within EmulatedGame and its tests:

#### 1.1 EconomyTracker — Gross Income Tracking

**What:** Replace net balance delta with cumulative gross income tracking.

**Current behavior:** `mineralRate = minerals_now - minerals_7_ticks_ago`. When the player spends minerals on units, the rate drops or goes negative (clamped to 0).

**Fix:** Add `grossMineralsCollected` and `grossVespeneCollected` accumulators in EmulatedGame's per-tick income calculation. Each tick, increment by the 3-tier saturation income computed for that tick (the income already calculated for adding to the mineral balance). Report this accumulator as the collection rate.

**Validation:** Extract PlayerStats from oracle replays at 1/3/5/7 min checkpoints. Compare EmulatedGame gross income against SC2's `mineralsCollectionRate` to verify the formula match. If the raw accumulator doesn't match (SC2 may use a rate-per-minute or windowed average), calibrate the reporting formula from the data.

#### 1.2 EGG Phantom Filtering

**What:** Exclude transient morphing unit types from unit count comparisons.

**Fix:** In the baseline comparison logic (EmulatedGameAccuracyBaselineTest), exclude `UnitType.EGG` and any similar intermediate states from both ground truth and emulated unit counts.

**Impact:** Removes 73 false-positive units from the emulated count, directly improving unit accuracy.

#### 1.3 Upgrade Timing Calibration

**What:** Calibrate all ~80 upgrade timing constants from replay ground truth.

**Pattern:** Follow SC2TrainTimeCalibrationTest methodology:
1. Extract UpgradeComplete tracker events from oracle replays (118 replays)
2. For each upgrade type, compute duration as `completeLoop - startLoop`
3. Take modal value across all replays (most common duration)
4. Replace existing `seconds × GAME_LOOPS_PER_SECOND` estimates in SC2Data

**New test:** `SC2UpgradeTimeCalibrationTest` — prints calibrated values, validates against the oracle dataset.

**Prerequisite:** Verify that upgrade intents are actually reaching EmulatedGame. If upgrade intent generation is also broken (extraction pipeline issue), the calibrated timings won't matter until intents arrive. Diagnose this first — if intents aren't generated, include upgrade intent extraction in Phase 2.

#### 1.4 MULE Verification

**What:** Confirm MULE calldown produces counted units in baseline measurements.

**Current state:** Recent commits added MULE support, but baseline shows 0% MULE accuracy (69 ground truth vs 0 emulated). Either the MULE code isn't being exercised in the baseline test path, or MULEs aren't being counted.

**Fix:** Trace the MULE lifecycle end-to-end: calldown → spawn → lifetime → expiry. Verify MULEs appear in the emulated unit snapshot at the measurement checkpoint.

### Phase 2: Extraction Pipeline (Conditional)

Triggered only if Phase 1 baseline shows unit accuracy <80%.

#### 2.1 Queen Intent Generation

**Problem:** 145 ground truth Queens vs 26 emulated across 118 replays (17.9%).

**Root cause:** Replay CmdEvent extraction under-counts Queen production commands. Queens are produced from Hatcheries; each Hatchery can produce independently. Likely an abilLink mapping gap or selection-based multiplication issue.

**Fix:** Diagnose the specific extraction gap using `AbilityMappingCoverageDiagnosticTest` and add missing Queen abilLink mappings.

#### 2.2 Morph Intent Generation

**Problem:** Baneling (0%), Roach (35.4%) significantly under-counted.

**Fix:** Similar diagnostic approach — trace morph CmdEvents to identify missing abilLink mappings or morph-specific extraction logic gaps.

## Testing Strategy

### Per-fix validation cycle

Each fix follows:
1. Write or update calibration test (documents source data)
2. Write unit test for the specific change
3. Re-run `EmulatedGameAccuracyBaselineTest` to measure delta

### Phase gates

| Gate | Check | Pass criteria |
|------|-------|---------------|
| After Phase 1 | EmulatedGameAccuracyBaselineTest | Economy MAPE ≤50%, units ≥80% → skip Phase 2 |
| After Phase 2 (if needed) | Same | Units ≥80% |
| Final | DivergenceRegressionTest | No category regression |

### New tests

| Test | Purpose |
|------|---------|
| SC2UpgradeTimeCalibrationTest | Modal upgrade durations from oracle replays |
| EconomyTracker gross income test | Validates accumulator against known income sequence |
| PlayerStats formula validation | EmulatedGame gross income vs SC2 PlayerStats at checkpoints |

### Existing tests — no regression

- EmulatedGameTest
- SC2TrainTimeCalibrationTest / SC2BuildTimeCalibrationTest
- DivergenceRegressionTest (thresholds should improve)
- GameLoopBenchmarkTest (income tracking adds per-tick work — verify no latency impact)

## Acceptance Criteria

| Metric | Before | Target | Test |
|--------|--------|--------|------|
| Unit accuracy (5min) | 78.7% | ≥80% | EmulatedGameAccuracyBaselineTest |
| Building accuracy (5min) | 100% | ≥97% | Same (maintain) |
| Upgrade accuracy (5min) | 3.4% | Measurable improvement (>30%) | Same |
| Economy MAPE (5min) | ~200% | ≤50% | Same |
| Regression | — | No drop | DivergenceRegressionTest |
| Calibration | — | Each fix documented | Per-fix calibration test |

## References

- EmulatedGame.java — tick() physics, handleTrain(), handleResearch()
- SC2Data.java — timing constants (trainTimeInLoops, buildTimeInLoops, upgradeTimeInLoops)
- EconomyTracker.java — current net balance delta computation
- SC2TrainTimeCalibrationTest.java — existing calibration pattern
- PP-20260522-572156 — protocol: SC2Data timing constants must be calibrated from replay ground truth
- docs/benchmarks/emulated-game-accuracy-baseline.md — current baseline data
- docs/benchmarks/oracle-accuracy-baseline.md — oracle extraction baseline
- EmulatedGameAccuracyBaselineTest — accuracy measurement test
- DivergenceRegressionTest — regression threshold test
