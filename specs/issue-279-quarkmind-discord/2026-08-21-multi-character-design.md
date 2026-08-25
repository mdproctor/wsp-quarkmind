# QuarkMind Chat — Multi-Character Per Server + Blocks Integration

**Date:** 2026-08-21 (revised 2026-08-25)
**Status:** Draft (R2)
**Issue:** #282
**Decisions:** See `decisions.md` (D25–D35)
**Depends on:** quarkmind-discord design (D1–D24)

## Overview

Extend quarkmind-chat to support multiple characters per Discord server, wired to the blocks social cognition stack. Each character is a separate Discord bot account with its own personality, memory, and identity, running in a single JVM process. Characters see each other through direct Gateway connections that include bot messages. The blocks `InnerLifeOrchestrator` replaces all custom proactive behavioral logic; `DriveOrchestrator` replaces `NeedState`; `CivilityConstraint` chain replaces `OutputGovernor`.

This follows WackyManor's proven pattern: shared CDI services + agentId-scoped operations = multi-character in one process. All blocks orchestrators use `agentId:tenantId` keying internally — multi-agent was designed in from the start.

## Architecture

### Deployment Model (D25)

Multiple Discord bot accounts in a single JVM. Each character has:
- Its own Discord bot token and Gateway WebSocket connection (D28)
- Its own `AgentDescriptor` (personality, disposition, goals, constraints)
- Its own memory space (scoped by `agentId` in `CaseMemoryStore`)
- Its own identity detector (knows its own Discord user ID)

Characters share:
- `InnerLifeOrchestrator` — proactive behavioral layer (per-agent state internally)
- `DriveOrchestrator` — four drives per agent (per-agent state internally)
- `LlmRequestQueue` — rate limiting and priority routing across all characters
- `CaseMemoryStore` / `GraphCaseMemoryStore` — one store, per-agent isolation via `agentId`
- `ChatPerceptionBridge`, `ObjectMapper` — stateless, reusable

### Two-Path Execution Model (D31)

The chat agent has two distinct paths:

**Reactive path (message wake — someone talks to/mentions the bot):**
1. Feed conversation events to `innerLifeOrchestrator.observe(event, descriptor)`
2. Get `driveOrchestrator.currentDrives(agentId, tenantId)` for prompt enrichment
3. Build conversation context, recall memories, call LLM → respond
4. `innerLifeOrchestrator.observeResponse(descriptor)` — records that the agent responded
5. Ingest memory via `ChatMemoryFacade`

**Proactive path (heartbeat wake — agent decides whether to speak unprompted):**
1. `innerLifeOrchestrator.tick(descriptor, channelContext)` → `Initiated(content, channelHint, score)` or `Silent(reason)`
2. If `Initiated`: dispatch as `ChatIntent.Send` — the content comes from InnerLifeOrchestrator's internal LLM motivation call
3. If `Silent`: no action

`InnerLifeOrchestrator` internally handles: drive ticking, civility constraints (pacing), reflection, content quality evaluation, and the proactive LLM call. `ChatAgencyLoop` retains the reactive LLM call for direct conversation responses where conversation-aware context (threads, @mentions, memory recall) is needed.

### CharacterContext (D26, revised D34)

Per-character identity and perception state — no behavioral state (blocks manages that internally per-agent):

```java
public class CharacterContext {
    private final String agentId;
    private final String tenantId;
    private final String systemPrompt;
    private final Supplier<AgentDescriptor> descriptorSupplier;
    private final BotIdentityDetector identityDetector;
    private final Set<String> participatedThreadIds;
    private final ChatWorldBridge worldBridge;
}
```

Simplified from the original D26 — `NeedState`, `IdleReflectionTrigger`, `consecutiveIdleTicks`, and `lastReflectionTimestamp` are removed. `InnerLifeOrchestrator` and `DriveOrchestrator` manage all behavioral state per-agent via `ConcurrentHashMap<String, AgentState>` keyed by `agentId:tenantId`.

### ChatCharacterManager

CDI `@ApplicationScoped` bean that manages all characters. Analogous to WackyManor's `ScenarioOrchestrator`.

```java
@ApplicationScoped
public class ChatCharacterManager {
    @Inject AgentRegistry agentRegistry;
    @Inject ChatMemoryFacade memoryFacade;
    @Inject LlmRequestQueue llmQueue;
    @Inject ObjectMapper mapper;
    @Inject ChatPerceptionBridge perceptionBridge;
    @Inject InnerLifeOrchestrator innerLifeOrchestrator;
    @Inject DriveOrchestrator driveOrchestrator;

    private ChatAgencyLoop agencyLoop;
    private final Map<String, CharacterContext> characters = new ConcurrentHashMap<>();
}
```

`ChatAgencyLoop` is constructed programmatically by the manager — it takes shared dependencies including `InnerLifeOrchestrator` and `DriveOrchestrator`. Per-character state flows through `AgencyContext` via `CharacterContext`.

