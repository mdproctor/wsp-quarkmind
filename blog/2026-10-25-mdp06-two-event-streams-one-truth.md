---
layout: post
title: "Two Event Streams, One Truth"
date: 2026-10-25
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [replay-reconstruction, emulated-game, sc2, selection-tracking]
series: issue-407-exact-building-routing
---

# Two Event Streams, One Truth

SC2 replays contain two independent event streams. Tracker events record *what happened* — a Marine was born at loop 3200, an upgrade completed at loop 4800. Game events record *how the player did it* — they selected a Barracks, pressed 'a', and the Marine started training.

Until now, QuarkMind's replay reconstruction used only tracker events. When the EmulatedGame needed to decide which Barracks should train the Marine, it picked the one with the shortest queue. Usually that's right. Sometimes it isn't — the player selected a specific Barracks via hotkey, maybe because they'd attached a Reactor to it, or because it was closer to the rally point. The "shortest queue" heuristic doesn't know any of that.

The result: about 16 units at the 5-minute mark complete 1-2 ticks late because they're training in the wrong building. Physics is correct — the validation harness proved that at 99.8% accuracy. But reconstruction fidelity is lower than it could be.

## The cross-reference

The fix is straightforward once you see it. Game events include `SelectionDeltaEvent` — a record of exactly which units the player has selected at any moment. When a `CmdEvent` fires (the player issues a command), the current selection tells you which building received the order.

So: walk the game events once, maintaining a `SelectionState`. At every `CmdEvent`, record the selected building tag in a `TreeMap<Integer, String>` keyed by game loop. Then, when processing tracker events, back-calculate the command loop from the completion time and look up the building tag within a ±5 loop window.

```java
String resolved = lookupBuildingTag(buildingIndex, commandLoop);
String tag = resolved != null ? resolved
    : "r-" + ue.getUnitTagIndex() + "-" + ue.getUnitTagRecycle();
```

If the index has a match, use the real building tag. If not — no game events, sparse CmdEvent coverage, or the window misses — fall back to the existing heuristic. Zero regression for replays without game events.

## The Zerg surprise

The first oracle replay I tested against was Zerg. Match rate: 1 out of 240 TrainIntents had a building tag matching a real production building. Concerning — until I thought about how Zerg actually works.

Terran and Protoss players select a production building and press the train hotkey. The selection contains the building. Zerg players select *Larva* and press the train hotkey. The selection contains Larva tags, not Hatchery tags. EmulatedGame won't find a building matching a Larva tag, so it falls back to auto-resolution — exactly the same as before.

The fix is race-sensitive by nature, not by code. Terran and Protoss get exact building routing. Zerg gets the same heuristic it always had. No special cases, no if-statements — the data structure handles it.

## Why it matters

This is replay reconstruction accuracy, not physics validation. The physics were already proven correct. But higher-fidelity reconstruction produces better training data. The ONNX binary detectors that classify opponent strategies at 3-minute and 5-minute marks are trained on reconstituted replay data. Every unit that completes in the right building, at the right tick, sharpens the signal they learn from.

The same applies to the CBR case base. When the strategy router retrieves similar games from the case base, the similarity metric compares game states at checkpoint intervals. More accurate game states mean better retrieval, which means better strategy selection.

Reconstruction fidelity is infrastructure. Every downstream system that consumes replay data benefits without being touched.
