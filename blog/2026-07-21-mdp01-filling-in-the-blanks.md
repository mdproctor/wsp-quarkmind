---
layout: post
title: "Filling in the Blanks"
date: 2026-07-21
type: phase-update
entry_type: note
subtype: diary
projects: [QuarkMind]
---

## Filling in the Blanks

Position-based compliance verification left the consumer side fully wired — `LocationResolver` could resolve watchtower positions, expansion ordinals, ramp locations — but the producer side was passing null. The domain model knew what a `NeutralFeature` was. Nobody was extracting them.

Two gaps made this obvious. First, `ObservationTranslator` filtered for `Alliance.SELF` and `Alliance.ENEMY` units but ignored `Alliance.NEUTRAL` entirely. Every geyser, mineral patch, watchtower, and destructible rock in a real SC2 observation was silently dropped. The fix follows the same pattern as the existing unit and building mappers: a set of ocraft enum entries per category, a classification method, and a filter in `translate()`. Watchtowers map to `XELNAGA_TOWER`, destructible barriers to `DESTRUCTIBLE_ROCK`, unbuildable debris to `DESTRUCTIBLE_DEBRIS`. Geysers and minerals now flow through as `Resource` records instead of the hardcoded `List.of()` that was there before.

Second, `IEM10JsonSimulatedGame` was dropping neutral units while its sibling `ReplaySimulatedGame` handled them correctly. The JSON replay path checked `controlPlayerId == 0` and moved on. The binary replay path checked `controlPlayerId == 0` and populated minerals and geysers. An inconsistency that only surfaces when you try to use IEM10 data for something that needs resource positions — like validating the expansion clustering radius.

That validation was the second task. `ExpansionLocation.fromResources()` uses a graph-connected-components algorithm with `CLUSTER_RADIUS = 12.0` map units to group nearby resources into expansion locations. The constant was chosen by design but never tested against real map layouts. With IEM10 now populating resources, a calibration test across all 59 replays (30 IEM10 JSON, 29 AI Arena binary) confirms the radius works: 12–20 expansions per map, averaging 15.4 for AI Arena and 16.5 for IEM10. No under-merging, no over-splitting.

The upstream `Uni<>` retirement commit had left four compilation errors — orphaned closing parentheses from wrapper removal. Those are fixed too, though they belong to a different story.

What's quietly happening here is the real SC2 observation path catching up to the replay path. The replay side has had resource extraction for months. The observation side was wired for units and buildings but treating neutrals as invisible. With `extractNeutralFeatures()` now available, the next step toward a complete `MapInfo` on the real SC2 path is constructing it in `SC2BotAgent.onStep()` — cached on first frame, reused thereafter. The coaching pipeline's `LocationResolver` is already waiting for it.
