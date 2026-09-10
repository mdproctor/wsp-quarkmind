---
title: "The Army That Never Moved"
date: 2026-09-10
author: mdp
entry_type: note
subtype: diary
series: issue-296-replay-workbench-cascade-empty
projects: [casehubio/quarkmind]
tags: [replay, sc2, scouting, testing, smoke-tests]
---

# The Army That Never Moved

Every enemy unit in replay playback was frozen at its birth position. Drones that should have been mining sat at (124, 156) forever. Overlords that should have been scouting hovered where they hatched. Queens that should have been injecting larvae stayed at the Hatchery — not because they were doing their job, but because the replay parser never told them to move.

The pattern classifier didn't notice. It works on unit types and counts — "I see 20 Drones and 3 Roaches" — and doesn't care where they are. The 70% accuracy target held. Everything downstream that uses positions — posture classification, timing attack detection, army centroid movement, map control assessment — was silently producing garbage from frozen coordinates.

## What the replay format actually does

SC2 replays have two event systems. Tracker events give you unit births, deaths, and economy. Game events give you player commands — move orders, build orders, ability casts. We were already handling both: tracker events for state, game events for the watched player's movement orders.

What we weren't handling was a third category: `UnitPositions` (tracker event ID 8). SC2 emits these periodically — 23 times across an 8-minute game — with updated coordinates for every unit that has moved since the last snapshot. Scelight parses them but provides no typed interface. Where `UnitBorn` has `IBaseUnitEvent` with `getXCoord()` and `getYCoord()`, `UnitPositions` gives you a raw `Map<String, Object>` with an `Integer[]` and a cryptic delta encoding.

I wrote a diagnostic test to dump the raw structure:

```java
Integer firstIndex = event.get("firstUnitIndex");
Integer[] items = event.get("items");
// items is a flat array of (unitIndex_delta, x, y) triplets
// firstIndex + cumulative deltas = the unit's tagIndex
```

The coordinates match tracker scale — same as birth positions, no conversion needed. The tag encoding uses only `unitTagIndex` (no recycle counter), so matching to existing units requires a lookup map maintained across birth/death events.

## The fix

Fifteen lines of production code. Add `applyUnitPositions()` to the tracker event switch, maintain a `tagIndexToTag` map populated on birth and cleaned on death, call the existing `replaceUnitPosition()`/`replaceEnemyPosition()` methods. The hard part was understanding the event format — the implementation was mechanical once that was clear.

## Why nobody noticed

The workbench visualiser showed enemy units. Pattern classification reported archetypes. Strategy selection picked responses. Commentary narrated the game. Everything looked functional. The spatial features were wrong, but they're one input among many — the pipeline produced output, just output based on stale positions.

This is the class of bug that survives indefinitely without targeted smoke tests. So I wrote ten of them: five against real replay data (enemy count grows, multiple unit types appear, positions change, economy is non-zero, buildings appear) and five against the CDI pipeline (scouting fires, strategy selects, WebSocket delivers, CaseHub solves). The position delta test would have caught this on the first run — assert that at least one enemy unit changes position across 200 ticks.

## What this opens up

Posture classification now has real movement data. When an army pushes across the map, the posture classifier can actually see AGGRESSIVE instead of MACRO. Timing attack detection can fire when units move toward the player base. Army centroid tracking in the coaching pipeline can measure real army movement, not a stationary blob. The spatial calibration tests all pass, but the downstream intelligence that depends on positions hasn't been recalibrated against the corrected data yet — that's a separate pass once the full calibration suite clears.
