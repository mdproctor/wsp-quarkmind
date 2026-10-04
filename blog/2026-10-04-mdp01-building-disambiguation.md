---
layout: post
title: "Which Building Did They Click?"
date: 2026-10-04
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [replay-parsing, abilLink, unitLink, upgrade-detection, sc2, onnx]
series: issue-339-onnx-coverage
---

# Which Building Did They Click?

Yesterday's version-dispatch work got upgrade detection to 78.6% across mixed-version replays. Today I wanted to find out why the remaining 21% was missing — and the answer wasn't what the issue description said it would be.

The hypothesis was neat: abilLink 177 is shared between CyberneticsCore and TwilightCouncil in tournament replays, so BlinkTech (27% detection rate) and Charge (65%) are being misidentified as WarpGateResearch. Add the building's unitLink to the dispatch interface, branch on it, done.

The data told a different story. I ran the accuracy breakdown and found the missing upgrades scattered across dozens of abilLinks, not concentrated on one collision. The original hypothesis was looking at the wrong layer of the problem.

## The generic research abilLink

What's actually happening is more interesting. In tournament replays, SC2 emits a **generic per-race research abilLink** that fires from any building. abilLink 177 isn't a CyberneticsCore command — it's a Protoss-wide "research something" command. abilLink 195 is the Zerg equivalent. They fire with whatever building happens to be selected, and the only way to know which upgrade is being researched is to check which building the player has selected via the selection delta's unitLink.

This is fundamentally different from the oracle replays (4.9.3), where each building has its own abilLink. The version-dispatch work from yesterday handled the shift for specific buildings that moved between patches. But the generic research abilLinks are a category the oracle replays don't have at all — they're a tournament replay-era addition.

## Discovery through tracker correlation

I needed the unitLink values for every Protoss and Zerg research building. The approach: correlate tracker UnitBorn events (which have building names) with selection delta subgroups (which have unitLink integers). When a player selects a CyberneticsCore, the selection delta carries its unitLink. Match the tag from the selection with the tag from the tracker, and the mapping falls out.

61 HSC XXVII replays, 24 buildings, every one produced a 100% unambiguous unitLink. CyberneticsCore is 95, TwilightCouncil is 88, Forge is 86, and so on down the list. Clean data — no collisions, no ambiguity.

With those constants, the `AbilityDispatch` interface gets a unitLink parameter. When abilLink 177 fires, the override checks the selected building's unitLink and routes to the correct upgrade map. CyberneticsCore → air weapons and WarpGate. TwilightCouncil → Charge, Blink, Adept. Forge → ground weapons, armour, shields. Seven Protoss buildings, each with its own upgrade table.

## The building-filtered diagnostic

The generic abilLink overrides recovered some accuracy, but the bigger gains came from a second discovery pass. Instead of correlating CmdEvents with UpgradeEvents through a time window (which catches whatever the player happened to click most recently, not what initiated the research), I filtered to only keep CmdEvents where the selection contained the **correct building** for that upgrade type.

This cut through the noise completely. The filtered results showed clean building-specific abilLinks I hadn't mapped: 238 for CyberneticsCore, 239 for TwilightCouncil, 187 for EvolutionChamber, 182 for Forge. These fire alongside the generic abilLinks — the SC2 client emits both a building-specific command and a generic one. The building-specific ones are the reliable signal.

Eleven new building-specific overrides, plus Terran abilLinks 167 (TechLab) and 171 (Armory). Each one a direct mapping — no unitLink disambiguation needed because they're already building-specific.

## Where it landed

78.6% → 87.7% across 179 replays. The biggest gains:

- BlinkTech: 27% → 84%
- ExtendedThermalLance: 47% → 88%
- CentrificalHooks: 64% → 96%
- GlialReconstitution: 63% → 95%
- DarkTemplarBlinkUpgrade: 0% → 50%
- PsiStormTech: 36% → 86%

The remaining 12.3% gap is a long tail — upgrades where the CmdEvent uses an abilLink not in the override set, or where the player issued the research via a control group hotkey that didn't emit a selection delta. That second category is an inherent ceiling for the unitLink approach. Some commands simply can't be resolved from the replay data.

I also decomposed the ONNX coverage epic into 16 child issues across the five phases. The phase-level issues existed but needed actionable children with dependency tracking. The mining model gap in the epic description turned out to be already closed — three-tier saturation mining has been in `SC2Data.mineralIncomePerTick()` for a while. The real gap I found and added: EmulatedGame has no `ResearchIntent` at all, which blocks the Phase 4 AI-vs-AI training loop.

The ONNX classifier's accuracy ceiling isn't about the model architecture or the training data volume. It's about whether the feature extraction pipeline can correctly identify what upgrades a player researched from stripped replay command data. Today moved that needle meaningfully — but the last few percent will need either wider replay datasets or acceptance that some commands are genuinely unrecoverable.
