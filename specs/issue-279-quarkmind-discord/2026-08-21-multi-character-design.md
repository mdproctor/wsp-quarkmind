# QuarkMind Chat — Multi-Character Per Server

**Date:** 2026-08-21
**Status:** Draft
**Issue:** #282
**Decisions:** See `decisions.md` (D25–D30)
**Depends on:** quarkmind-discord design (D1–D24)

## Overview

Extend quarkmind-chat to support multiple characters per Discord server. Each character is a separate Discord bot account with its own personality, memory, needs, and identity, running in a single JVM process. Characters see each other naturally through Discord — no special cross-character wiring — but share a pacing governor to prevent simultaneous responses.

This follows WackyManor's proven pattern: shared CDI services + agentId-scoped operations = multi-character in one process.

## Architecture

### Deployment Model (D25)

Multiple Discord bot accounts in a single JVM. Each character has:
- Its own Discord bot token and Gateway WebSocket connection
- Its own `AgentDescriptor` (personality, disposition, goals, constraints)
- Its own memory space (scoped by `agentId` in `CaseMemoryStore`)
- Its own need state (SOCIAL, CURIOSITY, EXPRESSION)
- Its own identity detector (knows its own Discord user ID)

Characters share:
- `LlmRequestQueue` — rate limiting and priority routing across all characters
- `CaseMemoryStore` / `GraphCaseMemoryStore` — one store, per-agent isolation via `agentId`
- `OutputGovernor` — cross-character pacing per server (D27)
- `ObjectMapper`, `ChatPerceptionBridge` — stateless, reusable

### CharacterContext (D26)

Per-character mutable state extracted from `ChatAgencyLoop` into a dedicated class:

```java
public class CharacterContext {
    private final String agentId;
    private final String tenantId;
    private final String systemPrompt;
    private final Supplier<AgentDescriptor> descriptorSupplier;
    private final BotIdentityDetector identityDetector;
    private final NeedState needState;
    private final IdleReflectionTrigger reflectionTrigger;
    private final Set<String> participatedThreadIds;
    private int consecutiveIdleTicks;
    private Instant lastReflectionTimestamp;
}
```

`ChatAgencyLoop` becomes stateless — `tick()` reads the `CharacterContext` from `AgencyContext` via `context.getAs("character", CharacterContext.class)` (D30). The loop no longer holds `agentId`, `tenantId`, `systemPrompt`, `consecutiveIdleTicks`, `lastReflectionTimestamp`, or `participatedThreadIds` as instance fields.

### ChatCharacterManager

CDI `@ApplicationScoped` bean that manages all characters. Analogous to WackyManor's `ScenarioOrchestrator`.

```java
@ApplicationScoped
public class ChatCharacterManager {
    @Inject AgentRegistry agentRegistry;
    @Inject ChatAgencyLoop agencyLoop;         // single shared instance
    @Inject ChatMemoryFacade memoryFacade;      // shared, agentId-scoped
    @Inject LlmRequestQueue llmQueue;           // shared rate limiter
    @Inject ObjectMapper mapper;
    @Inject ChatPerceptionBridge perceptionBridge;

    private final Map<String, CharacterContext> characters = new ConcurrentHashMap<>();
    private OutputGovernor sharedGovernor;       // cross-character pacing (D27)
}
```

**Startup responsibilities:**
1. Read character config from Quarkus `@ConfigMapping` — each entry has `agentId`, `token`, `channels`
2. For each configured character, look up `AgentDescriptor` from `AgentRegistry` via `agentRegistry.findById(agentId, tenantId)`
3. Create `CharacterContext` with per-character `NeedState`, `IdleReflectionTrigger`, `DiscordIdentityDetector`
4. Create per-character `DiscordEventSource` with the character's bot token (D28)
5. Wire each character's event source to drive `agencyLoop.tick()` with the right `CharacterContext` in `AgencyContext`

**Per-tick flow:**
1. Character's `DiscordEventSource` fires (message or heartbeat)
2. Character's `ChatWorldBridge` builds `ChatPerception` from its `DiscordGatewayMessageHistory`
3. Manager creates `AgencyContext` with the character's `NeedState`, puts `CharacterContext` and `ChatPerception`
4. Shared `OutputGovernor` checks if any character in this server acted recently — if so, delay
5. `agencyLoop.tick(context)` runs — reads `CharacterContext` for identity, prompt, memory scoping
6. Intents dispatched via the character's `ChatWorldBridge`

### Per-Character Gateway (D28)

Each character needs its own Discord Gateway connection because Discord requires one WebSocket session per bot token. The `ChatCharacterManager` creates per-character:

- `DiscordInboundConnector` — configured with the character's bot token
- `DiscordGatewayMessageHistory` — accumulates Gateway events for this bot
- `DiscordEventSource` — adapts the connector as an `EventSource`
- `DiscordIdentityDetector` — knows this bot's Discord user ID
- `ChatWorldBridge` — uses this bot's message history and identity detector

### Cross-Character Pacing (D27)

