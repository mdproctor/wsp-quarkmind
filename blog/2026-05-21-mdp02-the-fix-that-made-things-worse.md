---
layout: post
title: "The Fix That Made Things Worse"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: log
projects: [QuarkMind]
---

The previous entry ended with building costs as the remaining divergence cause. That turned out to be right about the cause, wrong about the fix — and investigating why led somewhere more interesting.

The hypothesis: EmulatedGame injects replay buildings for free, so the emulated mineral bank accumulates above ground truth. Enough excess that EM can afford a train one tick before the real player. The fix should be straightforward — deduct the building's mineral cost when the harness injects it.

We added `injectReplayBuildingWithCost(Building)` to `EmulatedGame`:

```java
public void injectReplayBuildingWithCost(Building building) {
    friendly.minerals -= SC2Data.mineralCost(building.type());
    friendly.buildings.add(building);
}
```

The first attempt clamped minerals at zero — if EM couldn't afford the full cost, floor it. `firstUnitDivergenceTick` moved from 86 to 49, and `maxUnitDelta` jumped from 2 to 48. Clearly wrong. Stripped the floor, let minerals go negative and recover through income. Still broken. `maxUnitDelta = 78`.

The deduction was reversing the direction of the divergence. Pre-fix, EM had too many minerals. Post-fix, EM was failing to train things it should.

The full divergence report for the pre-fix state explained it. At tick 86 — the tick under investigation — EM has 505 minerals and GT has 375. EM already has 130 more than ground truth. The unit that diverges there isn't a case of EM training it because it had just enough minerals earlier. The completion simply calculates to tick 86 in EM when the real SC2 observation places it at tick 87. That's a timing formula discrepancy, not a mineral one.

Building cost deductions can't fix a rounding issue in `completesAt`. The mineral excess is real, but it isn't the mechanism behind the specific divergence we were trying to close.

The deduction made things worse for a separate reason. SC2 `PlayerStats` events fire approximately every 10 game-time seconds — around 10 outer ticks at Faster speed. The GT mineral reading at the tick a building appears is stale by up to 10 ticks. The real player had accumulated that extra income before paying for the building. EM's continuous balance lagged behind, so deducting the full cost against it created artificial debt that blocked legitimate trains.

The correct picture has two independent causes. First: `completesAt` rounds one tick early for certain loop offsets — that's the tick-86 gap, and the fix belongs in the timing formula. Second: EmulatedGame has no vespene income model. Train commands for Stalkers (50 gas) and Immortals (100 gas) are rejected, adding 1 unit of divergence per gas unit the real player trains. Both filed.

`injectReplayBuildingWithCost` stays. The API is clean, the no-floor design is correct — negative minerals represent real debt that recovers through income, which is exactly what happened in the game. It just doesn't solve the problem I thought it would.
