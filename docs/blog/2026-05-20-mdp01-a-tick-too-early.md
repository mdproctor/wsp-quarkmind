---
layout: post
title: "A Tick Too Early"
date: 2026-05-20
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [emulation, sc2, java, replay-validation]
---

The last entry ended with a claim: the ≤ 2 unit delta was the mining model.
That turned out to be wrong — or at least, incomplete.

Issue #141 fixed saturation mining. Mineral delta dropped by 10×. Unit
divergence didn't move. Still diverging at tick 36, still ≤ 2 through three
minutes. Wrong hypothesis.

The actual cause was in `startTraining`. When a `TrainIntent` arrives,
the completion frame is computed as `gameFrame + trainTimeInTicks(PROBE)` —
which is 12. Twelve outer ticks. But a Probe trains in 268.8 game loops, and
one outer tick is exactly 22 loops. Twelve ticks is 264 loops. The real train
time is 268.8 — four loops past the integer boundary, which is 0.22 of a tick.

That gap matters. When a player's train command arrives at loop offset 18 or
more within a tick window, the real unit completes 13 ticks later.
`EmulatedGame` was predicting 12. One unit appears a tick early. Repeated
across the replay with different command offsets, that's `firstUnitDivergenceTick
= 36` and a growing delta.

The fix required three coordinated changes.

First: `SC2Data` needed exact loop counts. `trainTimeInLoops(PROBE) = 268.8`,
not a rounded integer. `trainTimeInTicks` now derives from it —
`(int)(268.8 / 22) = 12` — same value as before, but the two can't drift apart.
`LOOPS_PER_TICK = 22` and `GAME_LOOPS_PER_SECOND = 22.4` went in too; they
were implicit before, scattered as magic numbers.

Second: `TimedIntent` needed to move from `sc2.replay` to `sc2.intent`.
It's a general SC2 concept — an intent tagged with its raw game loop — not a
replay-parsing artefact. Leaving it in `sc2.replay` would have created a downward
package dependency if `EmulatedGame` accepted one directly. Moving it there was
the right call on its own terms; the package layering was just the immediate reason.

Third: `EmulatedGame.applyIntent(TimedIntent)`, which passes the absolute game
loop through to `startTraining`:

```java
int  loopOffset  = (int)(absLoop % SC2Data.LOOPS_PER_TICK);
long completesAt = gameFrame
    + (int)((loopOffset + SC2Data.trainTimeInLoops(unitType)) / SC2Data.LOOPS_PER_TICK);
```

For a command at offset 0: `(0 + 268.8) / 22 = 12.2` → floor 12 — same as
before. For offset 18: `(18 + 268.8) / 22 = 12.99` → floor 12. For offset 19:
`(19 + 268.8) / 22 = 13.08` → floor 13. The real boundary is at 17.2. Offsets
18 and above cross it.

We had Claude verify the formula by brute force — every unit type, every loop 0
through 3000 — and it came back clean. Zero cases where the emulated prediction
diverges from the real SC2 completion tick.

`firstUnitDivergenceTick` moved from 36 to 86. The ≤ 2 delta is still there.

The remaining divergence is resource-timing. `EmulatedGame` accumulates around
1800 more minerals than the real player by tick 86 — the flat mining model has
no saturation cap. When the emulated player issues a train command, it always has
enough minerals. Sometimes the real player didn't quite, and the unit trained one
tick later. That's #146, and it needs the saturation model from #141.

The tick rounding bug was hiding behind the mining model assumption.
The honest answer to "why ≤ 2 delta" is still "the mining model" — but the
divergence starting at tick 36 was something else entirely.
