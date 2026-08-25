# Multi-Character Per Server + Blocks Integration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #282 — quarkmind-chat — multi-character per server (D9 v2)
**Issue group:** #279, #280, #281, #282, #283

**Goal:** Support multiple AI characters per Discord server, each with its own bot account, personality, and memory, wired to the blocks social cognition stack (InnerLifeOrchestrator, DriveOrchestrator, CivilityConstraint) instead of custom behavioral code.

**Architecture:** Extract per-character identity state into a `CharacterContext` class. Wire blocks `InnerLifeOrchestrator` as the proactive behavioral layer and `DriveOrchestrator` for motivation (replacing NeedState). `ChatAgencyLoop` becomes stateless with two execution paths: reactive (LLM call for direct responses) and proactive (InnerLifeOrchestrator for unprompted initiation). `ChatCharacterManager` orchestrates N characters with shared CDI services and agentId-scoped isolation.

**Tech Stack:** Java 21, Quarkus CDI, casehub-blocks (social cognition stack), casehub-eidos (AgentDescriptor), casehub-neocortex (CaseMemoryStore), casehub-connectors (Discord Gateway)

## Global Constraints

- `AgencyLoop.tick(AgencyContext)` interface signature is unchanged — `CharacterContext` flows via `context.put("character", ctx)`
- All blocks orchestrators are CDI `@ApplicationScoped` — shared instances, per-agent isolation via `agentId:tenantId` keying in internal `ConcurrentHashMap`
- All memory operations already accept `agentId` as parameter — no neocortex changes needed
- Pre-release platform — breaking changes to quarkmind-chat are free
- `AgencyContext` constructor requires `NeedState` — chat passes an empty `NeedState` (SC2 still uses it; making it optional is out of scope)
- Update blocks dependency to latest snapshot before starting (types exist on blocks main but not in quarkmind's installed jar)

---

## Batch 0: Dependency update

### Task 0: Update blocks dependency + verify types resolve

**Files:**
- Modify: `quarkmind-chat/quarkmind-chat-agent/pom.xml`

**Interfaces:**
- Produces: blocks social cognition types on classpath for all subsequent tasks

- [ ] **Step 1: Install latest blocks snapshot**

```bash
mvn -f /Users/mdproctor/claude/casehub/blocks/pom.xml install -DskipTests -q
```

This publishes the latest blocks snapshot (with InnerLifeOrchestrator, DriveOrchestrator, etc.) to the local Maven repository.

- [ ] **Step 2: Add casehub-blocks dependency to quarkmind-chat-agent pom.xml**

Add to `quarkmind-chat/quarkmind-chat-agent/pom.xml` dependencies:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-blocks</artifactId>
</dependency>
```

Version is managed by the parent POM's `<dependencyManagement>`. If not present there, add with `<version>${casehub-blocks.version}</version>` and define the property.

- [ ] **Step 3: Verify types resolve**

```bash
mvn compile -pl quarkmind-chat/quarkmind-chat-agent -q
```

Expected: compiles successfully. If blocks types are not found, check `mvn dependency:tree -pl quarkmind-chat/quarkmind-chat-agent` for the blocks version.

- [ ] **Step 4: Commit**

```bash
git add quarkmind-chat/quarkmind-chat-agent/pom.xml
git commit -m "feat(#282): add casehub-blocks dependency to quarkmind-chat-agent

Wire blocks social cognition stack (InnerLifeOrchestrator,
DriveOrchestrator, CivilityConstraint) as dependency.

Refs #282"
```

---

## Batch 1: Foundation — CharacterContext + stateless loop

### Task 1: CharacterContext class and ChatAgencyLoop refactoring

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/CharacterContext.java`
- Modify: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatAgencyLoop.java`
- Modify: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatAgencyLoopTest.java`
- Modify: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatAgentEndToEndTest.java`

**Interfaces:**
- Produces: `CharacterContext` — used by `ChatCharacterManager` (Task 3) and all tests
- Produces: `ChatAgencyLoop` with constructor taking `LlmInvoker`, `LlmRequestQueue`, `ObjectMapper`, `ChatPerceptionBridge`, `ChatMemoryFacade`, `InnerLifeOrchestrator`, `DriveOrchestrator`

- [ ] **Step 1: Write CharacterContext class**

Use `ide_create_file`:

```java
package io.quarkmind.chat.agent;

import io.casehub.eidos.api.AgentDescriptor;
import io.quarkmind.agency.chat.BotIdentityDetector;

import java.util.HashSet;
import java.util.Set;
import java.util.function.Supplier;

public class CharacterContext {

    private final String agentId;
    private final String tenantId;
    private final String systemPrompt;
    private final Supplier<AgentDescriptor> descriptorSupplier;
    private final BotIdentityDetector identityDetector;
    private final Set<String> participatedThreadIds = new HashSet<>();

    public CharacterContext(String agentId, String tenantId, String systemPrompt,
                            Supplier<AgentDescriptor> descriptorSupplier,
                            BotIdentityDetector identityDetector) {
        this.agentId = agentId;
        this.tenantId = tenantId;
        this.systemPrompt = systemPrompt;
        this.descriptorSupplier = descriptorSupplier;
        this.identityDetector = identityDetector;
    }

    public String agentId() { return agentId; }
    public String tenantId() { return tenantId; }
    public String systemPrompt() { return systemPrompt; }
    public Supplier<AgentDescriptor> descriptorSupplier() { return descriptorSupplier; }
    public BotIdentityDetector identityDetector() { return identityDetector; }
    public Set<String> participatedThreadIds() { return participatedThreadIds; }
}
```

- [ ] **Step 2: Refactor ChatAgencyLoop — remove per-character fields, add blocks deps**

Remove fields: `systemPrompt`, `agentId`, `tenantId`, `participatedThreadIds`, `consecutiveIdleTicks`, `lastReflectionTimestamp`, `descriptorSupplier`, `dispositionActivator`, `identityDetector`, `reflectionTrigger`, `reflectionOrchestrator`, `evolutionPipeline`.

Remove setter methods: `setSystemPrompt`, `setAgentId`, `setTenantId`, `setDispositionActivator`.

Remove all constructors. Replace with a single constructor taking shared dependencies:

```java
private final LlmInvoker llmInvoker;
private final LlmRequestQueue llmQueue;
private final ObjectMapper mapper;
private final ChatPerceptionBridge perceptionBridge;
private final ChatMemoryFacade memoryFacade;
private final InnerLifeOrchestrator innerLifeOrchestrator;
private final DriveOrchestrator driveOrchestrator;

public ChatAgencyLoop(LlmInvoker llmInvoker, LlmRequestQueue llmQueue,
                      ObjectMapper mapper, ChatPerceptionBridge perceptionBridge,
                      ChatMemoryFacade memoryFacade,
                      InnerLifeOrchestrator innerLifeOrchestrator,
                      DriveOrchestrator driveOrchestrator) {
    this.llmInvoker = llmInvoker;
    this.llmQueue = llmQueue;
    this.mapper = mapper;
    this.perceptionBridge = perceptionBridge;
    this.memoryFacade = memoryFacade;
    this.innerLifeOrchestrator = innerLifeOrchestrator;
    this.driveOrchestrator = driveOrchestrator;
}
```

Imports to add: `io.casehub.blocks.agentic.social.InnerLifeOrchestrator`, `io.casehub.blocks.agentic.social.drive.DriveOrchestrator`.

Refactor `tick()` to read from `CharacterContext` and implement two-path execution:

```java
@Override
public void tick(AgencyContext context) {
    var perception = context.getAs("perception", ChatPerception.class);
    if (perception == null) return;

    var character = context.getAs("character", CharacterContext.class);
    if (character == null) return;

    if (perception.reason() == WakeReason.HEARTBEAT && !perception.hasActivity()) {
        handleProactiveTick(context, character);
        return;
    }

    if (!llmQueue.hasCapacity()) {
        context.put("intents", List.of());
        return;
    }

    handleReactiveTick(context, perception, character);
}
```

Add `handleReactiveTick` — the existing message-handling logic, reading from `character`:

```java
private void handleReactiveTick(AgencyContext context, ChatPerception perception,
                                 CharacterContext character) {
    ChatDeltaReport report = perceptionBridge.buildDelta(
            perception, character.identityDetector(), character.participatedThreadIds());
    String renderedContext = perceptionBridge.renderForLlm(report);

    List<Memory> memories = List.of();
    if (memoryFacade != null) {
        var participantIds = extractParticipantIds(perception);
        memories = memoryFacade.recall(character.agentId(), character.tenantId(),
                renderedContext, participantIds, new PersonalityWeights(Map.of()), Instant.now());
    }

    String userPrompt = buildUserPrompt(renderedContext, character, memories);
    String response = llmInvoker.invoke(character.systemPrompt(), userPrompt, character.agentId());
    ParsedResponse parsed = parseResponse(response);
    context.put("intents", parsed.intents());

    if (memoryFacade != null && parsed.observation() != null && !parsed.observation().isBlank()) {
        var sourceRefs = buildSourceRefs(perception);
        var participantIds = extractParticipantIds(perception);
        memoryFacade.ingest(character.agentId(), character.tenantId(),
                parsed.observation(), sourceRefs, participantIds);
    }
}
```

Add `handleProactiveTick` — delegates to InnerLifeOrchestrator:

```java
private void handleProactiveTick(AgencyContext context, CharacterContext character) {
    if (innerLifeOrchestrator == null) {
        context.put("intents", List.of());
        return;
    }
    var descriptor = character.descriptorSupplier() != null
            ? character.descriptorSupplier().get() : null;
    if (descriptor == null) {
        context.put("intents", List.of());
        return;
    }
    var result = innerLifeOrchestrator.tick(descriptor, null);
    if (result instanceof InnerLifeOrchestrator.InnerLifeTick.Initiated initiated) {
        var intents = List.<ChatIntent>of(
                new ChatIntent.Send(initiated.channelHint(), new ChatContent(initiated.content())));
        context.put("intents", intents);
    } else {
        context.put("intents", List.of());
    }
}
```

Update `buildUserPrompt` to take `CharacterContext` instead of `AgencyContext` for drive state:

```java
private String buildUserPrompt(String renderedContext, CharacterContext character,
                                List<Memory> memories) {
    var sb = new StringBuilder();

    var drives = driveOrchestrator != null
            ? driveOrchestrator.currentDrives(character.agentId(), character.tenantId())
            : java.util.Optional.empty();
    if (drives.isPresent()) {
        var dp = drives.get();
        sb.append("Drives: ").append(dp).append("\n");
    }

    if (!memories.isEmpty()) {
        sb.append("\nWhat I remember:\n");
        for (Memory m : memories) {
            sb.append("- ").append(m.text()).append("\n");
        }
    }

    sb.append("\n").append(renderedContext);
    sb.append("""

            Respond with JSON:
            {"action":"SEND|REPLY|REACT|WAIT","channel":"channel-id","text":"message","emoji":"emoji","messageId":"id-to-react-to","replyTo":"message-id","observation":"what I observed this tick"}
            Always include the observation field. Only include other fields relevant to the action.
            """);
    return sb.toString();
}
```

Remove `checkReflection()`, `checkEvolution()`, and `submitImportanceScoring()` methods — blocks handles these internally.

- [ ] **Step 3: Update ChatAgencyLoopTest — use CharacterContext**

Replace test helpers. Key changes:
- Tests create `CharacterContext` and put it in `AgencyContext` instead of calling setter methods
- Remove all tests for `checkReflection`, `checkEvolution`, `submitImportanceScoring` — these are deleted
- Add tests for the two-path execution model (reactive vs proactive)
- Mock `InnerLifeOrchestrator` and `DriveOrchestrator` — pass `null` for tests that don't need them

New helper methods:

```java
private CharacterContext stubCharacter() {
    return new CharacterContext("chat-agent", "default", "You are a bot.",
            null, stubDetector());
}

private AgencyContext contextWith(ChatPerception perception) {
    return contextWith(perception, stubCharacter());
}

private AgencyContext contextWith(ChatPerception perception, CharacterContext character) {
    var context = new AgencyContext(new NeedState());
    context.put("perception", perception);
    context.put("character", character);
    return context;
}
```

Update `createLoop`:

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

Add proactive tick test:

```java
@Test
void heartbeatWithNoActivityDelegatesToInnerLife() {
    // Verify that heartbeat ticks call innerLifeOrchestrator.tick()
    // and map Initiated result to ChatIntent.Send
}

@Test
void heartbeatWithNoInnerLifeProducesEmptyIntents() {
    var loop = createLoop((s, u, id) -> "");
    var perception = new ChatPerception(Map.of(), Map.of(), WakeReason.HEARTBEAT);
    var ctx = contextWith(perception);
    loop.tick(ctx);
    assertEquals(List.of(), ctx.get("intents"));
}
```

- [ ] **Step 4: Update ChatAgentEndToEndTest — use CharacterContext**

Apply the same pattern — create `CharacterContext` and put in `AgencyContext`.

- [ ] **Step 5: Delete superseded classes and their tests**

Delete production classes (use `ide_refactor_safe_delete` for each):
- `ChatNeedDefinitions.java`
- `LlmReflectionDispositionActivator.java`
- `DispositionAwareReflectionSynthesizer.java`
- `LlmReflectionSynthesizer.java`
- `ChatChannelPacing.java`

Delete test classes:
- `LlmReflectionSynthesizerTest.java`
- `LlmReflectionDispositionActivatorTest.java`
- `DispositionAwareReflectionSynthesizerTest.java`

If `ide_refactor_safe_delete` reports usages, examine each — they should only be test references or the ChatAgencyLoop references already removed in Step 2.

- [ ] **Step 6: Update CLAUDE.md — remove deleted classes from test listings**

Remove `LlmReflectionSynthesizerTest`, `LlmReflectionDispositionActivatorTest`, `DispositionAwareReflectionSynthesizerTest` from the unit test listings. Remove `ChatNeedDefinitionsTest`, `ChatChannelPacingTest` if they appear. Add `CharacterContext` to the code organisation section.

- [ ] **Step 7: Run all quarkmind-chat tests**

```bash
mvn test -pl quarkmind-chat/quarkmind-chat-protocol,quarkmind-chat/quarkmind-chat-agent -q
```

Expected: All tests pass.

- [ ] **Step 8: Run quarkmind-core tests (verify no breakage)**

```bash
mvn test -pl quarkmind-core -q
```

Expected: All tests pass (OutputGovernor, IdleReflectionTrigger, NeedState tests unchanged).

- [ ] **Step 9: Commit**

```bash
git add quarkmind-chat/ quarkmind-core/ CLAUDE.md
git commit -m "feat(#282): CharacterContext + stateless ChatAgencyLoop + blocks integration

Extract per-character state into CharacterContext. Wire InnerLifeOrchestrator
for proactive ticks and DriveOrchestrator for drive-enriched prompts.
Delete custom behavioral code replaced by blocks: ChatNeedDefinitions,
LlmReflectionDispositionActivator, DispositionAwareReflectionSynthesizer,
LlmReflectionSynthesizer, ChatChannelPacing.

Refs #282"
```

---

## Batch 2: Multi-character orchestration

### Task 2: CharacterConfig + ChatCharacterManager

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/CharacterConfig.java`
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatCharacterManager.java`
- Create: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatCharacterManagerTest.java`

**Interfaces:**
- Consumes: `CharacterContext` (Task 1), `ChatAgencyLoop` stateless constructor (Task 1)
- Produces: `ChatCharacterManager` — `addCharacter(CharacterContext)`, `tickCharacter(String agentId, ChatPerception perception)`, `character(String agentId)`, `characterCount()`

- [ ] **Step 1: Write CharacterConfig record**

Use `ide_create_file`:

```java
package io.quarkmind.chat.agent;

import java.util.List;

public record CharacterConfig(String agentId, String token, List<String> channels) {}
```

- [ ] **Step 2: Write ChatCharacterManagerTest**

Use `ide_create_file`:

```java
package io.quarkmind.chat.agent;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ReceivedMessage;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.ChatSender;
import io.quarkmind.agency.chat.BotIdentityDetector;
import io.quarkmind.agency.chat.ChatObservationRenderer;
import io.quarkmind.agency.llm.LlmRequestQueue;
import io.quarkmind.agency.needs.NeedState;
import io.quarkmind.chat.agent.discord.DiscordIdentityDetector;
import io.quarkmind.chat.protocol.ChatPerception;
import io.quarkmind.chat.protocol.WakeReason;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.concurrent.atomic.AtomicReference;

import static org.junit.jupiter.api.Assertions.*;

class ChatCharacterManagerTest {

    private ObjectMapper mapper;
    private LlmRequestQueue llmQueue;

    @BeforeEach
    void setUp() {
        mapper = new ObjectMapper();
        llmQueue = new LlmRequestQueue(10);
    }

    @Test
    void createsAndRetrievesCharacters() {
        var manager = createManager(dummyLlm());
        var luna = stubCharacter("luna", "bot-luna");
        var rex = stubCharacter("rex", "bot-rex");

        manager.addCharacter(luna);
        manager.addCharacter(rex);

        assertEquals(2, manager.characterCount());
        assertSame(luna, manager.character("luna"));
        assertSame(rex, manager.character("rex"));
    }

    @Test
    void tickRoutesToCorrectCharacterContext() {
        var capturedAgentId = new AtomicReference<String>();
        ChatAgencyLoop.LlmInvoker llm = (system, user, id) -> {
            capturedAgentId.set(id);
            return "{\"action\":\"WAIT\",\"observation\":\"idle\"}";
        };

        var manager = createManager(llm);
        manager.addCharacter(stubCharacter("luna", "bot-luna"));

        var msg = new ReceivedMessage(
                new ChatMessageRef(new ChatChannelRef("ch"), "m1"),
                new ChatSender("user1", "User"), "hello",
                java.time.Instant.now(), null);
        var perception = new ChatPerception(
                Map.of("ch", List.of(msg)), Map.of(), WakeReason.MESSAGE);

        manager.tickCharacter("luna", perception);

        assertEquals("luna", capturedAgentId.get());
    }

    @Test
    void tickForUnknownCharacterIsNoOp() {
        var manager = createManager(dummyLlm());
        var perception = new ChatPerception(Map.of(), Map.of(), WakeReason.MESSAGE);
        assertDoesNotThrow(() -> manager.tickCharacter("nonexistent", perception));
    }

    private ChatCharacterManager createManager(ChatAgencyLoop.LlmInvoker llm) {
        var loop = new ChatAgencyLoop(llm, llmQueue, mapper,
                new DefaultChatPerceptionBridge(new ChatObservationRenderer(10)),
                null, null, null);
        return new ChatCharacterManager(loop);
    }

    private CharacterContext stubCharacter(String agentId, String botUserId) {
        return new CharacterContext(agentId, "default", "You are " + agentId + ".",
                null, new DiscordIdentityDetector(botUserId));
    }

    private ChatAgencyLoop.LlmInvoker dummyLlm() {
        return (s, u, id) -> "{\"action\":\"WAIT\",\"observation\":\"idle\"}";
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=ChatCharacterManagerTest -q
```

Expected: Compilation error — `ChatCharacterManager` does not exist.

- [ ] **Step 4: Implement ChatCharacterManager**

Use `ide_create_file`:

```java
package io.quarkmind.chat.agent;

import io.quarkmind.agency.AgencyContext;
import io.quarkmind.agency.needs.NeedState;
import io.quarkmind.chat.protocol.ChatPerception;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class ChatCharacterManager {

    private final ChatAgencyLoop agencyLoop;
    private final Map<String, CharacterContext> characters = new ConcurrentHashMap<>();

    public ChatCharacterManager(ChatAgencyLoop agencyLoop) {
        this.agencyLoop = agencyLoop;
    }

    public void addCharacter(CharacterContext character) {
        characters.put(character.agentId(), character);
    }

    public void tickCharacter(String agentId, ChatPerception perception) {
        var character = characters.get(agentId);
        if (character == null) return;

        var context = new AgencyContext(new NeedState());
        context.put("perception", perception);
        context.put("character", character);

        agencyLoop.tick(context);
    }

    public CharacterContext character(String agentId) {
        return characters.get(agentId);
    }

    public int characterCount() {
        return characters.size();
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=ChatCharacterManagerTest -q
```

Expected: All pass.

- [ ] **Step 6: Run all quarkmind-chat tests**

```bash
mvn test -pl quarkmind-chat/quarkmind-chat-protocol,quarkmind-chat/quarkmind-chat-agent -q
```

Expected: All pass.

- [ ] **Step 7: Update CLAUDE.md — add ChatCharacterManagerTest to test listings, update code organisation**

Add `ChatCharacterManagerTest` to quarkmind-chat unit test listings. Add `CharacterConfig`, `ChatCharacterManager` to the `quarkmind-chat/` code organisation section.

- [ ] **Step 8: Commit**

```bash
git add quarkmind-chat/ CLAUDE.md
git commit -m "feat(#282): ChatCharacterManager — multi-character orchestration

ChatCharacterManager holds N CharacterContexts and routes ticks to the
correct character. CharacterConfig record defines per-character params.
Each character gets its own identity, system prompt, and thread tracking.

Refs #282"
```

---

## References

- `specs/issue-279-quarkmind-discord/2026-08-21-multi-character-design.md` — design spec (R2)
- `specs/issue-279-quarkmind-discord/decisions.md` D25–D35 — decisions
- `casehub/examples/wacky-manor` ScenarioOrchestrator — multi-character orchestration pattern
- `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatAgencyLoop.java` — primary refactoring target
- `quarkmind-core/src/main/java/io/quarkmind/agency/AgencyContext.java` — context container
- blocks `InnerLifeOrchestrator` (`io.casehub.blocks.agentic.social`) — proactive behavioral compositor
- blocks `DriveOrchestrator` (`io.casehub.blocks.agentic.social.drive`) — drive state management
- blocks `CivilityConstraint` (`io.casehub.blocks.agentic.social`) — pacing constraint chain
- GE-20260820-c19b68 — CbrQuery agentId filter gotcha
- GitHub #282
