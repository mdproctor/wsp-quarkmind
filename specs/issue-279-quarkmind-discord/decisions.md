# QuarkMind Discord — Decisions

## D1: Two sub-modules (protocol + agent)

**Choice:** `quarkmind-discord-protocol/` (shared types: DiscordPerception, DiscordIntent) + `quarkmind-discord-agent/` (agency loop, world bridge, Quarkus app). Mirrors QuarkVille's separation pattern.
**Alternatives:**
- Flat single module — simpler, but breaks pattern consistency across quarkmind worlds
**Rationale:** Consistency with QuarkVille's three-module structure (minus the server module Discord doesn't need). If a Discord dashboard or observer tool is added later, it can depend on protocol without pulling in agent dependencies.
**Trade-offs:** Slightly more Maven boilerplate for a module that may never have a third consumer.
**Exploration:** quick
**Status:** captured

## D2: Event-driven execution with blocks observation pipeline

**Choice:** Discord events feed into blocks `PartitionedObservationService<DiscordEvent, String>` partitioned by channel. Agent wakes on two triggers: (a) message arrives in a watched channel (immediate), (b) heartbeat timer fires to check needs. On wake, agent drains accumulated observations (tiered rendering handles volume), LLM processes context, agent decides: respond, react, initiate, or stay quiet.
**Alternatives:**
- Dual-loop (reactive + autonomous) — more complex, two loops sharing mutable state
- Single tick loop with fixed interval — adds unnecessary latency to message responses
**Rationale:** Reuses proven blocks observation pipeline from Wacky Manor. Event accumulation handles the flood problem (busy Discord channels produce many messages between agent actions). Tiered rendering (verbatim → grouped → summarised) naturally compresses high-volume channels. Drain pattern gives the agent exactly what happened since it last looked, at the right level of detail.
**Trade-offs:** Depends on blocks as a runtime dependency. Agent response time depends on LLM latency, not a fixed tick rate.
**Sources:** `casehub-blocks` PartitionedObservationService, TieredObservationRenderer; `wacky-manor` ObservationService, CharacterAgentLoop
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D3: Three-tier layering — blocks / quarkmind-core / quarkmind-discord

**Choice:** Push general event/observation machinery into blocks, bot-harness abstractions into quarkmind-core, world-specific code into quarkmind-discord.

**blocks additions:**
- Dual-wake combiner (`WakeSource`) — event OR timer wake as a general reactive pattern, extending blocks.agentic EventSource/ChoreographedDriver
- Chat-oriented observation rendering — tiered renderer variant that groups by sender or conversation thread

**quarkmind-core additions:**
- Observation → perception bridge — contract for how a `PartitionedObservationService` drain result becomes a `WorldPerception`
- Need-threshold wake — scheduler evaluates `NeedState`, fires proactive-wake event via blocks WakeSource when needs cross thresholds
- Proactive decision gate — agent-level logic for "should I actually act or stay quiet?"

**quarkmind-discord (world-specific):**
- Discord event → blocks event translation
- Discord-specific visibility policy (which channels to watch)
- Personality prompt construction, intent dispatch via casehub-connectors

**Alternatives:**
- Everything in quarkmind-discord — works but every future world rebuilds the same patterns
- Everything in quarkmind-core — violates blocks' role as the general event framework
**Rationale:** Follows existing platform layering: blocks is general-purpose, quarkmind-core is bot-harness, worlds are domain-specific. Discord becomes the second consumer of quarkmind-core SPIs (after QuarkVille), proving the abstractions.
**Trade-offs:** Requires changes to two repos (blocks, quarkmind) before quarkmind-discord is functional.
**Exploration:** quick
**Depends on:** D2
**Status:** captured
