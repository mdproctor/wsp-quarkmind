---
layout: post
title: "When Types Flatten, Costs Don't"
date: 2026-11-07
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [calibration, accuracy, simulation, starcraft2, domain-modelling, type-systems, digital-twin]
series: issue-534-emulated-game-100-accuracy
---

# When Types Flatten, Costs Don't

Technology spending was overcounting by 30%. The emulator charged 150 minerals every time a Terran player built a Reactor. The actual cost is 50.

The physics engine didn't have a bug. The cost lookup didn't have a bug. The problem was upstream — in a type-mapping function that deliberately discards variant information.

## The type flattening decision

StarCraft II has six add-on types: `BarracksReactor`, `BarracksTechLab`, `FactoryReactor`, `FactoryTechLab`, `StarportReactor`, `StarportTechLab`. Each is a distinct entity in the replay's tracker events. Each has its own cost — 50 minerals and either 50 gas (Reactor) or 25 gas (TechLab).

The emulator's `BuildingType` enum has no entries for add-ons. It tracks the parent production building — BARRACKS, FACTORY, STARPORT — and maps all add-on variants to the parent. `toBuildingType("BarracksReactor")` returns `BuildingType.BARRACKS`. This is a deliberate design choice. For entity counting, it's correct — the building accuracy baseline already tracks add-ons as instances of their parent type, and both the ground truth and the emulator agree on the count. 100% building accuracy at every checkpoint.

But `BuildingType` isn't just used for counting. It's also used for cost lookup. `SC2Data.mineralCost(BuildingType.BARRACKS)` returns 150. When `handleBuild` processes a `BuildIntent` for a Barracks Reactor, it deducts 150 minerals and 0 gas. The actual cost is 50 minerals and 50 gas.

Every add-on over-deducts roughly 100 minerals. Across 236 player runs, that compounds.

![Type flattening: add-on variants lose their cost identity when mapped to parent BuildingType](images/addon-type-flattening.svg)

## The fix: a parallel dimension

Adding REACTOR and TECH_LAB to `BuildingType` would fix the cost lookup but would also require updating every exhaustive switch expression over the enum — dozens of them, most of which correctly treat add-ons as their parent type.

The insight is that the type hierarchy is right for everything *except* cost. So the fix adds a parallel dimension: an `AddonKind` enum with three values — `NONE`, `REACTOR`, `TECH_LAB` — carried on the `BuildIntent` record alongside the existing `BuildingType`. A backward-compatible constructor means every existing call site continues to compile unchanged. Only `TrackerIntentExtractor` passes a non-NONE value, and only `handleBuild` reads it.

```java
if (b.addonKind() != AddonKind.NONE) {
    mCost = SC2Data.ADDON_MINERAL_COST;      // 50
    gCost = b.addonKind() == AddonKind.REACTOR
            ? SC2Data.REACTOR_GAS_COST        // 50
            : SC2Data.TECH_LAB_GAS_COST;      // 25
} else {
    mCost = SC2Data.mineralCost(b.buildingType());
    gCost = SC2Data.gasCost(b.buildingType());
}
```

Six lines in the cost path. One enum. No changes to entity tracking, building completion, supply calculation, or any other consumer of `BuildingType`.

## What it fixed

Technology spending MAPE dropped by roughly half at every checkpoint. Mineral technology: 30.3% → 17.0% at 7-minutes. Vespene technology: 46.1% → 19.1% at 3-minutes.

![Technology spending MAPE before and after the add-on cost fix](images/tech-spending-mape-improvement.svg)

But the more revealing number is the aggregate gap. Across all 236 player runs, the mineral technology spending totals are now within 2.6% of the ground truth at 5-minutes — 122,275 vs 119,150. The per-replay MAPE is higher because individual replays have larger errors in both directions, but the systematic bias is gone. The remaining error is variance, not drift.

## The hypothesis that didn't hold

With minerals nearly correct, the vespene gap was still 32% at 5-minutes. One hypothesis: SC2 might classify add-on gas costs as army spending rather than technology. If Reactor gas was filed under army instead of technology, the technology overcount would shrink and the army numbers would absorb it.

I tested this by routing add-on spending to the army bucket and re-running the full 118-replay baseline. The result was unambiguous. Army vespene MAPE at 3-minutes jumped from 4.5% to 44.4%. Technology vespene improved at some checkpoints but worsened at others. The hypothesis was wrong — SC2 classifies add-on costs as technology, same as we do.

The remaining vespene gap is structural. SC2's `PlayerStatsEvent` tracks separate `InProgress`, `Current`, and `Lost` buckets for each spending category. Our tracking matches `Current` semantics — spending is recorded at building completion. But SC2's internal vespene accounting appears to differ from a simple cost × completed-building summation. Without access to the game engine's source, we can document the gap and move on. The mineral side is solved.

## The pattern: type hierarchies and their shadows

Domain models flatten variant types all the time. An ORM maps subtypes to a single table. A message bus collapses event variants into a common envelope. A game engine maps six add-on types to three production building types. In each case, the flattening is justified — it simplifies queries, routing, or entity tracking by discarding distinctions that don't matter for the primary use case.

The trap is that the type carries more than one semantic. `BuildingType.BARRACKS` means "this is a Barracks" for entity counting, but it also means "this costs 150 minerals" for spending tracking. The flattening is correct for the first semantic and wrong for the second. The bug doesn't appear where the type is consumed — `handleBuild` looks correct, `SC2Data.mineralCost` looks correct. It appears in the *mapping* layer, where a decision made for one purpose silently corrupts another.

The fix pattern is always the same: don't fight the type hierarchy. Don't add add-on types to `BuildingType` and update forty switch statements. Instead, carry the missing dimension alongside the existing type — a parallel field that preserves exactly the information the primary type discards. The type hierarchy continues to serve its original purpose. The parallel field serves the secondary one. Neither contaminates the other.

Every domain model that flattens variant types has at least one shadow like this — a semantic that the primary type carries correctly for its original consumers and incorrectly for someone downstream. The cost just happened to be measurable.
