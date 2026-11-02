---
layout: post
title: "The Formula, the Replay, and the Floor"
date: 2026-11-02
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [emulator, calibration, empirical-measurement, accuracy-floor, starcraft2]
---

# The Formula, the Replay, and the Floor

Every StarCraft II wiki page lists unit train times in seconds. Stalker: 30 seconds. Siege Tank: 32 seconds. Marine: 25 seconds. Multiply by 22.4 (game loops per real-time second at Faster speed), and you have the value your emulator should use.

Except you don't. SC2 stores train times as integer game loops internally, and those integers do not follow the community formula. A Stalker isn't 30 × 22.4 = 672 loops. It's 676. A Siege Tank isn't 32 × 22.4 = 717. It's 720. The formula gets you close — within a few loops — but "close" compounds across a production chain, and by the five-minute mark of a fast game, the emulator's unit counts diverge from reality.

We've been building QuarkMind's EmulatedGame engine as a faithful replay harness — feed it tracker events from a real SC2 replay, let it reproduce the game state, and compare against ground truth at checkpoints. The accuracy number tells us whether the emulator's physics are correct. When it's wrong, we know the emulator has a bug. When it's right, we can trust it as a training environment for AI agents.

The question that drove this work: how accurate can the emulator get, and what's the difference between a fixable bug and a structural limit?

## The formula problem

![Formula-derived vs empirically calibrated train times](images/formula-vs-empirical.svg)

The community formula `seconds × 22.4` is an approximation. It's been repeated on wikis and in tools for years, and for casual use it's fine. For an emulator that needs to reproduce exact unit counts across 118 replays, it's not.

We ran a calibration test against 118 oracle replays — real human ladder games from patch 4.9.3, covering all three races. The approach: correlate each player's production commands (via abilLink in game events) with the unit-born tracker events. The time between command and completion, across hundreds of observations, gives the real train time as an integer modal value.

The results were systematic. Hellion: not 470 (21 × 22.4), but 480. Widow Mine: not 470, but 480. Siege Tank: not 717, but 720. Thor, Raven, Banshee, Liberator: not 963, but 960. Battlecruiser: not 1434, but 1440. SCV: not 275, but 272 — the same value as Probe, suggesting Blizzard uses a shared worker train time.

The pattern column is the interesting part. Many Terran values are exact multiples of 24: Hellion = 20 × 24, Siege Tank = 30 × 24, Thor = 40 × 24, Battlecruiser = 60 × 24. The community formula uses 22.4 as the base; the internal representation may use a different time base entirely. The formula gets you a float that you then round — but the game never used a float. It stores the integer directly.

Twenty-two values updated. The Marauder changed the most: 757 → 781, a 24-loop shift that moved its accuracy from 96% to 100% at the five-minute checkpoint. One unit recovered from a single constant.

## The invisible events

While calibrating, we found something unexpected. Warp-gate production — the Protoss mechanic where Gateways transform into WarpGates and produce units via warp-in instead of queued training — uses a different tracker event type. Normally trained units emit `UnitBorn` when they appear. Warped-in units emit `UnitInit` when warp-in starts, then `UnitDone` when it completes.

The extractor had been processing `UnitBorn` and explicitly skipping `UnitInit`. Every warped-in unit was invisible. Across 118 replays, 1502 gateway unit events were being discarded. The fix was straightforward: process `UnitInit` for gateway units after WarpGate research completes, use the 80-loop warp-in duration instead of the full 600+ loop train time, and mark the intent as warped-in so the emulator uses the right production speed.

The fix didn't change the accuracy headline — the train time calibration had already compensated for the warp-in timing at the five-minute mark. But the production model is now structurally correct. Units are produced through the right mechanism at the right speed, not accidentally arriving at the right time through a wrong calculation that happened to cancel out.

This matters for the AI agent. An agent reasoning about production rates needs to know that a WarpGate produces a Stalker every 230 loops (80 warp-in + 150 cooldown), not every 676 loops. The accuracy metric can't see the difference — but the agent's decision-making can.

## Finding the floor

![Tracker-driven accuracy — the calibration journey](images/calibration-journey.svg)

After the flag propagation fixes and the train time calibration, accuracy sat at 99.9% — 6803 units matched out of 6807 across 118 replays. Four units missing. The question: is 100% achievable, or is 99.9% a structural limit?

The hypothesis was tick quantization. EmulatedGame processes in 22-loop ticks; real SC2 processes individual game loops. A unit completing at loop 6601 would finish at tick 300 in the emulator (loop 6600), but the checkpoint is at tick 299. One tick late, one unit missed. If this were the cause, running two extra ticks past the checkpoint should recover the missing units.

We ran the diagnostic. Zero units recovered. Not one of the four missing units appeared at checkpoint + 2 ticks, across any of the 168 player runs. The gap is not tick-boundary timing. It's structural.

The root cause is building queue ordering. When the extractor can't match a production command to a specific building (about 43% of commands are missing from Blizzard's replay data), it falls back to auto-resolution — pick the building with the shortest queue. Real SC2 assigned the unit to a different building. The unit queues behind other units on the wrong building and finishes later than in the real game.

Four units out of 6807. Each from a different replay. Each a different unit type: Phoenix, Medivac, Reaper, Hellion. Fixing this would require matching each production command to the exact building the player used — information that isn't reliably available in the replay data. 99.9% is the floor.

## Why emulator accuracy matters for game AI

An AI agent that plays StarCraft doesn't need a perfect emulator. It needs one that's correct enough that the agent's strategy decisions are evaluated against reality, not against the emulator's quirks. When the emulator produces the wrong number of Marines, the strategy classifier labels the game state incorrectly, the case-based reasoner retrieves the wrong precedent, and the agent makes the wrong decision — all because of a three-loop error in a train time constant.

The calibration work isn't about achieving a number. It's about establishing trust. When the emulator reports that a strategy produced 14 Stalkers by minute five, we need to know whether the real game also produced 14 Stalkers — or whether the emulator's formula-derived constant shifted one Stalker past the checkpoint. At 99.9%, with the remaining gap explained and documented, we know exactly what the emulator gets right and what it approximates. The approximation is small, bounded, and understood. That's the difference between "close enough" and "verified."

The formula on the wiki is fine for napkin math. For an agent that needs to reason about production timings, resource trade-offs, and strategic responses — measure it from the replays.
