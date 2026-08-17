# QuarkMind Discord — Persistent Character Agent

**Date:** 2026-08-17
**Status:** Draft
**Issue:** #279
**Decisions:** See `decisions.md` (D1–D10)

## Overview

An autonomous AI character that lives in a Discord server. Not a command bot — a character with personality, moods, opinions, and its own social life. It remembers people, forms relationships, and grows over time.

The character uses quarkmind-core's agency framework over existing CaseHub foundation modules: casehub-eidos (personality), casehub-neocortex (memory), casehub-ledger (trust), casehub-connectors (Discord transport), and casehub-blocks (event handling + compression).

## Module Structure

Two sub-modules under `quarkmind-discord/` (D1):

```
quarkmind-discord/
├── pom.xml                           ← parent POM
├── quarkmind-discord-protocol/       ← shared message types
│   src/main/java/io/quarkmind/discord/protocol/
│     DiscordPerception.java          ← conversation delta + presence + needs
│     DiscordIntent.java              ← sealed: Send | Reply | React
│     DiscordEvent.java               ← gateway event envelope
│
└── quarkmind-discord-agent/          ← agency loop + Quarkus app
    src/main/java/io/quarkmind/discord/agent/
      DiscordWorldBridge.java         ← WorldBridge<DiscordPerception, DiscordIntent>
      DiscordAgencyLoop.java          ← AgencyLoop — direct LLM, no CaseEngine
      DiscordEventSource.java         ← adapts DiscordInboundConnector as EventSource
      DiscordCharacterApp.java        ← Quarkus application entry point
      DiscordNeedDefinitions.java     ← SOCIAL, CURIOSITY, EXPRESSION need config
      DiscordOutputGovernor.java      ← rate limiting + social pacing
```

Dependencies flow one way: `quarkmind-discord-protocol` has no runtime dependencies (plain Java records). `quarkmind-discord-agent` depends on protocol, quarkmind-core, casehub-connectors (chat-discord + discord), casehub-blocks, casehub-eidos-api, casehub-neocortex-api, casehub-ledger-api.

## Execution Model

**Push-wake, pull-context** (D2):

The agent wakes on two triggers, combined via blocks' existing `ChoreographedDriver`:

```java
new ChoreographedDriver(invoker, policy,
    discordEventSource,                          // push: Gateway MESSAGE_CREATE
    EventSource.ticker(heartbeatInterval, exec)  // timer: need evaluation
);
```

On wake:
1. `DriverEvent.source()` distinguishes message wake ("discord") from timer wake ("timer")
2. Agent calls `messageHistory.messages(channel, lastChecked)` per watched channel — structured delta with threading via `parentRef`
3. quarkmind-core conversation delta report reconstructs threads, classifies attention priority
4. If volume exceeds threshold, blocks `TieredObservationRenderer` compresses ambient messages while preserving elevated items verbatim
5. LLM processes context with personality system prompt
6. Agent emits intents (or stays quiet)

**MessageHistory implementation:** The existing `DiscordChatPlatform.getMessageHistory()` calls the REST API directly. A new Gateway-accumulating implementation is needed — either in quarkmind-discord (wrapping `DiscordInboundConnector` events) or as a connectors enhancement. REST fallback for cold start and gap detection.

## Perception

**Structured conversation delta** (D2 revised):

`DiscordPerception` wraps:
- `Map<String, List<ReceivedMessage>> channelDeltas` — messages per channel since last check, threaded via `parentRef`
- `Map<String, PresenceStatus> presenceChanges` — who came online/offline
- `NeedState currentNeeds` — agent's current need levels
- `WakeReason reason` — MESSAGE or HEARTBEAT

**Attention priority classification** (D3):
- **Direct address** — replies to bot, @mentions → always verbatim, never compressed
- **Active threads** — conversations the bot previously participated in → elevated
- **Ambient** — general channel chatter → compressible when volume is high

Bot identity detection (@mention parsing, reply-to-self detection) requires knowing the bot's Discord user ID — this is Discord-specific logic in quarkmind-discord, not quarkmind-core.

## Intent Model

**Sealed interface** with three variants (D4):

```java
sealed interface DiscordIntent extends Intent {
    record Send(String channelId, ChatContent content) implements DiscordIntent {}
    record Reply(ChatMessageRef parent, ChatContent content) implements DiscordIntent {}
    record React(ChatMessageRef message, String emoji) implements DiscordIntent {}
}
```

`ChatContent` supports markdown, attachments, and `RichCard` embeds. Validation (embed count, character limits) happens at dispatch through the chat SPI — the agent is free to express intent without pre-validating platform constraints.

Dispatch maps directly: `Send` → `Messaging.send()`, `Reply` → `Threading.reply()`, `React` → `Reactions.add()`.

## Agency Loop

**Direct LLM, no CaseEngine** (D8):

Follows QuarkVille's `VilleAgencyLoop` pattern — `AgencyLoop.tick(AgencyContext)` with an `LlmInvoker` for reasoning. A Discord character is a single agent making holistic decisions; CaseEngine's multi-plugin blackboard dispatch is unnecessary.

The loop per tick:
1. Read conversation delta (perception)
2. Retrieve relevant memories (episodic + relationship)
3. Evaluate needs (SOCIAL, CURIOSITY, EXPRESSION)
4. Compose LLM prompt: system prompt (Eidos personality) + conversation context + memories + needs + available actions
5. Parse LLM response into intents
6. Apply output governor (D7, D10) — rate limit, social pacing
7. Dispatch surviving intents via `WorldBridge.dispatch()`
8. Ingest experience into memory (with async importance scoring)

