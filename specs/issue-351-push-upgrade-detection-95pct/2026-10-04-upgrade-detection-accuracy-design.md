# Design: Push upgrade detection accuracy to 100%

**Issue:** #351 (child of #341 Phase 1: Feature extraction completeness)
**Branch:** issue-351-push-upgrade-detection-95pct
**Date:** 2026-10-04

## Goal

Push upgrade detection accuracy from 87.7% toward 100% across all 179 mixed-version replays (118 oracle 4.9.3 + 61 HSC tournament). Document inherent limitations where no CmdEvent signal exists.

## Current State

- **87.7% accuracy** after #350 unitLink-aware building-type disambiguation
- **83 UpgradeTypes** in the enum
- **~50 abilLink overrides** in `AbilityProfile.buildHsc2025Overrides()`
- Two dispatch paths: V4_9_3 direct abilLink constants (~30) and HSC_2025 profile overrides
- Existing discovery diagnostics: `discoverUpgradeResearchAbilLinks()` and `discoverUpgradeAbilLinksFrequencyBased()` in `AbilityDiscoveryCalibrationTest`
- **No upgrade accuracy assertion** — `StrippedReplayValidationTest` prints divergence but doesn't assert a threshold

## Design

### Phase 1: Per-upgrade-type accuracy report

Extend `StrippedReplayValidationTest` to compute and print a per-upgrade-type accuracy breakdown:

| UpgradeType | Oracle | Detected | Missed | Accuracy |
|-------------|--------|----------|--------|----------|
| CombatShield | 42 | 42 | 0 | 100% |
| ProtossAirWeapons1 | 18 | 12 | 6 | 66.7% |
| ... | | | | |
| **TOTAL** | N | M | N-M | M/N% |

This reveals which upgrades cluster behind shared abilLinks (e.g., all Forge upgrades missing → one generic abilLink fix).

The existing test iterates oracle `ITrackerEvents.ID_UPGRADE` events and keys by `"P" + playerId + ":" + upgradeTypeName`. Extend this to also track per-type totals and compute aggregate accuracy.

### Phase 2: Discovery — find abilLinks for missing upgrades

Run `discoverUpgradeResearchAbilLinks()` and `discoverUpgradeAbilLinksFrequencyBased()` against the full 179 replay set (currently only 118 oracle). These correlate oracle `IUpgradeEvent` with stripped `CmdEvent` in a 500–5000 loop window to identify the abilLink responsible for each upgrade.

For any upgrade with misses in Phase 1 but a discoverable abilLink, categorize as **recoverable**.

For any upgrade with misses but no correlating CmdEvent at all, categorize as **inherent gap** — no signal exists in stripped replay data.

### Phase 3: Add overrides for recoverable upgrades

For each recoverable upgrade:

1. Add the abilLink→UpgradeType mapping to `AbilityProfile.buildHsc2025Overrides()` (tournament replays) or `AbilityMapping` (4.9.3 direct constants)
2. Add a unit test in `AbilityMappingTest` verifying the new dispatch path
3. Rerun the accuracy report to confirm the fix

Follow the existing patterns:
- Generic abilLinks (shared by multiple upgrades from one building): switch on `abilCmdIndex` (`idx`)
- Building-disambiguated generics (177 Protoss, 195 Zerg): switch on `unitLink` then `idx`
- Building-specific tournament abilLinks: direct mapping

### Phase 4: Document known gaps

Create or update `docs/upgrade-detection-gaps.md` listing each inherent gap:

```
| UpgradeType | Gap reason | Replays affected |
|-------------|-----------|-----------------|
| SomeUpgrade | No CmdEvent emitted in stripped replay | 3/179 |
```

### Phase 5: Enforce threshold

Add an accuracy assertion to `StrippedReplayValidationTest`:

```java
double upgradeAccuracy = (double) totalDetected / totalOracle;
assertTrue(upgradeAccuracy >= 0.95,
    "Upgrade detection accuracy " + upgradeAccuracy + " below 95% threshold");
```

Set the threshold at the achieved accuracy (aspiring to 100%, asserting at the level we actually reach minus a small margin for inherent gaps).

## Scope boundaries

- **In scope:** abilLink overrides, accuracy reporting, threshold assertion, gap documentation
- **Out of scope:** Changes to `StrippedReplayFeatureExtractor` parsing logic, changes to `UpgradeType` enum (all 83 types already defined), changes to oracle replay infrastructure

## Testing

- `StrippedReplayValidationTest` (report mode) — primary validation, upgraded with accuracy assertion
- `AbilityMappingTest` — unit tests for each new override
- `AbilityDiscoveryCalibrationTest` — discovery diagnostics to find abilLinks

## References

- `AbilityProfile.java:50-348` — existing HSC_2025 overrides
- `AbilityMapping.java:39-763` — V4_9_3 abilLink constants
- `AbilityDiscoveryCalibrationTest.java:72-166` — existing discovery diagnostics
- `StrippedReplayValidationTest.java:104-223` — existing divergence report
- `UpgradeType.java` — 83 upgrade enum values
- Issue #350 — unitLink-aware building-type disambiguation
- Issue #338 — initial 78.6% accuracy with 6 tournament overrides
- Protocol `sc2data-train-times-require-calibration.md` — calibration methodology
