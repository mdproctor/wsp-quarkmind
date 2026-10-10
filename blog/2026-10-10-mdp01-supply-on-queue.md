---
title: "Where 10 SCVs Go To Die"
date: 2026-10-10
author: Mark Proctor
entry_type: note
subtype: diary
tags: [emulated-game, terran, economy, fidelity, sc2-physics]
projects: [casehubio/quarkmind]
series: issue-398-terran-scv-production-throughput
---

The Terran economy playbook was producing 40 SCVs at the 5-minute mark. Real SC2 Terran players hit 48-50. I expected the gap to be about supply depot timing — real players pre-build depots before hitting cap, our playbook was building them at the cap. Straightforward fix, maybe a session's work.

The depot timing was part of it, but the real finding was elsewhere entirely.

## The morph gap

When a Command Centre morphs to Orbital Command, the building goes offline for 25 ticks. In our EmulatedGame, `handleTrain()` correctly blocks new training requests on incomplete buildings — but `drainBuildingQueues()` didn't check `isComplete` at all. Queued SCVs would happily start training on a building that was mid-morph. In real SC2, the production queue pauses during a morph and resumes when it completes.

This was producing *phantom SCVs* — units that shouldn't exist according to SC2 rules. The fidelity bug was inflating our SCV count, not deflating it. Fixing it dropped us from 40 to 39 SCVs.

The fix itself was three lines:

```java
boolean buildingReady = state.buildings().stream()
    .anyMatch(b -> b.tag().equals(buildingTag) && b.isComplete());
if (!buildingReady) continue;
```

No test existed for the morph-during-train scenario. I added three: in-progress SCV completes during morph (correct behaviour, regression test), queued SCVs pause during morph (the bug), and queued SCVs resume after morph completes.

## The queue ate the supply

With the morph fixed and a 2nd OC morph added to the playbook, I started iterating on build orders to close the gap to 48. Tried earlier CC-2 (at 16 supply instead of 20 — failed, not enough minerals). Tried delayed OC morph (worse — losing MULE income hurts depot funding). Tried a 3rd CC (spent all the minerals on the building, didn't help). Every variant landed between 39 and 41 SCVs.

Then I looked at the supply numbers. At tick 305: `supplyUsed=50`, actual SCVs: 40. Ten SCVs were missing.

`handleTrain()` commits supply the moment a unit is *queued*, not when it starts training. With queue depth 5 per building and 2 production buildings, up to 10 supply slots are locked by SCVs that don't exist yet. Real SC2 commits supply when training starts — queued units are reservations, not consumers.

This single accounting difference means our EmulatedGame needs ~10 more supply cap than real SC2 to achieve the same worker count. The playbook was building depots, hitting supply triggers, consuming supply through the queue, and leaving completed SCVs 10 behind the committed count. The gap to 48 isn't a build order problem — it's a supply model fidelity issue.

## What landed

Two things shipped: the `drainBuildingQueues` fidelity fix (queue pauses during morphs) and a playbook rewrite with the 2nd OC morph. SCV count went from 40 to 41. Not the 48 I was aiming for, but the ceiling is structural — fixing it means changing when supply is committed in `handleTrain`, which touches every unit training path in the game.

The supply-on-queue fix is now its own issue. It's the kind of change where getting it right matters more than getting it fast — the supply accounting flows through queue management, the delivery handler's pre-checks, and potentially the replay validation harness. One line in the wrong place and every race's calibration shifts.
