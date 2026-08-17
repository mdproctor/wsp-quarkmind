# QuarkMind Discord — Decisions

## D1: Two sub-modules (protocol + agent)

**Choice:** `quarkmind-discord-protocol/` (shared types: DiscordPerception, DiscordIntent) + `quarkmind-discord-agent/` (agency loop, world bridge, Quarkus app).
**Alternatives:**
- Flat single module — simpler, but protocol types become entangled with agent runtime dependencies
**Rationale:** Two-module split stands on its own merits: (1) protocol types are shared contracts — a future dashboard or observer tool can depend on protocol without pulling in the agent's Quarkus runtime, (2) clear boundary between message types and application logic. QuarkVille uses three modules (protocol + server + agent) because it has a central game server; Discord has two because Discord IS the server — the structural difference reflects different architectural constraints, not inconsistency. The three other world modules (minecraft, evennia, sonaria) are stubs — there is no established multi-world pattern to claim consistency with; the two-module structure is justified by Discord's own constraints.
**Trade-offs:** Slightly more Maven boilerplate for a module that may never have a third consumer.
**Exploration:** quick
**Status:** revised (R1-02: rationale rewritten to stand on own merits; removed misleading "mirrors QuarkVille" claim)

## D2: Event-driven execution with chat SPI delta + blocks compression

**Choice:** Agent wakes on two triggers via existing blocks primitives: `new ChoreographedDriver(invoker, policy, discordEventSource, EventSource.ticker(heartbeat, executor))`. **Pattern: push-wake, pull-context** — Discord Gateway WebSocket push wakes the agent (via `DiscordInboundConnector` adapted as an `EventSource`); `MessageHistory.messages(channel, since)` provides structured conversation context with threading via `parentRef`. The `MessageHistory` implementation accumulates Gateway events locally, falling back to REST API only on cold start or detected gaps — avoids redundant REST calls and rate limit pressure. `DriverEvent.source` distinguishes timer wakes ("timer") from message wakes. Blocks `TieredObservationRenderer` acts as compression layer when volume exceeds thresholds.
**Alternatives:**
- Blocks observation pipeline as primary perception (Wacky Manor pattern) — rebuilds what chat SPI already provides
- Dual-loop (reactive + autonomous) — more complex, two loops sharing mutable state
- Single tick loop with fixed interval — adds unnecessary latency to message responses
- Gateway-only accumulation (no REST pull) — simpler but requires robust local gap detection; REST fallback still needed for cold start
**Rationale:** Push-wake, pull-context is a well-established pattern in distributed systems. The chat SPI's `MessageHistory` abstraction decouples the perception contract from the transport. The existing `DiscordChatPlatform.getMessageHistory()` calls the REST API directly; a new Gateway-accumulating `MessageHistory` implementation is needed (either in quarkmind-discord wrapping the `DiscordInboundConnector`'s Gateway events, or as a connectors enhancement). This is new implementation work, not existing infrastructure. Blocks' `ChoreographedDriver` with `EventSource` varargs already provides the dual-wake combiner — no new blocks abstractions needed (EventSource.merge handles combination, DriverEvent.source handles discrimination).
**Trade-offs:** Depends on both casehub-connectors (chat SPI) and blocks (compression). Agent response time depends on LLM latency, not a fixed tick rate.
**Sources:** `casehub-connectors` chat-spi MessageHistory, ReceivedMessage, Threading; `casehub-blocks` TieredObservationRenderer, EventSource, ChoreographedDriver, DriverEvent
**Exploration:** quick
**Depends on:** D1
**Status:** revised (R1-03: named push-wake/pull-context pattern, clarified MessageHistory backed by Gateway accumulation; R1-04: referenced existing EventSource.merge + ticker — no new blocks abstractions needed; R2-01: acknowledged Gateway-accumulating MessageHistory is new implementation work)

## D3: Two-tier layering — quarkmind-core / quarkmind-discord

**Choice:** Chat-agent abstractions into quarkmind-core, world-specific code into quarkmind-discord. No blocks additions needed — existing blocks primitives (`TieredObservationRenderer`, `EventSource`, `ChoreographedDriver`) are used directly.

