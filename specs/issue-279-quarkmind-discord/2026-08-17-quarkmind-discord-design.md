# QuarkMind Chat — Advanced AI Chat Bot Harness

**Date:** 2026-08-17
**Status:** Draft
**Issue:** #279
**Decisions:** See `decisions.md` (D1–D10, D11)

## Overview

A harness for building advanced AI chat bots — autonomous characters that live in chat platforms. Not command bots — characters with personality, moods, opinions, and their own social lives. They remember people, form relationships, and grow over time.

Built on the casehub-connectors `ChatPlatform` SPI, the harness works with any supported chat platform (Discord, Slack, IRC). Discord is the first target. The platform-specific code is a thin adapter layer; the agency loop, personality, memory, needs, and rate governing are platform-agnostic.

The harness uses quarkmind-core's agency framework over existing CaseHub foundation modules: casehub-eidos (personality), casehub-neocortex (memory), casehub-ledger (trust), casehub-connectors (chat transport), and casehub-blocks (event handling + compression).

## Module Structure

Two sub-modules under `quarkmind-chat/` (D1, D11):

```
quarkmind-chat/
├── pom.xml                           ← parent POM
├── quarkmind-chat-protocol/          ← shared message types (platform-agnostic)
│   src/main/java/io/quarkmind/chat/protocol/
│     ChatPerception.java             ← conversation delta + presence
│     ChatIntent.java                 ← sealed: Send | Reply | React
│
└── quarkmind-chat-agent/             ← agency loop + Quarkus app
    src/main/java/io/quarkmind/chat/agent/
      ChatWorldBridge.java            ← WorldBridge<ChatPerception, ChatIntent>
      ChatAgencyLoop.java             ← AgencyLoop — direct LLM, no CaseEngine
      ChatCharacterApp.java           ← Quarkus application entry point
      ChatNeedDefinitions.java        ← SOCIAL, CURIOSITY, EXPRESSION need config
      ChatChannelPacing.java          ← channel-aware rate pacing (layer 3)
    src/main/java/io/quarkmind/chat/agent/discord/
      DiscordEventSource.java         ← adapts DiscordInboundConnector as EventSource
      DiscordGatewayMessageHistory.java ← Gateway-accumulating MessageHistory impl
      DiscordIdentityDetector.java    ← @mention parsing, bot user ID detection
```

Platform-specific adapters live in a sub-package (`discord/`, `slack/`, `irc/`). The core agent classes are platform-agnostic — they depend on the `ChatPlatform` SPI, not on any specific connector implementation.

Dependencies flow one way: `quarkmind-chat-protocol` has no runtime dependencies (plain Java records). `quarkmind-chat-agent` depends on protocol, quarkmind-core, casehub-connectors (chat-spi + platform-specific connector via profile), casehub-blocks, casehub-eidos-api, casehub-neocortex-api, casehub-ledger-api.

### quarkmind-core additions

New types added to quarkmind-core for chat-agent support:

```
io.quarkmind.agency.chat/
  ChatDeltaReport.java              ← structured conversation delta: threads, new vs continuing, attention priority
  ChatPerceptionBridge.java         ← SPI: ChatDeltaReport → WorldPerception translation
  AttentionClassifier.java          ← classifies messages: DIRECT (mentions/replies) | ELEVATED (active threads) | AMBIENT
  ChatObservationRenderer.java      ← configures TieredObservationRenderer for chat messages

io.quarkmind.agency.schedule/
  NeedThresholdWake.java            ← evaluates NeedState, emits EventSource signal when needs cross thresholds
  IdleReflectionTrigger.java        ← fires reflection when accumulated importance exceeds threshold during idle
  OutputGovernor.java               ← global rate limiter: max actions per window, minimum interval

io.quarkmind.agency.personality/
  ReflectionDispositionActivator.java ← LLM-based: classifies which disposition function term a reflection activates
  ProactiveDecisionGate.java        ← evaluates "should I act?" given social context (time since last post, channel activity)
```

