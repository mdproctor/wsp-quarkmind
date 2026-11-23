---
layout: post
title: "The Assumption That Wasn't"
date: 2026-11-23
author: Mark Proctor
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [quarkmind, sc2, emulated-game, economy, cycle-variance, diagnostics]
series: issue-534-emulated-game-100-accuracy
---

# The Assumption That Wasn't

*Continues the [100% accuracy series](2026-11-23-mdp01-halving-the-distance.md).*

For months, the working hypothesis was clear: the emulator's worker cycles were too uniform. SC2 workers showed cycle variance of σ≈3+ ticks while the emulator sat at σ≈0.5. The fix path was equally clear — dynamic patch selection, walk-path collisions, contention modelling. Three issues filed, each addressing a different variance source.

The hypothesis was wrong.

When I finally measured like-for-like — same scenario, same worker count, same map geometry — the emulator's cycle variance was *higher* than SC2's. Controlled 12-worker economy: SC2 σ=0.56, emulator σ=0.85. The emulator was more jittery, not less.

The original comparison had been apples-to-oranges. The σ≈3+ figure came from real game replays where workers transfer between expansions, get pulled for building, and walk across maps. The σ≈0.5 came from a controlled calibration. Comparing a controlled emulator against wild SC2 data, then concluding the emulator needed more variance — that's the kind of mistake that feels obvious in hindsight and invisible while you're making it.

The oracle replay data told the rest of the story. When we ran the emulator in standalone mode across 118 real game replays, cycle σ was 7-10 ticks. High variance already existed. But the variance came from walk distances — workers moving between bases at different distances produce cycle times ranging from 2.5 to 236 ticks. The phase decomposition made it unambiguous:

| Phase | σ (ticks) |
|---|---|
| Walk to patch | 4.4-5.7 |
| Walk to base | 3.3-3.6 |
| Patch contention wait | 0.5-0.6 |
| Mining | 0.0 |
| Deposit | 0.0 |

Walk distance dominates. Mining and deposit phases are invariant — fixed durations, zero variance contribution. The contention wait adds modest jitter. But the overwhelming variance source is simply how far workers have to walk, which varies as expansions come online and workers redistribute.

Three issues closed in one diagnostic. Dynamic patch selection, walk-path collisions, contention modelling — none of them would have moved the needle on the metric we cared about. The 38% mineral rate MAPE has a different root cause, and we're closer to finding it now that we're not chasing phantom variance.

The uncomfortable lesson: a hypothesis that explains the data well enough to plan three issues of work can still be entirely wrong. The measurements that validated it were real. The comparison that produced them wasn't.
