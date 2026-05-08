---
layout: post
title: "StarCraft II Quarkus Agent — Day Two: No More Stubs"
date: 2026-04-07
---
# StarCraft II Quarkus Agent — Day Two: No More Stubs

**Date:** 2026-04-07
**Type:** phase-update

---

## What I was trying to achieve: make the agent actually play

Yesterday's work built the scaffold — mock architecture, real SC2 wiring, replay parsing. Impressive plumbing. The agent still did nothing intelligent. Today's goal: all four plugins real, all stubs gone, the loop closed.

I brought Claude back in for the full session. We started with 38 tests and four pass-through stubs. We finished with 94 tests and no stubs left.

## The engine was three seams pretending to be one

The first thing I wanted to fix was structural. `SC2Client`, `GameObserver`, and `CommandDispatcher` always moved together — any profile swap replaced all three. That's not three seams. We merged them into `SC2Engine`: one `connect()`, one `tick()`, one `observe()`, one `dispatch()`. `AgentOrchestrator` went from four injections to one.

The `@UnlessBuildProfile` annotation needed to exclude both `%sc2` and `%replay`. The obvious approaches — `@UnlessBuildProfile({"sc2","replay"})`, or stacking the annotation twice — both fail to compile. Claude found the answer by running `javap` on the `quarkus-arc-3.34.2.jar` bytecode: the annotation has an undocumented `anyOf` attribute. `@UnlessBuildProfile(anyOf = {"sc2", "replay"})` compiles and works. It's not in the docs anywhere.

## Four plugins doing real things

With the engine clean, we turned to the plugins.

`BasicEconomicsTask` trains Probes until 22 workers and builds Pylons when supply headroom drops to 4. `BasicStrategyTask` runs the gateway opener into Cybernetics Core into Stalkers, and writes `STRATEGY` (MACRO/DEFEND/ATTACK) to the CaseFile each tick. `BasicTacticsTask` reads that key and acts — AttackIntent toward the nearest threat, or MoveIntent back to the Nexus on DEFEND. `BasicScoutingTask` does passive intel from visible units and, after 20 ticks with no contacts, dispatches a Probe toward the estimated enemy base.

The scouting plugin has the most interesting implementation detail. It needs to remember which probe was sent — across ticks, not within one. Since plugins are `@ApplicationScoped` CDI singletons, a `volatile String scoutProbeTag` field works perfectly. The probe dies, the field is cleared, a new scout is assigned next tick.

## The arbitration problem

Both Economics and Strategy read minerals from the same CaseFile snapshot. At 200 minerals, both would see 200 and independently queue 150-mineral builds — spending 300 total. The CaseFile snapshot can't prevent this because plugins run against the same read.

The fix: `ResourceBudget` is a mutable object written fresh each tick by `GameStateTranslator` and consumed in place by plugins. `budget.spendMinerals(150)` deducts and returns true if affordable, returns false and skips if not. First plugin wins, second skips. No inter-plugin communication needed.

Realistic build times came with the same refactor — replacing the 1-tick instant build queue with `PendingCompletion` records carrying `completesAtTick`. Buildings appear as `isComplete=false` immediately so strategy plugins can see them under construction; supply from a Pylon only arrives when it actually completes.

## What the replay endpoint shows

The replay profile auto-starts, ticks through `Nothing_4720936.SC2Replay` at 500ms intervals, and serves the current state via REST. Curling it mid-game:

```json
{
  "minerals": 350, "supply": 23, "supplyUsed": 18,
  "myUnits": [{"type": "PROBE", ...}, {"type": "STALKER", ...}],
  "myBuildings": [{"type": "NEXUS", "isComplete": true}, {"type": "GATEWAY", "isComplete": true}],
  "gameFrame": 47
}
```

Real positions from a real bot game. Stalkers appearing as the Cybernetics Core completes. The agent's plugins are running against actual game data and the intents they produce — `ATTACK` toward the Zerg base — are being logged for offline evaluation. The loop isn't closed into the replay (it can't be), but the agent is thinking.

Closing the loop with real intelligence is next. Drools, then Quarkus Flow.