`ChatDeltaReport`, `AttentionClassifier`, and `ChatObservationRenderer` are chat-world abstractions — reusable by any quarkmind world built on a chat platform. `NeedThresholdWake`, `IdleReflectionTrigger`, `OutputGovernor`, `ReflectionDispositionActivator`, and `ProactiveDecisionGate` are general agency abstractions usable by any quarkmind world.

## Execution Model

**Push-wake, pull-context** (D2):

The agent wakes on two triggers, combined via blocks' existing `ChoreographedDriver`:

```java
new ChoreographedDriver(invoker, policy,
    platformEventSource,                         // push: platform-specific event source
    EventSource.ticker(heartbeatInterval, exec)  // timer: need evaluation
);
```

On wake:
1. `DriverEvent.source()` distinguishes message wake from timer wake
2. Agent calls `messageHistory.messages(channel, lastChecked)` per watched channel — structured delta with threading via `parentRef`
3. quarkmind-core conversation delta report reconstructs threads, classifies attention priority
4. If volume exceeds threshold, blocks `TieredObservationRenderer` compresses ambient messages while preserving elevated items verbatim
5. LLM processes context with personality system prompt
6. Agent emits intents (or stays quiet)

**MessageHistory implementation (Discord):** The existing `DiscordChatPlatform.getMessageHistory()` calls the REST API directly. `DiscordGatewayMessageHistory` wraps `DiscordInboundConnector`'s Gateway events into a local message buffer with REST API fallback for cold start and detected gaps. Other platforms use their own `MessageHistory` implementations (Slack has event subscription, IRC has the read loop).

## Perception

**Structured conversation delta** (D2 revised):

`ChatPerception` wraps:
- `Map<String, List<ReceivedMessage>> channelDeltas` — messages per channel since last check, threaded via `parentRef`
- `Map<String, PresenceStatus> presenceChanges` — who came online/offline
- `WakeReason reason` — MESSAGE or HEARTBEAT (derived from `DriverEvent.source()`)

`NeedState` is agent-internal state held by `AgencyContext`, not world perception. Needs are consulted during the agency loop's reasoning phase, not delivered as perception.

**Attention priority classification** (D3):
- **Direct address** — replies to bot, @mentions → always verbatim, never compressed
- **Active threads** — conversations the bot previously participated in → elevated
- **Ambient** — general channel chatter → compressible when volume is high

Bot identity detection (platform-specific @mention syntax, bot user ID) lives in the platform adapter sub-package, not in quarkmind-core. `AttentionClassifier` accepts a `BotIdentityDetector` SPI for platform-specific detection.

## Intent Model

**Sealed interface** with three variants (D4):

```java
sealed interface ChatIntent extends Intent {
    record Send(String channelId, ChatContent content) implements ChatIntent {}
    record Reply(ChatMessageRef parent, ChatContent content) implements ChatIntent {}
    record React(ChatMessageRef message, String emoji) implements ChatIntent {}
}
```

`ChatContent` supports markdown, attachments, and `RichCard` embeds. Validation (platform-specific limits) happens at dispatch through the chat SPI — the agent is free to express intent without pre-validating platform constraints.

Dispatch maps directly: `Send` → `Messaging.send()`, `Reply` → `Threading.reply()`, `React` → `Reactions.add()`.

## Agency Loop

**Direct LLM, no CaseEngine** (D8):

Follows QuarkVille's `VilleAgencyLoop` pattern — `AgencyLoop.tick(AgencyContext)` with direct LLM invocation for reasoning. A chat character is a single agent making holistic decisions; CaseEngine's multi-plugin blackboard dispatch is unnecessary.

**LLM invocation:** All LLM calls (agency reasoning, importance scoring, reflection, disposition activation) go through quarkmind-core's `LlmRequestQueue` for rate limiting, prioritisation, and token budgeting. Agency reasoning calls are `LlmPriority.HIGH`; importance scoring and reflection are `LlmPriority.LOW` (async, non-blocking). This ensures a single shared queue — no competing callers.

