# QuarkMind Discord — Decisions

## D1: Two sub-modules (protocol + agent)

**Choice:** `quarkmind-chat-protocol/` (shared types: ChatPerception, ChatIntent) + `quarkmind-chat-agent/` (agency loop, world bridge, Quarkus app).
**Alternatives:**
- Flat single module — simpler, but protocol types become entangled with agent runtime dependencies
**Rationale:** Two-module split stands on its own merits: (1) protocol types are shared contracts — a future dashboard or observer tool can depend on protocol without pulling in the agent's Quarkus runtime, (2) clear boundary between message types and application logic. QuarkVille uses three modules (protocol + server + agent) because it has a central game server; Discord has two because Discord IS the server — the structural difference reflects different architectural constraints, not inconsistency. The three other world modules (minecraft, evennia, sonaria) are stubs — there is no established multi-world pattern to claim consistency with; the two-module structure is justified by Discord's own constraints.
**Trade-offs:** Slightly more Maven boilerplate for a module that may never have a third consumer.
**Exploration:** quick
**Status:** revised (R1-02: rationale rewritten to stand on own merits; removed misleading "mirrors QuarkVille" claim; R1-10: module names updated to match D11/code — quarkmind-chat-protocol, quarkmind-chat-agent)

## D2: Event-driven execution with chat SPI delta + blocks compression

**Choice:** Agent wakes on two triggers via existing blocks primitives: `new ChoreographedDriver(invoker, policy, discordEventSource, EventSource.ticker(heartbeat, executor))`. **Pattern: push-wake, pull-context** — Discord Gateway WebSocket push wakes the agent (via `DiscordInboundConnector` adapted as an `EventSource`); `MessageHistory.messages(channel, since)` provides structured conversation context with threading via `parentRef`. The `MessageHistory` implementation accumulates Gateway events locally, falling back to REST API only on cold start or detected gaps — avoids redundant REST calls and rate limit pressure. `DriverEvent.source` distinguishes timer wakes ("timer") from message wakes. Blocks `TieredObservationRenderer` acts as compression layer when volume exceeds thresholds.
**Alternatives:**
- Blocks observation pipeline as primary perception (Wacky Manor pattern) — rebuilds what chat SPI already provides
- Dual-loop (reactive + autonomous) — more complex, two loops sharing mutable state
- Single tick loop with fixed interval — adds unnecessary latency to message responses
- Gateway-only accumulation (no REST pull) — simpler but requires robust local gap detection; REST fallback still needed for cold start
**Rationale:** Push-wake, pull-context is a well-established pattern in distributed systems. The chat SPI's `MessageHistory` abstraction decouples the perception contract from the transport. The existing `DiscordChatPlatform.getMessageHistory()` calls the REST API directly; a new Gateway-accumulating `MessageHistory` implementation is needed (either in quarkmind-discord wrapping the `DiscordInboundConnector`'s Gateway events, or as a connectors enhancement). This is new implementation work, not existing infrastructure. Blocks' `ChoreographedDriver` with `EventSource` varargs already provides the dual-wake combiner — no new blocks abstractions needed (EventSource.merge handles combination, DriverEvent.source handles discrimination).
**Message buffer eviction:** `DiscordGatewayMessageHistory` drains consumed messages after each agency tick. Once `ChatPerceptionBridge.buildDelta()` has processed a channel's messages for a given tick, those messages are removed from the buffer. The tick boundary is the natural eviction point — messages have been perceived and potentially stored as observations. This prevents unbounded buffer growth while preserving all messages until they have been consumed by the agency loop.

**Trade-offs:** Depends on both casehub-connectors (chat SPI) and blocks (compression). Agent response time depends on LLM latency, not a fixed tick rate.
**Sources:** `casehub-connectors` chat-spi MessageHistory, ReceivedMessage, Threading; `casehub-blocks` TieredObservationRenderer, EventSource, ChoreographedDriver, DriverEvent
**Exploration:** quick
**Depends on:** D1
**Status:** revised (R1-03: named push-wake/pull-context pattern, clarified MessageHistory backed by Gateway accumulation; R1-04: referenced existing EventSource.merge + ticker — no new blocks abstractions needed; R2-01: acknowledged Gateway-accumulating MessageHistory is new implementation work; R1-09: drain-after-consume eviction added to prevent unbounded buffer growth)

## D3: Two-tier layering — quarkmind-core / quarkmind-discord

**Choice:** Chat-agent abstractions into quarkmind-core, world-specific code into quarkmind-discord. No blocks additions needed — existing blocks primitives (`TieredObservationRenderer`, `EventSource`, `ChoreographedDriver`) are used directly.

**blocks additions:** None. Existing `TieredObservationRenderer<E>` with custom `Function<E, String> eventRenderer`, `Function<E, String> groupKeyExtractor`, and `Summariser<E, String>` provides compression. `ChoreographedDriver` with `EventSource.merge()` provides dual-wake. No new blocks types or chat-domain dependencies.

**quarkmind-core additions:**
- Conversation-aware delta report (`ChatDeltaReport`) — takes messages from any chat source, reconstructs thread structure from `parentRef`, classifies new vs continuing conversations, provides pre-delta context. General capability for any chat-based quarkmind world.
- Bot identity detection (`BotIdentityDetector`) — interface for identifying the bot's own messages in a conversation stream. Platform-agnostic contract; implementations are platform-specific.
- Attention-priority classification — messages classified by relevance: direct address (replies to bot, @mentions) → always verbatim; active threads (bot participated) → elevated; ambient → compressible. The classification produces priority tags that drive `TieredObservationRenderer` grouping/summarisation.
- Need-threshold wake (`NeedThresholdWake`) — scheduler evaluates `NeedState`, fires proactive-wake event via `EventSource` when needs cross thresholds.
- Proactive decision gate — agent-level logic for "should I actually act or stay quiet?" Considers time since last post, channel activity, and social context.

**quarkmind-chat-agent (shared chat harness, per D11):**
- Chat perception bridge (`ChatPerceptionBridge`, `DefaultChatPerceptionBridge`) — contract for how a conversation delta becomes a `WorldPerception`. Wires chat SPI + blocks compression into the agency loop. Lives in quarkmind-chat-agent because it composes chat-agent-level concerns (blocks rendering, LLM prompt construction) — not a quarkmind-core primitive.
- Thread participation state — `ChatAgencyLoop.participatedThreadIds` tracks which threads the bot has contributed to. Reconstructed on cold start from `MessageHistory` by filtering recent messages via `BotIdentityDetector` (configurable lookback window).

