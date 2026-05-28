---
layout: post
title: "Three Ways to Extract Nothing"
date: 2026-05-28
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
---

The IEM10 dataset has 30 tournament replays from 2016. Until now we'd used
them for tracker events only — building state, counting units, tracking births.
The `gameEvents` array was sitting there the whole time, encoding the commands
players actually issued: which building, which unit, at which loop. Exactly
what the validation model needs.

The problem with reading `gameEvents` is that everything fails silently.

The first failure mode is patch-specific abilLink constants. `AbilityMapping`
was calibrated from 2023 AI Arena replays; IEM10 is a 2016 build, and the
ability IDs shifted between patches. If you use the wrong constants you get
zero training commands with no error — we'd documented this one already.
What we didn't have was the full 2016 table.

So before writing any extraction code we ran a discovery test: narrow-window
modal correlation across all 30 games. For each unit type visible in tracker
event births, find the abilLink that most consistently predicts those births
within the expected training-time window. The table came back with NEXUS=167,
GATEWAY=164, ROBOTICS=166, STARGATE=165, LARVA=185. All different from the
AI Arena values. All verifiable against birth counts. We hardcoded those
constants with the patch version in a comment, and a diagnostic test that can
re-derive the table when needed.

The second failure mode is the player filter. `gameEvents` entries carry
`userid.userId` to identify the player who issued the command. The natural
assumption — `userId = playerID - 1` — holds in some games and silently
returns zero commands in others. SC2EGSet's `ToonPlayerDescMap` has a
separate `userID` field that is the correct key; it's independent of
`playerID`, and in the IEM10 dataset some players have `userID=4` where
their `playerID=2`. We'd seen this before (GE-20260522-f0d1ec) but it
needed explicit handling in the new extractor.

The third failure mode was discovered mid-implementation. Claude reached for
`Sc2ReplayShared.makeTag()` to construct building tags from selection event
data — package-private, same package, same `makeTag(int index, int recycle)`
signature. The natural reuse. It returned zero intents. The problem:
`Sc2ReplayShared.makeTag()` produces `r-index-recycle` tags, the format for
binary SC2Replay files. IEM10 JSON uses `j-index-recycle`. Both are valid
strings; neither matches the other; no exception is thrown. The fix is
inlining `"j-" + index + "-" + recycle` directly and adding a comment
explaining why not to call the shared method here.

Three silent failures, all navigated. We also had to handle that `gameEvents`
SelectionDelta entries carry unit tags as packed 32-bit integers — `index =
packed >> 18; recycle = packed & 0x3FFFF` — which we verified against tracker
event tags before trusting. And about 13 of 30 games have an extra UnitBorn
at loop 0 that puts the initial unit count at 13 instead of 12; the
validation harness needed a ±1 tolerance for that.

I wanted the harness to work with IEM10 games directly. `ReplayValidationHarness`
previously expected a binary `.SC2Replay` path, but `ReplaySimulatedGame`
and `IEM10JsonSimulatedGame` both extend `SimulatedGame`, so the fix was one
new overload: `run(SimulatedGame groundTruth, List<TimedIntent> intents, int
tickLimit)`. The binary overload delegates to it now.

The multi-game run finished all 30 games. Baseline: PvP averages 9.6 units
off, PvT 22.6, PvZ 40.1. Zerg larva training is less complete in the 2016
ability table — Mutalisk and Hydralisk still need disambiguation — so the PvZ
number is expected to be higher. It's a starting point.