## Memory Architecture

**Four layers, existing foundation** (D5):

| Layer | Purpose | Foundation module |
|-------|---------|-------------------|
| Working memory | Current conversation context | LLM context window |
| Episodic memory | Conversation cases | `CbrCaseMemoryStore` + `TemporalDecayCbrCaseMemoryStore` |
| Semantic memory | Reflections / insights | `ReflectionService` → high-importance CBR cases |
| Relationship memory | Per-person profiles | `GraphitiCaseMemoryStore` or `Mem0CaseMemoryStore` + `casehub-ledger` |

**Three capability additions required:**

1. **Importance scoring at ingest** (neocortex enhancement) — LLM-rated significance 1–10 at memory creation. Runs asynchronously — memory creation returns immediately; score attached when LLM call completes.

2. **Idle-time reflection trigger** (quarkmind-core) — scheduler fires when accumulated importance exceeds threshold during idle periods. Drives both memory consolidation (episodic → semantic) and personality growth (D6 Layer 3).

3. **Relationship schema** (Graphiti/Mem0) — per-person model: trust score (from ledger), interaction count, opinion summary, shared topics, last interaction timestamp.

## Personality System

**Three layers** (D6):

**Layer 1 — Prompt-driven (Eidos → LLM):** `AgentDescriptor` defines character via disposition axes, style profile, briefing, goals, constraints. `SystemPromptRenderer` renders into system prompt.

**Layer 2 — Mechanical (quarkmind-core NeedState):** Dispositions affect behavior through `DispositionNeedModifier`:
- SOCIAL_ORIENTATION → social need decay rate
- AUTONOMY → heartbeat frequency for unprompted action
- RISK_APPETITE → engagement willingness with strangers

**Layer 3 — Growth (reflection → disposition evolution):** The full pipeline:

```
Discord interactions
  → episodic memory (ingest)
  → importance accumulates
  → idle-time reflection trigger fires
  → ReflectionService produces insights
  → ReflectionDispositionActivator classifies which disposition function term is activated
  → DispositionSignalStore.recordActivation()
  → DispositionHealth.probe() checks threshold crossings
  → DispositionEvolution.evaluate() → Evolved(newProfile) or Dampened(decayFactor)
```

Discord is the first consumer of this full end-to-end chain. Each link uses existing foundation types; the integration is new.

## Need System

**Three needs** (D7):

**SOCIAL** — decays when idle, satisfied by conversation. Three sub-drives (prompt-level, not mechanical):
1. Learn about people — understand who they are, what they care about
2. Help people — look for opportunities to be useful
3. Meaningful contribution — add value to the group (connect people, remember things)

**CURIOSITY** — decays slowly, spikes on interesting topics. Drives questions and follow-ups.

**EXPRESSION** — builds over time, satisfied by sharing opinions/reactions. Drives unprompted posting and emoji reactions.

## Rate Governing

**Three independent layers** (D7, D10):

1. **Global output governor** (quarkmind-core) — max actions per time window. Default: 1 unprompted post per 5 minutes, hard minimum 30-second interval.

2. **Proactive decision gate** (quarkmind-core) — evaluates social context before need-driven actions: time since last post, channel activity, typing indicators, conversational flow.

3. **Channel-aware pacing** (quarkmind-discord) — frequency scales with channel activity. Quiet channel → fill silence; busy channel → wait for gaps.

Need threshold crossing is necessary but not sufficient for action. Multiple simultaneous threshold crossings produce a single prioritized action.

**API rate limiting** (D10) is the connector's responsibility — `DiscordChatPlatform` tracks Discord rate limit headers and applies back-pressure. Behavioral pacing is the binding constraint in normal operation.

## Scope

**v1: Single character per server** (D9). Multi-character scenarios (multiple characters interacting, per-channel profiles) are explicitly out of scope.

## Layering Summary

**Two-tier** (D3 revised):

| Layer | Owns | Uses |
|-------|------|------|
| **quarkmind-core** | Chat perception bridge, conversation delta report, attention priority classification, need-threshold wake, proactive decision gate, idle-time reflection trigger, output governor | blocks (TieredObservationRenderer, EventSource, ChoreographedDriver), eidos-api, neocortex-api, ledger-api |
| **quarkmind-discord** | DiscordEventSource, Discord-specific detection (@mentions, bot user ID), channel visibility policy, personality prompt construction, intent dispatch | quarkmind-core, connectors (chat-discord, discord), quarkmind-discord-protocol |

Blocks receives **no new types** — existing `TieredObservationRenderer<E>`, `EventSource`, `ChoreographedDriver` are used directly with custom renderers and event sources.

## Cross-Repo Work

| Repo | What changes |
|------|-------------|
| `casehub-blocks` | Nothing — existing primitives used directly |
| `casehub-connectors` | Possible: Gateway-accumulating `MessageHistory` implementation (or quarkmind-discord wraps it) |
| `casehub-neocortex` | Importance scoring at ingest (async LLM rating) |
| `quarkmind` (quarkmind-core) | Chat perception bridge, conversation delta report, attention priority, need-threshold wake, proactive decision gate, idle-time reflection trigger, output governor, ReflectionDispositionActivator |
| `quarkmind` (quarkmind-discord) | New module: protocol + agent |

## References

- `casehub-connectors` chat-discord — `DiscordChatPlatform.java`, `DiscordInboundConnector.java`, `ChatPlatform` SPI
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
