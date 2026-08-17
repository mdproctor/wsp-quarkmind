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

## D4: Intent model — text + reactions

**Choice:** `DiscordIntent` is a sealed interface with three variants: `Send(channelId, content)` — post a message; `Reply(parentRef, content)` — reply in a thread; `React(messageRef, emoji)` — add an emoji reaction. Plus implicit `Stay quiet` (no intent emitted). Maps directly to chat SPI: `Messaging.send()`, `Threading.reply()`, `Reactions.add()`.
**Alternatives:**
- Text only — simpler but loses lightweight engagement (reactions feel alive with no compose cost)
- Full action surface (presence, channel creation, member management) — over-scoped for v1
**Rationale:** Reactions are nearly free to implement (one-liner through the SPI) and add natural expressiveness. A character reacting with :eyes: or :thinking: without composing a full message is a distinctly non-bot behaviour. Text covers communication; reactions cover ambient personality.
**Trade-offs:** Three intent types to handle in dispatch vs one. Minimal cost.
**Exploration:** quick
**Depends on:** D2
**Status:** captured

## D5: Four-layer memory — existing foundation, no new modules

**Choice:** Four memory layers, all backed by existing foundation modules:

| Layer | Purpose | Backed by |
|-------|---------|-----------|
| Working memory | Current conversation context | LLM context window (no persistence) |
| Episodic memory | Conversation cases — who, what, how it went | `CbrCaseMemoryStore` with `TemporalDecayCbrCaseMemoryStore` + importance scoring |
| Semantic memory | Reflections — generalized insights about people and topics | `ReflectionService` → stored as high-importance CBR cases |
| Relationship memory | Per-person profiles — trust, history, opinion | `GraphitiCaseMemoryStore` or `Mem0CaseMemoryStore`; `casehub-ledger` for trust scores |

**Small gaps to fill (enhancements, not new modules):**
- Importance scoring at ingest in neocortex — LLM-rated significance 1–10 at memory creation time (Park et al. retrieval composite = recency + importance + relevance; neocortex has recency and relevance but not importance)
- Idle-time reflection trigger in quarkmind-core — "when to reflect" scheduler, fires when accumulated importance exceeds threshold during idle periods (analogous to Park et al.'s reflection threshold)
- Relationship schema for Graphiti/Mem0 — per-person model with trust score, interaction count, opinion, shared topics (schema on existing infrastructure)

**Alternatives:**
- CBR only (flat case store) — loses relationship modeling and reflection/consolidation
- Custom memory system — duplicates what neocortex already provides
- New foundation module for social memory — unnecessary; Graphiti + Mem0 + ledger already cover the infrastructure
**Rationale:** Research survey (Park et al. Generative Agents, MemGPT/Letta, CoALA framework, Mem0 graph memory) confirms three-tier episodic/semantic/relationship model as field consensus. Neocortex already provides all three tiers via CBR, reflection, and graph memory stores. Platform also has novel capabilities not found in surveyed frameworks: mood-modulated retrieval, personality-weighted retrieval, trust-weighted CBR. No new foundation modules needed — the work is wiring and schema.
**Trade-offs:** Importance scoring at ingest adds an LLM call per memory creation (latency + cost). Acceptable — memory creation is not on the hot path.
**Sources:** Park et al. 2023 Generative Agents; MemGPT/Letta tiered memory; Mem0 graph memory; CoALA cognitive architecture; casehub-neocortex existing stores
**Exploration:** deep-analysis
**Depends on:** D2, D3
**Status:** captured

## D6: Personality — Eidos prompt + quarkmind-core mechanical needs + reflection-driven growth

**Choice:** Three-layer personality system using existing foundation:

**Layer 1 — Prompt-driven (Eidos → LLM):** `AgentDescriptor` defines character personality via disposition axes (SOCIAL_ORIENTATION, RULE_FOLLOWING, RISK_APPETITE, AUTONOMY, CONFLICT_MODE), `styleProfile`, `dispositionProfile`, `briefing`, `goals`, and `constraints`. `SystemPromptRenderer` renders this into the LLM system prompt.

**Layer 2 — Mechanical (quarkmind-core NeedState):** Dispositions affect non-LLM behavior via `DispositionNeedModifier`:
- SOCIAL_ORIENTATION → social need decay rate (extrovert drops fast when alone)
- AUTONOMY → heartbeat frequency for unprompted action
- RISK_APPETITE → willingness to engage with strangers/controversial topics

**Layer 3 — Growth (reflection → disposition evolution):** Discord interactions → episodic memory → reflection threshold → semantic insights → `BehavioralSignal` → Eidos `DispositionEvolution`. The idle-time reflection trigger (D5) is the shared mechanism driving both memory consolidation AND personality growth.

**Alternatives:**
- Prompt-only personality (no mechanical needs) — character sounds consistent but doesn't behave consistently
- Static personality (no evolution) — misses the "grows over time" requirement
**Rationale:** Each layer serves a different concern. Prompt shapes what the character says. Mechanical needs shape when and whether it acts. Reflection-driven growth makes the character change over time. All three use existing foundation modules.
**Trade-offs:** Disposition evolution is subtle — may be hard to observe. Acceptable — the point is long-term believability, not visible personality swings.
**Sources:** casehub-eidos AgentDescriptor, DispositionEvolution, BehavioralSignal; quarkmind-core DispositionNeedModifier, NeedState; casehub-neocortex ReflectionOrchestrator
**Exploration:** deep-analysis
**Depends on:** D3, D5
**Status:** captured

## D7: Three needs — SOCIAL (with sub-drives), CURIOSITY, EXPRESSION

**Choice:** Three needs drive the Discord character's autonomous behavior:

**SOCIAL** — decays when idle, satisfied by conversation. The primary engagement driver. Three sub-drives shape how social need is satisfied:
1. *Learn about people* — the character seeks to understand who people are, what they care about, their history
2. *Help people* — the character looks for opportunities to be useful, answer questions, offer support
3. *Meaningful contribution* — the character seeks to add value to the social group as a whole, not just individual exchanges (sharing relevant information, connecting people, remembering things the group forgets)

Sub-drives are not separate needs — they're strategies for satisfying SOCIAL, weighted by disposition. A high-AUTONOMY character leans toward meaningful contribution; a high-SOCIAL_ORIENTATION character leans toward learning about people.

**CURIOSITY** — decays slowly, spikes when interesting topics appear. Drives questions, topic exploration, following up on things mentioned days ago.

**EXPRESSION** — builds up over time, satisfied by sharing opinions/reactions. Drives unprompted posting, emoji reactions, and unsolicited commentary.

All three map through `DispositionNeedModifier` — disposition axes affect decay rates and satisfaction thresholds.

**Alternatives:**
- More granular needs (separate HELP, LEARN, CONTRIBUTE) — over-specified; the LLM handles strategy selection within the social drive better than mechanical sub-needs
- Fewer needs (SOCIAL only) — loses the distinction between wanting to talk (social), wanting to know (curiosity), and wanting to share (expression)
**Rationale:** Three needs are enough to produce varied autonomous behavior. The sub-drives under SOCIAL give the LLM guidance on *how* to be social without mechanizing strategy selection — the character might satisfy its social need by helping someone one moment and by asking a question the next, depending on context.
**Trade-offs:** Sub-drives are prompt-level guidance, not mechanical — they don't have independent decay rates. If finer control is needed later, they can be promoted to full needs.
**Exploration:** quick
**Depends on:** D6
**Status:** captured
