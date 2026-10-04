# HANDOFF — quarkmind

## Last Session

Closed #351 (P1-2: upgrade detection accuracy). Started at 87.7%, ended at 100% gameplay accuracy (759/759 events across 118 oracle replays, patch 4.9.3). Three phases: fixed over-detection from duplicate abilLink mappings (WarpGateResearch +43, Charge +7, PunisherGrenades +12) → 97.8%; discovered abilLink 191 (HydraliskDen alt) via per-replay CmdEvent dump diagnostic → 99.7%; found abilLinks 608 (DarkShrine) and 69 (FleetBeacon) → 100%. Filed #363 (stress-test across patch versions) as prerequisite for #352 (ONNX expansion).

## Decisions

- AbiLinks 608/69 based on 1 data point each — threshold assertion at 99% not 100%
- Stress-test (#363) before ONNX expansion (#352) — validate detection reliability before wiring into classifier
- Blizzard replay API key available; SC2EGSet (17,930 replays) largely untouched

## References

| What | Where |
|------|-------|
| Gap docs | `docs/upgrade-detection-gaps.md` |
| Diagnostic | `AbilityDiscoveryCalibrationTest.dumpAllCmdEventsNearMissedUpgrades` |
| Garden entry | `GE-20261004-d738a8` — temporal filter technique |
| Epic queue | #363 (stress-test) → #352 (ONNX expansion) |