**blocks additions:** None. Existing `TieredObservationRenderer<E>` with custom `Function<E, String> eventRenderer`, `Function<E, String> groupKeyExtractor`, and `Summariser<E, String>` provides compression. `ChoreographedDriver` with `EventSource.merge()` provides dual-wake. No new blocks types or chat-domain dependencies.

**quarkmind-core additions:**
- Chat perception bridge — contract for how a conversation delta becomes a `WorldPerception`. Wires chat SPI + blocks compression into the agency loop.
- Conversation-aware delta report — takes messages from any chat source, reconstructs thread structure from `parentRef`, classifies new vs continuing conversations, provides pre-delta context. General capability for any chat-based quarkmind world.
- Attention-priority classification — messages classified by relevance: direct address (replies to bot, @mentions) → always verbatim; active threads (bot participated) → elevated; ambient → compressible. The classification produces priority tags that drive `TieredObservationRenderer` grouping/summarisation.
- Need-threshold wake — scheduler evaluates `NeedState`, fires proactive-wake event via `EventSource` when needs cross thresholds.
- Proactive decision gate — agent-level logic for "should I actually act or stay quiet?" Considers time since last post, channel activity, and social context.

**quarkmind-discord (world-specific):**
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
**Status:** revised (R1-05: moved chat-domain logic from blocks to quarkmind-core — blocks stays domain-agnostic; R1-06: dropped WakeSource — EventSource.merge suffices; original three-tier became two-tier)

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
- Importance scoring at ingest in neocortex — LLM-rated significance 1–10 at memory creation time (Park et al. retrieval composite = recency + importance + relevance; neocortex has recency and relevance but not importance). Importance scoring runs asynchronously — memory creation returns immediately; the score is attached when the LLM call completes. Not on the hot path for response latency.
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
**Status:** revised (R1-08: scope characterization corrected — "small gaps" → "three capability additions" with honest complexity acknowledgment; importance scoring confirmed async)

## D6: Personality — Eidos prompt + quarkmind-core mechanical needs + reflection-driven growth

**Choice:** Three-layer personality system using existing foundation:

**Layer 1 — Prompt-driven (Eidos → LLM):** `AgentDescriptor` defines character personality via disposition axes (SOCIAL_ORIENTATION, RULE_FOLLOWING, RISK_APPETITE, AUTONOMY, CONFLICT_MODE), `styleProfile`, `dispositionProfile`, `briefing`, `goals`, and `constraints`. `SystemPromptRenderer` renders this into the LLM system prompt.

**Layer 2 — Mechanical (quarkmind-core NeedState):** Dispositions affect non-LLM behavior via `DispositionNeedModifier`:
- SOCIAL_ORIENTATION → social need decay rate (extrovert drops fast when alone)
- AUTONOMY → heartbeat frequency for unprompted action
- RISK_APPETITE → willingness to engage with strangers/controversial topics

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
**Status:** revised (R1-09: reflection→BehavioralSignal mapping was wrong — BehavioralSignal is capability health, not personality evolution; R2 corrected to use DispositionSignalStore.recordActivation → DispositionHealth.probe → DispositionEvolution.evaluate pipeline; R1-20: D5 dependency upgraded to CRITICAL)

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

**Alternatives:**
- More granular needs (separate HELP, LEARN, CONTRIBUTE) — over-specified; the LLM handles strategy selection within the social drive better than mechanical sub-needs
- Fewer needs (SOCIAL only) — loses the distinction between wanting to talk (social), wanting to know (curiosity), and wanting to share (expression)
**Rationale:** Three needs are enough to produce varied autonomous behavior. The sub-drives under SOCIAL give the LLM guidance on *how* to be social without mechanizing strategy selection — the character might satisfy its social need by helping someone one moment and by asking a question the next, depending on context. Rate governing ensures the character's behavior is socially appropriate regardless of need pressure.
**Trade-offs:** Sub-drives are prompt-level guidance, not mechanical — they don't have independent decay rates. If finer control is needed later, they can be promoted to full needs.
**Exploration:** quick
**Depends on:** D6
**Status:** revised (R1-10: rate-governing mechanisms added — global governor, proactive decision gate, channel-aware pacing; R1-14: Discord-specific rate constraints addressed)

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
