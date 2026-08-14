# QuarkMind Restructure — Autonomous Minds in Virtual Worlds

**Date:** 2026-08-14
**Status:** Draft
**Decisions:** See `decisions.md` (D1–D17)

## Overview

QuarkMind evolves from a StarCraft II game AI into a platform for building autonomous agents that live in virtual worlds. The existing SC2 codebase is split into a shared agency framework (`quarkmind-core`) and the SC2-specific implementation (`quarkmind-sc2`). New world implementations build on the same framework.

The framework sits on top of CaseHub foundations (engine, neocortex, ledger, eidos, qhorus, blocks) — it does not duplicate them. It adds an agency-specific composition layer: the perceive/need/goal/plan/act/reflect loop, needs/drives, spatial awareness, intent buffering, and interaction pipelines.

## Module Structure

Single Maven repo, multi-module:

```
quarkmind/
├── pom.xml                        ← parent POM
├── quarkmind-core/                ← agency framework
├── quarkmind-sc2/                 ← StarCraft II (refactored from current code)
├── quarkmind-town/                ← Sims-like 3D life simulation
│   ├── src/main/java/             ← Quarkus server
│   └── godot/                     ← Godot 4 client project
├── quarkmind-minecraft/           ← Minecraft agent
│   ├── src/main/java/             ← Quarkus agent backend
│   └── bridge/                    ← Node.js Mineflayer bridge
├── quarkmind-evennia/             ← MUD agent
│   └── src/main/java/
├── quarkmind-sonaria/             ← Roblox/Sonaria agent
│   └── src/main/java/
└── quarkmind-godot-mcp/           ← Godot EditorPlugin MCP (tooling)
    └── godot-plugin/
```

Dependencies flow one way: each world module depends on `quarkmind-core`. No world depends on another world. `quarkmind-core` depends on CaseHub foundations.

## quarkmind-core: Agency Framework

### Design Principles

1. **Composition over prescription** — the framework provides building blocks (needs, intents, perception, spatial memory), not a rigid execution model. Sequential tick loops (SC2, Wacky Manor style), fully async independent clients (Town, Minecraft), and hybrids are all valid compositions.

2. **Generic SPIs** — world implementors can use simple strings or rich typed domain classes. The framework defines contracts and orchestration (queue, dispatch, lifecycle), not concrete types. Java generics on SPI interfaces preserve type safety per-world.

3. **Thin wrapper over CaseEngine** — the agency loop maps agency phases (perceive, need, goal, plan, act, reflect) to CaseEngine TaskDefinitions internally. World implementors see agency vocabulary, not engine internals.

### What quarkmind-core Contains

Extracted from current QuarkMind, generalised:

| Component | Source in current QuarkMind | Generalised form |
|---|---|---|
| Agency loop | `AgentOrchestrator` + `GameTickExecutor` | `AgencyLoop` — configurable phase pipeline over CaseEngine |
| Needs/drives | (new) | `NeedState`, `DispositionNeedModifier` — decaying values modified by Eidos dispositions |
| World bridge SPI | SC2Engine interface + GameStateTranslator | `WorldBridge<P, I>` — connect, perceive, translate intents |
| Intent buffer | `IntentQueue` + sealed `Intent` | `IntentQueue<I>` — generic buffered intent dispatch |
| Spatial awareness SPIs | `AStarPathfinder`, `VisibilityGrid` concepts | `NavigationSPI`, `VisibilitySPI`, `SpatialMemory` — contracts only |
| Interaction pipeline | Advisory/Coaching trigger→worker→channel pattern | `InteractionTrigger`, `InteractionPipeline` — generic trigger→LLM→response |
| Moment detection | `MomentDetectionTask` pattern (over casehub-blocks) | `MomentDetector` — significant event detection contract |
| Plugin seam | `TaskDefinition` with `activateIf`/`requires`/`produces` | Same pattern, decoupled from SC2 types |
| LLM request queue | Wacky Manor chat pipeline pattern | `LlmRequestQueue` — rate limiting, prioritisation, token budgeting |

### What quarkmind-core Does NOT Contain

- Pathfinding algorithms (each world implements NavigationSPI differently)
- World models (blocks, rooms, tiles, units — domain-specific)
- Action libraries (mine, craft, attack, cook — domain-specific)
- Drools rules (domain-specific)
- Game-specific GOAP action definitions (generic GOAP planner is in casehub-engine-api)

### Key SPIs

```
WorldBridge<P extends WorldPerception, I extends Intent>
  connect()
  disconnect()
  perceive() → P
  dispatch(IntentQueue<I>)

NeedDefinition
  need() → String or enum
  baseDecayRate() → float
  dispositionModifiers() → Map<DispositionTerm, float>

InteractionTrigger
  evaluate(AgencyContext) → Optional<TriggerEvent>

NavigationSPI
  pathTo(target) → Path
  isReachable(target) → boolean

VisibilitySPI
  visible() → Set<Entity>
  remembered() → Set<Entity>
  unexplored() → Set<Region>
```

