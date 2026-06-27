---
layout: post
title: "Emulated SC2 as a Real Server"
date: 2026-06-27
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [sc2, protocol, emulation, ocraft, websocket, protobuf]
series: issue-171-emulated-sc2-protocol-wrapper
---

## The gap that shouldn't exist

EmulatedGame has been the workhorse for physics testing since April. It simulates combat, pathfinding, enemy AI, mining — everything the agent stack needs to make decisions. But it bypasses the SC2 network layer entirely. `EmulatedEngine` calls `game.snapshot()` directly and hands the result to the orchestrator as a `GameState`.

Meanwhile, `RealSC2Engine` talks to actual StarCraft II over a raw WebSocket, exchanging protobuf messages through `QuarkusSC2Transport`. Two completely different code paths. If something works in emulated mode but breaks in the real SC2 profile, you don't find out until you have a StarCraft II installation and a Mac that can run it.

I wanted to close that gap. Wrap EmulatedGame in the same SC2 WebSocket/protobuf protocol that real SC2 speaks. `RealSC2Engine` connects to it identically — same transport, same protobuf, same ocraft parsing. The agent stack never knows it's talking to a simulation.

## What the wrapper actually does

The core idea: promote `FakeSC2Server` — a test inner class that already speaks the full SC2 protocol — into a production CDI bean backed by EmulatedGame. The test server handles WebSocket handshakes, frame encoding, request routing. The delta is replacing its static `buildResponse()` with one that delegates to the game.

Two pure-function translators bridge the gap. `GameStateToProtobuf` takes a `GameState` and builds an SC2 protobuf observation — the reverse of `ObservationTranslator`. `ProtobufToIntent` takes an incoming protobuf action command and produces the domain `Intent` the game understands — the reverse of `ActionTranslator`. Both are static, no CDI, no state. They mirror the translators they invert.

The round-trip test is the enforcement mechanism. `GameState` → protobuf → ocraft parsing → `ObservationTranslator` → `GameState`. If the two sides drift, the test breaks. The ocraft parsing layer is critical — it validates every field with `orElseThrow`. A protobuf observation that looks right but omits `display_type` on a unit, or `camera` on the player, or any of the five `StartRaw` fields, crashes at runtime. The round-trip catches it at test time.

## ocraft's hidden contracts

The design review rounds surfaced several ocraft validation requirements that aren't visible from the proto schema. `StartRaw` is optional in `ResponseGameInfo` — but once you include it, all five sub-fields are mandatory. `ImageData` validates `size.x * size.y * bpp / 8 == data.length` using integer division — so a 1×1 image at 1 bit per pixel requires zero bytes. The workaround is 8×1 instead, giving a clean single-byte payload.

Each `Raw.Unit` requires `display_type` and `alliance` set — fields `ObservationTranslator` never reads, but ocraft's `UnitSnapshot.from()` crashes without them. `ObservationRaw` needs a `PlayerRaw` with a `camera` position. None of these appear in the SC2 protocol documentation. They're ocraft-level invariants discoverable only from the source.

## The profile picture

The new `%emulated-sc2` profile sits alongside the existing six. `%emulated` stays for pure physics benchmarking — no protocol overhead, no serialization cost. `%emulated-sc2` exercises the full stack: WebSocket framing, protobuf marshalling, ocraft validation, the same `RealSC2Engine` and `QuarkusSC2Transport` that talk to real SC2.

Anyone can clone the repo, run `mvn quarkus:dev -Dquarkus.profile=emulated-sc2`, and test strategies against a physically accurate simulation — without installing StarCraft II. That includes CI. GitHub Actions can run the full agent loop without a third-party binary.

The `SC2WebSocketCodec` extraction was a natural dedup. The WebSocket handshake and frame encode/decode code was duplicated across the transport client, the test server, and now the emulated server. Three copies of the same RFC 6455 byte manipulation. One shared utility — pure static methods, no state.

The protocol wrapper doesn't try to be a complete SC2 server. Cooldown data, geysers, and mineral patches don't survive the round trip — matching real SC2 behaviour. Enemy buildings collapse into the enemy unit list. The known differences are documented and tested explicitly. The point isn't perfect fidelity — it's exercising the code path that matters.
