---
title: "Bot replays lied about ability IDs"
author: mdp
date: 2026-09-28
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [sc2, replay-parsing, calibration, ability-mapping]
refs:
  - "#320"
  - "#319"
  - "#318"
---

The StrippedReplayFeatureExtractor from yesterday's session could parse human replays and reconstruct tracker events from game commands. Coverage sat at 24.4%. I assumed the remaining 75% was missing abilLink mappings — add more ability IDs, get more events. The truth was worse: the mappings we had were actively wrong.

The AbilityMapping tables were derived from AI Arena bot replays. Those bots run on a different SC2 protocol version than the Blizzard ladder replays (4.9.3). Same ability names, different integer IDs. abilLink=173 means Robotics Facility in the bot replays; in 4.9.3, it means Stargate. The Gateway train indices are swapped — idx=0 is Zealot in 4.9.3, Stalker in bot replays. The Larva train table was completely reshuffled: idx=9 maps to Roach in 4.9.3 but Viper in bot replays.

That last one was the smoking gun. The validation report showed 484 Viper events against an oracle count of 10. Vipers are rare — maybe one or two per game in the matchups that use them. Four hundred and eighty four meant every Roach train command was being tagged as Viper. A single wrong index was producing the entire false population.

The broader discovery — using the existing `AbilityDiscoveryCalibrationTest` — was too noisy to trust for precise mappings. It correlates tracker events with the closest preceding game command, but players issue hundreds of move and attack-move commands per minute. Those dominate every time window. We needed a different approach.

The fix was to reverse the correlation direction. Instead of starting from commands and searching forward for births, start from each oracle UnitBorn event and search backward for the closest preceding command. Then add a filter: train commands don't target a location, so exclude any command with a target point. That single filter — no target point — eliminates move orders, attack-move, and building placement, leaving only train and research commands. The signal cleaned up immediately.

We ran this against all 118 oracle replays for each race. The Terran results were the clearest: abilLink=161 is Starport (not previously mapped at all), abilLink=160 is Factory with an expanded index range that includes WidowMine at the unusual idx=24. For Protoss, abilLinks 173 and 174 are swapped compared to bot replays — Stargate and Robotics switched places. That explained why Oracle was over-counted: Robotics Colossus commands were being mapped through the Stargate table to Oracle.

There was also a structural problem beyond the index values. SC2 reuses the same integer abilLink values across races for completely different abilities. Without filtering by the player's race, a Terran player's abilLink=172 commands — whatever that Terran ability is — would produce false Gateway unit events. I added a race parameter to AbilityMapping. Each race-specific dispatch branch is now guarded by an `isRace()` check. The old constructors still work (null race means no filtering), so the bot replay code path is unaffected.

The combined effect: UnitBorn coverage went from 24.4% to 27.9%. Eighteen unit types went from zero to coverage — the entire Terran Factory and Starport roster, plus Overlord, Carrier, Observer, WarpPrism, Corruptor, Infestor, and Reaper. The Viper over-count dropped from 484 to 0. Oracle now matches the oracle exactly — 26 events, 26 expected.

The remaining gap to 80%+ is mostly command duplication — the extractor counts every train command, including ones that fail due to supply cap or insufficient resources. That's a resource-tracking problem, not a mapping problem. Different issue, different approach.
