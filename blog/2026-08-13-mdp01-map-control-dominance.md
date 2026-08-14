---
layout: post
title: "Map Control as a Dominance Signal"
date: 2026-08-13
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [dominance, map-control, expansion, calibration]
---

The `MultiFactorDominanceAssessor` has measured dominance across four dimensions since it replaced the supply-only assessor: economy, army value, tech tier, base count. Each factor contributes to a weighted score that drives milestone trust attestations — the mechanism by which the bot evaluates whether its chosen strategy is working mid-game.

Base count was always the weakest of the four. It measures how many completed base buildings each side has, but says nothing about *where* they are. A player with three bases clustered at the natural gets the same score as one with three bases spread across the map. In SC2, territorial coverage matters — controlling distant expansions means controlling resources, vision, and map presence.

The expansion location infrastructure landed a few weeks ago: resource clustering produces a list of all viable base locations on any map, ordered by distance from the start position. With that in place, the question became whether map control should replace the base count factor or sit alongside it.

We went with both — a 5th factor in `DominanceWeights`, configurable at runtime. Set the bases weight to near-zero and you're effectively replacing it. Keep both non-zero and they contribute independently. The sum-to-1.0 invariant on the weights record means the total budget is fixed; redistribution is a config change, not a code change.

The factor itself measures expansion-site occupation: how many of the map's available base locations have a completed base building within 10.0 map units. The score is `(ownControlled - enemyControlled) / maxExpectedDelta`, clamped to [-1, 1] — the same normalization pattern as every other factor. The fog-of-war guard returns neutral when no enemy buildings are visible, acknowledging that unscouted enemy expansions would otherwise inflate the perceived advantage.

The 10.0 proximity threshold needed calibration against replay data — the spatial constants protocol requires it for any distance-based heuristic. We ran it against all 59 replays (IEM10 + AI Arena). Every single base building fell within 10.0 units of its nearest expansion centroid. The actual distances ranged from 4.4 to 6.0, averaging 5.1. The threshold has comfortable headroom.

The blast radius was significant — `DominanceWeights`, `WeightModifier`, `MilestoneConfig`, all three weight strategies, the `AnchorInterpolator`, every Drools rule in `DominanceWeightAdjustment.drl`, and every test that constructs any of these types. All mechanical, all following the established pattern. The Drools rules all get `mapControlDelta = 0.00` for now — tuning how tactical situations should shift map control weight needs runtime data we don't have yet.

The interesting design question that surfaced: map control and base count overlap significantly for standard play but diverge for non-standard builds. A Terran with three orbital commands at the natural gets high base count but low map control. The default weights (bases=0.05, mapControl=0.10) are a bet that the spatial signal is more informative than the absolute count — but the runtime configurability means we can test this without touching code.
