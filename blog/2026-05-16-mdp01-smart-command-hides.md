---
layout: post
title: "What the Smart Command Hides"
date: 2026-05-16
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [emulation, sc2, java, replay-validation]
---

Phase 6 was about feeding real player commands into `EmulatedGame` and watching the unit counts match the replay. The question was how to get those commands out.

SC2 replays store GAME_EVENTS — a stream of `SelectionDeltaEvent`s (player selects units) and `CmdEvent`s (player issues a command). Every `CmdEvent` has an `abilLink` identifying the ability: move is abilLink=42, train Probe from Nexus is abilLink=175, train Zealot from Gateway is abilLink=172. I brought Claude in to build the extraction pipeline. We split `GameEventStream` into a thin MPQ reader, added `AbilityMapping` to translate command events into `Intent` objects, and wired it up through a `ReplayCommandExtractor` facade.

The design decision that mattered for `AbilityMapping` was making it stateful. To interpret any `CmdEvent` you need to know what's currently selected — that comes from preceding `SelectionDeltaEvent`s. A stateless lookup table can't work. `AbilityMapping` owns selection state and you feed it events sequentially: `onSelection()`, then `process()`.

The ability IDs aren't documented anywhere — only discoverable by running real replays through the parser and observing what appears. We built `AbilityDiscoveryTest` for this: a permanent JUnit test that scans replay files and prints all observed `(abilLink, abilCmdIndex, count)` tuples sorted by frequency. One run produces the mapping table; every future run confirms nothing new appeared when new replays are added.

Running it on the aiarena PvZ dataset revealed the central problem.

Building commands don't appear. There's no abilLink for "Build Pylon" or "Build Gateway" in bot replays. Instead, every build command — and every movement command — shows up as abilLink=42. That's the SC2 "Smart" ability, the generic right-click that means "do the contextually appropriate thing." In human play through the game client, specific build ability IDs are issued. In bot play through the SC2 API, bots issue Smart commands for everything.

The two are indistinguishable in the event stream. You cannot tell "probe moves to this location" from "probe builds Pylon at this location" from GAME_EVENTS alone.

The tracker events know. `ReplaySimulatedGame` is already driven by tracker events — `UnitInit` (construction started), `UnitDone` (complete). They're unambiguous about what exists and when. So instead of extracting `BuildIntent`s from game events, we inject buildings from `ReplaySimulatedGame.snapshot()` directly into `EmulatedGame` each tick, using the same tracker-event tags. `TrainIntent`s carry those tags — a Nexus that appeared as `"r-12-1"` in the tracker events will be found by `handleTrain()` because the injected building carries the same tag.

Two new methods on `EmulatedGame` support this: `injectReplayBuilding(Building)` adds a building and updates supply for completed structures, `markReplayBuildingComplete(String)` handles buildings that finish construction mid-run. Narrow and explicit.

The result: `ReplayValidationTest` runs the harness against `Nothing_4720936.SC2Replay` to three minutes and asserts unit counts stay within 2 of ground truth at every tick. The ≤ 2 delta is the mining model, not the ability mapping — the flat income rate accumulates more minerals than the real player had, occasionally training a unit a tick early. The ability IDs are right; the resource dynamics are approximate. Closing that gap is issue #141.

The pipeline is in place. Everything Phase 6 still needs builds on it.
