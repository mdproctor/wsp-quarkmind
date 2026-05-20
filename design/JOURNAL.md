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
