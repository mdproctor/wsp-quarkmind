# QuarkMind Chat — Multi-Character Per Server

**Date:** 2026-08-21
**Status:** Draft
**Issue:** #282
**Decisions:** See `decisions.md` (D25–D30)
**Depends on:** quarkmind-discord design (D1–D24)

## Overview

Extend quarkmind-chat to support multiple characters per Discord server. Each character is a separate Discord bot account with its own personality, memory, needs, and identity, running in a single JVM process. Characters see each other through direct Gateway connections that include bot messages (the existing `DiscordInboundConnector` filters bot messages and cannot be used). A shared pacing governor prevents simultaneous responses.

This follows WackyManor's proven pattern (`casehub/examples/wacky-manor`): shared CDI services + agentId-scoped operations = multi-character in one process. WackyManor's `ScenarioOrchestrator` manages N characters via `agentId`-scoped calls to shared services (`AgentRegistry`, `CaseMemoryStore`, `DispositionSignalStore`). QuarkMind chat adapts this to event-driven Discord execution.

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
    // Per-character WorldBridge, EventSource, MessageHistory (see Per-Character Gateway)
    private final ChatWorldBridge worldBridge;
}
```

Note: `CharacterContext` is a mutable class, not a record — `consecutiveIdleTicks` and `lastReflectionTimestamp` are mutated per tick.

`ChatAgencyLoop` becomes stateless — `tick()` reads the `CharacterContext` from `AgencyContext` via `context.getAs("character", CharacterContext.class)` (D30). The loop no longer holds `agentId`, `tenantId`, `systemPrompt`, `consecutiveIdleTicks`, `lastReflectionTimestamp`, or `participatedThreadIds` as instance fields.

### ChatCharacterManager

CDI `@ApplicationScoped` bean with `@Observes StartupEvent` that manages all characters. Analogous to WackyManor's `ScenarioOrchestrator` (`casehub/examples/wacky-manor`).

```java
@ApplicationScoped
public class ChatCharacterManager {
    @Inject AgentRegistry agentRegistry;
    @Inject ChatMemoryFacade memoryFacade;      // shared, agentId-scoped
    @Inject LlmRequestQueue llmQueue;           // shared rate limiter
    @Inject ObjectMapper mapper;
    @Inject ChatPerceptionBridge perceptionBridge;

    private ChatAgencyLoop agencyLoop;           // single shared instance, constructed programmatically
    private final Map<String, CharacterContext> characters = new ConcurrentHashMap<>();
    private OutputGovernor sharedGovernor;       // cross-character pacing (D27)

