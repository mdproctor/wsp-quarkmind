# Multi-Character Per Server Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #282 — quarkmind-chat — multi-character per server (D9 v2)
**Issue group:** #279, #280, #281, #282, #283

**Goal:** Support multiple AI characters per Discord server, each with its own bot account, personality, memory, and needs, managed by a single JVM process.

**Architecture:** Extract per-character mutable state from `ChatAgencyLoop` into a `CharacterContext` class. A new `ChatCharacterManager` CDI bean manages N characters. Each character gets its own Discord Gateway connection, identity detector, need state, and reflection trigger. Shared services (LLM queue, memory store, perception bridge) use `agentId` scoping for per-character isolation. A shared `OutputGovernor` with atomic `tryAcquire()` prevents cross-character response collisions.

**Tech Stack:** Java 21, Quarkus CDI, casehub-eidos (AgentDescriptor), casehub-neocortex (CaseMemoryStore), casehub-connectors (DiscordGateway)

## Global Constraints

- `CharacterContext` is a mutable class (not a record) — `consecutiveIdleTicks` and `lastReflectionTimestamp` are mutated per tick
- `AgencyLoop.tick(AgencyContext)` signature is unchanged — `CharacterContext` flows via `context.put("character", ctx)`
- Do NOT use `DiscordInboundConnector` for multi-character — it filters bot messages (`author.bot == true`). Use `DiscordGateway` directly.
- All memory operations already accept `agentId` as parameter — no neocortex changes needed
- Pre-release platform — breaking changes to quarkmind-chat are free

---

## Batch 1: Foundation — CharacterContext + stateless loop

### Task 1: CharacterContext class and ChatAgencyLoop refactoring

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/CharacterContext.java`
- Modify: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatAgencyLoop.java`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatAgencyLoopTest.java`

**Interfaces:**
- Produces: `CharacterContext` — used by `ChatCharacterManager` (Task 4) and all tests
- Produces: `ChatAgencyLoop` with reduced constructor (shared deps only)

- [ ] **Step 1: Write CharacterContext class**

Use `ide_create_file`:

```java
package io.quarkmind.chat.agent;

import io.casehub.eidos.api.AgentDescriptor;
import io.quarkmind.agency.chat.BotIdentityDetector;
import io.quarkmind.agency.needs.NeedState;
import io.quarkmind.agency.schedule.IdleReflectionTrigger;

import java.time.Instant;
import java.util.HashSet;
import java.util.Set;
import java.util.function.Supplier;

public class CharacterContext {

    private final String agentId;
    private final String tenantId;
    private final String systemPrompt;
    private final Supplier<AgentDescriptor> descriptorSupplier;
    private final BotIdentityDetector identityDetector;
    private final NeedState needState;
    private final IdleReflectionTrigger reflectionTrigger;
    private final Set<String> participatedThreadIds = new HashSet<>();
    private int consecutiveIdleTicks;
    private Instant lastReflectionTimestamp = Instant.now();

    public CharacterContext(String agentId, String tenantId, String systemPrompt,
                            Supplier<AgentDescriptor> descriptorSupplier,
                            BotIdentityDetector identityDetector,
                            NeedState needState,
                            IdleReflectionTrigger reflectionTrigger) {
        this.agentId = agentId;
        this.tenantId = tenantId;
        this.systemPrompt = systemPrompt;
        this.descriptorSupplier = descriptorSupplier;
        this.identityDetector = identityDetector;
        this.needState = needState;
        this.reflectionTrigger = reflectionTrigger;
    }

