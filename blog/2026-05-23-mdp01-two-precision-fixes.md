---
layout: post
title: "Two Precision Fixes"
date: 2026-05-23
type: phase-update
entry_type: note
subtype: log
projects: [quarkmind]
---

The previous entry ended with a ±1 tick residual in the replay validation — Stalkers completing one tick early because `drainBuildingQueues` passes `0L` to `startTraining` for the next queued unit, losing the sub-tick loop offset. I'd documented it in a comment and moved on. This session, we fixed it.

## Propagating the completion loop

The mechanism is straightforward. When `startTraining` fires, it computes the absolute game loop at which the unit will complete: `absLoop + trainTimeInLoops(unitType)`. We now store that value in a new `buildingCompletionAtLoop` map on `PlayerState`. When `drainBuildingQueues` starts the next queued unit, it reads the stored loop and passes it through instead of `0L`.

The effect: if Probe A was trained at loop 10, it completes at loop 282. The sub-tick offset of 282 — that's `282 % 22 = 18` — now carries forward to Probe B's completion calculation. `(18 + 272) / 22 = 13` ticks, not the `(0 + 272) / 22 = 12` ticks you get with the zeroed offset. One tick of difference, exactly the ±1 we'd been seeing.

Claude caught a leak during code review: when a building trains a single unit with no queue behind it, the completion callback clears `buildingTrainingUntil` but leaves the `buildingCompletionAtLoop` entry orphaned. The drain path only runs for buildings with queues, so non-queued buildings accumulate stale entries. Functionally harmless — overwrite-on-reuse prevents incorrect behaviour — but semantically dirty. The fix: a conditional cleanup in the completion callback that only removes the entry when no queue exists, preserving it for drain when one does.

## Multi-base mining was a framing problem

Issue #143 looked like a function signature change: extend `mineralIncomePerTick(int probeCount)` to accept a base count, distribute probes evenly. I almost went with that until I asked: what is this actually for?

It's for replay validation. And in a replay, the player doesn't distribute probes evenly. They might have 16 on their main and 4 on their natural. Even distribution models optimal play, not actual play — and the whole point of the harness is matching what actually happened.

Once I reframed it as a replay accuracy fix, the design inverted. `mineralIncomePerTick` is already a per-base function — it computes income for N probes mining one base. The change isn't in SC2Data at all. It's in the caller.

`ReplayValidationHarness` now has `countProbesPerBase(GameState)`, which assigns each probe to its nearest complete Nexus by Euclidean distance. `EmulatedGame` takes `int... probesPerBase` instead of a scalar count, and `tick()` sums income across bases. The single-base case — one element in the array — behaves identically to before.

The approximation: probes in transit (scouting, transferring between bases) get assigned to their nearest nexus. The current model already counts all probes as mining regardless of what they're doing, so this is the same level of approximation, just distributed. When the replay corpus extends past the 3-minute single-base window, the per-base distribution should visibly reduce mineral divergence.
