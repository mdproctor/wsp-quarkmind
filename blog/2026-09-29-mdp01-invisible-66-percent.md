---
layout: post
title: "The Invisible 66%"
date: 2026-09-29
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [sc2, replay-parsing, scelight, warp-in, rapid-fire]
series: issue-326-warp-in-abillink-coverage
---

# The Invisible 66%

The previous session established that human replays use abilLink=214 for warp-in commands, and wired up the `StrippedReplayFeatureExtractor` to emit `UnitInit` events for gateway units after WarpGate research completes. It worked — but only for about a third of the warp-ins. The oracle replays showed 464 Stalker UnitInit events; we were producing 152.

The obvious hypothesis was wrong. I assumed there were additional abilLinks for warp-in — maybe a WarpPrism variant, maybe a rapid-fire shortcut. The discovery test said otherwise: across 118 oracle replays, every single warp-in UnitInit event correlated with abilLink=214. No alternatives. The mapping was already correct.

So I started counting. 514 CmdEvents with abilLink=214 across all replays, both players. The oracle showed 1502 gateway UnitInit events. The CmdEvents were identical between stripped and oracle replays — game events aren't affected by the restoration process. The commands genuinely weren't there.

The answer was hiding in the SC2 protocol definition files. Event ID 104 — `SCmdUpdateTargetPointEvent` — is how SC2 records rapid-fire commands. When a player holds a warp-in hotkey and clicks five locations, the replay records one CmdEvent followed by four CmdUpdateTargetPointEvents. Each one means "same command, new target." But Scelight's `GameEventFactory` has no case for ID 104. These events fall through to the base parser and emerge as plain `Event` objects — invisible to any code dispatching on `instanceof CmdEvent`.

2,213 of these events followed abilLink=214 commands from Protoss players. Combined with the 514 initial CmdEvents, that's 2,727 warp-in signals for 1,502 oracle events — more commands than actual warp-ins, because some clicks are target adjustments (the player re-clicking before the warp-in starts) or failed attempts (no available WarpGate).

The fix tracks the last warp-in `TrainIntent` per player. When event 104 arrives after an abilLink=214 CmdEvent from the same player, it re-emits the same unit type at the new loop. Coverage jumped from 28–55% to 100–114% across all gateway units. The ~10% over-emission comes from those target adjustments — a known trade-off without SC2 engine simulation.

A smaller fix went in alongside: `AbilityMapping.process()` had a selection guard that dropped commands when the selection state was empty. Human-mode commands — builds, warp-ins, archon merges — are self-identifying via their abilLink and don't need selection context. Moving `dispatchHuman()` before the guard fixed edge cases where rapid-fire warp-ins arrived without a preceding `SelectionDeltaEvent`.

The interesting part isn't the fix. It's that the symptom pointed in exactly the wrong direction. Low warp-in coverage looks like missing ability mappings. The real cause was an entirely different event type that the replay parser never surfaces as a typed object. Without the diagnostic step — counting raw CmdEvents against oracle tracker events — I'd have spent hours searching for abilLinks that don't exist.
