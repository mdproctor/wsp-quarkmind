---
layout: post
title: "Two Bugs Make a Right"
date: 2026-09-29
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [sc2, replay-parsing, scelight, selection-tracking, multiplication]
series: issue-318-stripped-replay-coverage
---

# Two Bugs Make a Right

The `StrippedReplayFeatureExtractor` had a tracker corruption bug. P2:Marine was at 178% of oracle — nearly double the real count. The `SelectionUnitLinkTracker` maintained a flat list of unitLinks from SC2 selection delta events, and when the `removeMask` was `None`, it appended new entries without clearing old ones. Over hundreds of selection events, unitLink=70 (Marine) accumulated from 2 to 76 in the worst replay.

The fix seemed straightforward: rewrite the tracker with tag-based deduplication, mirroring `AbilityMapping`'s pattern. Each unit in SC2 has a unique tag (including a recycle counter), so pairing tags with unitLinks and skipping duplicates prevents accumulation. It worked. P2:Marine dropped from 178% to 93.5%.

Then P1:Marine fell off a cliff. From 97.6% to 70.6%.

I ran the diagnostic to confirm `addUnitTags` was present in stripped replays — it is, for every single selection event across the dataset. The tags include recycle counters, so different Marines have genuinely different tags. The dedup was correct: it wasn't filtering legitimate units.

The real story was more interesting. `PRODUCTION_UNIT_LINKS` mapped Barracks to unitLink=70 — which is the Marine unit, not the Barracks building. The multiplication logic was counting Marines in the selection and using that as a proxy for how many Barracks the player had selected. The `BarracksUnitLinkDiscoveryTest` confirmed it: unitLink=42 is the actual Barracks building. But building unitLinks rarely appear in selections — players select mixed control groups where units outnumber buildings.

With the old inflated tracker, the Marine count grew past reality due to the accumulation bug, and that inflated count happened to approximate the real Barracks count. Two bugs cancelling out: the tracker accumulated when it shouldn't, and the mapping pointed at the wrong unit type. Together, they produced 97.6% accuracy. Separately, each one breaks.

We tried four approaches to satisfy both P1 and P2 simultaneously. Using the correct building unitLinks (42 for Barracks) dropped both to ~44% — buildings are almost invisible in selection data. Using cumulative building counts directly gave P1=95% but P2=137% — the count never subtracts for destroyed buildings, and P2 players apparently lose more production buildings over the course of a game. Clearing the tracker on `removeMask=None` made everything worse.

The fix that landed: use the cumulative building count for all production multiplication (replacing the selection-based approach entirely), capped at 4. The cap accounts for building destruction — most competitive players have 3-5 production buildings of any type active at once, and the cumulative count overestimates when replacements are built. P1:Marine settled at 94.7%, P2:Marine at 109.7%.

The tag-based tracker rewrite is still there — structurally correct, fully tested, available for future use. But the production path no longer queries it. The multiplication signal turned out to be the building count tracked from `BuildCommand` events, not the selection state at command time. Selection tracking was always an indirect proxy for something we already had directly.

What I'll remember from this: a system that looks 97.6% accurate can be 100% wrong about why. The old code worked not because it correctly understood SC2 selection semantics, but because two independent errors produced a near-correct output. Fixing either error alone makes the output worse. You can only see this by fixing the first bug and watching something unrelated break — then tracing the dependency chain back to the second bug you didn't know existed.