One `OutputGovernor` per server (tenantId), shared across all characters. Prevents multiple characters from responding simultaneously.

The existing three-layer pacing from D7 becomes four layers:
1. **Server-wide governor** (new) — minimum interval between any two character actions in the same server
2. **Per-character output governor** (existing) — per-character action frequency cap
3. **Proactive decision gate** (existing) — per-character social context evaluation
4. **Channel-aware pacing** (existing) — per-character channel activity scaling

The server-wide governor is the only new layer. It wraps the existing per-character `OutputGovernor` — a character must pass BOTH the server-wide check AND its own per-character check before acting.

### Configuration (D29)

Character configuration uses standard Quarkus `@ConfigMapping`:

```yaml
# application.yaml
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

Each character's `agentId` in the config must match an `AgentDescriptor.agentId()` in `META-INF/eidos/descriptors.yaml`. The descriptor defines personality (disposition axes, style profile, briefing, goals, constraints). The config defines operational parameters (bot token, watched channels).

A single-character deployment is a special case: one entry in the `characters` map. Backwards-compatible with D12.

## Changes Required

### quarkmind-chat-agent (primary changes)

| Class | Change |
|-------|--------|
| `ChatAgencyLoop` | Remove instance fields (`agentId`, `tenantId`, `systemPrompt`, `consecutiveIdleTicks`, `lastReflectionTimestamp`, `participatedThreadIds`, `descriptorSupplier`, `dispositionActivator`). Read from `CharacterContext` in `AgencyContext`. Remove setter methods. |
| `CharacterContext` | **New class.** Holds all per-character mutable state. |
| `ChatCharacterManager` | **New class.** Manages N characters, creates per-character stacks, holds shared `OutputGovernor`. |
| `ChatWorldBridge` | No structural change — already parameterized. One instance per character created by manager. |
| `ChatMemoryFacade` | No change — already takes `agentId`/`tenantId` as method params. |
| `ChatNeedDefinitions` | No structural change — one instance per character, configured from per-character descriptor. |
| `ChatChannelPacing` | No change — one instance per character. |
| `LlmReflectionSynthesizer` | No change — stateless, takes `agentId` as param. |
| `DispositionAwareReflectionSynthesizer` | No change — delegates to synthesizer and activator with `agentId`. |
| `LlmReflectionDispositionActivator` | No change — stateless, takes params. |

### quarkmind-chat-agent/discord (per-character instances)

| Class | Change |
|-------|--------|
| `DiscordEventSource` | No structural change — one instance per character. |
| `DiscordGatewayMessageHistory` | No structural change — one instance per character. |
| `DiscordIdentityDetector` | No structural change — one instance per character with its own `botUserId`. |

### quarkmind-core

| Class | Change |
|-------|--------|
| `OutputGovernor` | Make thread-safe (if not already) for shared cross-character use. |
| `AgencyContext` | No change — `CharacterContext` goes in the existing map. |
| `AgencyLoop` | No change — `tick(AgencyContext)` signature unchanged. |

### Tests

| Test | Change |
|------|--------|
| `ChatAgencyLoopTest` | Update to put `CharacterContext` in `AgencyContext` instead of using setter methods. |
| `ChatAgentEndToEndTest` | Update to use `CharacterContext`. |
| `ChatCharacterManagerTest` | **New test.** Multi-character lifecycle: startup creates N contexts, tick dispatches to correct character, shared governor enforces pacing. |
| `DispositionAwareReflectionSynthesizerTest` | Minor — verify agentId flows correctly through `CharacterContext`. |

## What Does NOT Change

- `ChatPerception`, `ChatIntent`, `WakeReason` (protocol types) — character-agnostic
- `ChatDeltaReport`, `AttentionClassifier`, `ChatObservationRenderer` (quarkmind-core) — stateless
- `PersonalityEvolutionPipeline`, `IdleReflectionTrigger` (quarkmind-core) — already parameterized or per-instance
- `CaseMemoryStore`, `GraphCaseMemoryStore` (neocortex) — already scoped by `agentId`
- `AgentDescriptor`, `AgentRegistry` (eidos) — already supports multiple descriptors
- `ReflectionOrchestrator` (neocortex) — already takes `agentId` as parameter

## References

- WackyManor `ScenarioOrchestrator` — multi-character orchestration pattern (shared services, agentId-scoped operations, virtual threads)
- WackyManor `CharacterState` — per-character mutable state record
- WackyManor `ProfileAwareDescriptorRegistrar` — profile-aware multi-descriptor loading
- `ClasspathYamlDescriptorRegistrar` (casehub-eidos) — loads N descriptors from one YAML
- GE-20260820-c19b68 — CbrQuery lacks producerAgentId filter (memory isolation by agentId)
- GE-20260811-3bf675 — AgentDescriptorRegistrar producer pattern
- D1–D24 in `decisions.md` — quarkmind-discord design decisions this extends
- blocks `WorldObservationProvider` (blocks#127) — available SPI for structured observation rendering