**Memory retrieval:** Uses composite retrieval (Park et al. model) via neocortex's existing stores: `TemporalDecayCbrCaseMemoryStore` for recency-weighted episodic recall, `PersonalityWeightedRetrieval` for disposition-aware filtering, `MoodModulatedRetrieval` for emotional-state-influenced recall. Relationship context retrieved from `GraphitiCaseMemoryStore` by person entity.

The loop per tick:
1. Read conversation delta (perception)
2. Retrieve relevant memories — episodic via composite retrieval, relationship via Graphiti person lookup
3. Evaluate needs (SOCIAL, CURIOSITY, EXPRESSION)
4. Compose LLM prompt: system prompt (Eidos personality) + conversation context + memories + needs + available actions
5. Submit to `LlmRequestQueue` at HIGH priority, parse response into intents
6. Apply output governor (D7, D10) — rate limit, social pacing
7. Dispatch surviving intents via `WorldBridge.dispatch()`
8. Ingest experience into memory (importance scoring submitted to `LlmRequestQueue` at LOW priority, async)

## Memory Architecture

**Four layers, existing foundation** (D5):

| Layer | Purpose | Foundation module |
|-------|---------|-------------------|
| Working memory | Current conversation context | LLM context window |
| Episodic memory | Conversation cases | `CbrCaseMemoryStore` + `TemporalDecayCbrCaseMemoryStore` |
| Semantic memory | Reflections / insights | `ReflectionService` → high-importance CBR cases |
| Relationship memory | Per-person profiles | `GraphitiCaseMemoryStore` + `casehub-ledger` |

**Three capability additions required:**

1. **Importance scoring at ingest** (neocortex enhancement) — LLM-rated significance 1–10 at memory creation. Runs asynchronously — memory creation returns immediately; score attached when LLM call completes.

2. **Idle-time reflection trigger** (quarkmind-core) — scheduler fires when accumulated importance exceeds threshold during idle periods. Drives both memory consolidation (episodic → semantic) and personality growth (D6 Layer 3).

3. **Relationship schema** (Graphiti) — per-person model: trust score (from ledger), interaction count, opinion summary, shared topics, last interaction timestamp. Graphiti is the default for entity-relationship graphs; Mem0 is an alternative if Graphiti's entity extraction proves insufficient.

## Personality System

**Three layers** (D6):

**Layer 1 — Prompt-driven (Eidos → LLM):** `AgentDescriptor` defines character via disposition axes, style profile, briefing, goals, constraints. `SystemPromptRenderer` renders into system prompt.

**Layer 2 — Mechanical (quarkmind-core NeedState):** Dispositions affect behavior through `DispositionNeedModifier`:
- SOCIAL_ORIENTATION → social need decay rate
- AUTONOMY → heartbeat frequency for unprompted action
- RISK_APPETITE → engagement willingness with strangers

**Layer 3 — Growth (reflection → disposition evolution):** The full pipeline:

```
Chat interactions
  → episodic memory (ingest)
  → importance accumulates
  → idle-time reflection trigger fires
  → ReflectionService produces insights
  → ReflectionDispositionActivator classifies which disposition function term is activated
  → DispositionSignalStore.recordActivation()
  → DispositionHealth.probe() checks threshold crossings
  → DispositionEvolution.evaluate() → Evolved(newProfile) or Dampened(decayFactor)
```

This is the first consumer of the full end-to-end chain. Each link uses existing foundation types; the integration is new.

**ReflectionDispositionActivator classification**: LLM-based. Receives a reflection summary and the agent's `dispositionProfile()` terms. Submits an LLM request (via `LlmRequestQueue` at LOW priority) asking: "Given this reflection and these disposition function terms, which term does this reflection most strongly activate?" Returns a single function term. Classification is async and best-effort — a failed LLM call means no activation recorded for that reflection, not a system error.

