---
layout: post
title: "Two Wrong Build Times"
date: 2026-05-25
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
---

The sub-tick timing fix for unit training had one known gap: buildings still used integer tick counts, no loop offset, whatever we'd written down.

Writing the building calibration test turned out to be simpler than the training version. For units you need to correlate GAME_EVENTS commands with tracker UnitBorn events — two streams, missed selections creating noise, a range-bounded modal to find the correct pairs. For buildings, both endpoints live in tracker events: `UnitInit.getLoop()` when construction starts, `UnitDone.getLoop()` when it finishes. T_real = done − init, matched by tag. No cross-stream matching needed.

First run across 30 replays: Nexus, 1600 loops, n=71.

The `buildTimeInTicks` switch had no NEXUS case. It fell through to `default → 40 ticks` — 880 loops. The actual value is 1600 loops, 72 ticks. A 32-tick error sitting in the default case, undetected because the replay harness always injects Nexus constructions from ground truth; only agent-issued builds go through `handleBuild`. Every test covering `buildTimeInTicks(NEXUS)` ticked 40 times against the wrong constant, and the building completed in 40 ticks against the same wrong constant. The test suite agreed with itself. Neither was right.

Factory came up next: 400 loops, n=32. Thirty-two observations looks solid. We updated the constant and moved on.

The calibration test then failed. Factory at Faster speed is ~960 loops (~43 seconds). Four hundred is the addon build time.

`Sc2ReplayShared.toBuildingType` maps `FactoryTechLab`, `FactoryReactor`, and `FactoryFlying` all to `BuildingType.FACTORY`. In these replays, Terran bots place factories at game start — those arrive via `UnitBorn` events, which generate no UnitInit/UnitDone pairs. During the game, the bots complete addons. All 32 observations were addon completions. The modal was confident and wrong.

Filtering addon names before the type lookup changed the picture: 16 clean factory structures at exactly 960 loops — the estimate we'd just discarded.

```java
private static final Set<String> ADDON_OR_MORPH_NAMES = Set.of(
    "BarracksTechLab", "BarracksReactor", "BarracksFlying",
    "FactoryTechLab",  "FactoryReactor",  "FactoryFlying",
    "StarportTechLab", "StarportReactor",  "StarportFlying", ...
);
```

The sub-tick fix itself is the same formula as training — `gameFrame + (loopOffset + buildTimeInLoops(bt)) / LOOPS_PER_TICK`. `applyIntent(TimedIntent)` now passes `ti.loop()` to `handleBuild`; the enemy AI path defaults to loopOffset=0, same tick-aligned behaviour as before. For Nexus, the offset matters: 1600 % 22 = 16, so a build placed late in a tick completes a full tick later than one placed at the start.
