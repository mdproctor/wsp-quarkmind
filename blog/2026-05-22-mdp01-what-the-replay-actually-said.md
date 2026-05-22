---
layout: post
title: "What the Replay Actually Said"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: log
projects: [QuarkMind]
---

The previous entry ended with a diagnosis: the `completesAt` formula uses float arithmetic (268.8 for Probe — derived from 12 seconds × 22.4 loops/second), which causes floor division to give the wrong tick at certain loop offsets. I expected the fix to be simple. Find the nearest integer, apply it, done.

The right integer turned out not to be what I expected.

## 268.8 wasn't being rounded the wrong way — it was just wrong

I brought Claude in to build a calibration test. We'd pair training commands from GAME_EVENTS with unit-born tracker events from 29 AI Arena replays and compute `T_real = born_loop − command_loop` for each unit type.

The first matching attempt was straightforward: sort commands by loop, sort born events by loop, pair them i-th to i-th. Non-queued training gives a diff of exactly T_real; queued units give a larger diff. Take the minimum. Simple.

It produced `PROBE=182` and `STALKER=7`. A Stalker supposedly training in 7 game loops.

The cause was AbilityMapping. It tracks the currently-selected building to associate train commands with a specific gateway or nexus. When a selection event is missed, subsequent commands are dropped from the stream. The remaining commands then pair with the wrong born events in the sorted list, producing nonsense diffs.

## The modal approach

We dropped FIFO entirely. For each unit type, compute all `(born_loop − command_loop)` pairs where the diff falls in the expected range — say [256, 282] for Probe. Count how often each value appears. The modal value is T_real.

```java
Map<Integer, Integer> freq = new TreeMap<>();
for (int born : borns) {
    for (int cmd : cmds) {
        int diff = born - cmd;
        if (diff >= tMin && diff <= tMax) {
            freq.merge(diff, 1, Integer::sum);
        }
    }
}
```

Non-matching pairs produce random diffs that won't cluster. Correct pairs all produce exactly T_real for non-queued training. The mode wins cleanly over the noise.

Across 29 replays: `PROBE=272` with 499 observations. `ZEALOT=618` with 7. `STALKER=698` with 2 — low confidence, but directionally consistent. The single default replay agreed on 272 with 56 observations.

I'd expected 269 (ceiling of 268.8). SC2 stores training times as integer game loops, not derived from a seconds formula, and the integers are not consistently ceil or floor — they're just whatever the game engine uses.

## IEM10 was the wrong era

I'd planned to cross-validate against the IEM10 JSON dataset, 30 professional games from 2016. The SC2EGSet format includes both `gameEvents` and `trackerEvents`, so I assumed the same abilLink constants would work.

They don't. The IEM10 replays are from build 39948 (2016). The AI Arena replays are from 2023+ builds. SC2 patches change abilLink IDs — Nexus training is abilLink=175 in the AI Arena data and abilLink=167 in IEM10. We returned 0 training commands across 30 games before Claude surfaced the diagnosis: wrong `userId` field (the player map's `userID` ≠ `playerID − 1` in SC2EGSet), then wrong abilLink constants entirely once we fixed the filter.

The second source became the other 28 AI Arena replays instead. They agreed on PROBE=272.

## The fix is two lines

`SC2Data.trainTimeInLoops` changes from `double` to `int`:

```java
case PROBE    -> 272;  // empirical (499 obs)
case ZEALOT   -> 618;  // empirical (7 obs)
case STALKER  -> 698;  // empirical (2 obs)
```

And `EmulatedGame.startTraining` drops the float cast:

```java
long completesAt = gameFrame
    + (loopOffset + SC2Data.trainTimeInLoops(unitType)) / SC2Data.LOOPS_PER_TICK;
```

Before: `(17 + 268.8) / 22 = 12.99... → 12`. After: `(17 + 272) / 22 = 13.14... → 13`. One tick at the boundary offsets, which was the entire divergence at tick 86.

`firstUnitDivergenceTick` is now 150. The remaining delta is gas units — Stalkers and Immortals are rejected because EmulatedGame has no vespene model. Each one the real player trains adds one to the delta. That's the next problem.