**Startup (`onStart`):**
1. Read character config from Quarkus `@ConfigMapping` — each entry has `agentId`, `token`, `channels`
2. For each configured character, look up `AgentDescriptor` from `AgentRegistry`
3. Create `CharacterContext` with per-character `DiscordIdentityDetector`, system prompt, world bridge
4. Create per-character `DiscordGateway` connection with the character's bot token (D28)
5. Wire each character's Gateway events to its `DiscordGatewayMessageHistory` and `DiscordEventSource`
6. Construct shared `ChatAgencyLoop` with shared dependencies

**Per-tick flow:**
1. Character's `DiscordEventSource` fires (message or heartbeat)
2. Character's `ChatWorldBridge` builds `ChatPerception`
3. Manager creates `AgencyContext`, puts `CharacterContext` and `ChatPerception` (D30)
4. `agencyLoop.tick(context)` runs — reads `CharacterContext` for identity, prompt, memory scoping
5. On message wake: reactive path (ChatAgencyLoop calls LLM, checks drives for prompt context)
6. On heartbeat wake: proactive path (ChatAgencyLoop calls `innerLifeOrchestrator.tick()`)
7. Intents dispatched via the character's `ChatWorldBridge`

### Per-Character Gateway (D28)

Each character needs its own Discord Gateway connection (one WebSocket per bot token — Discord API requirement). **NOT `DiscordInboundConnector`** — the connector filters bot messages (`author.bot == true`), which would prevent characters from seeing each other.

Per character:
- `DiscordGateway` — direct Gateway connection with the character's bot token
- `DiscordGatewayMessageHistory` — accumulates all messages including from other characters
- `DiscordEventSource` — receives events from the Gateway listener
- `DiscordIdentityDetector` — knows this bot's Discord user ID, filters own messages from perception
- `ChatWorldBridge` — uses this bot's message history and identity detector

The per-character `GatewayEventListener`:
1. Receives all `MESSAGE_CREATE` events (no bot filter)
2. Accumulates into `messageHistory`
3. Fires `eventSource.onMessage()` for messages not from this character's own bot

### Drive-Enriched Reactive Prompts (D32)

The reactive LLM prompt replaces `Needs: SOCIAL=X, CURIOSITY=Y, EXPRESSION=Z` with drive state from `DriveOrchestrator`:

```
Drives: CURIOSITY=0.7, COMPETENCE=0.5, AFFILIATION=0.8, AUTONOMY=0.3
```

`driveOrchestrator.currentDrives(agentId, tenantId)` returns a `DriveProfile` with the current drive intensities. The four drives (CURIOSITY, COMPETENCE, AFFILIATION, AUTONOMY) are richer than the original three needs — they compose mood, narrative modulation, and disposition internally.

### Cross-Character Pacing (D27, revised D33)

`CivilityConstraint` chain replaces `OutputGovernor`. `InnerLifeOrchestrator` runs the constraint chain internally for proactive initiation. The three-layer pacing from D7 maps to `CivilityConstraint` implementations:

| Original layer | Blocks replacement |
|---|---|
| Global output governor (D7 layer 1) | `MaxPerWindowConstraint` |
| Minimum interval (D7 layer 1) | `MinimumGapConstraint` |
| Proactive decision gate (D7 layer 2) | `ConsecutiveInitiationCooldownConstraint` + InnerLifeOrchestrator internal evaluation |
| Channel-aware pacing (D7 layer 3) | InnerLifeOrchestrator channel context evaluation |

For cross-character shared pacing (D27), a single `MinimumGapConstraint` instance shared across all characters achieves the server-wide pacing. If the built-in constraint doesn't support shared-state across agents (each sees one agent's `InitiationContext`), a custom `SharedMinimumGapConstraint` CDI bean may be needed — verify during implementation.

### Configuration (D29)

Character configuration via Quarkus `@ConfigMapping`:

```yaml
quarkmind:
  chat:
    tenant-id: my-server
    characters:
      luna:
        token: ${DISCORD_TOKEN_LUNA}
        channels: general,lounge
      rex:
        token: ${DISCORD_TOKEN_REX}
        channels: general,workshop
```

Each `agentId` in config must match an `AgentDescriptor.agentId()` in `META-INF/eidos/descriptors.yaml`. Single-character deployment is a special case: one entry in the `characters` map.

## Changes Required

### quarkmind-chat-agent — modified

| Class | Change |
|-------|--------|
| `ChatAgencyLoop` | Remove per-character instance fields. Read from `CharacterContext` in `AgencyContext`. Add `InnerLifeOrchestrator` and `DriveOrchestrator` as constructor params. Implement two-path execution (reactive/proactive). |
| `ChatWorldBridge` | No structural change — already parameterized. One instance per character. |
| `ChatMemoryFacade` | No change — already takes `agentId`/`tenantId` as method params. |