Type parameters allow SC2 to use `WorldBridge<SC2Perception, SC2Intent>` while Evennia uses `WorldBridge<RoomPerception, TextIntent>`. The framework orchestrates the lifecycle without knowing the shapes.

### Execution Models

The framework supports different compositions:

**Sequential tick (SC2, Wacky Manor style):**
```
while (running) {
  for (agent : agents) {
    perception = bridge.perceive()
    agencyLoop.tick(agent, perception)
    bridge.dispatch(agent.intents())
  }
  sleep(tickRate)
}
```

**Async independent clients (Town, Minecraft):**
```
// Each agent on its own virtual thread
void agentLifecycle(Agent agent) {
  bridge.connect()
  while (running) {
    perception = bridge.perceive()        // blocks until world sends update
    agencyLoop.tick(agent, perception)    // LLM calls go through shared queue
    bridge.dispatch(agent.intents())
  }
}
```

**Hybrid:** any mix. The building blocks are the same — composition is configuration.

## quarkmind-sc2: StarCraft II

All existing SC2-specific code moves here. Package structure mirrors current layout:

```
io.quarkmind.sc2/
  domain/              ← TerrainGrid, AStarPathfinder, ExpansionLocation, etc.
  sc2/                 ← SC2Engine, IntentQueue, sealed Intent, GameStarted/Stopped
  sc2/real/            ← QuarkusSC2Transport, SC2BotAgent, ObservationTranslator
  sc2/emulated/        ← EmulatedGame, EmulatedSC2Server, PathfindingMovement, VisibilityGrid
  sc2/mock/            ← SimulatedGame, ScenarioLibrary
  agent/               ← QuarkMindCaseHub, QuarkMindCaseFile, GameStateTranslator, AgentOrchestrator
  agent/plugin/        ← StrategyTask, EconomicsTask, TacticsTask, ScoutingTask
  agent/cbr/           ← SC2GameCbrCase, SC2CbrRetentionObserver, SC2StrategyRouterTask
  plugin/              ← DroolsStrategyTask, FlowEconomicsTask, DroolsTacticsTask
  plugin/scouting/     ← DroolsScoutingTask, PatternClassificationRuleUnit
  plugin/tactics/      ← SC2 GOAP actions, WorldState
  plugin/coaching/     ← CoachingTriggerBuilder, CoachingWorkerFactory, CoachingStyle
  plugin/advisory/     ← AdvisoryWorkerFactory, AdvisoryChannelBroker
  plugin/commentary/   ← CommentaryWorkerFactory, CommentaryChannelBroker
  plugin/summarisation/ ← MomentDetectionTask, GameArcSummariser
  plugin/flow/         ← EconomicsFlow
  qa/                  ← QA endpoints, workbench
```

SC2 implements the quarkmind-core SPIs:
- `SC2WorldBridge implements WorldBridge<SC2Perception, SC2Intent>`
- `SC2NavigationSPI implements NavigationSPI` (wraps AStarPathfinder + TerrainGrid)
- `SC2VisibilitySPI implements VisibilitySPI` (wraps VisibilityGrid)
- Advisory/coaching/commentary implement `InteractionPipeline`

All existing tests must pass against the refactored module structure.

## quarkmind-town: Sims-Like 3D Life Simulation

### Architecture

```
┌─────────────────────────────────┐
│       Town Server (Quarkus)      │
│  World tick, needs decay,        │
│  weather, day/night, objects     │
└──┬────┬────┬────┬────┬─────────┘
   WS   WS   WS   WS   WS
   │    │    │    │    │
  AI   AI   AI  Human Observer
  Bob  Kit  Sam (player) (dashboard)
```

- **Server:** Quarkus application. Ticks the world (needs decay, weather, time). Manages object state (stove on/off, crops growing, resources available). Serves world state over WebSocket.
- **AI clients:** Each agent is an independent WebSocket client on its own virtual thread. Runs the AgencyLoop from quarkmind-core. LLM calls go through a shared `LlmRequestQueue`.
- **Human clients:** Same WebSocket protocol as AI clients. Server doesn't distinguish.
- **Observer:** Receives all agent thought streams. Powers the dashboard.

### World Systems

- **Needs:** HUNGER, ENERGY, SOCIAL, FUN, HYGIENE, BLADDER. Decay rates modified by Eidos dispositions (extrovert social need decays fast when alone, introvert's overflows around people).
- **Objects:** Defined in data files (YAML/JSON). Each object maps to need satisfaction values and optional skill requirements. `oven: { satisfies: {hunger: 5}, requires_skill: cooking }`.
- **Skills:** Improve through use (backed by CBR — case-based reasoning from casehub-neocortex).
- **World state:** Persistent. Stove stays on. Crops grow. Things built stay built. Resources deplete and regenerate.
- **Weather/time:** Day/night cycle, seasons. Affect what's possible (can't garden at night, need shelter in winter).
- **Economy:** Objects have value. Trade with other agents and players.