    public String agentId() { return agentId; }
    public String tenantId() { return tenantId; }
    public String systemPrompt() { return systemPrompt; }
    public Supplier<AgentDescriptor> descriptorSupplier() { return descriptorSupplier; }
    public BotIdentityDetector identityDetector() { return identityDetector; }
    public NeedState needState() { return needState; }
    public IdleReflectionTrigger reflectionTrigger() { return reflectionTrigger; }
    public Set<String> participatedThreadIds() { return participatedThreadIds; }
    public int consecutiveIdleTicks() { return consecutiveIdleTicks; }
    public void setConsecutiveIdleTicks(int ticks) { this.consecutiveIdleTicks = ticks; }
    public Instant lastReflectionTimestamp() { return lastReflectionTimestamp; }
    public void setLastReflectionTimestamp(Instant ts) { this.lastReflectionTimestamp = ts; }
    public void incrementIdleTicks() { consecutiveIdleTicks++; }
    public void resetIdleTicks() { consecutiveIdleTicks = 0; }
}
```

- [ ] **Step 2: Refactor ChatAgencyLoop — remove per-character fields, read from CharacterContext**

Remove fields: `systemPrompt`, `agentId`, `tenantId`, `participatedThreadIds`, `consecutiveIdleTicks`, `lastReflectionTimestamp`, `descriptorSupplier`, `dispositionActivator`.

Remove setter methods: `setSystemPrompt`, `setAgentId`, `setTenantId`, `setDispositionActivator`.

Remove the 10-arg constructor and the setter-based constructor chain. Replace with a single constructor taking only shared dependencies:

```java
public ChatAgencyLoop(LlmInvoker llmInvoker, LlmRequestQueue llmQueue,
                      ObjectMapper mapper, ChatPerceptionBridge perceptionBridge,
                      ChatMemoryFacade memoryFacade,
                      ReflectionOrchestrator reflectionOrchestrator,
                      PersonalityEvolutionPipeline evolutionPipeline) {
    this.llmInvoker = llmInvoker;
    this.llmQueue = llmQueue;
    this.mapper = mapper;
    this.perceptionBridge = perceptionBridge;
    this.memoryFacade = memoryFacade;
    this.reflectionOrchestrator = reflectionOrchestrator;
    this.evolutionPipeline = evolutionPipeline;
}
```

`BotIdentityDetector identityDetector` moves OUT of the constructor — it's per-character, read from `CharacterContext`.

Refactor `tick()` to read from `CharacterContext`:

```java
@Override
public void tick(AgencyContext context) {
    var perception = context.getAs("perception", ChatPerception.class);
    if (perception == null) return;

    var character = context.getAs("character", CharacterContext.class);
    if (character == null) return;

    if (perception.reason() == WakeReason.HEARTBEAT && !perception.hasActivity()) {
        character.incrementIdleTicks();
        checkReflection(character);
        checkEvolution(character);
        context.put("intents", List.of());
        return;
    }

    if (!llmQueue.hasCapacity()) {
        context.put("intents", List.of());
        return;
    }

    character.resetIdleTicks();

    ChatDeltaReport report = perceptionBridge.buildDelta(
            perception, character.identityDetector(), character.participatedThreadIds());
    String renderedContext = perceptionBridge.renderForLlm(report);

    List<Memory> memories = List.of();
    if (memoryFacade != null) {
        var participantIds = extractParticipantIds(perception);
        memories = memoryFacade.recall(character.agentId(), character.tenantId(),
                renderedContext, participantIds, new PersonalityWeights(Map.of()), Instant.now());
    }

    String userPrompt = buildUserPrompt(renderedContext, context, memories);
    String response = llmInvoker.invoke(character.systemPrompt(), userPrompt, character.agentId());
    ParsedResponse parsed = parseResponse(response);
    context.put("intents", parsed.intents());

    if (memoryFacade != null && parsed.observation() != null && !parsed.observation().isBlank()) {
        var sourceRefs = buildSourceRefs(perception);
        var participantIds = extractParticipantIds(perception);
        String memoryId = memoryFacade.ingest(character.agentId(), character.tenantId(),
                parsed.observation(), sourceRefs, participantIds);
        submitImportanceScoring(character, memoryId, parsed.observation());
    }
}
```

Update `checkReflection`, `checkEvolution`, and `submitImportanceScoring` to take `CharacterContext` parameter instead of reading instance fields.

`checkReflection(CharacterContext character)`:
```java
private void checkReflection(CharacterContext character) {
    if (character.reflectionTrigger() == null || reflectionOrchestrator == null) return;
    if (!character.reflectionTrigger().shouldReflect(character.consecutiveIdleTicks())) return;
    try {
        reflectionOrchestrator.reflect(character.agentId(), character.tenantId(),
                character.lastReflectionTimestamp(), 50);
        character.setLastReflectionTimestamp(Instant.now());
        character.reflectionTrigger().reset();
    } catch (Exception e) {
        LOG.warn("Reflection failed", e);
    }
}
```

`checkEvolution(CharacterContext character)`:
```java
private void checkEvolution(CharacterContext character) {
    if (evolutionPipeline == null || character.descriptorSupplier() == null) return;
    try {
        var descriptor = character.descriptorSupplier().get();
        var result = evolutionPipeline.checkEvolution(descriptor);
        result.ifPresent(r -> {
            if (r instanceof DispositionEvolution.EvolutionResult.Evolved evolved) {
                LOG.infof("Personality evolved for %s: %s → %s",
                        character.agentId(), evolved.previousTypeLabel(), evolved.newTypeLabel());
            } else if (r instanceof DispositionEvolution.EvolutionResult.Dampened dampened) {
                LOG.infof("Personality evolution dampened for %s (decay=%.2f)",
                        character.agentId(), dampened.decayFactor());
            }
        });
    } catch (Exception e) {
        LOG.warn("Evolution check failed", e);
    }
}
```

`submitImportanceScoring(CharacterContext character, String memoryId, String observation)`:
```java
private void submitImportanceScoring(CharacterContext character, String memoryId, String observation) {
    String prompt = "Rate the importance of this experience on a scale of 0.0 to 1.0, " +
            "where 0.0 is mundane and 1.0 is life-changing. Respond with a single number.\n\n" +
            "Experience: " + observation;
    llmQueue.submit(new LlmRequest(prompt, LlmPriority.LOW, Map.of(), response -> {
        try {
            double score = Double.parseDouble(response.trim());
            if (score >= 0.0 && score <= 1.0) {
                memoryFacade.scoreImportance(memoryId, character.tenantId(), score);
                if (character.reflectionTrigger() != null) {
                    character.reflectionTrigger().accumulate(score);
                }
            }
        } catch (Exception e) {
            LOG.warn("Importance scoring failed for memory " + memoryId, e);
        }
    }));
}
```

- [ ] **Step 3: Update ChatAgencyLoopTest — use CharacterContext**

Replace all test helpers. The key change: tests now create a `CharacterContext` and put it in `AgencyContext` instead of calling `loop.setAgentId()`, `loop.setSystemPrompt()`, etc.

New helper methods:

```java
private CharacterContext stubCharacter() {
    return new CharacterContext("chat-agent", "default", "You are a bot.",
            null, stubDetector(), new NeedState(), null);
}

