# QuarkMind Restructure — Decisions

## D1: QuarkMind becomes an autonomous agent platform

**Choice:** QuarkMind evolves from "SC2 game AI" to "autonomous minds in virtual worlds" — a platform for building AI agents that live in any game/world.
**Alternatives:**
- Keep QuarkMind SC2-only, build new projects separately — loses the common patterns
- Build the framework in CaseHub foundations directly — too generic, too far from the domain
**Rationale:** QuarkMind already solved most autonomous agent problems in the SC2 context. The patterns (GOAP, CBR, trust, advisory pipelines) transfer to other worlds. Extracting a shared framework from a working system beats designing in a vacuum.
**Trade-offs:** QuarkMind loses its single-purpose identity. Refactoring risk during extraction.
**Exploration:** quick
**Status:** captured

## D2: Multi-module mono-repo

**Choice:** Single Maven repo with modules: quarkmind-core, quarkmind-sc2, quarkmind-town, quarkmind-minecraft, quarkmind-evennia, quarkmind-sonaria.
**Alternatives:**
- Separate repos per world — independent versioning but harder dependency management
- Hybrid (framework + SC2 together, worlds separate) — pragmatic but messy long-term
**Rationale:** Mono-repo enables atomic refactors across modules during extraction. Simpler CI. All worlds share the same framework version by definition.
**Trade-offs:** All worlds share release cadence. Repo grows larger over time.
**Exploration:** quick
**Status:** captured

## D3: Five target platforms

**Choice:** SC2, Town (Sims-like 3D), Minecraft (Mineflayer + Luanti for CI), Evennia (MUD), Sonaria (Roblox partnership).
**Alternatives:**
- Fewer platforms (just Town + one game) — less proof of portability
- More platforms (add custom Roblox town) — too much scope now; park as Phase 2
**Rationale:** Each platform demonstrates different agent-world relationships: omniscient commander (SC2), embodied survivor (Minecraft), conversationalist (Evennia), creature in ecosystem (Sonaria), autonomous life (Town). Together they prove the framework is genuinely portable.
**Trade-offs:** Five platforms is ambitious. Sonaria requires a developer partnership.
**Exploration:** quick
**Status:** captured

## D4: CaseHub foundations provide the heavy infrastructure

**Choice:** The agency framework builds ON TOP of CaseHub foundations (engine, neocortex/CBR, ledger/trust, qhorus, eidos, blocks/summarisation) — does not duplicate them.
**Alternatives:**
- Build standalone framework independent of CaseHub — massive duplication
- Merge agency concepts directly into CaseHub foundations — too game-specific for the enterprise platform
**Rationale:** CaseEngine (blackboard), CBR (learning), trust scoring (Bayesian beta), GOAP planner, summarisation, messaging, dispositions are all already foundation modules. The agency framework is a thin layer that composes them for the "autonomous agent in a world" use case.
**Trade-offs:** Dependency on CaseHub release cadence. New worlds must be Quarkus applications.
**Exploration:** quick
**Status:** captured

## D5: Framework extraction scope

**Choice:** Extract from current QuarkMind into quarkmind-core:
- Agency loop abstraction (perceive → need → goal → plan → act → reflect)
- Needs/drives system (decaying floats modified by dispositions and object interactions)
- WorldState SPI (each world provides "what I can see" in its own shape)
- Intent buffer (decide → act separation, before world-specific translation)
- Interaction pipeline pattern (trigger detection → LLM reasoning → response delivery)
- Spatial awareness SPIs (navigation, visibility, memory-of-places)
- Plugin seam with activateIf/requires/produces (more expressive than foundation WorkerFunctionProvider)
- Moment detection patterns (built on casehub-blocks summarisation framework)
**Alternatives:**
- Extract more (include pathfinding algorithms) — too domain-specific, different per world
- Extract less (just the loop, leave SPIs to each world) — loses the common contract
**Rationale:** These are the patterns that transfer across all five platforms. The SPIs define what a "world bridge" looks like without prescribing implementation. Pathfinding algorithms, world models, action libraries stay game-specific.
**Trade-offs:** SPI design is based on one concrete example (SC2). May need adjustment when the second world (Town) is built. Accept this — extract now, adjust during Town implementation.
**Exploration:** quick
**Depends on:** D1, D4
**Status:** captured

## D6: SC2 code stays in quarkmind-sc2