## Need System

**Three needs** (D7):

**SOCIAL** — decays when idle, satisfied by conversation. Three sub-drives (prompt-level, not mechanical):
1. Learn about people — understand who they are, what they care about
2. Help people — look for opportunities to be useful
3. Meaningful contribution — add value to the group (connect people, remember things)

Boredom (mentioned in issue #279) maps to low SOCIAL + low CURIOSITY — the character has nothing to engage with and nothing interesting to explore. The need system generates engagement pressure; the sub-drives shape how it responds.

**CURIOSITY** — decays slowly, spikes on interesting topics. Drives questions and follow-ups.

**EXPRESSION** — builds over time, satisfied by sharing opinions/reactions. Drives unprompted posting and emoji reactions.

## Rate Governing

**Three independent layers** (D7, D10):

1. **Global output governor** (quarkmind-core) — max actions per time window. Default: 1 unprompted post per 5 minutes, hard minimum 30-second interval.

2. **Proactive decision gate** (quarkmind-core) — evaluates social context before need-driven actions: time since last post, channel activity, typing indicators, conversational flow.

3. **Channel-aware pacing** (quarkmind-chat) — frequency scales with channel activity. Quiet channel → fill silence; busy channel → wait for gaps.

Need threshold crossing is necessary but not sufficient for action. Multiple simultaneous threshold crossings produce a single prioritized action.

**API rate limiting** (D10) is the connector's responsibility — each `ChatPlatform` implementation tracks its platform's rate limit headers and applies back-pressure. Behavioral pacing is the binding constraint in normal operation.

## Configuration-Driven Deployment

**No Java code required for new characters** (D12). A deployment needs:

1. **Character YAML** — Eidos `AgentDescriptor` loaded by `ClasspathYamlDescriptorRegistrar`. Defines personality (disposition axes, style, briefing, goals, constraints).

2. **application.yaml** — Quarkus config for everything else:

```yaml
# Platform (profile-scoped)
%discord.casehub.discord.token: ${DISCORD_BOT_TOKEN}
%discord.quarkmind.chat.channels: general,random,tech

# Needs
quarkmind.chat.needs.social.decay-rate: 0.5
quarkmind.chat.needs.social.satisfaction-threshold: 30
quarkmind.chat.needs.curiosity.decay-rate: 0.1
quarkmind.chat.needs.expression.decay-rate: 0.2

# Rate governing
quarkmind.chat.governor.max-unprompted-per-window: 1
quarkmind.chat.governor.window-seconds: 300
quarkmind.chat.governor.min-interval-seconds: 30

# Heartbeat
quarkmind.chat.heartbeat-seconds: 60

# LLM
quarkmind.chat.llm.model: claude-sonnet-5
```

All tuning knobs externalized as config properties. No constants.

**Personality generator** (future): LLM-powered wizard that helps non-technical users create character YAML descriptors. Out of v1 scope — filed as a separate issue.

## Scope

**v1: Single character per server** (D9). Multi-character scenarios (multiple characters interacting, per-channel profiles) are explicitly out of scope.

**v1: Discord as first platform.** The harness is platform-agnostic; Discord is the validation target. Slack and IRC adapters can be added with minimal code — they implement `EventSource` and `BotIdentityDetector` for their platform, and configure the appropriate `ChatPlatform` bean.

## Layering Summary

**Two-tier** (D3 revised):

| Layer | Owns | Uses |
|-------|------|------|
| **quarkmind-core** | Chat perception bridge, conversation delta report, attention priority classification, need-threshold wake, proactive decision gate, idle-time reflection trigger, output governor | blocks (TieredObservationRenderer, EventSource, ChoreographedDriver), eidos-api, neocortex-api, ledger-api |
| **quarkmind-chat** | ChatWorldBridge, ChatAgencyLoop, platform adapters (Discord/Slack/IRC event sources, message history, identity detection), channel-aware pacing, intent dispatch | quarkmind-core, connectors (chat-spi + platform connector), quarkmind-chat-protocol |

Chat-agent abstractions (thread reconstruction, attention priority, conversation delta) live in quarkmind-core because they generalize to any chat-based world. These may not generalize to non-chat worlds (SC2, Godot) — quarkmind-core accepts this specialization as the cost of reuse across chat worlds.

Blocks receives **no new types** — existing `TieredObservationRenderer<E>`, `EventSource`, `ChoreographedDriver` are used directly with custom renderers and event sources.

## Testing Strategy

Tests run without a live chat platform connection (parallel to SC2's "tests run without a live SC2 binary"):

- **Unit tests** (plain JUnit): `ChatAgencyLoopTest`, `AttentionClassifierTest`, `ChatDeltaReportTest`, `OutputGovernorTest`, `ProactiveDecisionGateTest`, `NeedThresholdWakeTest`. Mock `ChatPlatform`, mock `LlmRequestQueue`.
- **Integration tests** (`@QuarkusTest`): full agency loop with in-memory `ChatPlatform` (ref implementation from connectors), in-memory neocortex stores, mock LLM. Verify: perception → reasoning → intent dispatch cycle.
- **Personality evolution tests**: accelerated time — inject high-importance memories, trigger reflection, verify disposition activation and evolution. Weeks of interaction compressed to seconds.

## Cross-Repo Work

| Repo | What changes |
|------|-------------|
| `casehub-blocks` | Nothing — existing primitives used directly |
| `casehub-connectors` | Nothing for v1 — Gateway-accumulating MessageHistory lives in quarkmind-chat |
| `casehub-neocortex` | Importance scoring at ingest (async LLM rating) |
| `quarkmind` (quarkmind-core) | Chat perception bridge, conversation delta report, attention priority, need-threshold wake, proactive decision gate, idle-time reflection trigger, output governor, ReflectionDispositionActivator |
| `quarkmind` (quarkmind-chat) | New module: protocol + agent + Discord adapter |

## References

- `casehub-connectors` chat-spi — `ChatPlatform.java`, `MessageHistory`, `Threading`, `Messaging`, `Reactions`, `Presence`, `Members`, `ReceivedMessage`
- `casehub-connectors` chat-discord — `DiscordChatPlatform.java`, `DiscordInboundConnector.java`
- `casehub-connectors` discord — `DiscordClient.java`, `DiscordGateway.java`, `DiscordGatewayPresenceCache`
- `casehub-blocks` — `TieredObservationRenderer`, `EventSource`, `ChoreographedDriver`, `PartitionedObservationService`
- `casehub-eidos` — `AgentDescriptor`, `AgentDisposition`, `DispositionAxis`, `DispositionSignalStore`, `DispositionHealth`, `DispositionEvolution`, `SystemPromptRenderer`
- `casehub-neocortex` — `CbrCaseMemoryStore`, `TemporalDecayCbrCaseMemoryStore`, `ReflectionService`, `GraphitiCaseMemoryStore`, `Mem0CaseMemoryStore`, `MoodModulatedRetrieval`, `PersonalityWeightedRetrieval`
- `casehub-ledger` — Bayesian beta trust scoring
- `quarkmind-core` — `AgencyLoop`, `AgencyContext`, `WorldBridge`, `NeedState`, `DispositionNeedModifier`, `IntentQueue`, `LlmRequestQueue`
- `quarkmind-ville` — `VilleAgencyLoop`, `VilleWorldBridge` (reference pattern)
- `wacky-manor` — `CharacterAgentLoop`, `ObservationService`, `ManorLlmSummariser` (reference pattern)
- Park et al. 2023 "Generative Agents" — three-tier memory model (episodic/semantic/procedural)
- MemGPT/Letta — tiered memory with self-directed consolidation
- Mem0 — graph memory with entity extraction and conflict detection
- CoALA — cognitive architecture framework for LLM agents
