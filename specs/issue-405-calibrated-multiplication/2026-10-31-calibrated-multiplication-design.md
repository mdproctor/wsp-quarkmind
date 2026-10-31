# Calibrated Multiplication — Design Spec

**Issue:** casehubio/quarkmind#405
**Branch:** issue-405-calibrated-multiplication
**Date:** 2026-10-31
**Parent:** #366 (Phase 2.5 — Reconstitution accuracy gate)
**Epic:** #481 (EmulatedGame playbook calibration)

---

## Problem

`StrippedReplayFeatureExtractor` compensates for the ~43% CmdEvent detection rate in stripped replays by multiplying each detected train command by the number of production buildings of that type (capped at `MAX_MULTIPLICATION = 4`). This uniform multiplication over-counts types with many production buildings:

| Type | Oracle | Java | Diff | Cause |
|------|--------|------|------|-------|
| SCV | 2636 | 3498 | +862 | 3 CCs × multiplication |
| Overlord | 487 | 837 | +350 | Multiple Hatcheries |
| Hydralisk | 323 | 509 | +186 | Multiple Hatcheries |

And under-counts types the multiplication cannot help:

| Type | Oracle | Java | Diff | Cause |
|------|--------|------|------|-------|
| Zergling | 4288 | 3752 | -698 (net) | CmdEvent gap + morph losses |
| MULE | 623 | 0 | -623 | No CmdEvent mapping |
| Baneling | 454 | 225 | -229 | Morph detection gap |
| Drone | 1377 | 1133 | -244 | CmdEvent gap |

The current 99.3% T1 accuracy relies on these over- and under-counts balancing out. But the per-type distortions may cause systematic misclassification: vs_zerg classifier accuracy is 47% at 3min and 61% at 5min (fine-grained), significantly worse than vs_protoss (85-87%) and vs_terran (64-91%).

## Approach

Replace uniform building-count multiplication with per-type calibration factors derived from oracle comparison data, stored in a configurable resource file.

### Phase 1: Assessment — Baseline Capture

**Goal:** Establish the current state as a reference point.

1. Record current classifier accuracy from `cross-version-generalisation.md` as the baseline
2. Run `OracleAccuracyBaselineTest` to capture per-type divergence data
3. Run the extractor with multiplication disabled (`repeatCount = 1` for all types) against the 118 oracle replays to get uncalibrated counts — this is the denominator for factor derivation

**Output:** Baseline accuracy numbers and uncalibrated per-type counts.

### Phase 2: Factor Derivation

**Goal:** Compute per-unit-type correction factors.

For each unit type `T` that appears in both oracle and stripped extraction:

```
factor(T) = sum_across_replays(oracle_count(T)) / sum_across_replays(uncalibrated_count(T))
```

Where `uncalibrated_count(T)` is the count with multiplication disabled (repeatCount=1). The ratio is computed over aggregate sums, not as a mean of per-replay ratios — this prevents outlier replays (e.g., one game with 0 SCVs detected) from skewing the factor.

Special cases:
- Types with `uncalibrated_count = 0` and `oracle_count > 0`: these are structurally unmappable (e.g., MULE has no CmdEvent). Factor is undefined — leave as-is (no multiplication can help).
- Types with `oracle_count = 0`: skip — no oracle data to calibrate against.
- TECHLAB_UNITS: keep at factor 1.0 (they already bypass multiplication correctly).

**Storage format:** JSON resource file at `quarkmind-sc2/src/main/resources/calibration/multiplication-factors.json`:

```json
{
  "version": "oracle-v4.9.3-118",
  "dataset": "oracle",
  "replay_count": 118,
  "factors": {
    "SCV": 1.44,
    "MARINE": 2.1,
    "OVERLORD": 0.87,
    "DRONE": 1.5
  }
}
```

The `version` field identifies the dataset and replay count used to derive the factors. Different datasets can produce different factor files.

**Derivation tool:** A new test class `CalibrationFactorDerivationTest` (tagged `@Tag("report")`) that:
1. Runs the extractor with multiplication disabled across oracle replays
2. Compares against oracle ground truth per type
3. Computes factors
4. Writes the JSON file

### Phase 3: Extractor Integration

**Goal:** Replace building-count multiplication with factor-based multiplication.

**Change location:** `StrippedReplayFeatureExtractor.java` lines 210-226.

**Current logic:**
```java
int repeatCount = 1;
if (abilLink != null && !TECHLAB_UNITS.contains(train.unitType())) {
    int capKey = ...;
    int buildingCount = state.productionBuildingCounts.getOrDefault(capKey, 0);
    if (buildingCount > 0) {
        repeatCount = Math.min(buildingCount, MAX_MULTIPLICATION);
    }
}
```

