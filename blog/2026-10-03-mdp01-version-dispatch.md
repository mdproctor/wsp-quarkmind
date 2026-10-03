---
layout: post
title: "The Same Number Means Different Things"
date: 2026-10-03
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [replay-parsing, version-dispatch, sc2, calibration]
---

# The Same Number Means Different Things

SC2 replays encode player commands as ability IDs — `abilLink` integers that identify which button the player pressed. Train a Probe from a Nexus? That's abilLink 175. Research WarpGate at a CyberneticsCore? That's 236, index 6. I'd spent weeks building the mapping table from 118 oracle replays (patch 4.9.3), cross-referencing every command event with tracker events to discover which abilLink means what. 82 upgrade research abilities, all wired and working.

Then I ran the accuracy test across tournament replays from HomeStory Cup XXVII — a different SC2 patch — and hit 65.9%. A third of the upgrade events were invisible.

The problem is that Blizzard reassigns ability IDs between patches. The same integer means a different building in a different version. abilLink 182 is TemplarArchive in 4.9.3 (PsiStorm research at index 4) but it's Forge in the HSC patch (ground weapons upgrades across indices 0-8). abilLink 192 is Spire in one version, SpawningPool in another. These aren't edge cases — they're the main upgrade buildings for each race.

## The sparse override pattern

I didn't want to duplicate the entire 700-line dispatch switch for each version. Most abilLinks are stable across patches — maybe 95% of them don't move. The conflicts are sparse.

So I built `AbilityProfile` as an enum with override maps. `V4_9_3` has an empty override map — its abilLinks are the base case in the main switch. `HSC_2025` carries a `Map<Integer, AbilityDispatch>` of lambda overrides for the ~15 abilLinks that differ. Before the switch fires, the override map is checked first. If the profile has a lambda for that abilLink, the lambda handles it. If not, the base switch runs.

Adding a new SC2 version is one enum entry plus a handful of lambdas. No switch modification. The callers just pass `baseBuild` from the replay header and `AbilityProfile.resolve()` picks the right profile — threshold at 75689, calibrated from a diagnostic test that scanned all 179 replays.

That got us from 65.9% to 74.2%. Three conflict overrides (Forge, SpawningPool, CyberneticsCore) plus a full EngBay tournament map recovered 108 previously invisible upgrade detections.

## Selection state as a disambiguation signal

The remaining gap needed more tournament-era abilLink mappings, but discovering them was harder than the oracle replays. The technique I'd used before — find the nearest CmdEvent preceding each UpgradeEvent — breaks down for human replays. Human players issue movement commands constantly. The nearest command is almost always abilLink 42 (Smart/Move), not the research command.

The fix was filtering by selection state. When a player researches an upgrade, they have exactly one building selected. By tracking `SelectionDeltaEvent` and filtering to CmdEvents where `selectionSize == 1`, the noise drops dramatically. abilLink 177 at index 0 emerged as WarpGateResearch with 17 clean hits — invisible in the unfiltered data because 177/0 was drowning in movement commands.

Six more overrides from the selection-aware discovery brought accuracy to 78.6%. That's 165 additional upgrade detections across 179 replays compared to where we started.

## Where it stops

The last 1.4% to 80% hits a wall: abilLink 177 means both WarpGateResearch (from CyberneticsCore) and BlinkTech (from TwilightCouncil) in the tournament patch. Same number, same index, different building. The current override system picks one — and WarpGateResearch wins on volume (94/110 vs BlinkTech's 14/51). To disambiguate, the dispatch would need to know which building is selected by unitLink, not just what abilLink was pressed. That's a deeper change to `AbilityDispatch` — the interface currently takes `(idx, event, loop, race)` and would need the selected building's identity as a fifth parameter.

The Zerg side has a similar problem with abilLink 195, which appears across BanelingNest, Hatchery, EvolutionChamber, and more — all with overlapping index values. Same fundamental issue: multiple buildings sharing one abilLink, distinguishable only by selection state.

Both are tractable with building-type-aware dispatch. But 78.6% across mixed-version replays, from infrastructure that took two focused sessions to build, is a reasonable place to stop and let the data tell us when the last 1.4% matters.