**quarkmind-discord (world-specific, sub-package of quarkmind-chat-agent):**
- Discord event → blocks event translation (DiscordInboundConnector adapted as EventSource)
- Discord-specific visibility policy (which channels to watch)
- Bot-identity-specific detection: @mention parsing, bot-reply detection (requires knowing the bot's own user ID)
- Personality prompt construction, intent dispatch via casehub-connectors

**Alternatives:**
- Everything in quarkmind-discord — works but every future chat-based world rebuilds the same patterns
- Chat-domain logic in blocks — violates blocks' domain-agnostic boundary (APPLICATIONS.md: "domain logic stays in the application"); would create blocks → connectors dependency
**Rationale:** Blocks is general-purpose event/observation machinery (`EventSource`, `TieredObservationRenderer`, `ChoreographedDriver`). Chat semantics (thread reconstruction, @mention detection, conversation delta) are domain knowledge that belongs in the application tier. quarkmind-core is the right home: it's the bot-harness layer shared by all quarkmind worlds, and any chat-based world (Discord, IRC, Slack) would reuse these abstractions. Discord-specific detection (bot user ID, Discord @mention syntax) stays in quarkmind-discord.
**Trade-offs:** Two-tier instead of three-tier — fewer moving parts, but quarkmind-core accumulates chat-agent abstractions that may not generalize to non-chat worlds (SC2, Godot).
**Exploration:** quick
**Depends on:** D2
**Status:** revised (R1-05: moved chat-domain logic from blocks to quarkmind-core — blocks stays domain-agnostic; R1-06: dropped WakeSource — EventSource.merge suffices; original three-tier became two-tier; R1-07: ChatPerceptionBridge moved from quarkmind-core to quarkmind-chat-agent listing — it's a chat-agent composition concern; R1-13: thread participation cold-start reconstruction added)

## D4: Intent model — text + reactions

**Choice:** `DiscordIntent` is a sealed interface with three variants: `Send(channelId, content: ChatContent)` — post a message; `Reply(parentRef, content: ChatContent)` — reply in a thread; `React(messageRef, emoji)` — add an emoji reaction. Plus implicit `Stay quiet` (no intent emitted). Maps directly to chat SPI: `Messaging.send()`, `Threading.reply()`, `Reactions.add()`. `ChatContent` supports markdown, attachments, and `RichCard` embeds (Discord embeds with titles, descriptions, fields, thumbnails, footers) — the agent retains the full Discord expressive surface. Validation (embed limits: max 10 embeds, 256-char titles, 4096-char descriptions, 6000 total chars) happens at dispatch through the chat SPI, not at intent creation — the SPI knows the platform constraints; the agent should be free to express intent without pre-validating limits.
**Alternatives:**
- Text only — simpler but loses lightweight engagement (reactions feel alive with no compose cost)
- Full action surface (presence, channel creation, member management) — over-scoped for v1
- String content instead of ChatContent — loses embeds, attachments, formatted markdown
**Rationale:** Reactions are nearly free to implement (one-liner through the SPI) and add natural expressiveness. A character reacting with :eyes: or :thinking: without composing a full message is a distinctly non-bot behaviour. Text covers communication; reactions cover ambient personality. ChatContent preserves the full Discord surface — constraining to String would artificially limit the character's expressive range.
**Trade-offs:** Three intent types to handle in dispatch vs one. Minimal cost.
**Exploration:** quick
**Depends on:** D2
**Status:** revised (R1-07: content type explicit as ChatContent; validation at dispatch, not intent creation)

## D5: Four-layer memory — existing foundation, no new modules

**Choice:** Four memory layers, all backed by existing foundation modules:

| Layer | Purpose | Backed by |
|-------|---------|-----------|
| Working memory | Current conversation context | LLM context window (no persistence) |
| Episodic memory | Conversation cases — who, what, how it went | `CbrCaseMemoryStore` with `TemporalDecayCbrCaseMemoryStore` + importance scoring |
| Semantic memory | Reflections — generalized insights about people and topics | `ReflectionService` → stored as high-importance CBR cases |
| Relationship memory | Per-person profiles — trust, history, opinion | `GraphitiCaseMemoryStore` or `Mem0CaseMemoryStore`; `casehub-ledger` for trust scores |

**Three capability additions (enhancements to existing modules, not new modules):**
- Importance scoring at ingest in neocortex — LLM-rated significance 0.0–1.0, mapping directly to `MemoryInput.importance` without normalization (Park et al. retrieval composite = recency + importance + relevance; neocortex has recency and relevance but not importance). Importance scoring runs asynchronously — memory creation returns immediately with a default importance of 0.5 (neutral); the LLM-scored value replaces the default when the async call completes. Default 0.5 ensures unscored memories participate in retrieval at neutral rank rather than being excluded or unpredictably ordered. Not on the hot path for response latency.
- Idle-time reflection trigger in quarkmind-core — "when to reflect" scheduler, fires when accumulated importance exceeds threshold during idle periods (analogous to Park et al.'s reflection threshold). Non-trivial scheduler with state management: tracks accumulated importance since last reflection, monitors idle intervals, fires reflection when both conditions met.
- Relationship schema for Graphiti/Mem0 — per-person model with trust score, interaction count, opinion, shared topics (schema on existing infrastructure). Requires schema design and migration, not a trivial enhancement.

**Alternatives:**
- CBR only (flat case store) — loses relationship modeling and reflection/consolidation
- Custom memory system — duplicates what neocortex already provides
- New foundation module for social memory — unnecessary; Graphiti + Mem0 + ledger already cover the infrastructure
**Rationale:** Research survey (Park et al. Generative Agents, MemGPT/Letta, CoALA framework, Mem0 graph memory) confirms three-tier episodic/semantic/relationship model as field consensus. Neocortex already provides all three tiers via CBR, reflection, and graph memory stores. Platform also has novel capabilities not found in surveyed frameworks: mood-modulated retrieval, personality-weighted retrieval, trust-weighted CBR. No new foundation modules needed — the work is wiring, schema, and the three capability additions described above.
**Trade-offs:** Importance scoring adds an async LLM call per memory creation (cost, not latency). The idle-time reflection trigger and relationship schema are non-trivial enhancements that require design attention beyond simple wiring.
**Sources:** Park et al. 2023 Generative Agents; MemGPT/Letta tiered memory; Mem0 graph memory; CoALA cognitive architecture; casehub-neocortex existing stores
**Exploration:** deep-analysis
**Depends on:** D2, D3
**Status:** revised (R1-08: scope characterization corrected — "small gaps" → "three capability additions" with honest complexity acknowledgment; importance scoring confirmed async; R1-03: importance scale corrected from 1–10 to 0.0–1.0; R1-11: default importance 0.5 for unscored memories)

## D6: Personality — Eidos prompt + quarkmind-core mechanical needs + reflection-driven growth

**Choice:** Three-layer personality system using existing foundation:

**Layer 1 — Prompt-driven (Eidos → LLM):** `AgentDescriptor` defines character personality via disposition axes (SOCIAL_ORIENTATION, RULE_FOLLOWING, RISK_APPETITE, AUTONOMY, CONFLICT_MODE), `styleProfile`, `dispositionProfile`, `briefing`, `goals`, and `constraints`. `SystemPromptRenderer` renders this into the LLM system prompt.

**Layer 2 — Mechanical (quarkmind-core NeedState):** Dispositions affect non-LLM behavior through three distinct mechanisms:
- SOCIAL_ORIENTATION → social need **decay rate** via `DispositionNeedModifier.modifyDecayRate()` (extrovert drops fast when alone). This is the only effect routed through the DispositionNeedModifier SPI.
- AUTONOMY → **heartbeat frequency** for unprompted action — configuration-time mapping from disposition profile to `EventSource.ticker(interval, executor)` interval. Set at startup from the character's disposition axes, not a runtime modifier.
- RISK_APPETITE → **willingness to engage** with strangers/controversial topics — evaluated by the proactive decision gate (D7/D3) at decision time. The gate reads risk appetite from the disposition profile when evaluating "should I act?" Not a need decay concern.

**Layer 3 — Growth (reflection → disposition activation → evolution):** Discord interactions → episodic memory → reflection threshold → semantic insights → classify activated disposition function term → `DispositionSignalStore.recordActivation(agentId, tenancyId, functionTerm)` → periodic `DispositionHealth.probe(descriptor, context)` → on `EvolutionPending` → `DispositionEvolution.evaluate(descriptor, pending)` → `Evolved` with new disposition profile. The idle-time reflection trigger (D5) is the shared mechanism driving both memory consolidation AND disposition activation.

**Reflection → disposition activation mapping:** `ReflectionService` produces insight summaries (e.g., "User X has been consistently helpful and trustworthy across 12 interactions"). A new quarkmind-core `ReflectionDispositionActivator` classifies which disposition function term from the agent's `AgentDescriptor.disposition().dispositionProfile()` a reflection activates — e.g., a reflection about consistent empathy activates the "empathetic" function term; a reflection about analytical conversations activates "analytical". `DispositionSignalStore.recordActivation()` records the activation; `DefaultDispositionHealth.probe()` computes effective weights as `base + count × delta` and checks threshold crossings (DOMINANT_AUXILIARY_SWAP, DOMINANT_REPLACEMENT, AUXILIARY_REPLACEMENT, STRUCTURAL_REORGANIZATION). When a threshold is crossed, `DispositionEvolution.evaluate()` produces either `Evolved(newProfile)` or `Dampened(decayFactor)`.

`BehavioralSignal` (`DECLINE`, `SUCCESS`, `COMPLIANT`, `VIOLATED`) is a capability health signal used by `CapabilityHealth` to track operational health — it is NOT part of the personality evolution pipeline and is removed from D6.

**Alternatives:**
- Prompt-only personality (no mechanical needs) — character sounds consistent but doesn't behave consistently
- Static personality (no evolution) — misses the "grows over time" requirement
**Rationale:** Each layer serves a different concern. Prompt shapes what the character says. Mechanical needs shape when and whether it acts. Reflection-driven disposition activation makes the character change over time. All three use existing foundation modules. Discord is the first consumer of the full reflection → disposition activation → evolution chain — each link uses existing foundation types (`ReflectionService`, `DispositionSignalStore`, `DispositionHealth`, `DispositionEvolution`), but the end-to-end integration has not been exercised in any existing world (QuarkVille uses direct LLM without reflection; SC2 uses CaseEngine without personality evolution). The integration chain will be validated incrementally during implementation.
**Trade-offs:** Disposition evolution is subtle — may be hard to observe. Acceptable — the point is long-term believability, not visible personality swings.
**Sources:** casehub-eidos AgentDescriptor, DispositionSignalStore, DispositionHealth, DispositionEvolution; quarkmind-core DispositionNeedModifier, NeedState; casehub-neocortex ReflectionService
**Exploration:** deep-analysis
**Depends on:** D3, D5 (CRITICAL — D5's reflection trigger is the sole input to Layer 3; if reflection never fires, personality never evolves)
**Status:** revised (R1-09: reflection→BehavioralSignal mapping was wrong — BehavioralSignal is capability health, not personality evolution; R2 corrected to use DispositionSignalStore.recordActivation → DispositionHealth.probe → DispositionEvolution.evaluate pipeline; R1-20: D5 dependency upgraded to CRITICAL; R1-04: Layer 2 corrected — only SOCIAL_ORIENTATION uses DispositionNeedModifier; AUTONOMY and RISK_APPETITE use separate mechanisms)

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

**Rate governing — three layers prevent anti-social output volume:**
1. **Global output governor** (quarkmind-core) — maximum actions per time window, configurable per world. Default: 1 unprompted post per 5 minutes, hard minimum interval of 30 seconds between any two character actions.
2. **Proactive decision gate** (quarkmind-core, from D3) — evaluates "should I actually act or stay quiet?" before any need-driven action. Considers: time since last post, channel activity level, whether others are typing, conversational flow (don't interrupt).
3. **Channel-aware pacing** (quarkmind-discord) — unprompted post frequency scales with channel activity. In a quiet channel, the character can post to fill silence; in a busy channel, it waits for natural gaps and participates reactively. Activity-relative pacing prevents the character from dominating conversation regardless of need pressure.

These constraints are independent of need mechanics. Need threshold crossing is necessary but not sufficient for action: EXPRESSION crosses threshold → proactive decision gate evaluates social context → channel-aware pacing governs timing → global governor caps frequency. Multiple simultaneous threshold crossings (SOCIAL + EXPRESSION + CURIOSITY) produce a single prioritized action, not three concurrent actions.

**Rate governing alternatives considered:**
- **Single priority queue with configurable policies** — conflates output throttling with social decision-making. "Am I rate-limited?" and "should I speak?" are orthogonal concerns that change independently. A policy-based queue cannot evaluate conversational flow or whether others are typing.
- **Token bucket rate limiter** — the global output governor IS a token bucket (max actions per window, hard minimum interval). But token buckets handle rate compliance, not social context. A token bucket alone permits posting every 30 seconds into a silent channel — technically compliant, socially obnoxious. The proactive decision gate adds the social judgment layer.
- **Intent filter chain** — architecturally equivalent to the three-layer model. The three layers ARE an ordered filter pipeline: governor (rate cap) → gate (social context evaluation) → pacer (channel activity scaling). The filter-chain framing is the more precise description of the same architecture.

**Alternatives:**
- More granular needs (separate HELP, LEARN, CONTRIBUTE) — over-specified; the LLM handles strategy selection within the social drive better than mechanical sub-needs
- Fewer needs (SOCIAL only) — loses the distinction between wanting to talk (social), wanting to know (curiosity), and wanting to share (expression)
**Rationale:** Three needs are enough to produce varied autonomous behavior. The sub-drives under SOCIAL give the LLM guidance on *how* to be social without mechanizing strategy selection — the character might satisfy its social need by helping someone one moment and by asking a question the next, depending on context. Rate governing ensures the character's behavior is socially appropriate regardless of need pressure.
**Trade-offs:** Sub-drives are prompt-level guidance, not mechanical — they don't have independent decay rates. If finer control is needed later, they can be promoted to full needs.
**Exploration:** quick
**Depends on:** D6
**Status:** revised (R1-10: rate-governing mechanisms added — global governor, proactive decision gate, channel-aware pacing; R1-14: Discord-specific rate constraints addressed; R1-05: rate governing alternatives analysis added — three-layer model confirmed as ordered filter pipeline)

## D8: Direct LLM agency loop, no CaseEngine

**Choice:** Discord character uses direct LLM invocation via quarkmind-core's `AgencyLoop` SPI, not CaseEngine blackboard dispatch. Follows QuarkVille's `VilleAgencyLoop` pattern: `AgencyLoop.tick(AgencyContext)` with an `LlmInvoker` for reasoning.
**Alternatives:**
- CaseEngine with `@CaseType("discord-character")` plugins — adds blackboard coordination overhead for a single-agent character that doesn't need multi-plugin dispatch
- Hybrid (CaseEngine for some phases, direct LLM for others) — unnecessary complexity for v1
**Rationale:** A Discord character is a single agent making holistic decisions about how to participate in conversation. CaseEngine's value is coordinating multiple specialist plugins against a shared blackboard — SC2's four-plugin model (strategy, economics, tactics, scouting) needs this. A chat character doesn't decompose into specialist plugins; it perceives, reasons, and acts as one unit. QuarkVille's `VilleAgencyLoop` proves `AgencyLoop.tick(AgencyContext)` with direct LLM invocation is sufficient for character-driven agents. ARC42STORIES.MD's "CaseHub harness pattern" framing applies to the SC2 track; Discord is the agency track, using quarkmind-core SPIs directly.
**Trade-offs:** No CaseFile blackboard means no formal plugin attribution, no binding-condition dispatch, no trust-weighted routing. These capabilities are not needed for a single-agent character.
**Exploration:** implicit (surfaced by R1-12)
**Status:** captured

## D9: Single-character per server (v1 scope)

**Choice:** v1 targets one Discord character per server. Multi-character scenarios (multiple characters interacting, per-channel personality profiles) are explicitly out of scope.
**Alternatives:**
- Multi-character from day one — affects module structure (D1), memory (D5), personality (D6), and need mechanics (D7); each system needs per-character isolation
- Per-channel personality profiles — lighter than multi-character but still adds schema complexity
**Rationale:** Single-character simplifies every design decision. One identity, one memory store, one need state, one personality. Multi-character introduces identity isolation, inter-character interaction, and memory boundaries that should be designed once the single-character experience is validated.
**Trade-offs:** If multi-character is needed later, memory and personality systems may need per-character scoping that would have been cheaper to build in from the start.
**Exploration:** implicit (surfaced by R1-13)
**Status:** captured

## D10: Discord API rate limiting strategy

**Choice:** Rate limiting operates at two independent levels:
1. **API compliance** (casehub-connectors) — the `DiscordChatPlatform` implementation tracks rate limit headers from Discord API responses and applies back-pressure. This is the connector's responsibility, not quarkmind's. Gateway reconnection with backoff on disconnect.
2. **Behavioral pacing** (quarkmind-core + quarkmind-discord) — the output governor (D7) limits character action frequency independent of API rate limits. Prevents the character from generating more intents than any platform can handle socially, even when API limits are not being hit.

**Alternatives:**
- Application-level rate limiting only — duplicates connector's responsibility
- No behavioral limiting (rely on connector) — connector handles API limits but not behavioral spam; character could post every 5 seconds and still be within Discord rate limits
**Rationale:** API rate limits and behavioral rate limits are orthogonal concerns. Discord allows ~5 REST requests/5s per route — the connector enforces this. But a character posting every 6 seconds (just under the limit) is technically compliant and socially obnoxious. The output governor ensures the character behaves like a person, not like a bot operating at maximum allowed throughput.
**Trade-offs:** Two-layer rate limiting means the slower limit (behavioral) is the binding constraint in normal operation. API limits only matter during burst scenarios (cold start catchup, reconnection).
**Exploration:** implicit (surfaced by R1-14)
**Depends on:** D7
**Status:** captured

## D11: Rename to quarkmind-chat — platform-agnostic chat bot harness

**Choice:** Module is `quarkmind-chat` (not `quarkmind-discord`). The harness works with any `ChatPlatform` SPI implementation — Discord, Slack, IRC. Platform-specific adapters (event sources, message history, identity detection) live in a sub-package within quarkmind-chat-agent. Discord is the first target; other platforms add only the thin adapter layer.
**Alternatives:**
- quarkmind-discord (Discord-specific) — build for one platform, generalize later. Premature in this case — the design is already platform-agnostic.
**Rationale:** The entire design (agency loop, perception, intents, needs, personality, memory) depends on the `ChatPlatform` SPI, not on any Discord-specific type. The only Discord-specific code is 3 classes: event source, gateway message history, identity detector. Naming the module after the abstraction level it operates at (chat) rather than one implementation (Discord) is accurate.
**Trade-offs:** None — no additional implementation cost.
**Exploration:** quick
**Status:** captured

## D12: Configuration-driven deployment — no Java code required for new characters

**Choice:** All character customization is file-driven:
- **Personality** (already exists): Eidos `AgentDescriptor` loaded from YAML via `ClasspathYamlDescriptorRegistrar` — disposition axes, style profile, briefing, goals, constraints
- **Needs**: decay rates, satisfaction values, thresholds defined in `application.yaml` (Quarkus config)
- **Platform**: `ChatPlatform` selected by Quarkus profile (`%discord`, `%slack`, `%irc`) — bot token, guild/channel config in profile-scoped properties
- **Channels**: watch list configured in properties — which channels to observe, which to actively participate in
- **Rate governing**: output governor window, minimum interval, channel-pacing parameters in properties
- **Heartbeat**: interval configured in properties

A new character deployment requires: one YAML personality descriptor + one `application.yaml` with platform credentials and tuning. No Java code.

**Personality generator** (future, out of v1 scope): LLM-powered wizard that helps non-technical users create character descriptors. "Describe your character" → generates the YAML descriptor with appropriate disposition axes, style terms, and briefing. Filed as a separate issue.

**Alternatives:**
- Java code required for new characters — limits audience to developers
**Rationale:** Configuration-driven deployment removes the need to write Java code for new characters. Developers who can run a JVM application and configure YAML/environment variables can deploy a character without touching the codebase. The deployment audience is developers comfortable with JVM deployment and Discord Developer Portal setup, not end users — but the barrier is configuration, not code. The personality generator (future) further lowers the character-creation bar within that audience.
**Trade-offs:** Need to ensure all tuning knobs are externalized as config properties, not hardcoded constants.
**Exploration:** quick
**Depends on:** D11
**Status:** revised (R1-15: deployment audience language corrected — "anyone who can write YAML" → "developers who can run JVM apps")

## D13: ChatMemoryFacade in quarkmind-chat-agent

**Choice:** A single `ChatMemoryFacade` class in quarkmind-chat-agent that composes neocortex stores behind two methods: `recall(conversationContext, participants)` → `List<Memory>` and `ingest(observation, sourceRefs)` → void. ChatAgencyLoop depends only on the facade, not on neocortex types directly. Each quarkmind world gets its own facade with world-specific shapes — ville would get VilleMemoryFacade for game-event shapes.
**Alternatives:**
- Direct injection of CaseMemoryStore + GraphCaseMemoryStore + re-rankers into ChatAgencyLoop — simpler but loop grows complex and harder to test
- quarkmind-core MemoryBridge SPI — no realistic second consumer; chat platforms all use quarkmind-chat via different connectors, and Ville has fundamentally different perception shapes
**Rationale:** The neocortex stores are already the shared foundation layer. What varies per world is composition: what to store, what to query for, how to format results. A per-world facade handles this without premature generalization. No second chat world exists that isn't covered by quarkmind-chat + a different connector.
**Trade-offs:** One more class to maintain. Minimal cost.
**Exploration:** quick
**Depends on:** D5
**Status:** captured

## D14: CaseMemoryStore for episodic memory — not CbrCaseMemoryStore

**Choice:** Episodic chat memories stored via `CaseMemoryStore.store(MemoryInput)` with semantic retrieval via `ExperienceQuery.search()` (MemoryOrder.RELEVANCE). This corrects D5 which named CbrCaseMemoryStore — that store is for structured feature-vector cases (like SC2GameCbrCase with supply_count, unit_composition). Chat conversations are unstructured text where semantic embedding search is the right retrieval model.
**Alternatives:**
- CbrCaseMemoryStore with a chat-specific CbrFeatureSchema — requires defining explicit features (topic, sentiment, participant count) for structured similarity, which is artificial for free-form conversation
**Rationale:** ExperienceQuery and ReflectionService already use CaseMemoryStore. PersonalityWeightedRetrieval and MoodModulatedRetrieval re-rank CaseMemoryStore results. The existing retrieval pipeline is built for text memories with semantic search. CbrCaseMemoryStore adds schema and feature-vector overhead with no benefit for text.
**Trade-offs:** Temporal decay is handled by re-rankers (PersonalityWeightedRetrieval already has recency decay with HALF_LIFE_HOURS=168) rather than by TemporalDecayCbrCaseMemoryStore decorator. This is fine — the decorator only applies to CbrCaseMemoryStore results.
**Exploration:** quick
**Depends on:** D5, D13
**Status:** captured

## D15: Observation summaries with platform source references

**Choice:** Memory text is an LLM-generated observation summary of what happened — not verbatim messages. The observation includes source references in CaseMemoryStore attributes (`source.channelId`, `source.firstMessageId`, `source.lastMessageId`, `source.timestamp.start`, `source.timestamp.end`). Raw messages remain in the platform (Discord) as the source of truth. When verification or revision is needed, the facade queries `MessageHistory` using the source references to retrieve the raw conversation.
**Alternatives:**
- Verbatim message storage — high fidelity but high volume, poor semantic embedding, duplicates platform storage
- Dual storage (observation + raw transcript as linked memories) — durable but doubles storage cost for data the platform already keeps
- Reference store (separate transcript log) — adds another store to manage
**Rationale:** The platform already has durable message storage. The agent needs to remember the meaning of what happened, not the raw bytes. Summaries embed better for semantic search than raw multi-turn chat. Source references maintain provenance — if an observation needs verification, the raw messages are queryable.
**Trade-offs:** If platform messages are deleted, source references are broken. Acceptable — this mirrors human memory (you can't always go back and verify). If durability beyond platform retention is needed later, add a transcript store as a separate concern.
**Exploration:** quick
**Depends on:** D14
**Status:** captured

## D16: Per-tick ingestion — one observation per channel per tick

**Choice:** Each agency tick produces one observation per active channel. If multiple channels have activity in one tick, each gets its own CaseMemoryStore entry with its own source references. This is the natural ingestion boundary — one tick, one perception, one set of intents.
**Alternatives:**
- Per-message ingestion — too granular, floods the store
- Per-conversation ingestion — conversations don't have clear boundaries in async chat
**Rationale:** The agency tick is the atomic unit of perception-reasoning-action. The observation captures what the agent saw, thought, and did in one cycle. Multiple channels in one tick are independent observations.
**Trade-offs:** High-activity channels with frequent ticks may generate many observations. Mitigated by the tick rate (governed by heartbeat + event wake).
**Exploration:** quick
**Depends on:** D15
**Status:** captured

## D17: updateImportance() on CaseMemoryStore — cross-repo neocortex API addition

**Choice:** Add `updateImportance(String memoryId, String tenantId, double importance)` to `CaseMemoryStore`. Importance is already a first-class field on MemoryInput and Memory. The method updates the stored importance score after initial storage. All implementations (JPA, InMemory, NoOp) gain the method. The async flow: store observation with importance=null → submit LLM importance scoring at LOW priority via LlmRequestQueue → on LLM response, call updateImportance(). This is cross-repo work in casehub-neocortex, done before quarkmind wiring.
**Alternatives:**
- Generic updateAttributes() — importance is a dedicated field, not an attribute; a generic method obscures the intent
- Store with importance upfront — blocks the tick on an LLM call; importance scoring must be async
- Quarkmind-only stub — avoids cross-repo but builds a local workaround for a foundation capability
**Rationale:** Importance scoring is fundamental to retrieval quality (Park et al. composite = recency × importance × relevance). Making it async keeps the agent responsive while still scoring every memory. The API change is small and clean.
**Failure handling:** `updateImportance()` is idempotent. If the async LLM scoring call fails (unreachable, queue full, process restart), the memory retains its default importance of 0.5 (D5). A backfill mechanism during idle periods (piggybacking on the reflection trigger from D5) scans for memories still at default importance older than a configurable threshold and resubmits them to the scoring queue at LOW priority. This ensures importance scores eventually converge even under transient LLM failures.
**Trade-offs:** Cross-repo coordination. Small API surface change. Backfill adds periodic scan cost during idle periods — bounded by the same idle-time trigger that drives reflection.
**Exploration:** quick
**Depends on:** D14
**Status:** revised (R1-11: failure handling and backfill mechanism added; idempotent updateImportance)

## D18: Observation generated from same LLM call as action decision

**Choice:** The LLM prompt includes a second instruction asking for a one-line observation alongside the action decision. Single LLM call produces both the intent (SEND/REPLY/REACT/WAIT) and the experience observation. No separate async LLM call for observation generation. JSON response gains an `observation` field. **Critical: observation extraction is decoupled from action processing.** The response parser extracts the `observation` field unconditionally before evaluating the `action` field. On WAIT (the most common action), the observation is still captured and ingested as episodic memory. Without this decoupling, observations would be discarded for every WAIT tick — the agent would only have episodic memories of ticks where it spoke, not ticks where it observed.
**Alternatives:**
- Separate async LLM call for observation — doubles LLM cost per tick, adds latency
- Template-based observation (no LLM) — loses nuance, can't capture tone or social dynamics
**Rationale:** The LLM already has the full conversation context during the tick. Asking it to describe what happened is nearly free (adds ~50 tokens to the response). The observation quality is high because the model just reasoned about the conversation.
**Trade-offs:** Slightly larger response to parse. Observation quality depends on the action model (not a dedicated summarization model). Acceptable for v1.
**Exploration:** quick
**Depends on:** D16
**Status:** revised (R1-06: observation extraction decoupled from action — observation captured on WAIT, not discarded)

## D19: LLM-based ReflectionSynthesizer in quarkmind-chat-agent

**Choice:** Implement `ReflectionSynthesizer` in quarkmind-chat-agent using `LlmRequestQueue`. The impl submits an LLM request at LOW priority with recent episodic memories and asks for generalized insights. Triggered by the idle-time reflection trigger (NeedThresholdWake in quarkmind-core, already built in phase 1). ReflectionService orchestrates: queries experience memories → passes to synthesizer → stores reflections back via CaseMemoryStore → fires ReflectionRecorded CDI event.
**Alternatives:**
- NoOp (skip reflections) — misses the memory consolidation layer; episodic memories grow without semantic compression
- Implement in neocortex — closer to the store but neocortex has no LLM integration; quarkmind-chat has LlmRequestQueue
**Rationale:** quarkmind-chat is the first consumer of the full reflection pipeline. LlmRequestQueue provides rate limiting and prioritization. The synthesizer is a thin adapter between the neocortex SPI and the quarkmind LLM infrastructure.
**Trade-offs:** Implementation lives in quarkmind, not foundation. If a second consumer appears, extract to a shared module. Acceptable — the spec acknowledges this is the first end-to-end exercise of the reflection chain.
**Exploration:** quick
**Depends on:** D5, D17
**Status:** captured

## D20: Graphiti-optional relationship memory

**Choice:** ChatMemoryFacade checks if `GraphCaseMemoryStore` resolves to a real implementation (not NoOp). With Graphiti deployed: `graphQuery()` by participant entity IDs for rich relationship facts (entity extraction, temporal graph). Without Graphiti: fall back to `CaseMemoryStore.query()` with participant attributes on observations (`participant.<userId>`). Bot works either way — Graphiti adds depth but is not required. Participant IDs use Discord user IDs for consistent entity identification.
**Alternatives:**
- Graphiti-required — richer results but requires graph database + Graphiti service for all environments including dev/test
- No graph, attributes only — simpler but loses entity extraction, relationship inference, and temporal fact tracking
**Rationale:** Graphiti infrastructure may not be available in all deployments. The bot should function with basic relationship memory (attribute-filtered queries) and gain richer memory when Graphiti is present. NoOpCaseMemoryStore already implements GraphCaseMemoryStore (per the interface's Javadoc), so CDI injection is always satisfied.
**Trade-offs:** Two code paths in ChatMemoryFacade (graph vs attribute fallback). Tested independently.
**Exploration:** quick
**Depends on:** D5, D13
**Status:** captured

## D21: LLM invocation contract — provider-agnostic string SPI

**Choice:** `ChatAgencyLoop.LlmInvoker` is `(systemPrompt, userPrompt, agentId) → String` — a provider-agnostic string interface. Structured output enforcement is an implementation concern: a `StructuredLlmInvoker` decorator validates JSON schema conformance and retries on parse failure, but the core SPI stays string-based for provider portability. Model selection per task type (action decision vs importance scoring vs reflection synthesis) is configured through `LlmRequestQueue` priority routing — LOW priority tasks (importance scoring, reflection) can route to cheaper/slower models; NORMAL priority (action decision) routes to the primary model. Fallback on LLM unavailability: the agent marks itself as degraded via a health check endpoint; the needs system continues to decay but the proactive decision gate (D7) suppresses all actions until LLM connectivity is restored. Parse failures in `parseResponse()` return empty intents (equivalent to WAIT) — logged as warnings, not silently swallowed.
**Alternatives:**
- Typed LLM SPI with structured output (tool calling / function calling) — couples the SPI to a specific LLM provider's API surface. Provider portability is a v1 requirement (support OpenAI, Anthropic, local models).
- Per-task model configuration in the SPI — adds model selection to every LlmInvoker call signature. Better handled at the queue routing layer where priorities already separate task types.
- Explicit circuit breaker for LLM unavailability — the proactive decision gate already serves this function. When the LLM is unreachable, `llmQueue.hasCapacity()` returns false, preventing action attempts. A dedicated circuit breaker would duplicate the gate's suppression logic.
**Rationale:** The string SPI is the minimal viable contract for LLM integration. Provider-specific features (structured output, streaming, vision) are implementation details of the `LlmInvoker` instance, not the contract. This keeps the agency loop testable with a simple lambda and portable across providers.
**Trade-offs:** Parse failures are indistinguishable from deliberate WAIT at the intent level (both produce empty intents). Logging distinguishes them operationally.
**Exploration:** implicit (surfaced by R1-12)
**Depends on:** D8
**Status:** captured

## D22: Gateway connection lifecycle — delegated to casehub-connectors

**Choice:** Discord Gateway lifecycle management (heartbeating, session resumption on disconnect, shard management) is owned by `casehub-connectors-discord`'s `DiscordInboundConnector`. quarkmind-chat specifies required Gateway intents via configuration: `MESSAGE_CONTENT` (privileged — requires Discord Developer Portal approval), `GUILD_MESSAGES`, `GUILD_MESSAGE_REACTIONS`. On reconnection, `DiscordGatewayMessageHistory` detects gaps between the last buffered message timestamp and the first post-reconnection message, and falls back to REST API via `DiscordChatPlatform.getMessageHistory()` for gap filling (D2's cold-start fallback path). quarkmind-chat does not manage the WebSocket connection directly — it consumes events from the connector's event stream.
**Alternatives:**
- quarkmind-chat owns the Gateway connection — duplicates connector responsibility; quarkmind-chat has no reason to know about WebSocket heartbeating or session resumption
- No intent specification — bot receives all events; wastes Gateway bandwidth and memory on irrelevant events (presence updates, voice state, etc.)
**Rationale:** The connector already handles Gateway lifecycle for the existing `DiscordChatPlatform` implementation. quarkmind-chat's responsibility is to specify WHAT events it needs (via intents configuration), not HOW to maintain the connection.
**Trade-offs:** quarkmind-chat depends on connector quality for reconnection resilience. If the connector drops events silently, the agent has gaps in perception with no direct visibility.
**Exploration:** implicit (surfaced by R1-19)
**Depends on:** D2
**Status:** captured

## D23: Context window management — ChatPerceptionBridge as compression boundary

**Choice:** `ChatPerceptionBridge.renderForLlm()` is the compression boundary. The bridge uses blocks `TieredObservationRenderer` with chat-specific configuration: per-channel message limits (most recent N messages verbatim, older messages summarized), total token budget for the rendered context, and D3's attention-priority tiers driving which channels get more context space. Configuration: reserve a configurable token budget for conversation context (e.g., 4K tokens for context in an 8K total budget). Messages beyond the per-channel verbatim limit are compressed via the blocks `Summariser` interface. Priority tiers (D3) determine allocation: direct-address channels get more verbatim messages; ambient channels may be summarized entirely.
**Alternatives:**
- Fixed truncation (most recent N messages, drop the rest) — loses conversational context from earlier messages; poor for multi-turn threads
- Client-side truncation (truncate the final prompt string) — cuts mid-message; loses structure; may break JSON response format instructions
- No compression (pass everything) — exceeds context window on active servers; fails hard instead of degrading gracefully
**Rationale:** `TieredObservationRenderer` already provides the compression infrastructure with configurable tiers and summarisation. The bridge composes this with D3's attention-priority classification to allocate context space based on relevance. This is the intended use of the blocks compression layer.
**Trade-offs:** Summarised messages lose detail that may be relevant. Acceptable — the agent can query `MessageHistory` for raw messages if deeper context is needed during a specific interaction.
**Exploration:** implicit (surfaced by R1-20)
**Depends on:** D3
**Status:** captured

## D24: GDPR and data retention — erasure via CaseMemoryStore entity operations

**Choice:** User erasure uses `CaseMemoryStore.eraseEntity(entityId, tenantId)` where `entityId` maps to the Discord user ID. Erasure scope:
- **Episodic memories:** erased by entity ID (memories where the user is a participant, tagged via `participant.<userId>` attribute on `MemoryInput`)
- **Relationship memories:** erased by entity ID in `GraphCaseMemoryStore` (Graphiti/Mem0 entity nodes and edges involving the user)
- **Reflections derived from user interactions:** reflections are tagged with participant entity IDs in their attributes. `eraseEntity()` removes reflections where the target user was a participant. Conservative strategy: reflections that synthesized insights from mixed-user interactions but where the target user was a participant are erased entirely — prefer over-erasure to under-erasure.
- **Multi-server erasure:** `eraseEntityAcrossTenants()` handles the case where a user interacts with the bot across multiple Discord servers (tenants).

The derivation chain (episodic → semantic/reflection → relationship) means a single `eraseEntity()` call must traverse all three memory layers. The attribute-based tagging (`participant.<userId>`) is the key — it enables the store to identify all memories involving a given user regardless of memory type.
**Alternatives:**
- Erasure only for direct memories (skip reflections) — non-compliant; derived data containing personal data is still personal data under GDPR
- Per-memory erasure tracking (record which memories contributed to which reflections) — correct but adds significant bookkeeping overhead. The attribute-based approach achieves the same result by tagging reflections with their source participants at creation time.
**Rationale:** The existing `CaseMemoryStore` API already provides `eraseEntity()` and `eraseEntityAcrossTenants()`. The design work is ensuring every memory and reflection is tagged with participant entity IDs at creation time, so erasure queries find everything. No new API needed — the tagging discipline is the architectural requirement.
**Trade-offs:** Over-erasure of mixed-user reflections removes insights about other users that happened to be in the same conversation. Acceptable — privacy takes precedence over memory completeness.
**Exploration:** implicit (surfaced by R1-21)
**Depends on:** D5, D14, D20
**Status:** captured

## D25: Multi-bot, single JVM deployment model

**Choice:** Multiple Discord bot accounts running in a single JVM process. Each character has its own bot token, Gateway WebSocket connection, Discord user identity, and full agent stack. Characters share the JVM for LLM queue management, memory stores, and resource efficiency. Each character appears as a separate Discord user with its own username and avatar.
**Alternatives:**
- Multi-bot, multi-process — one JVM per character. Zero code changes but loses shared resource management (LLM rate limiting, memory store connections, cross-character pacing coordination)
- Single bot, channel-scoped personalities — one bot account switches personality per channel. Characters can't interact (one bot = one Discord identity), the illusion breaks visibly
**Rationale:** Multi-bot single JVM gives the best balance: characters are fully independent Discord identities (separate usernames, avatars, message attribution) while sharing expensive resources (LLM API connections, memory store connections). Characters see each other's messages naturally through Discord — no special cross-character wiring needed. WackyManor demonstrates this pattern: shared CDI services + agentId-scoped operations = multi-character in one process.
**Trade-offs:** Requires multiple Discord bot tokens and Developer Portal setups. More Gateway WebSocket connections per process. Each bot must be invited to the server separately.
**Sources:** WackyManor ScenarioOrchestrator (shared services, per-agentId scoping), Discord API (one token = one Gateway session)
**Exploration:** quick
**Depends on:** D9
**Status:** captured

## D26: CharacterContext class + stateless ChatAgencyLoop

**Choice:** Extract all per-character mutable state from `ChatAgencyLoop` into a `CharacterContext` class (not a record — has mutable fields). `ChatAgencyLoop` becomes stateless — `tick()` reads character identity, system prompt, need state, and other per-character state from the `CharacterContext` stored in `AgencyContext`. `ChatCharacterManager` (CDI `@ApplicationScoped`) holds a `Map<String, CharacterContext>` and orchestrates all characters.

**CharacterContext holds:**
- `agentId`, `tenantId`, `systemPrompt`
- `Supplier<AgentDescriptor>` (per-character descriptor from registry)
- `BotIdentityDetector` (per-character Discord user ID)
- `NeedState` (per-character needs — SOCIAL, CURIOSITY, EXPRESSION)
- `IdleReflectionTrigger` (per-character reflection state)
- `Set<String> participatedThreadIds` (per-character thread tracking)
- `int consecutiveIdleTicks`, `Instant lastReflectionTimestamp`

**ChatCharacterManager responsibilities:**
- On startup: read all `AgentDescriptor` entries from `AgentRegistry` with the configured tenancyId, create a `CharacterContext` per descriptor, wire each to its Discord bot token from config
- Per tick: set the appropriate `CharacterContext` into `AgencyContext` before calling `tick()`
- Hold the shared `OutputGovernor` (D27) for cross-character pacing

**Alternatives:**
- N independent ChatAgencyLoop instances — keeps mutable per-character state in the loop, duplicates shared service references, approach B territory
- Central tick loop iterating over characters — doesn't fit event-driven Discord model where each bot gets events independently
**Rationale:** Mirrors WackyManor's pattern where `ScenarioOrchestrator` manages N characters, each identified by `agentId`, using shared CDI services with agentId-scoped operations. Separating character state from loop logic makes the loop reusable and testable — one loop instance serves all characters. The `ChatCharacterManager` is the quarkmind-chat equivalent of WackyManor's `ScenarioOrchestrator`.
**Trade-offs:** Requires refactoring ChatAgencyLoop to remove instance fields (`agentId`, `tenantId`, `systemPrompt`, `consecutiveIdleTicks`, `lastReflectionTimestamp`, `participatedThreadIds`). All tests that set these fields need updating.
**Exploration:** quick
**Depends on:** D25
**Status:** captured

## D27: Shared OutputGovernor per server for cross-character pacing

**Choice:** One `OutputGovernor` instance per Discord server (scoped by tenantId), shared across all characters in that server. Enforces a global minimum interval between any two character actions in the same server. Character A posts → Character B must wait the minimum interval before posting. Individual character pacing (`ChatChannelPacing`) still applies per-character on top.
**Alternatives:**
- Turn-taking coordinator — assigns "who speaks next" based on priority and fairness. More sophisticated but over-constrains natural conversation flow and adds significant complexity
- No cross-character pacing — characters might respond simultaneously, creating awkward double-posts. Simplest but worst UX
**Rationale:** The shared governor prevents the worst multi-character UX problem (simultaneous responses) with minimal complexity. It's a simple extension of the existing `OutputGovernor` — instead of per-character, it's per-server. The per-character `ChatChannelPacing` layer still handles individual character behavior within the global constraint.
**Trade-offs:** Characters may be delayed when another character just posted. Acceptable — a brief delay feels natural; simultaneous posting feels broken.
**Exploration:** quick
**Depends on:** D25, D26
**Status:** captured

## D28: Per-character DiscordEventSource — own Gateway connection

**Choice:** Each `CharacterContext` gets its own `DiscordEventSource` wrapping its own `DiscordInboundConnector` (its own bot token → its own Gateway WebSocket). `ChatCharacterManager` creates all connections on startup. Each character's Gateway events drive its own agency tick independently. This is the natural extension of the current single-character model.
**Alternatives:**
- Shared connector with filtered events — not possible with Discord. Each bot token requires its own Gateway session. One WebSocket per bot is a Discord API requirement, not a design choice.
**Rationale:** Discord's API model requires one Gateway session per bot token. Each character IS a separate bot. The current single-character model already has one DiscordEventSource — multi-character simply creates N of them, each with its own bot token.
**Trade-offs:** N Gateway WebSocket connections per JVM. For 2-3 characters this is negligible. At scale (10+ characters), connection management becomes a concern, but that's beyond v2 scope.
**Exploration:** quick
**Depends on:** D25
**Status:** captured

## D29: Multi-descriptor YAML + per-character bot tokens in config

**Choice:** One `descriptors.yaml` with N `AgentDescriptor` entries (already supported by `ClasspathYamlDescriptorRegistrar`). Each descriptor's `agentId` maps to a bot token in `application.yaml` via Quarkus config:

```yaml
quarkmind:
  chat:
    characters:
      luna:
        token: ${DISCORD_TOKEN_LUNA}
        channels: general,lounge
      rex:
        token: ${DISCORD_TOKEN_REX}
        channels: general,workshop
```

`ChatCharacterManager` reads all descriptors from the registry, matches each `agentId` to its config block, and creates the character stack. A character without a matching token config is skipped with a warning.
**Alternatives:**
- Separate config directory per character — more filesystem structure to manage, no clear benefit
- Database-driven registration — hot-reloadable but adds persistence dependency beyond v2 scope
**Rationale:** Extends D12's configuration-driven deployment naturally. `ClasspathYamlDescriptorRegistrar` already loads multiple descriptors from one YAML file. The per-character config block is standard Quarkus `@ConfigMapping` pattern. No new infrastructure needed.
**Trade-offs:** Adding a character requires: (1) add descriptor to YAML, (2) add config block with bot token, (3) invite the bot to the Discord server. Three steps, all configuration.
**Exploration:** quick
**Depends on:** D12, D25
**Status:** captured

## D30: CharacterContext in AgencyContext map

**Choice:** `CharacterContext` is placed into the `AgencyContext` map via `context.put("character", characterContext)` before each `tick()` call. `ChatAgencyLoop.tick()` reads it with `context.getAs("character", CharacterContext.class)`. The `AgencyLoop.tick(AgencyContext)` interface signature stays unchanged. `ChatCharacterManager` sets the right `CharacterContext` before each character's tick.
**Alternatives:**
- CharacterContext as constructor param — creates N loop instances (approach B), duplicates shared service references
- Direct parameter on tick() — breaks `AgencyLoop` interface, requires quarkmind-core SPI change
**Rationale:** `AgencyContext` is already a heterogeneous map (`put`/`getAs` pattern). Adding `CharacterContext` follows the established pattern — perception is already passed this way (`context.put("perception", perception)`). No interface changes, no quarkmind-core SPI changes. The manager is responsible for populating the right context before each tick.
**Trade-offs:** Runtime type safety — `getAs` returns null if the key is missing or the type doesn't match. Acceptable — the manager always sets it, and a null check at the top of `tick()` guards against misconfiguration.
**Exploration:** quick
**Depends on:** D26
**Status:** captured
