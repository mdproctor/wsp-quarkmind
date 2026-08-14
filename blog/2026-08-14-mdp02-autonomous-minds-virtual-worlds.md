---
layout: post
title: "Autonomous Minds in Virtual Worlds"
date: 2026-08-14
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [architecture, multi-module, agency-framework, spi-extraction]
---

QuarkMind started as a StarCraft II game AI — strategy, economics, scouting, trust-weighted plugin dispatch, all running at game-loop granularity over the CaseHub engine. It was always a living lab for the agentic harness, but the domain was narrow. Today it became a platform.

The restructure splits the mono-module into seven Maven modules. `quarkmind-core` holds the agency framework — the SPIs that any world implementation needs. `quarkmind-sc2` gets everything SC2-specific. Five stub modules (`quarkmind-town`, `quarkmind-minecraft`, `quarkmind-evennia`, `quarkmind-sonaria`, `quarkmind-godot-mcp`) declare the intent: five virtual worlds, each demonstrating a different agent-world relationship.

The interesting architectural bet is extracting the framework from one working implementation rather than designing it in a vacuum. SC2 already solved most of the autonomous agent problems — GOAP planning, case-based reasoning, trust scoring, advisory pipelines, needs-driven goal selection. The patterns transfer. A `WorldBridge<P extends WorldPerception, I extends Intent>` connects to any world; an `IntentQueue<I>` buffers decisions before world-specific translation. `NeedState` decays hunger, energy, social needs at rates modified by personality dispositions from Eidos. These SPIs are generic enough to cover a StarCraft commander, a Sims-like townsperson, a Minecraft survivor, and a MUD conversationalist — because they describe the agency loop, not the world.

What I like about this approach: the SPIs will be wrong. They're derived from a single concrete example. When the Town module starts implementing its `WorldBridge<RoomPerception, TextIntent>`, something won't fit. That's the point. Extract from a working system, validate against a second, adjust. Framework design by extraction is slower than greenfield but the frameworks actually survive contact with reality.

The execution model flexibility matters too. SC2 runs a sequential tick loop — perceive, decide, act, sleep, repeat. Town agents will be independent WebSocket clients on virtual threads, each running their own agency loop asynchronously. The server doesn't know if a client is AI or human — Turing test by architecture. Both compositions use the same building blocks from `quarkmind-core`. Sequential and async are configurations, not architectures.

`AgencyLoop` itself is a thin wrapper over CaseEngine. It maps agency phases (perceive, need, goal, plan, act, reflect) to TaskDefinitions internally. World implementors see agency vocabulary — "what does this agent need, what does it intend to do" — not engine vocabulary. The wrapper is mostly naming. CaseEngine already provides the orchestration, the blackboard, the plugin dispatch, the signal settlement. Adding an agency-shaped API on top of a case-shaped runtime is a few type mappings and a lot of naming.

Phase 1 is structural only — no behavioural changes. SC2 tests stayed green throughout. Phase 2 is where it gets interesting: all five worlds build in parallel, each one testing the SPIs from a different angle. Town is the primary validation — if a personality-driven townsperson can live autonomously using the same framework as an omniscient StarCraft commander, the extraction worked.