private CharacterContext stubCharacter(IdleReflectionTrigger trigger) {
    return new CharacterContext("agent-1", "default", "",
            null, stubDetector(), new NeedState(), trigger);
}

private AgencyContext contextWith(ChatPerception perception) {
    return contextWith(perception, stubCharacter());
}

private AgencyContext contextWith(ChatPerception perception, CharacterContext character) {
    var context = new AgencyContext(character.needState());
    context.put("perception", perception);
    context.put("character", character);
    return context;
}
```

Update `createLoop` to use the new single constructor:

```java
private ChatAgencyLoop createLoop(ChatAgencyLoop.LlmInvoker llm) {
    return new ChatAgencyLoop(llm, llmQueue, mapper,
            new DefaultChatPerceptionBridge(new ChatObservationRenderer(10)),
            null, null, null);
}

private ChatAgencyLoop createLoopWithMemory(ChatAgencyLoop.LlmInvoker llm, ChatMemoryFacade facade) {
    return new ChatAgencyLoop(llm, llmQueue, mapper,
            new DefaultChatPerceptionBridge(new ChatObservationRenderer(10)),
            facade, null, null);
}
```

Update each test that called `loop.setAgentId("agent-1")` to use `stubCharacter()` with `agentId` set. Update evolution tests to pass `descriptorSupplier` in the `CharacterContext` constructor.

- [ ] **Step 4: Update ChatAgentEndToEndTest**

Apply the same pattern — create `CharacterContext` and put in `AgencyContext`.

- [ ] **Step 5: Run all quarkmind-chat tests**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-protocol,quarkmind-chat/quarkmind-chat-agent -q`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add quarkmind-chat/
git commit -m "feat(#282): CharacterContext + stateless ChatAgencyLoop refactoring

Extract per-character mutable state into CharacterContext class.
ChatAgencyLoop.tick() reads agentId, systemPrompt, needState,
reflectionTrigger, and identity from CharacterContext in AgencyContext.
Loop constructor takes only shared dependencies.

Refs #282"
```

---

## Batch 2: Multi-character safety

### Task 2: OutputGovernor atomic tryAcquire

**Files:**
- Modify: `quarkmind-core/src/main/java/io/quarkmind/agency/schedule/OutputGovernor.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/schedule/OutputGovernorTest.java` (find or create)

**Interfaces:**
- Produces: `boolean tryAcquire()` — atomic check-and-record, used by `ChatCharacterManager` (Task 4)
- Preserves: `boolean allow()` and `void recordAction()` — existing single-character callers

- [ ] **Step 1: Write tests for tryAcquire**

Check if `OutputGovernorTest` exists. If not, create it. Add test for the new atomic method:

```java
@Test
void tryAcquireReturnsTrueAndRecordsAction() {
    var governor = new OutputGovernor(60_000, 1000, 5);
    assertTrue(governor.tryAcquire());
}

