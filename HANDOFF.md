# HANDOFF — quarkmind

## Last Session

Branch `issue-368-abilityprofile-expansion` — **closed as not needed**, diagnostic tests landed on main (`3efe9a14`).

### What happened

Investigated #368 (AbilityProfile expansion for intermediate SC2 patches). Built 5 diagnostic tests. Found two things:

1. **Infeasible:** Intermediate patches (IEM 2018 baseBuild=60321, ASUS ROG 2020 baseBuild=82457) use generic abilLinks (177, 195, 157) shared across all races and buildings. No linear offset works (brute-force best: 1.5-6.0%). Selection-based unitLink dispatch fails — players issue research via hotkeys without selecting the building.

2. **Unnecessary:** `ReplayFeatureExtractor.java` auto-routes full replays to `TrackerEventFeatureExtractor` (ground truth). All intermediate patch replays have tracker events. Only stripped Blizzard ladder replays (all patch 4.9.3) use `StrippedReplayFeatureExtractor`.

### Decisions

- #368 closed with detailed diagnostic evidence
- #366 epic acceptance criteria updated: "AbilityProfile expanded" → "intermediate patches validated via TrackerEventFeatureExtractor"

### Epic #366 state

| # | Issue | Status |
|---|-------|--------|
| 367, 369, 370, 373, 374, 376, 378 | Baseline + extraction improvements | **Closed** |
| **368** | AbilityProfile expansion | **Closed (not needed)** |
| 371 | Cross-patch regression suite | **Open** — next priority |
| 372 | Re-reconstitute training data | **Open** — blocked on #371 |
| 379 | EmulatedGame accuracy baseline | **Open** — independent |

### Accuracy summary (training data quality)

| Extractor | Used for | Accuracy |
|-----------|----------|----------|
| TrackerEventFeatureExtractor | Full replays (all tournament datasets) | 100% (ground truth) |
| StrippedReplayFeatureExtractor | Stripped ladder replays (4.9.3 only) | 99.7% upgrades, ~78% units/buildings (CmdEvent gap) |

Training data uses TrackerEventFeatureExtractor for all full replays. The 78% unit/building gap in StrippedReplayFeatureExtractor only affects stripped ladder replays.

### Suggested next work

**#371 (Cross-patch regression suite)** — validate TrackerEventFeatureExtractor ≥99% across all 4 categories and all patch eras. This is the formal gate for ONNX retraining (#343).

## References

| What | Where |
|------|-------|
| Commit on main | `3efe9a14` |
| Diagnostic tests | `CrossPatchUpgradeAccuracyTest`, `AbilLinkOffsetCalibrationTest`, `CrossPatchAbilLinkDiscoveryTest`, `AbilLinkEnumerationTest` |
| #368 closing comment | Full diagnostic findings and reasoning |
| ReplayFeatureExtractor auto-routing | `quarkmind-sc2/.../replay/ReplayFeatureExtractor.java` (30 lines) |
