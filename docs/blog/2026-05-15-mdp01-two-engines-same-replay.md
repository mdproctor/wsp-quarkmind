---
layout: post
title: "Two Engines, Same Replay"
date: 2026-05-15
type: phase-update
entry_type: note
subtype: log
projects: [QuarkMind]
---

The last entry ended with EmulatedGame finally behaving like a real SC2 engine
at the structural level: parallel building queues, correct supply timing,
auto-engage. The question was whether the physics actually match.

The only way to answer it honestly is to run the same game through both
EmulatedGame and a real SC2 replay, then compare them tick by tick.

The ground truth side was already in place. `ReplaySimulatedGame` reads tracker
events from an `.SC2Replay` file — unit-born, unit-died, resource tick — and
provides per-tick snapshots of what actually happened.

The emulation side required a new pipeline. To drive EmulatedGame, I needed the
commands the human player issued: train a Probe, start a Stalker, begin a
Gateway. Those commands live in GAME_EVENTS as `CmdEvent`s, each carrying an
`abilLink` — a raw integer. `175` means train Probe from a Nexus. `172` with
`abilCmdIndex=1` means train Zealot from a Gateway; `abilCmdIndex=0` means
Stalker. The distinction between abilLink=172 and abilLink=173 is the difference
between a Gateway and a Robotics Facility. No enum, no type field.

I brought Claude in to build the translation layer. Before we could write the
map, we needed to discover the values. `AbilityDiscoveryTest` scans the replay's
GAME_EVENTS and prints every (abilLink, abilCmdIndex) pair observed for the human
player, grouped by frequency. Cross-referencing with tracker events — seeing that
abilLink=175 appears consistently alongside Probe-born events — confirmed the
mapping. We repeated this for every building type in the Protoss build.

With the table confirmed, we built `AbilityMapping`:

```java
return switch (abilLink) {
    case ABIL_NEXUS   -> trainIntent(loop, UnitType.PROBE);     // 175
    case ABIL_GATEWAY -> {
        UnitType unit = GATEWAY_UNITS.get(idx);                 // 172; idx selects unit
        yield unit != null ? trainIntent(loop, unit) : unknown(abilLink, idx);
    }
    default -> unknown(abilLink, idx);
};
```

The stateful part was less obvious. SC2 commands are issued relative to the
currently-selected building. "Train Probe" means whichever Nexus the player
clicked last — the ability ID doesn't embed a building tag. We tracked selection
events: each `SelectionDeltaEvent` updates `currentSelection`, and `trainIntent`
reads `currentSelection.get(0)` to attach the building tag to the intent.

`ReplayCommandExtractor` walks the GAME_EVENTS stream, filters for the human
player, and produces a `TimedIntent` stream. `ReplayValidationHarness` then runs
`ReplaySimulatedGame` and EmulatedGame in parallel on the same replay, feeding
those intents into the emulated engine at the correct game loop. A
`DivergenceReport` records the per-tick delta in unit counts and resources.

First run on 29 AI Arena replays: first divergence at game tick 86, unit delta
growing from there. Probes completing at slightly wrong ticks. That points to a
timing error — which is the next entry.