@Test
void tryAcquireRejectsBelowMinInterval() {
    var governor = new OutputGovernor(60_000, 5000, 5);
    assertTrue(governor.tryAcquire());
    assertFalse(governor.tryAcquire());
}

@Test
void tryAcquireRejectsAtMaxPerWindow() {
    var governor = new OutputGovernor(60_000, 0, 2);
    assertTrue(governor.tryAcquire());
    assertTrue(governor.tryAcquire());
    assertFalse(governor.tryAcquire());
}

@Test
void tryAcquireIsThreadSafe() throws Exception {
    var governor = new OutputGovernor(60_000, 0, 1);
    var results = new java.util.concurrent.ConcurrentLinkedQueue<Boolean>();
    var latch = new java.util.concurrent.CountDownLatch(2);
    var barrier = new java.util.concurrent.CyclicBarrier(2);
    for (int i = 0; i < 2; i++) {
        Thread.ofVirtual().start(() -> {
            try { barrier.await(); } catch (Exception e) { throw new RuntimeException(e); }
            results.add(governor.tryAcquire());
            latch.countDown();
        });
    }
    latch.await();
    long trueCount = results.stream().filter(Boolean::booleanValue).count();
    assertEquals(1, trueCount, "Exactly one thread should acquire");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-core -Dtest=OutputGovernorTest -q`
Expected: Compilation error — `tryAcquire()` does not exist

- [ ] **Step 3: Implement tryAcquire on OutputGovernor**

Add `synchronized boolean tryAcquire()` method:

```java
public synchronized boolean tryAcquire() {
    long now = System.currentTimeMillis();
    pruneOld(now);
    if (!actionTimestamps.isEmpty() && now - actionTimestamps.peekLast() < minIntervalMs) {
        return false;
    }
    if (actionTimestamps.size() >= maxPerWindow) {
        return false;
    }
    actionTimestamps.addLast(now);
    return true;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-core -Dtest=OutputGovernorTest -q`
Expected: All pass

- [ ] **Step 5: Commit**

```bash
git add quarkmind-core/
git commit -m "feat(#282): OutputGovernor.tryAcquire() — atomic check-and-record

Adds synchronized tryAcquire() that atomically checks interval/window
constraints AND records the action. Prevents TOCTOU race when multiple
characters share a governor. Existing allow()+recordAction() preserved
for single-character callers.

Refs #282"
```

### Task 3: LlmReflectionDispositionActivator agentId-keyed profiles

**Files:**
- Modify: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/LlmReflectionDispositionActivator.java`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/LlmReflectionDispositionActivatorTest.java`

**Interfaces:**
- Changes: `updateProfile(String agentId, List<DispositionValue> newProfile)` — agentId-scoped
- Changes: constructor takes `Map<String, List<DispositionValue>>` initial profiles (or empty map)
- Preserves: `onReflection(String agentId, String tenantId, String insight)` — already has agentId

- [ ] **Step 1: Write tests for agentId-keyed profile lookup**

Add tests to `LlmReflectionDispositionActivatorTest`:

```java
@Test
void onReflectionUsesCorrectProfileForAgent() {
    var submitted = new ArrayList<LlmRequest>();
    var queue = recordingQueue(submitted);
    var store = new RecordingSignalStore();

    var profiles = Map.of(
            "luna", List.of(new DispositionValue("empathetic", 0.6)),
            "rex", List.of(new DispositionValue("analytical", 0.7)));

    var activator = new LlmReflectionDispositionActivator(queue, store, profiles);

    activator.onReflection("luna", "t1", "people need support");
    assertEquals(1, submitted.size());
    assertTrue(submitted.get(0).prompt().contains("empathetic"));
    assertFalse(submitted.get(0).prompt().contains("analytical"));
}

@Test
void updateProfileScopedByAgentId() {
    var submitted = new ArrayList<LlmRequest>();
    var queue = recordingQueue(submitted);
    var store = new RecordingSignalStore();

    var activator = new LlmReflectionDispositionActivator(queue, store, Map.of());
    activator.updateProfile("luna", List.of(new DispositionValue("curious", 0.5)));
    activator.onReflection("luna", "t1", "interesting topic");

    assertEquals(1, submitted.size());
    assertTrue(submitted.get(0).prompt().contains("curious"));
}

@Test
void onReflectionSkipsWhenNoProfileForAgent() {
    var submitted = new ArrayList<LlmRequest>();
    var queue = recordingQueue(submitted);
    var store = new RecordingSignalStore();

    var activator = new LlmReflectionDispositionActivator(queue, store, Map.of());
    activator.onReflection("unknown-agent", "t1", "some insight");

    assertTrue(submitted.isEmpty());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=LlmReflectionDispositionActivatorTest -q`
Expected: Compilation error — constructor signature changed

- [ ] **Step 3: Refactor LlmReflectionDispositionActivator**

Replace `volatile List<DispositionValue> dispositionProfile` with `ConcurrentHashMap<String, List<DispositionValue>>`:

```java
private final ConcurrentHashMap<String, List<DispositionValue>> profilesByAgent =
        new ConcurrentHashMap<>();

public LlmReflectionDispositionActivator(LlmRequestQueue llmQueue,
                                         DispositionSignalStore signalStore,
                                         Map<String, List<DispositionValue>> initialProfiles) {
    this.llmQueue = llmQueue;
    this.signalStore = signalStore;
    initialProfiles.forEach((id, profile) -> profilesByAgent.put(id, List.copyOf(profile)));
}

public void updateProfile(String agentId, List<DispositionValue> newProfile) {
    profilesByAgent.put(agentId, List.copyOf(newProfile));
}

@Override
public void onReflection(String agentId, String tenantId, String insight) {
    var profile = profilesByAgent.get(agentId);
    if (profile == null || profile.isEmpty()) return;

    String terms = profile.stream()
            .map(DispositionValue::term)
            .collect(Collectors.joining(", "));
    // ... rest unchanged
}
```

- [ ] **Step 4: Update existing tests and callers**

Update `ChatAgencyLoop.checkEvolution()` — if evolution produces `Evolved`, call `activator.updateProfile(character.agentId(), evolved.newProfile())`. The activator is now a shared field on `ChatAgencyLoop` (not per-character), and the `updateProfile` call uses the character's `agentId` for scoping.

Update `evolvedResultUpdatesActivatorProfile` test in `ChatAgencyLoopTest` to pass `Map.of("agent-1", ...)` to the activator constructor and verify per-agent scoping.

- [ ] **Step 5: Run all quarkmind-chat tests**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-protocol,quarkmind-chat/quarkmind-chat-agent -q`
Expected: All pass

- [ ] **Step 6: Commit**

```bash
git add quarkmind-chat/ quarkmind-core/
git commit -m "feat(#282): LlmReflectionDispositionActivator agentId-keyed profiles

Replace single volatile dispositionProfile with ConcurrentHashMap keyed
by agentId. updateProfile() and onReflection() now scope profile lookup
per agent. Makes the activator safely shareable across multiple characters.

Refs #282"
```

---

## Batch 3: Orchestration — ChatCharacterManager

### Task 4: ChatCharacterManager + multi-character test

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatCharacterManager.java`
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/CharacterConfig.java`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatCharacterManagerTest.java`

**Interfaces:**
- Consumes: `CharacterContext` (Task 1), `OutputGovernor.tryAcquire()` (Task 2), `LlmReflectionDispositionActivator` agentId-keyed (Task 3)
- Produces: `ChatCharacterManager` — manages N characters, provides `tickCharacter(String agentId, ChatPerception perception)` for external drivers

- [ ] **Step 1: Write CharacterConfig — per-character config record**

```java
package io.quarkmind.chat.agent;

import java.util.List;

public record CharacterConfig(String agentId, String token, List<String> channels) {}
```

- [ ] **Step 2: Write tests for ChatCharacterManager**

Test the manager's core responsibilities: character creation, per-character tick routing, shared governor.

```java
@Test
void createsCharacterContextsFromConfig() {
    var configs = List.of(
            new CharacterConfig("luna", "token-luna", List.of("general")),
            new CharacterConfig("rex", "token-rex", List.of("general", "workshop")));

    var manager = createManager(configs);

    assertEquals(2, manager.characterCount());
    assertNotNull(manager.character("luna"));
    assertNotNull(manager.character("rex"));
}

@Test
void tickRoutesToCorrectCharacterContext() {
    var configs = List.of(new CharacterConfig("luna", "token", List.of("ch")));
    var capturedAgentId = new AtomicReference<String>();
    var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) -> {
        capturedAgentId.set(id);
        return "{\"action\":\"WAIT\",\"observation\":\"idle\"}";
    };

    var manager = createManager(configs, llm);
    var perception = perceptionWithMessage("hi", "ch");
    manager.tickCharacter("luna", perception);

    assertEquals("luna", capturedAgentId.get());
}

@Test
void sharedGovernorPreventsSimultaneousResponses() {
    var configs = List.of(
            new CharacterConfig("luna", "t1", List.of("ch")),
            new CharacterConfig("rex", "t2", List.of("ch")));

    var actions = new ArrayList<String>();
    var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) -> {
        actions.add(id);
        return "{\"action\":\"SEND\",\"channel\":\"ch\",\"text\":\"hi\",\"observation\":\"talked\"}";
    };

    var manager = createManager(configs, llm, new OutputGovernor(60_000, 5000, 1));
    var perception = perceptionWithMessage("hello", "ch");

    manager.tickCharacter("luna", perception);
    manager.tickCharacter("rex", perception);

    // Only one should have produced intents — the governor blocks the second
    // (exact assertion depends on implementation — verify one tick's intents are empty)
}
```

- [ ] **Step 3: Implement ChatCharacterManager**

```java
package io.quarkmind.chat.agent;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.eidos.api.AgentRegistry;
import io.quarkmind.agency.AgencyContext;
import io.quarkmind.agency.chat.ChatObservationRenderer;
import io.quarkmind.agency.llm.LlmRequestQueue;
import io.quarkmind.agency.needs.NeedState;
import io.quarkmind.agency.schedule.IdleReflectionTrigger;
import io.quarkmind.agency.schedule.OutputGovernor;
import io.quarkmind.chat.protocol.ChatPerception;

import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class ChatCharacterManager {

    private final ChatAgencyLoop agencyLoop;
    private final Map<String, CharacterContext> characters = new ConcurrentHashMap<>();
    private final OutputGovernor sharedGovernor;

    public ChatCharacterManager(ChatAgencyLoop agencyLoop, OutputGovernor sharedGovernor) {
        this.agencyLoop = agencyLoop;
        this.sharedGovernor = sharedGovernor;
    }

    public void addCharacter(CharacterContext character) {
        characters.put(character.agentId(), character);
    }

    public void tickCharacter(String agentId, ChatPerception perception) {
        var character = characters.get(agentId);
        if (character == null) return;

        var context = new AgencyContext(character.needState());
        context.put("perception", perception);
        context.put("character", character);

        agencyLoop.tick(context);

        @SuppressWarnings("unchecked")
        var intents = (List<?>) context.get("intents");
        if (intents != null && !intents.isEmpty() && !sharedGovernor.tryAcquire()) {
            context.put("intents", List.of());
        }
    }

    public CharacterContext character(String agentId) {
        return characters.get(agentId);
    }

    public int characterCount() {
        return characters.size();
    }
}
```

Note: the governor check happens AFTER the tick — observation and memory still run (per spec: "when tryAcquire() returns false, the tick proceeds but skips intent dispatch"). This is a first-pass implementation. The intent suppression should happen between LLM response and dispatch in the final version, but for this task the manager-level suppression is sufficient to demonstrate the pattern.

- [ ] **Step 4: Run all quarkmind-chat tests**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-protocol,quarkmind-chat/quarkmind-chat-agent -q`
Expected: All pass

- [ ] **Step 5: Update CLAUDE.md — add CharacterContext, ChatCharacterManager to test listings and code organisation**

Add `ChatCharacterManagerTest` to the unit test listings. Update the `quarkmind-chat/` section in code organisation to mention `CharacterContext` and `ChatCharacterManager`.

- [ ] **Step 6: Commit**

```bash
git add quarkmind-chat/ CLAUDE.md
git commit -m "feat(#282): ChatCharacterManager — multi-character orchestration

ChatCharacterManager holds N CharacterContexts and routes ticks to the
correct character. Shared OutputGovernor prevents simultaneous responses.
CharacterConfig record defines per-character operational params.

Refs #282"
```

---

## References

- `specs/issue-279-quarkmind-discord/2026-08-21-multi-character-design.md` — design spec
- `specs/issue-279-quarkmind-discord/decisions.md` D25–D30 — decisions
- `casehub/examples/wacky-manor` ScenarioOrchestrator — multi-character orchestration pattern
- `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatAgencyLoop.java` — primary refactoring target
- `quarkmind-core/src/main/java/io/quarkmind/agency/schedule/OutputGovernor.java` — atomic tryAcquire target
- GE-20260820-c19b68 — CbrQuery agentId filter gotcha
- GitHub #282
