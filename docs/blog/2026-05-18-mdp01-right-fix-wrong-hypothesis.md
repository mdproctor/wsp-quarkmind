---
layout: post
title: "The right fix, the wrong hypothesis"
date: 2026-05-18
type: phase-update
entry_type: note
subtype: diary
projects: [QuarkMind]
tags: [sc2, emulation, mining]
---

The hypothesis going into this session was simple: EmulatedGame's flat mining
rate accumulates minerals too fast, the surplus lets it afford units one tick
early, and unit counts diverge. Fix the mining model, fix the divergence.

It was wrong. The mining model needed fixing regardless — but not for that reason.

## The flat model's actual problem

EmulatedGame calculated mineral income as `miningProbes * MINERALS_PER_PROBE_PER_TICK`,
where `MINERALS_PER_PROBE_PER_TICK` is 50 min/min scaled to a per-game-loop rate.
The harness set `miningProbes = probeCount * 22` — 22 game loops per outer tick — so
`miningProbes` stored probe-count-times-22, not probe count. A confusing unit with
no obvious meaning.

More importantly, the rate was wrong. Real SC2 has a three-tier saturation model:
the first 8 probes per base earn full rate, the next 8 earn half, the next 8 earn
roughly 10%. Beyond 24 probes per Nexus, additional income approaches zero. The flat
model ignores this — probe 20 earns the same as probe 1. Over the full replay, that
produces an 11,564-mineral overestimate.

## The tier model

I worked through the design with Claude. We replaced the flat multiplication with a
piecewise function iterating over an array of rates:

```java
public static final double[] MINERAL_TIER_RATES_PER_TICK = {
    50.0 / 60.0 * 22.0 / 22.4,   // first worker per patch
    25.0 / 60.0 * 22.0 / 22.4,   // second worker per patch
     5.0 / 60.0 * 22.0 / 22.4,   // third worker per patch
};

public static double mineralIncomePerTick(final int probeCount) {
    double income = 0;
    for (int tier = 0; tier < MINERAL_TIER_RATES_PER_TICK.length; tier++) {
        int tierStart    = tier * MINERAL_PATCHES_PER_BASE;
        int probesInTier = Math.min(Math.max(probeCount - tierStart, 0), MINERAL_PATCHES_PER_BASE);
        income += probesInTier * MINERAL_TIER_RATES_PER_TICK[tier];
    }
    return income;
}
```

The array structure was a deliberate choice over three named constants. The length of
the array defines how many tiers exist — adding a fourth tier for a different game
mechanic is one element, not a structural change. `MINERAL_PATCHES_PER_BASE` (8)
determines tier width. The harness change was a single line: stop multiplying probe
count by 22 before passing it in.

## The hypothesis fails the test

We ran the full-replay divergence report with the new model. Mineral delta: 11,564 → 1,243.
Roughly ten times better.

First unit divergence: tick 36. Same as the flat model.

I compared both models' tick-by-tick reports. The unit divergence appears at the same
tick, with the same pattern, in both. That proves the hypothesis by elimination —
minerals aren't the cause.

The actual cause is train-timing precision. `SC2Data.trainTimeInTicks` uses integer
outer ticks: a Probe is 12, which is 264 game loops. The real SC2 value is 268.8
loops — 12.22 outer ticks. When a player's train command falls late in a tick window,
EmulatedGame completes the unit one tick early. The fractional part gets truncated.

The mineral improvement stands on its own — economic accuracy matters for the strategy
plugin's reasoning, and the saturation model is the correct representation of how SC2
works. The unit count divergence is a different bug, filed separately.

## What the model still doesn't know

One Nexus base has 8 mineral patches. The saturation model handles that correctly.
Once a second Nexus is active, the function doesn't know — all probes are treated as
mining one base, understating income above about 24 workers. Multi-base is tracked
but out of scope here.
