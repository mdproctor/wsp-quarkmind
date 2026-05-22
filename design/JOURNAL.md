# Design Journal — issue-149-completesat-rounding

### 2026-05-22 · §Mock Infrastructure

`SC2Data.trainTimeInLoops(UnitType)` changed from `double` to `int`. The float literals (e.g. 268.8 for Probe) were derived from `seconds × 22.4 loops/sec` but SC2 stores training times as integer game loops internally, not as floats. Using floats caused floor division to give 12 ticks instead of 13 for loop offsets in [14, 17], producing a systematic 1-tick-early unit completion that showed as `firstUnitDivergenceTick = 86` in `ReplayValidationHarness`. The empirical integer values (PROBE=272, ZEALOT=618, STALKER=698) were calibrated directly from 29 AI Arena replays by pairing GAME_EVENTS training commands with tracker UnitBorn events. `EmulatedGame.startTraining` removes the `(int)` cast so arithmetic is fully integer — floor is exact and no floating-point boundary errors can occur. The type change is the authoritative fix; the formula change is a consequence.

### 2026-05-22 · §Domain Model

In the SC2Data description, update `trainTimeInLoops(UnitType)` example from "e.g. Probe=268.8" to "e.g. Probe=272 (empirically calibrated)". The float literal was wrong — SC2 stores training times as integers, and the calibrated value is 272 loops for Probe, not 268.8.

### 2026-05-22 · §Next Steps

Remove the #142 train-timing precision item — resolved by #149 (empirical calibration and integer formula fix). The `completesAt` rounding issue at tick 86 is fixed; `firstUnitDivergenceTick` is now 150 (sole remaining cause: #148 vespene gap). Replace with: "**#148 Vespene income model** — EmulatedGame starts with 0 vespene; train commands for Stalkers (50 gas) and Immortals (100 gas) are rejected, causing growing unit delta proportional to gas units trained."

### 2026-05-22 · §Testing Strategy

`SC2TrainTimeCalibrationTest` establishes a two-source calibration methodology for ground-truth train-time validation. Source A (a single replay via the scelight binary parser) extracts training commands by abilLink directly — bypassing `AbilityMapping` selection tracking which can miss commands — and pairs them with UnitBorn tracker events using a range-bounded modal algorithm. Source C (28 other AI Arena replays, same patch) aggregates the same extraction across diverse game patterns for statistical confidence. The IEM10 JSON dataset was evaluated but is from a 2016 SC2 patch with different abilLink values (e.g. 167 for Nexus vs 175 in AI Arena), making it incompatible for direct cross-validation. The test currently covers PROBE (499 observations, high confidence), ZEALOT (7 obs), and STALKER (2 obs, low confidence); IMMORTAL and OBSERVER remain uncalibrated until replay corpora with Robotics Facility usage are available.
