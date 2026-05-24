# Design Journal — quarkmind

> Consolidated from closed issue branches. Each section is one branch's journal.

---

# Design Journal — issue-142-tick-rounding

### 2026-05-20 · §Domain Model

`SC2Data` now carries SC2 timing constants (`LOOPS_PER_TICK=22`,
`GAME_LOOPS_PER_SECOND=22.4`) and `trainTimeInLoops(UnitType)` as the single
source of truth for unit train durations in game loops. `trainTimeInTicks`
derives from it, eliminating the risk of drift between the integer-tick and
loop-count representations. The root issue was that integer-tick train times
caused systematic 1-tick-early unit completion when replay commands arrived
late in a tick window (offset ≥18 within a 22-loop tick). Exact loop values
(Probe=268.8, Zealot=627.2, etc.) let `EmulatedGame` compute the correct
fractional completion tick from the command's absolute game loop.

### 2026-05-20 · §Component Structure

`TimedIntent` moved from `sc2.replay` to `sc2.intent`, alongside `TrainIntent`,
`BuildIntent`, and the other intent types. It is a general SC2 concept — an
intent tagged with its absolute game-loop timestamp — not a replay-parsing
artefact. Moving it there enabled `EmulatedGame` to accept `TimedIntent` via a
new `applyIntent(TimedIntent)` overload without creating a package coupling from
`sc2.emulated` into `sc2.replay`. The `ReplayValidationHarness` now passes
`TimedIntent` directly to the engine, preserving the game loop for sub-tick
train-completion precision.

---

# Design Journal — issue-143-multi-base-mining

---

# Design Journal — issue-146-building-cost-mineral-timing

### 2026-05-21 · §Mock Infrastructure

`EmulatedGame` gains `injectReplayBuildingWithCost(Building)` alongside the existing free `injectReplayBuilding(Building)`. The Component Structure table entry for `EmulatedGame` should be updated to list `injectReplayBuildingWithCost(Building)` next to the existing harness API methods (`injectReplayBuilding`, `markReplayBuildingComplete`, `setMiningProbes`, `setSupplyCapForHarness`). The two methods represent distinct injection semantics: free injection is correct for buildings gifted at game start (the initial Nexus, which no player purchases); cost-deducting injection is the right model for buildings ordered during the game. Minerals may go negative — this is intentional, representing a short-lived mineral debt repaid through income. Clamping at 0 was evaluated and rejected: it masks the debt, causes EM to recover too quickly, and produces worse unit-count divergence than the unfixed baseline.

The harness routing between the two paths was designed but not wired in. Investigation showed that deducting building costs from EM's continuous mineral balance against stale GT PlayerStats readings creates artificial debt that blocks trains EM should not miss, reversing the direction of the divergence. The API is ready for future use when the harness mineral model is more precisely aligned with the real player's per-loop balance.

### 2026-05-21 · §Testing Strategy

Investigation into the residual unit-count divergence in `ReplayValidationHarness` clarified that `firstUnitDivergenceTick = 86` has two independent causes, neither of which is purely mineral-timing. (1) A completion-time formula gap in `startTraining`: `completesAt` rounds to 86 while SC2 places the unit's appearance at tick 87 for the specific loop offset of that train command. This is independent of mineral balance — deducting building costs does not affect it. (2) EmulatedGame has no vespene income model; Stalker and Immortal train commands (50 and 100 gas respectively) are always rejected, causing the harness to fall 1 unit behind GT each time the real player trains a gas unit. Filed as #148.

The validation test docstring was updated to document both causes explicitly so future contributors do not re-investigate the mineral path as a fix for the timing gap.

---

# Design Journal — issue-149-completesat-rounding

### 2026-05-22 · §Mock Infrastructure

`SC2Data.trainTimeInLoops(UnitType)` changed from `double` to `int`. The float literals (e.g. 268.8 for Probe) were derived from `seconds × 22.4 loops/sec` but SC2 stores training times as integer game loops internally, not as floats. Using floats caused floor division to give 12 ticks instead of 13 for loop offsets in [14, 17], producing a systematic 1-tick-early unit completion that showed as `firstUnitDivergenceTick = 86` in `ReplayValidationHarness`. The empirical integer values (PROBE=272, ZEALOT=618, STALKER=698) were calibrated directly from 29 AI Arena replays by pairing GAME_EVENTS training commands with tracker UnitBorn events. `EmulatedGame.startTraining` removes the `(int)` cast so arithmetic is fully integer — floor is exact and no floating-point boundary errors can occur. The type change is the authoritative fix; the formula change is a consequence.

### 2026-05-22 · §Domain Model

In the SC2Data description, update `trainTimeInLoops(UnitType)` example from "e.g. Probe=268.8" to "e.g. Probe=272 (empirically calibrated)". The float literal was wrong — SC2 stores training times as integers, and the calibrated value is 272 loops for Probe, not 268.8.

### 2026-05-22 · §Next Steps

Remove the #142 train-timing precision item — resolved by #149 (empirical calibration and integer formula fix). The `completesAt` rounding issue at tick 86 is fixed; `firstUnitDivergenceTick` is now 150 (sole remaining cause: #148 vespene gap). Replace with: "**#148 Vespene income model** — EmulatedGame starts with 0 vespene; train commands for Stalkers (50 gas) and Immortals (100 gas) are rejected, causing growing unit delta proportional to gas units trained."

### 2026-05-22 · §Testing Strategy

`SC2TrainTimeCalibrationTest` establishes a two-source calibration methodology for ground-truth train-time validation. Source A (a single replay via the scelight binary parser) extracts training commands by abilLink directly — bypassing `AbilityMapping` selection tracking which can miss commands — and pairs them with UnitBorn tracker events using a range-bounded modal algorithm. Source C (28 other AI Arena replays, same patch) aggregates the same extraction across diverse game patterns for statistical confidence. The IEM10 JSON dataset was evaluated but is from a 2016 SC2 patch with different abilLink values (e.g. 167 for Nexus vs 175 in AI Arena), making it incompatible for direct cross-validation. The test currently covers PROBE (499 observations, high confidence), ZEALOT (7 obs), and STALKER (2 obs, low confidence); IMMORTAL and OBSERVER remain uncalibrated until replay corpora with Robotics Facility usage are available.

---

# Design Journal — issue-153-152-review-nits-and-probe-dist