    void onStart(@Observes StartupEvent evt) { ... }
}
```

`ChatAgencyLoop` is constructed programmatically by the manager (not CDI-managed) — it takes only shared, stateless dependencies. Per-character state flows through `AgencyContext`.

**Startup responsibilities (`onStart`):**
1. Read character config from Quarkus `@ConfigMapping` — each entry has `agentId`, `token`, `channels`
2. For each configured character, look up `AgentDescriptor` from `AgentRegistry` via `agentRegistry.findById(agentId, tenantId)`
3. Create `CharacterContext` with per-character `NeedState`, `IdleReflectionTrigger`, `DiscordIdentityDetector`
4. Create per-character `DiscordGateway` connection with the character's bot token (D28) — NOT `DiscordInboundConnector` (see Per-Character Gateway)
5. Wire each character's Gateway events to its `DiscordGatewayMessageHistory` and `DiscordEventSource`
6. Construct shared `ChatAgencyLoop` with shared dependencies (`llmQueue`, `mapper`, `perceptionBridge`, `memoryFacade`)

**Per-tick flow:**
1. Character's `DiscordEventSource` fires (message or heartbeat)
2. Character's `ChatWorldBridge` builds `ChatPerception` from its `DiscordGatewayMessageHistory`
3. Manager creates `AgencyContext` with the character's `NeedState`, puts `CharacterContext` and `ChatPerception`
4. Shared `OutputGovernor.tryAcquire()` atomically checks AND records — if denied, skip this tick
5. `agencyLoop.tick(context)` runs — reads `CharacterContext` for identity, prompt, memory scoping
6. Intents dispatched via the character's `ChatWorldBridge`

**Concurrency model:** Each character's Gateway events fire on separate virtual threads. Same-character ticks are serialized by the `ChoreographedDriver` event dispatch model (single-threaded per driver instance). Cross-character ticks are concurrent — the shared `OutputGovernor` uses atomic `tryAcquire()` (not separate allow/record) to prevent TOCTOU races. When `tryAcquire()` returns false, the tick proceeds but skips intent dispatch (observation and memory still run).

### Per-Character Gateway (D28)

Each character needs its own Discord Gateway connection because Discord requires one WebSocket session per bot token. The `ChatCharacterManager` creates per-character:

- `DiscordGateway` — direct Gateway connection with the character's bot token. **NOT `DiscordInboundConnector`** — the connector filters out bot messages (`author.bot == true` at line 124), which would prevent characters from seeing each other. Direct Gateway access receives ALL messages including other bots.
- `DiscordGatewayMessageHistory` — accumulates Gateway events for this bot (all messages, including from other characters)
- `DiscordEventSource` — receives events from the Gateway listener
- `DiscordIdentityDetector` — knows this bot's Discord user ID, used to filter the character's OWN messages from perception (a character shouldn't perceive its own output as external input)
- `ChatWorldBridge` — uses this bot's message history and identity detector

The per-character `GatewayEventListener` wired by `ChatCharacterManager`:
1. Receives all `MESSAGE_CREATE` events (no bot filter)
2. Converts to `ReceivedMessage` and calls `messageHistory.accumulate()`
3. Fires `eventSource.onMessage()` for messages not from this character's own bot (filtered by `identityDetector.botUserId()`)

This means Character A's listener accumulates Character B's messages into its history (so A can see B's posts) but does NOT fire a wake event for A's own messages.

### Cross-Character Pacing (D27)

One `OutputGovernor` per server (tenantId), shared across all characters. Prevents multiple characters from responding simultaneously.

The existing three-layer pacing from D7 becomes four layers:
1. **Server-wide governor** (new) — minimum interval between any two character actions in the same server
2. **Per-character output governor** (existing) — per-character action frequency cap
3. **Proactive decision gate** (existing) — per-character social context evaluation
4. **Channel-aware pacing** (existing) — per-character channel activity scaling

The server-wide governor is the only new layer. A character must pass BOTH the server-wide check AND its own per-character check before acting.

**Thread safety:** `OutputGovernor.allow()` + `recordAction()` is a TOCTOU race under concurrent character ticks. Replace with a single atomic `boolean tryAcquire()` that checks AND records in one synchronized operation. The existing separate `allow()`/`recordAction()` pattern is safe for single-character (one tick at a time) but breaks with concurrent multi-character ticks.

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
| `LlmReflectionDispositionActivator` | **Refactor.** Currently holds mutable `volatile List<DispositionValue> dispositionProfile` — per-character state. Change to `Map<String, List<DispositionValue>>` keyed by `agentId`. `updateProfile()` becomes `updateProfile(String agentId, ...)`. `onReflection(agentId, ...)` already receives `agentId` — look up profile per call. This makes the activator genuinely shareable across characters. |

### quarkmind-chat-agent/discord (per-character instances)

| Class | Change |
|-------|--------|
| `DiscordEventSource` | No structural change — one instance per character. |
| `DiscordGatewayMessageHistory` | No structural change — one instance per character. |
| `DiscordIdentityDetector` | No structural change — one instance per character with its own `botUserId`. |

### quarkmind-core

| Class | Change |
|-------|--------|
| `OutputGovernor` | Replace `allow()` + `recordAction()` with atomic `boolean tryAcquire()`. Make thread-safe for shared cross-character use. |
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

## Known Debt

`DiscordGatewayMessageHistory` accumulates messages without eviction — the buffer grows indefinitely. With N characters, each holding its own buffer, this is amplified. The `CopyOnWriteArrayList` backing structure copies the entire array on every `add()`. This is existing debt (pre-dates multi-character) but should be addressed: add a `drain(Instant before)` method or switch to a bounded deque. Filed separately from this issue.

## References

- WackyManor `ScenarioOrchestrator` — multi-character orchestration pattern (shared services, agentId-scoped operations, virtual threads)
- WackyManor `CharacterState` — per-character mutable state record
- WackyManor `ProfileAwareDescriptorRegistrar` — profile-aware multi-descriptor loading
- `ClasspathYamlDescriptorRegistrar` (casehub-eidos) — loads N descriptors from one YAML
- GE-20260820-c19b68 — CbrQuery lacks producerAgentId filter (memory isolation by agentId)
- GE-20260811-3bf675 — AgentDescriptorRegistrar producer pattern
- D1–D24 in `decisions.md` — quarkmind-discord design decisions this extends
- blocks `WorldObservationProvider` (blocks#127) — available SPI for structured observation rendering
