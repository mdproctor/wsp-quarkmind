---
layout: post
title: "Human Replays Speak a Different Language"
date: 2026-09-28
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [sc2, replay-parsing, classifier, abilLink]
series: issue-317-blizzard-ladder-restore
---

# Human Replays Speak a Different Language

The whole point of issue #317 is to train the ONNX classifier on 151K stripped Blizzard ladder replays — human games, not bot games. The previous session designed the pipeline and got 118 oracle replays restored through Docker. This session ran the discovery test against those oracles and found something I hadn't expected: human replays use completely different ability IDs from bot replays for the same actions.

In SC2 replays, every player command carries an `abilLink` — an integer that identifies the ability being used. When a bot places a Pylon, it issues `abilLink=42` (Smart), the same command it uses for movement. The unit then walks to the target location and places the building. Indistinguishable from a move order. That's why the existing `AbilityMapping` class treats building placement as invisible — in bot replays, it literally is.

Human players don't work that way. When a human selects a Probe and clicks "Build Pylon", the game client emits `abilLink=170, abilCmdIndex=1` — a distinct, typed command with the building identity baked in. Each Protoss building gets its own index under abilLink 170: Nexus is 0, Pylon is 1, Assimilator is 2, Gateway is 3, all the way through ShieldBattery at 15. The same pattern holds for Terran (abilLink 129, SCV build) and Zerg (abilLink 183, Drone morph). Every building in the game has a deterministic, discoverable constant.

The discovery data was clean. Modal counts of 250+ for Gateway placement, 478 for Pylon, 519 for SupplyDepot. No ambiguity.

Here's where it gets interesting: `abilLink=170` is already in the codebase as `ABIL_WARPGATE` — the WarpGate warp-in ability from bot replays. Same integer, completely different meaning depending on who played the game. In human replays, WarpGate warp-in uses a different constant entirely: `abilLink=214`. The distinction was invisible as long as we only looked at bot games.

The fix was straightforward — a boolean `humanReplay` mode on the constructor. In human mode, 170 routes to building placement; in bot mode, it routes to movement as before. Not elegant, but correct and backward-compatible.

Upgrades were a different story. The temporal proximity approach — find the closest preceding command before each tracker event — works brilliantly for buildings because placement is near-instantaneous. For upgrades, the research command happens thousands of game loops before the completion event. By the time the upgrade finishes, the player has issued dozens of unrelated commands. The modal match for Stimpack research was "train Marine from Barracks" — the player was macro-ing while the upgrade cooked. The upgrade constants need a different discovery strategy, probably filtering by building selection state rather than pure temporal proximity.

The building constants are the real prize for the feature extractor. Stripped ladder replays have no tracker events — no UnitBorn, no UnitInit, nothing but raw player commands. With the building abilLink maps, we can reconstruct what was built and where from commands alone. That's 53+ building types across all three races, plus addons, plus WarpGate warp-ins.