### Rendering (Godot 4)

- **3D low-poly** using Kenney CC0 assets + CC0 character animation libraries.
- **Dual camera:** Isometric overview (whole town, select characters) + click-to-inhabit first-person (see through a character's eyes).
- **Transparent walls:** Building walls fade when camera approaches, revealing room interiors with furniture.
- **Buildings with floors and rooms:** Kitchen, bedroom, living room, bathroom, garden — each with objects that satisfy different needs.
- **Thought stream panel:** Selected character's inner monologue — perceptions, decisions, goals, mood.
- **HTML5 export:** Accessible via browser URL. Always running. Anyone can visit.

### Godot MCP

EditorPlugin exposing MCP tools: `place_node`, `move_node`, `take_screenshot`, `set_camera`, `list_assets`, `modify_property`. Generates both the Godot scene AND server data files from the same MCP session.

### Town Data Files (generated by MCP, loaded by server)

```yaml
# buildings.yaml
bakery:
  position: [12, 0, 8]
  rooms:
    kitchen:
      objects:
        - type: oven
          satisfies: {hunger: 5}
          requires_skill: cooking
        - type: counter
          satisfies: {social: 2}
    shop:
      objects:
        - type: display_case
          satisfies: {fun: 1}
```

## quarkmind-minecraft

- **Agent backend:** Quarkus application running AgencyLoop from quarkmind-core.
- **Bridge:** Node.js Mineflayer process connecting to Minecraft server as a player. Translates between QuarkMind intents and Minecraft protocol.
- **Testing:** Luanti (open-source Minecraft) server in Docker for CI. Deterministic seed worlds for reproducible tests.
- **Observer:** Minecraft spectator mode (join with real client) + thought stream dashboard.
- **Showcase:** Embodied survival. The agent passes as human among real players.

## quarkmind-evennia

- **Agent backend:** Quarkus application running AgencyLoop.
- **Bridge:** Connects to Evennia MUD server via bot API. Room/exit spatial model.
- **Showcase:** Conversational depth. Personality shines in text. Deepest Turing test for character.

## quarkmind-sonaria

- **Agent backend:** Quarkus application running AgencyLoop.
- **Bridge:** HttpService integration via Roblox developer partnership with Sonar Studios.
- **Showcase:** Creature surviving in an ecosystem. Hunger, thirst, pack dynamics, trading.
- **Dependency:** Requires external partnership — timeline is less predictable.

## Phasing

**Phase 1 — Extract (gates everything)**
- Convert quarkmind to multi-module Maven project
- Extract quarkmind-core from current codebase
- Refactor remaining code into quarkmind-sc2
- All existing SC2 tests pass
- IntelliJ MCP refactoring only, no subagents
- Stub modules for town/minecraft/evennia/sonaria (pom.xml + empty SPI implementations)
- Milestone: `mvn test` green across all modules

**Phase 2 — All worlds in parallel**
- quarkmind-town: server, Godot client, Godot MCP, first character living autonomously
- quarkmind-minecraft: Mineflayer bridge, Luanti CI, first agent joining a server
- quarkmind-evennia: Evennia bridge, first agent in a MUD
- quarkmind-sonaria: partnership outreach, bridge design
- Each world works independently — no dependencies between them

## Refactoring Constraints

- No execution without explicit approval
- All QuarkMind refactoring via IntelliJ MCP (type-safe rename, move, find references)
- No subagents for refactoring — main session with human oversight
- SC2 test suite is the safety net — must stay green throughout
- Incremental commits at each stable point

## What Each Platform Demonstrates

| Platform | Agent relationship | Key capability showcased |
|---|---|---|
| SC2 | Omniscient commander | Strategy, learning, trust-weighted decisions |
| Town | Autonomous life | Needs-driven goals, personality, observation |
| Minecraft | Embodied survivor | Passes as human, survival, social |
| Evennia | Conversationalist | Personality depth, relationship building |
| Sonaria | Creature in ecosystem | Instinct, pack dynamics, survival |

## CaseHub Foundation Dependencies

quarkmind-core composes these existing foundation modules:

| Foundation Module | What it provides |
|---|---|
| casehub-engine | CaseEngine, CaseFile blackboard, GOAP planner, plugin dispatch |
| casehub-neocortex | CBR (learning from experience), memory stores |
| casehub-ledger | Trust scoring (Bayesian beta), outcome recording |
| casehub-eidos | Disposition vocabulary, personality descriptors |
| casehub-qhorus | Channel-based messaging, typed delivery |
| casehub-blocks | Summarisation framework, event accumulation |
| casehub-worker | Worker execution, LLM pipeline integration |