**Choice:** All SC2-specific code moves to quarkmind-sc2 module: TerrainGrid, VisibilityGrid, SC2 game state model, SC2 GOAP actions, Drools rules, advisory/coaching/commentary implementations, EmulatedGame, replay harness, SC2 protocol.
**Alternatives:**
- Leave SC2 code in place, extract framework alongside — muddies boundaries
**Rationale:** Clean separation. SC2 module depends on quarkmind-core, not the other way around. SC2 continues to work exactly as before, just with explicit framework dependency.
**Trade-offs:** Refactoring a working system risks breaking it. Mitigated by comprehensive test suite.
**Exploration:** quick
**Depends on:** D5
**Status:** captured

## D7: Town uses Godot 4 + Quarkus backend

**Choice:** quarkmind-town uses Godot 4 for 3D rendering (HTML5 export for browser access), Quarkus backend for world simulation and agency, Kenney CC0 assets for 3D models, Mixamo or CC0 libraries for character animations.
**Alternatives:**
- Phaser (2D) — simpler but loses first-person mode and transparent walls
- Babylon.js / Three.js (browser-native 3D) — no visual editor, more manual work
- LibGDX (Java 3D) — browser export via GWT is fragile, fighting the framework
**Rationale:** Godot 4 provides a visual scene editor (design towns visually), built-in character controllers, camera systems, physics, animation, and HTML5 export. MIT licensed. The Godot project IS the town — non-developers could design new towns. Client connects to Quarkus backend via WebSocket.
**Trade-offs:** Adds Godot to the tech stack (GDScript or C# client-side). Team needs Godot knowledge.
**Exploration:** quick
**Depends on:** D3
**Status:** captured

## D8: Each agent is an independent client

**Choice:** Client/server architecture — the world is a server that ticks constantly, each AI agent is an independent client connecting via WebSocket with its own perception-decision-action loop on its own virtual thread. Human players use the same protocol.
**Alternatives:**
- Sequential processing (like current Wacky Manor) — doesn't scale, not realistic
- Shared-thread agents with round-robin — simulates concurrency but isn't real
**Rationale:** Matches how real games work (Minecraft, Roblox). Agents are logically concurrent. The server doesn't know or care whether a client is AI or human — Turing test by architecture. Same pattern works across all five platforms.
**Trade-offs:** More complex than sequential. Need shared LLM request queue for rate limiting (D9).
**Exploration:** quick
**Status:** captured

## D9: Shared LLM request queue

**Choice:** Physically co-located agents share a central request queue for LLM calls — rate limiting, prioritisation (human-interaction priority), token budgeting, potential batching.
**Alternatives:**
- Each agent manages its own LLM calls — races, rate limit violations, cost overruns
**Rationale:** Agents are logically independent but physically sharing resources. Five agents thinking simultaneously = five API calls. The queue serialises LLM access while agents remain logically concurrent. From the agent's perspective, it's just async latency.
**Trade-offs:** Queue introduces latency. Acceptable — mirrors "thinking time."
**Depends on:** D8
**Exploration:** quick
**Status:** captured

## D10: Needs driven by Eidos dispositions

**Choice:** Personality traits (from casehub-eidos) act as coefficients on need decay rates. An extrovert's social need decays faster when alone; an introvert's overflows faster around people. Same needs system, different rates per personality.
**Alternatives:**
- Fixed need decay rates for all agents — loses personality-driven behaviour
- Fully LLM-driven needs (no mechanical system) — unpredictable, expensive
**Rationale:** Simple, elegant, testable. Personality creates genuinely different life patterns from the same agency framework. The mechanical needs system is deterministic and fast; the LLM handles the "what to do about it" decision.
**Trade-offs:** Requires mapping Eidos disposition vocabulary to need decay modifiers — new integration work.
**Depends on:** D4, D5
**Exploration:** quick
**Status:** captured

## D11: Phase order — base harness first, then worlds in parallel

**Choice:** Phase 1: Extract quarkmind-core from current QuarkMind, keep SC2 working. Phase 2: Build quarkmind-town (primary new showcase) alongside stubs for minecraft/evennia/sonaria. Phase 3: Flesh out remaining worlds in parallel.
**Alternatives:**
- Build Town first, extract framework after — premature abstraction risk from one example
- Build all worlds sequentially — too slow
**Rationale:** Framework extraction from a working system is safer than greenfield. SC2 is the validation — if SC2 still works, the extraction is correct. Town is second validation — if Town works on the framework, the SPIs are right. Then remaining worlds build in parallel with confidence.
**Trade-offs:** SC2 refactoring delays new world work. Acceptable — it's the foundation.
**Exploration:** quick
**Status:** captured

## D12: Town is 3D with dual camera modes

**Choice:** Isometric overview (see whole town, select characters) + click-to-inhabit first-person mode (see through character's eyes). Transparent walls when camera is near buildings to show interiors. Buildings have floors and rooms.
**Alternatives:**
- 2D pixel art (Smallville/AI Town style) — simpler but loses the immersive first-person moment
- 3D only first-person — loses the observation overview
**Rationale:** The isometric overview is the demo/observation mode (paired with thought-stream dashboard). The first-person mode is the "wow moment" — drop into a character's eyes while reading their thoughts. Both are just camera positions in the same 3D scene. 3D is also easier for stock assets (one model vs multiple sprite directions).
**Trade-offs:** 3D is more engineering work than 2D. Justified by the first-person experience and easier asset pipeline.
**Depends on:** D7
**Exploration:** quick
**Status:** captured

## D13: Godot MCP for visual world building

**Choice:** Build an MCP server as a Godot EditorPlugin that exposes tools for scene manipulation (place_node, move_node, take_screenshot, set_camera, list_assets, modify_property). Claude designs towns visually through iterative placement and screenshot feedback. MCP session generates both the Godot scene AND the data files (YAML/JSON) the Quarkus server loads.
**Alternatives:**
- Hand-code coordinates in data files — error-prone, no visual feedback
- Human designs in Godot editor, exports for server — requires Godot expertise from the human
**Rationale:** Closes the loop — Claude can design, see, and adjust the world. Data files are generated alongside visual placement, keeping server and client in sync. Reusable beyond this project.
**Trade-offs:** Building the MCP is additional work. Pays for itself across all future Godot work.
**Depends on:** D7
**Exploration:** quick
**Status:** captured

## D14: Agency loop wraps CaseEngine with agency vocabulary

**Choice:** Thin wrapper — AgencyLoop maps agency phases (perceive, need, goal, plan, act, reflect) to CaseEngine TaskDefinitions internally. World implementors see agency concepts, not engine concepts. CaseEngine is the runtime underneath.
**Alternatives:**
- Own tick abstraction independent of CaseEngine — duplicates orchestration
- Expose CaseEngine directly to world implementors — leaks engine concepts, steeper learning curve
**Rationale:** Best of both worlds. Agency vocabulary is intuitive for game developers. CaseEngine provides the proven orchestration (blackboard, plugin dispatch, signal settlement). The wrapper is thin — mostly type mapping and naming.
**Trade-offs:** Wrapper adds a layer of indirection. Acceptable — it's naming, not logic.
**Depends on:** D4, D5
**Exploration:** quick
**Status:** captured

## D15: Refactoring constraints

**Choice:** No execution until explicitly approved. All QuarkMind refactoring uses IntelliJ MCP for type-safe operations (rename, move, find references). No subagents for refactoring work — must be done in the main session with full IDE context.
**Alternatives:**
- Subagent-driven refactoring — loses IDE type safety, risks broken references
- Text-based grep/sed refactoring — error-prone for Java, misses type hierarchies
**Rationale:** QuarkMind has a comprehensive test suite but the codebase is complex (sealed types, CDI injection, Drools rules, plugin seams). IDE-driven refactoring respects the type system. Main session ensures the human can watch and intervene.
**Trade-offs:** Slower than parallel subagent work. Worth it — a broken SC2 build blocks everything.
**Exploration:** quick
**Status:** captured

## D16: SPIs support both strings and rich models

**Choice:** Framework SPIs are generic — worlds can use simple string keys or proper typed domain classes. quarkmind-core defines contracts and orchestration patterns (queue, dispatch, lifecycle), not concrete types. SC2 brings its sealed Intent hierarchy. Evennia might use strings. Both are valid implementations of the same SPI.
**Alternatives:**
- Force rich models everywhere — too rigid for text-based worlds
- Force strings everywhere — loses type safety for complex worlds
**Rationale:** Different worlds have different type complexity. A MUD is naturally string-based. An RTS has complex typed commands. The framework shouldn't dictate the shape — it provides the plumbing.
**Trade-offs:** Generic SPIs require careful design to avoid becoming stringly-typed. Java generics on the SPI interfaces keep type safety per-world while allowing different shapes.
**Exploration:** quick
**Status:** captured

## D17: Two-phase plan — extraction gates, then all worlds parallel

**Choice:** Phase 1: Extract quarkmind-core + quarkmind-sc2 (gates everything). Phase 2: All worlds build in parallel (town, minecraft, evennia, sonaria). Not sequential — truly parallel once the core exists.
**Alternatives:**
- Town first then others — unnecessary sequencing
- All at once including extraction — risky, core isn't stable yet
**Rationale:** The core framework is the only gate. Once it exists with proven SPIs (validated by SC2 passing all tests), every world can proceed independently. No world depends on another.
**Trade-offs:** None significant — this is strictly better than sequential.
**Depends on:** D11 (supersedes the three-phase ordering)
**Exploration:** quick
**Status:** captured