**New logic:**
```java
int repeatCount = 1;
if (calibrationFactors != null && calibrationFactors.containsKey(train.unitType())) {
    double factor = calibrationFactors.get(train.unitType());
    repeatCount = Math.max(1, (int) Math.round(factor));
} else if (abilLink != null && !TECHLAB_UNITS.contains(train.unitType())) {
    // Fallback: existing building-count multiplication
    int capKey = ...;
    int buildingCount = state.productionBuildingCounts.getOrDefault(capKey, 0);
    if (buildingCount > 0) {
        repeatCount = Math.min(buildingCount, MAX_MULTIPLICATION);
    }
}
```

**Factor injection:** The extractor constructor or a factory method accepts an optional `Map<UnitType, Double>` of calibration factors. When null/empty, the existing building-count logic applies unchanged.

**Rounding strategy:** Deterministic `Math.round()`. Fractional factors like 1.44 round to 1 (no extra repetition). Factors like 1.6 round to 2. This is simpler than probabilistic rounding and the extractor must be reproducible for test assertions. Aggregate accuracy across 118+ replays smooths out per-command rounding.

**Morph multiplication:** Morph commands (Zergling→Baneling) use `morphMultiplier()` which is based on selection size, not building count. This is a separate mechanism and is not changed by this design. If morph-specific calibration is needed, that's a follow-up.

### Phase 4: Validation

**Goal:** Measure whether calibration improves classifier accuracy.

1. **Oracle accuracy baseline** — Run `OracleAccuracyBaselineTest` with calibrated factors. Target: reduce SCV/Overlord over-count. Monitor T1 aggregate — some regression is expected as compensating errors are disrupted.
2. **Reconstitution export** — Run `ReconstitutionExportTest` to re-extract all reconstituted JSON with calibrated factors
3. **Retrain classifier** — Full training pipeline: `python3 -m src.run_pipeline --data combined`
4. **Cross-version validation** — Run `python3 -m src.cross_version_validation --write-report` and compare per-matchup accuracy

**Regression guard:** `DivergenceRegressionTest` must still pass. If calibration changes push unit accuracy below the existing thresholds, the test will catch it.

### Phase 5: Decision Gate

Compare calibrated vs uncalibrated classifier accuracy:

| Metric | Baseline | Target |
|--------|----------|--------|
| vs_zerg fine 3min | 47% | >55% |
| vs_zerg fine 5min | 61% | >70% |
| vs_terran fine 5min | 91% | ≥90% (no regression) |
| vs_protoss fine 5min | 87% | ≥85% (no regression) |
| Coarse accuracy 5min | 92% | ≥90% (no regression) |

**If targets met:** Keep calibrated factors, update baselines, close #405.
**If not met:** Revert calibration changes, close #405 as won't-fix with assessment data documenting that per-type distortions don't meaningfully affect classification.

Per protocol `calibration-milestones-are-waypoints`: closing assessment documents baseline at close, what improved, what didn't, methodology verdict, and next milestone recommendation.

## Components Modified

| Component | Change |
|-----------|--------|
| `StrippedReplayFeatureExtractor` | Accept optional calibration factors; use factor-based multiplication when available |
| `CalibrationFactorDerivationTest` (new) | `@Tag("report")` — derive factors from oracle comparison |
| `multiplication-factors.json` (new) | Resource file with per-type factors |
| `OracleAccuracyBaselineTest` | Run with calibrated factors (optional parameter) |
| `ReconstitutionExportTest` | Pass calibration factors to extractor |

## What This Does NOT Change

- **TrackerIntentExtractor** — unaffected, it uses ground-truth tracker events
- **EmulatedGame** — unaffected, it receives intents from the extractor
- **Morph multiplication** — separate mechanism, not changed here
- **MULE/auto-spawn under-counts** — structurally unmappable from CmdEvents, not addressable by multiplication calibration
- **Economy emulation** — separate concern (mineralsCurrent, collection rates)

## Risks

1. **T1 aggregate regression** — Fixing over-counts disrupts compensating errors. Mitigated by: the decision gate allows reverting if aggregate accuracy drops.
2. **Dataset overfitting** — Factors derived from 118 oracle replays may not generalise to tournament/ladder replays. Mitigated by: cross-version validation tests against 6 different tournament datasets.
3. **Factor staleness** — If the extractor changes (new morph mappings, abilLink discovery), factors may need re-derivation. Mitigated by: `CalibrationFactorDerivationTest` can be re-run at any time.

## References

- `StrippedReplayFeatureExtractor.java:210-226` — current multiplication logic
- `StrippedReplayFeatureExtractor.java:143` — MAX_MULTIPLICATION constant
- `docs/benchmarks/oracle-accuracy-baseline.md` — per-type divergence data
- `docs/benchmarks/cross-version-generalisation.md` — classifier accuracy by matchup
- `docs/protocols/calibration-milestones-are-waypoints.md` — waypoint assessment protocol
- `docs/protocols/extractor-separate-from-simulated-game.md` — extractor boundary
- Issue #404 — pool-based approach failure (88% → 56%)
- Issue #357 — ONNX retraining (closed, precondition met)
- Issue #366 — Phase 2.5 reconstitution accuracy gate (parent)
