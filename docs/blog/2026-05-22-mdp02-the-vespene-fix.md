---
layout: post
title: "The Vespene Fix"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
---

The previous entry ended with gas units as the remaining divergence: `EmulatedGame` starts with 0 vespene and earns none, so every `TrainIntent` for a Stalker (50 gas) or Immortal (100 gas) is silently rejected. Each rejection adds one to the unit-count delta. At three minutes with two gas units trained, `maxUnitDelta` sat at 2 and climbing.

The fix looked simple: sync vespene from the replay ground truth, the same way we already sync probe count and supply cap.

## Pre-tick or post-tick: more consequential than it looks

The harness loop runs: snapshot ground truth before the tick, tick both games, snapshot after, then apply `TimedIntent`s. Probes are synced pre-tick (so mineral income uses realistic inputs); supply is synced post-tick. Vespene should be pre-tick — the player had a certain amount of gas *before* issuing train commands, and the `TimedIntent`s we apply represent those same commands. Syncing from `gtBefore` means the resource check sees what the player actually had; the TrainIntent deduction then mirrors the real expenditure without double-counting.

The first implementation used `gt.vespene()` — post-tick. The delta shifted but didn't go away. Claude had taken the simpler-looking option. A spec review caught it: `gt` vs `gtBefore`, one variable. Fixed.

## The residual wasn't from gas

With the pre-tick sync in place, `maxUnitDelta` dropped from ≤15 to ≤2. Not zero.

The 2 remaining units are Stalkers completing ±1 tick off from the replay at the edge of the three-minute window. Same sub-tick imprecision we'd already documented in the `drainBuildingQueues` comment — when a train command fires at loop 13 within an outer tick, we lose that offset when computing the completion frame. For the gas units in this particular replay, the imprecision surfaces because they're trained near the 150-tick mark, close enough to the boundary to show.

The `economicallyAccurate()` assertion — which I'd written expecting zero divergence after the vespene sync — would always be false. We tightened to `maxUnitDelta ≤ 2` instead. An honest bound, not a concession.

## A Java 21 exhaustiveness gap, quietly closed

Two `applyIntent` overloads in `EmulatedGame` dispatch over the sealed `Intent` interface using switch statements. Java 21 switch *statements* with pattern matching compile cleanly even if a new sealed subtype is added without updating the switch — `MatchException` at runtime, not a compile error. Switch *expressions* enforce exhaustiveness. For void dispatch, the idiomatic route is a `Runnable` switch:

```java
Runnable action = switch (intent) {
    case MoveIntent   m -> () -> setTarget(m.unitTag(), m.targetLocation(), false, state);
    case AttackIntent a -> () -> setTarget(a.unitTag(), a.targetLocation(), true,  state);
    case TrainIntent  t -> () -> handleTrain(t, state);
    case BuildIntent  b -> () -> handleBuild(b, state);
    case BlinkIntent  b -> () -> executeBlink(b.unitTag(), state);
};
action.run();
```

Each arm wraps the void call in a lambda. The switch is now an expression; the compiler enforces exhaustiveness against the `permits` clause. No behaviour change; adding a new `Intent` subtype without updating either dispatch method is now a compile error.