### quarkmind-chat-agent — new classes

| Class | Purpose |
|-------|---------|
| `CharacterContext` | Per-character identity + perception state (D26/D34) |
| `ChatCharacterManager` | Manages N characters, creates per-character stacks (D26) |
| `CharacterConfig` | Per-character config record (`agentId`, `token`, `channels`) |

### quarkmind-chat-agent — deleted (D35)

| Class | Replaced by |
|-------|-------------|
| `ChatNeedDefinitions` | `DriveOrchestrator` (blocks) |
| `LlmReflectionDispositionActivator` | `TraitPressureSource` impls (blocks) |
| `DispositionAwareReflectionSynthesizer` | `InnerLifeOrchestrator` reflection (blocks) |
| `LlmReflectionSynthesizer` | `InnerLifeOrchestrator` reflection (blocks) |
| `ChatChannelPacing` | `CivilityConstraint` chain (blocks) |

### quarkmind-chat-agent/discord

| Class | Change |
|-------|--------|
| `DiscordEventSource` | No structural change — one instance per character |
| `DiscordGatewayMessageHistory` | No structural change — one instance per character |
| `DiscordIdentityDetector` | No structural change — one instance per character |

### quarkmind-core

| Class | Change |
|-------|--------|
| `OutputGovernor` | No change — stays for SC2. No longer used by quarkmind-chat. |
| `IdleReflectionTrigger` | No change — stays for SC2. No longer used by quarkmind-chat. |
| `NeedState` | No change — stays for SC2. No longer used by quarkmind-chat. |
| `AgencyContext` | No change — `CharacterContext` goes in the existing map. |

### Tests

| Test | Change |
|------|--------|
| `ChatAgencyLoopTest` | Major rewrite — use `CharacterContext` in `AgencyContext`, mock `InnerLifeOrchestrator`/`DriveOrchestrator`, test reactive/proactive paths |
| `ChatAgentEndToEndTest` | Update to use `CharacterContext` and blocks orchestrators |
| `ChatCharacterManagerTest` | **New.** Multi-character lifecycle: startup creates N contexts, tick dispatches to correct character |
| `LlmReflectionSynthesizerTest` | **Deleted** — class deleted |
| `LlmReflectionDispositionActivatorTest` | **Deleted** — class deleted |
| `DispositionAwareReflectionSynthesizerTest` | **Deleted** — class deleted |

### Dependencies — pom.xml

`quarkmind-chat-agent` gains dependency on `casehub-blocks` for:
- `InnerLifeOrchestrator`
- `DriveOrchestrator`
- `CivilityConstraint`
- `ImportanceScorer`
- `PersonalityEvolutionOrchestrator`
- `TraitPressureSource`
- `NarrativeOrchestrator`
- `GoalProposalOrchestrator`

## What Does NOT Change

- `ChatPerception`, `ChatIntent`, `WakeReason` (protocol types) — character-agnostic
- `ChatDeltaReport`, `AttentionClassifier`, `ChatObservationRenderer` (quarkmind-core) — stateless
- `CaseMemoryStore`, `GraphCaseMemoryStore` (neocortex) — already scoped by `agentId`
- `AgentDescriptor`, `AgentRegistry` (eidos) — already supports multiple descriptors
- `ReflectionOrchestrator` (neocortex) — already takes `agentId` as parameter

## Known Debt

- `DiscordGatewayMessageHistory` accumulates messages without eviction — buffer grows indefinitely. With N characters, each holding its own buffer, this is amplified (#284).
- `DiscordIdentityDetector.botMessageIds` is an unbounded `ConcurrentHashMap.newKeySet()` — grows indefinitely as the bot sends messages (#285).

## References

- WackyManor `ScenarioOrchestrator` — multi-character orchestration pattern
- WackyManor `CharacterState` — per-character mutable state
- WackyManor `ProfileAwareDescriptorRegistrar` — profile-aware multi-descriptor loading
- blocks `InnerLifeOrchestrator` — proactive behavioral compositor (`io.casehub.blocks.agentic.social`)
- blocks `DriveOrchestrator` — drive architecture (`io.casehub.blocks.agentic.social.drive`)
- blocks `CivilityConstraint` — pacing constraint chain (`io.casehub.blocks.agentic.social`)
- blocks `NarrativeOrchestrator` — narrative identity (`io.casehub.blocks.agentic.social.narrative`)
- blocks `GoalProposalOrchestrator` — autonomous goals (`io.casehub.blocks.agentic.social.goal`)
- `ClasspathYamlDescriptorRegistrar` (casehub-eidos) — loads N descriptors from one YAML
- GE-20260820-c19b68 — CbrQuery lacks producerAgentId filter (memory isolation by agentId)
- D1–D24 in `decisions.md` — quarkmind-discord design decisions this extends
- D25–D35 in `decisions.md` — multi-character + blocks integration decisions
