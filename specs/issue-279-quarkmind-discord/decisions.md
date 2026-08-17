# QuarkMind Discord — Decisions

## D1: Two sub-modules (protocol + agent)

**Choice:** `quarkmind-discord-protocol/` (shared types: DiscordPerception, DiscordIntent) + `quarkmind-discord-agent/` (agency loop, world bridge, Quarkus app). Mirrors QuarkVille's separation pattern.
**Alternatives:**
- Flat single module — simpler, but breaks pattern consistency across quarkmind worlds
**Rationale:** Consistency with QuarkVille's three-module structure (minus the server module Discord doesn't need). If a Discord dashboard or observer tool is added later, it can depend on protocol without pulling in agent dependencies.
**Trade-offs:** Slightly more Maven boilerplate for a module that may never have a third consumer.
**Exploration:** quick
**Status:** captured

## D2: Event-driven execution with chat SPI delta + blocks compression

**Choice:** Agent wakes on two triggers: (a) message arrives in a watched channel (immediate via `DiscordInboundConnector`), (b) heartbeat timer fires to check needs. On wake, agent reads the structured conversation delta from the chat SPI's `MessageHistory.messages(channel, since)` — already threaded via `parentRef`. A new blocks capability reconstructs thread structure, classifies new vs continuing conversations, and produces a structured LLM-consumable delta report. Blocks tiered rendering acts as a compression layer only when volume exceeds thresholds.
**Alternatives:**
- Blocks observation pipeline as primary perception (Wacky Manor pattern) — rebuilds what chat SPI already provides
- Dual-loop (reactive + autonomous) — more complex, two loops sharing mutable state
- Single tick loop with fixed interval — adds unnecessary latency to message responses
**Rationale:** The chat SPI already solves the delta problem (`MessageHistory.messages(channel, since)` gives `List<ReceivedMessage>` with threading). The agent reads conversations structurally rather than through an event accumulation layer. Blocks provides conversation-aware delta reporting (thread reconstruction, new vs continuing, pre-delta context) and compression for high-volume channels — both general capabilities useful to any LLM-powered chat agent, not just Discord.
**Trade-offs:** Depends on both casehub-connectors (chat SPI) and blocks (delta report + compression). Agent response time depends on LLM latency, not a fixed tick rate.
**Sources:** `casehub-connectors` chat-spi MessageHistory, ReceivedMessage, Threading; `casehub-blocks` TieredObservationRenderer; `wacky-manor` ObservationService
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D3: Three-tier layering — blocks / quarkmind-core / quarkmind-discord

**Choice:** Push general event/observation machinery into blocks, bot-harness abstractions into quarkmind-core, world-specific code into quarkmind-discord.

**blocks additions:**
- Dual-wake combiner (`WakeSource`) — event OR timer wake as a general reactive pattern, extending blocks.agentic EventSource/ChoreographedDriver
- Conversation-aware delta report — takes `List<ReceivedMessage>` from any `ChatPlatform`, reconstructs thread structure from `parentRef`, classifies new vs continuing conversations, provides pre-delta context for continuing threads. General capability for any LLM chat consumer.
- Attention-priority classification — messages classified by relevance to the agent: direct address (replies to bot, @mentions) → always verbatim; active threads (bot previously participated) → elevated; ambient → compressible. Compression layer respects priority: elevated items survive summarisation.
- Tiered compression for chat — applies only when volume exceeds thresholds. Compresses ambient conversation while preserving elevated items verbatim.

**quarkmind-core additions:**
- Chat perception bridge — contract for how a blocks conversation delta report becomes a `WorldPerception`. Wires chat SPI + blocks delta into the agency loop.
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
