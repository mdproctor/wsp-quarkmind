# Memory Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #280 — quarkmind-chat memory integration (D5 four-layer memory)
**Issue group:** #279, #280, #281, #282, #283

**Goal:** Wire episodic, semantic, and relationship memory into ChatAgencyLoop so the chat agent remembers conversations, learns about people, and develops insights over time.

**Architecture:** ChatMemoryFacade in quarkmind-chat-agent composes neocortex CaseMemoryStore + optional GraphCaseMemoryStore behind recall() + ingest(). ChatAgencyLoop gains memory retrieval before LLM, observation ingestion after, and async importance scoring via LlmRequestQueue. IdleReflectionTrigger + LlmReflectionSynthesizer handle memory consolidation during idle periods.

**Tech Stack:** Java 21, Quarkus, casehub-neocortex (CaseMemoryStore, ExperienceQuery, PersonalityWeightedRetrieval, ReflectionService), casehub-eidos (personality), quarkmind-core (LlmRequestQueue, AgencyLoop)

## Global Constraints

- Pre-release platform — breaking changes are free
- IntelliJ MCP mandatory for all Java file operations
- `ExperienceEvents.DOMAIN` ("experience") for episodic memories — not a custom domain
- `ReflectionEvents.DOMAIN` ("reflection") for reflections — existing domain
- Importance range [0.0, 1.0] — matches MemoryInput validation
- Unscored memories default to importance 1.0 in retrieval re-rankers
- No `@QuarkusTest` for tests that can be plain JUnit

---

## Batch 1: Foundation — neocortex updateImportance + quarkmind-core enhancements

### Task 1: updateImportance() on CaseMemoryStore (cross-repo: casehub-neocortex)

**Files:**
- Modify: `casehub-neocortex/memory-api/src/main/java/io/casehub/neocortex/memory/CaseMemoryStore.java`
- Modify: `casehub-neocortex/memory-jpa/src/main/java/io/casehub/neocortex/memory/jpa/JpaMemoryStore.java`
- Modify: `casehub-neocortex/memory-sqlite/src/main/java/io/casehub/neocortex/memory/sqlite/SqliteMemoryStore.java`
- Modify: `casehub-neocortex/memory-inmem/src/main/java/io/casehub/neocortex/memory/inmem/InMemoryMemoryStore.java`
- Modify: `casehub-neocortex/memory/src/main/java/io/casehub/memory/runtime/NoOpCaseMemoryStore.java`
- Modify: `casehub-neocortex/memory/src/main/java/io/casehub/memory/runtime/CaseEnrichmentDecorator.java`
- Modify: `casehub-neocortex/memory-mem0/src/main/java/io/casehub/neocortex/memory/mem0/Mem0CaseMemoryStore.java`
- Modify: `casehub-neocortex/memory-graphiti/src/main/java/io/casehub/neocortex/memory/graphiti/GraphitiCaseMemoryStore.java`
- Modify: `casehub-neocortex/memory-testing/src/main/java/io/casehub/memory/testing/CaseMemoryStoreContractTest.java`
- Test: `casehub-neocortex/memory-testing/src/main/java/io/casehub/memory/testing/CaseMemoryStoreContractTest.java`

**Interfaces:**
- Produces: `CaseMemoryStore.updateImportance(String memoryId, String tenantId, double importance)` — default method with range validation; implementations override

- [ ] **Step 1: Add contract tests to CaseMemoryStoreContractTest**

Add three test methods to `CaseMemoryStoreContractTest`:

```java
@Test
void updateImportance_updatesStoredScore() {
    var id = store().store(input("important event"));
    store().updateImportance(id, TENANT, 0.85);
    var results = store().query(new MemoryQuery(
        List.of("entity-1"), DOMAIN, TENANT, null, null, 10, null, MemoryOrder.CHRONOLOGICAL));
    assertFalse(results.isEmpty());
    assertEquals(0.85, results.get(0).importance(), 0.001);
}

@Test
void updateImportance_unknownMemory_noError() {
    store().updateImportance("nonexistent-id", TENANT, 0.5);
}

@Test
void updateImportance_outOfRange_throwsIllegalArgument() {
    assertThrows(IllegalArgumentException.class, () ->
        store().updateImportance("any", TENANT, 1.5));
    assertThrows(IllegalArgumentException.class, () ->
        store().updateImportance("any", TENANT, -0.1));
}
```

- [ ] **Step 2: Run InMemoryMemoryStore contract tests to verify they fail**

Run: `mvn test -pl memory-inmem -Dtest=InMemoryMemoryStoreTest -q` (from neocortex root)
Expected: FAIL — `updateImportance` throws `UnsupportedOperationException` from default method

- [ ] **Step 3: Add default method to CaseMemoryStore interface**

Use `ide_insert_member` to add to `CaseMemoryStore.java`:

```java
default void updateImportance(String memoryId, String tenantId, double importance) {
    if (importance < 0.0 || importance > 1.0) {
        throw new IllegalArgumentException("importance must be in [0, 1], got " + importance);
    }
}
```

The default silently ignores the update (no-op after validation). Implementations that support persistence override this.

- [ ] **Step 4: Implement in InMemoryMemoryStore**

Use `ide_insert_member` to add to `InMemoryMemoryStore.java`:

```java
@Override
public void updateImportance(String memoryId, String tenantId, double importance) {
    CaseMemoryStore.super.updateImportance(memoryId, tenantId, importance);
    buckets.values().stream()
        .flatMap(List::stream)
        .filter(e -> memoryId.equals(e.memoryId()) && tenantId.equals(e.tenantId()))
        .findFirst()
        .ifPresent(e -> e.setImportance(importance));
}
```

Note: Check `InMemoryMemoryStore`'s internal storage structure — it may use a different field access pattern. The key is: find the entry by memoryId + tenantId, set its importance.

- [ ] **Step 5: Run InMemoryMemoryStore contract tests**

Run: `mvn test -pl memory-inmem -Dtest=InMemoryMemoryStoreTest -q`
Expected: PASS — all three new tests pass

- [ ] **Step 6: Implement in JpaMemoryStore**

Use `ide_insert_member` to add to `JpaMemoryStore.java`:

```java
@Override
@Transactional
public void updateImportance(String memoryId, String tenantId, double importance) {
    CaseMemoryStore.super.updateImportance(memoryId, tenantId, importance);
    entityManager.createQuery(
        "UPDATE MemoryEntry m SET m.importance = :importance " +
        "WHERE m.id = :memoryId AND m.tenantId = :tenantId")
        .setParameter("importance", importance)
        .setParameter("memoryId", memoryId)
        .setParameter("tenantId", tenantId)
        .executeUpdate();
}
```

- [ ] **Step 7: Implement in SqliteMemoryStore**

Use `ide_insert_member`. Similar to JPA but using the SQLite-specific column name (`memory_id`):

```java
@Override
public void updateImportance(String memoryId, String tenantId, double importance) {
    CaseMemoryStore.super.updateImportance(memoryId, tenantId, importance);
    // Use the SQLite-specific update pattern matching the store's existing methods
}
```

- [ ] **Step 8: Implement CaseEnrichmentDecorator delegation**

Use `ide_insert_member` to add to `CaseEnrichmentDecorator.java`:

```java
@Override
public void updateImportance(String memoryId, String tenantId, double importance) {
    delegate.updateImportance(memoryId, tenantId, importance);
}
```

This is critical — without it, the decorator inherits the interface default (no-op after validation) and swallows CDI-injected calls even when the underlying store supports persistence.

- [ ] **Step 9: Implement no-op in NoOpCaseMemoryStore, Mem0CaseMemoryStore, GraphitiCaseMemoryStore**

NoOp: inherits default (silent no-op) — no override needed.

Mem0 and Graphiti: use `ide_insert_member` to add warning-log no-ops:

```java
// Mem0CaseMemoryStore
@Override
public void updateImportance(String memoryId, String tenantId, double importance) {
    CaseMemoryStore.super.updateImportance(memoryId, tenantId, importance);
    LOG.warnv("updateImportance not supported by Mem0 — memory {0} importance unchanged", memoryId);
}

// GraphitiCaseMemoryStore
@Override
public void updateImportance(String memoryId, String tenantId, double importance) {
    CaseMemoryStore.super.updateImportance(memoryId, tenantId, importance);
    LOG.warnv("updateImportance not supported by Graphiti — memory {0} importance unchanged", memoryId);
}
```

- [ ] **Step 10: Run full neocortex test suite**

Run: `mvn test -q` (from neocortex root)
Expected: PASS

- [ ] **Step 11: Install neocortex SNAPSHOT**

Run: `mvn install -DskipTests -q` (from neocortex root)

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/neocortex add -A
git -C /Users/mdproctor/claude/casehub/neocortex commit -m "feat: add updateImportance() to CaseMemoryStore + all implementations Refs casehubio/quarkmind#280"
```

### Task 2: LlmRequest response handler + IdleReflectionTrigger (quarkmind-core)

**Files:**
- Modify: `quarkmind-core/src/main/java/io/quarkmind/agency/llm/LlmRequest.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/schedule/IdleReflectionTrigger.java`
- Create: `quarkmind-core/src/test/java/io/quarkmind/agency/schedule/IdleReflectionTriggerTest.java`

**Interfaces:**
- Produces: `LlmRequest(String prompt, LlmPriority priority, Map<String, Object> metadata, Consumer<String> responseHandler)` — backward-compatible 3-arg constructor retained
- Produces: `IdleReflectionTrigger.accumulate(double importance)`, `shouldReflect(int consecutiveIdleTicks)`, `reset()`

- [ ] **Step 1: Write IdleReflectionTrigger test**

Create `quarkmind-core/src/test/java/io/quarkmind/agency/schedule/IdleReflectionTriggerTest.java`:

```java
package io.quarkmind.agency.schedule;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class IdleReflectionTriggerTest {

    @Test
    void doesNotReflectBelowThreshold() {
        var trigger = new IdleReflectionTrigger(3.0, 5);
        trigger.accumulate(0.5);
        trigger.accumulate(0.5);
        assertFalse(trigger.shouldReflect(10));
    }

    @Test
    void reflectsWhenThresholdCrossedAndIdle() {
        var trigger = new IdleReflectionTrigger(3.0, 5);
        trigger.accumulate(1.0);
        trigger.accumulate(1.0);
        trigger.accumulate(1.0);
        assertFalse(trigger.shouldReflect(3));
        assertTrue(trigger.shouldReflect(5));
    }

    @Test
    void resetClearsAccumulator() {
        var trigger = new IdleReflectionTrigger(3.0, 5);
        trigger.accumulate(1.0);
        trigger.accumulate(1.0);
        trigger.accumulate(1.0);
        trigger.reset();
        assertFalse(trigger.shouldReflect(10));
    }

    @Test
    void doesNotReflectWhenNotIdle() {
        var trigger = new IdleReflectionTrigger(3.0, 5);
        trigger.accumulate(5.0);
        assertFalse(trigger.shouldReflect(2));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-core -Dtest=IdleReflectionTriggerTest -q`
Expected: FAIL — class not found

- [ ] **Step 3: Implement IdleReflectionTrigger**

Create `quarkmind-core/src/main/java/io/quarkmind/agency/schedule/IdleReflectionTrigger.java`:

```java
package io.quarkmind.agency.schedule;

public class IdleReflectionTrigger {

    private double accumulatedImportance = 0.0;
    private final double threshold;
    private final int idleHeartbeats;

    public IdleReflectionTrigger(double threshold, int idleHeartbeats) {
        this.threshold = threshold;
        this.idleHeartbeats = idleHeartbeats;
    }

    public void accumulate(double importance) {
        accumulatedImportance += importance;
    }

    public boolean shouldReflect(int consecutiveIdleTicks) {
        return accumulatedImportance >= threshold && consecutiveIdleTicks >= idleHeartbeats;
    }

    public void reset() {
        accumulatedImportance = 0.0;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-core -Dtest=IdleReflectionTriggerTest -q`
Expected: PASS

- [ ] **Step 5: Update LlmRequest with responseHandler**

Use `ide_replace_member` on `LlmRequest.java` to replace the entire record:

```java
package io.quarkmind.agency.llm;

import java.util.Map;
import java.util.function.Consumer;

public record LlmRequest(String prompt, LlmPriority priority,
                          Map<String, Object> metadata,
                          Consumer<String> responseHandler) {
    public LlmRequest(String prompt, LlmPriority priority, Map<String, Object> metadata) {
        this(prompt, priority, metadata, null);
    }
    public LlmRequest {
        metadata = metadata != null ? Map.copyOf(metadata) : Map.of();
    }
}
```

- [ ] **Step 6: Verify existing tests still pass**

Run: `mvn test -pl quarkmind-core -q`
Expected: PASS — backward compatible (existing 3-arg constructor still works)

- [ ] **Step 7: Commit**

```bash
git add quarkmind-core/
git commit -m "feat(#280): LlmRequest responseHandler + IdleReflectionTrigger Refs #280"
```

---

## Batch 2: ChatMemoryFacade + ChatAgencyLoop integration

### Task 3: ChatMemoryFacade — recall, ingest, scoreImportance

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatMemoryFacade.java`
- Create: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatMemoryFacadeTest.java`
- Modify: `quarkmind-chat/quarkmind-chat-agent/pom.xml` (add neocortex memory-api dependency if not present)

**Interfaces:**
- Consumes: `CaseMemoryStore.store()`, `.query()`, `.updateImportance()` from Task 1; `GraphCaseMemoryStore.graphQuery()`; `ExperienceQuery.search()`; `PersonalityWeightedRetrieval.reweight()`
- Produces: `ChatMemoryFacade.recall(agentId, tenantId, conversationContext, participantIds, weights, now)` → `List<Memory>`; `ChatMemoryFacade.ingest(agentId, tenantId, observationText, sourceRefs, participantIds)` → `String memoryId`; `ChatMemoryFacade.scoreImportance(memoryId, tenantId, importance)`

- [ ] **Step 1: Write ChatMemoryFacadeTest**

Create `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatMemoryFacadeTest.java`:

```java
package io.quarkmind.chat.agent;

import io.casehub.neocortex.memory.*;
import io.casehub.neocortex.memory.experience.ExperienceEvents;
import io.casehub.neocortex.memory.personality.PersonalityWeights;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;

class ChatMemoryFacadeTest {

    @Test
    void ingestStoresObservationWithExperienceDomain() {
        var store = new RecordingMemoryStore();
        var facade = new ChatMemoryFacade(store, store, false);

        var refs = Map.of("source.channelId", "ch-1", "source.firstMessageId", "m1",
                "source.lastMessageId", "m3");
        var participants = Set.of("user-123");

        String memoryId = facade.ingest("agent-1", "tenant-1",
                "Talked to Bob about ML", refs, participants);

        assertNotNull(memoryId);
        assertEquals(1, store.stored.size());
        var input = store.stored.get(0);
        assertEquals(ExperienceEvents.DOMAIN, input.domain());
        assertEquals("agent-1", input.entityId());
        assertNull(input.importance());
        assertEquals("ch-1", input.attributes().get("source.channelId"));
        assertTrue(input.attributes().containsKey("participant.user-123"));
    }

    @Test
    void scoreImportanceDelegatesToUpdateImportance() {
        var store = new RecordingMemoryStore();
        var facade = new ChatMemoryFacade(store, store, false);

        facade.scoreImportance("mem-1", "tenant-1", 0.75);

        assertEquals("mem-1", store.lastUpdatedMemoryId);
        assertEquals(0.75, store.lastUpdatedImportance, 0.001);
    }

    @Test
    void recallQueriesStoreWithSemanticSearch() {
        var store = new RecordingMemoryStore();
        store.queryResults = List.of(
            new Memory("m1", "agent-1", ExperienceEvents.DOMAIN, "t1", null,
                "Talked about ML", Map.of(), Instant.now().minusSeconds(3600), 0.8));
        var facade = new ChatMemoryFacade(store, store, false);

        var results = facade.recall("agent-1", "t1", "machine learning",
                Set.of(), PersonalityWeights.uniform(), Instant.now());

        assertFalse(results.isEmpty());
        assertNotNull(store.lastQuery);
    }

    @Test
    void recallSkipsGraphQueryWhenNotAvailable() {
        var store = new RecordingMemoryStore();
        store.queryResults = List.of();
        var facade = new ChatMemoryFacade(store, store, false);

        facade.recall("agent-1", "t1", "hello",
                Set.of("user-1"), PersonalityWeights.uniform(), Instant.now());

        assertFalse(store.graphQueried);
    }

    static class RecordingMemoryStore implements CaseMemoryStore, GraphCaseMemoryStore {
        List<MemoryInput> stored = new ArrayList<>();
        MemoryQuery lastQuery;
        List<Memory> queryResults = List.of();
        String lastUpdatedMemoryId;
        double lastUpdatedImportance;
        boolean graphQueried;
        int storeCounter;

        @Override public String store(MemoryInput input) {
            stored.add(input);
            return "mem-" + (++storeCounter);
        }
        @Override public List<Memory> query(MemoryQuery query) {
            lastQuery = query;
            return queryResults;
        }
        @Override public void updateImportance(String id, String t, double imp) {
            lastUpdatedMemoryId = id;
            lastUpdatedImportance = imp;
        }
        @Override public int erase(EraseRequest r) { return 0; }
        @Override public List<Memory> graphQuery(GraphMemoryQuery q) {
            graphQueried = true;
            return List.of();
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=ChatMemoryFacadeTest -q`
Expected: FAIL — ChatMemoryFacade class not found

- [ ] **Step 3: Check if neocortex memory-api dependency exists in quarkmind-chat-agent pom.xml**

Read `quarkmind-chat/quarkmind-chat-agent/pom.xml` and check for `casehub-neocortex-memory-api`. If missing, add it.

- [ ] **Step 4: Implement ChatMemoryFacade**

Create `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatMemoryFacade.java`:

```java
package io.quarkmind.chat.agent;

import io.casehub.neocortex.memory.*;
import io.casehub.neocortex.memory.experience.ExperienceEvents;
import io.casehub.neocortex.memory.experience.ExperienceQuery;
import io.casehub.neocortex.memory.personality.PersonalityWeightedRetrieval;
import io.casehub.neocortex.memory.personality.PersonalityWeights;

import java.time.Instant;
import java.util.*;
import java.util.stream.Stream;

public class ChatMemoryFacade {

    private static final int DEFAULT_MAX_MEMORIES = 15;

    private final CaseMemoryStore memoryStore;
    private final GraphCaseMemoryStore graphStore;
    private final boolean graphAvailable;
    private int maxMemories = DEFAULT_MAX_MEMORIES;

    public ChatMemoryFacade(CaseMemoryStore memoryStore, GraphCaseMemoryStore graphStore,
                            boolean graphAvailable) {
        this.memoryStore = memoryStore;
        this.graphStore = graphStore;
        this.graphAvailable = graphAvailable;
    }

    public List<Memory> recall(String agentId, String tenantId,
                               String conversationContext, Set<String> participantIds,
                               PersonalityWeights weights, Instant now) {
        var episodic = memoryStore.query(
                ExperienceQuery.search(agentId, tenantId, conversationContext));

        List<Memory> relationship = List.of();
        if (graphAvailable && !participantIds.isEmpty()) {
            relationship = participantIds.stream()
                    .flatMap(pid -> graphStore.graphQuery(
                            GraphMemoryQuery.forEntity(pid,
                                    ExperienceEvents.DOMAIN, tenantId,
                                    "what do I know about this person?")).stream())
                    .toList();
        }

        var merged = Stream.concat(episodic.stream(), relationship.stream()).toList();
        var ranked = PersonalityWeightedRetrieval.reweight(merged, weights, now);
        return ranked.size() <= maxMemories ? ranked : ranked.subList(0, maxMemories);
    }

    public String ingest(String agentId, String tenantId,
                         String observationText, Map<String, String> sourceRefs,
                         Set<String> participantIds) {
        var attrs = new HashMap<>(sourceRefs);
        for (String pid : participantIds) {
            attrs.put("participant." + pid, pid);
        }
        var input = new MemoryInput(agentId, ExperienceEvents.DOMAIN, tenantId,
                null, observationText, attrs, null);
        return memoryStore.store(input);
    }

    public void scoreImportance(String memoryId, String tenantId, double importance) {
        memoryStore.updateImportance(memoryId, tenantId, importance);
    }

    public void setMaxMemories(int max) { this.maxMemories = max; }
}
```

- [ ] **Step 5: Run tests**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=ChatMemoryFacadeTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add quarkmind-chat/
git commit -m "feat(#280): ChatMemoryFacade — recall, ingest, scoreImportance Refs #280"
```

### Task 4: ChatAgencyLoop memory integration

**Files:**
- Modify: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatAgencyLoop.java`
- Modify: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatAgencyLoopTest.java`

**Interfaces:**
- Consumes: `ChatMemoryFacade.recall()`, `.ingest()`, `.scoreImportance()` from Task 3; `LlmRequest(prompt, priority, metadata, responseHandler)` from Task 2; `IdleReflectionTrigger.accumulate()`, `.shouldReflect()`, `.reset()` from Task 2; `ReflectionOrchestrator.reflect()` from neocortex
- Produces: Updated `ChatAgencyLoop.tick()` with memory retrieval, observation ingestion, async importance scoring, and reflection check on heartbeat

- [ ] **Step 1: Write test for memory retrieval in tick**

Add to `ChatAgencyLoopTest.java`:

```java
@Test
void tickRetrievesMemoriesBeforeLlm() {
    var recallCalled = new AtomicBoolean(false);
    var facade = new ChatMemoryFacade(
            new ChatMemoryFacadeTest.RecordingMemoryStore(),
            new ChatMemoryFacadeTest.RecordingMemoryStore(), false) {
        @Override
        public List<Memory> recall(String agentId, String tenantId,
                String ctx, Set<String> pids, PersonalityWeights w, Instant now) {
            recallCalled.set(true);
            return List.of();
        }
    };

    var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) ->
            "{\"action\":\"WAIT\",\"observation\":\"Saw a greeting\"}";
    var loop = createLoopWithMemory(llm, facade);
    loop.tick(contextWith(perceptionWithMessage("hi", "ch")));
    assertTrue(recallCalled.get());
}
```

- [ ] **Step 2: Write test for observation ingestion after tick**

```java
@Test
void tickIngestsObservationFromLlmResponse() {
    var store = new ChatMemoryFacadeTest.RecordingMemoryStore();
    var facade = new ChatMemoryFacade(store, store, false);

    var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) ->
            "{\"action\":\"SEND\",\"channel\":\"ch-1\",\"text\":\"hi\",\"observation\":\"Greeted the channel\"}";
    var loop = createLoopWithMemory(llm, facade);
    loop.tick(contextWith(perceptionWithMessage("hello", "ch-1")));

    assertEquals(1, store.stored.size());
    assertTrue(store.stored.get(0).text().contains("Greeted the channel"));
}
```

- [ ] **Step 3: Write test for async importance scoring submission**

```java
@Test
void tickSubmitsAsyncImportanceScoring() {
    var store = new ChatMemoryFacadeTest.RecordingMemoryStore();
    var facade = new ChatMemoryFacade(store, store, false);
    var submitted = new ArrayList<LlmRequest>();
    var queue = new LlmRequestQueue() {
        @Override public void submit(LlmRequest r) { submitted.add(r); }
        @Override public int pendingCount() { return 0; }
        @Override public boolean hasCapacity() { return true; }
    };

    var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) ->
            "{\"action\":\"WAIT\",\"observation\":\"Nothing happened\"}";
    var loop = new ChatAgencyLoop(llm, detector, queue, mapper,
            new DefaultChatPerceptionBridge(new ChatObservationRenderer(10)),
            facade, new IdleReflectionTrigger(3.0, 5), null);
    loop.tick(contextWith(perceptionWithMessage("hey", "ch")));

    assertTrue(submitted.stream().anyMatch(r ->
            r.priority() == LlmPriority.LOW && r.responseHandler() != null));
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=ChatAgencyLoopTest -q`
Expected: FAIL — new constructor and methods don't exist

- [ ] **Step 5: Update ChatAgencyLoop with memory integration**

Modify `ChatAgencyLoop.java`:

1. Add constructor parameter: `ChatMemoryFacade facade`, `IdleReflectionTrigger reflectionTrigger`, `ReflectionOrchestrator reflectionOrchestrator`
2. Add backward-compatible constructor (no memory — passes null facade)
3. In `tick()`:
   - After perception bridge, call `facade.recall()` if facade is not null
   - Add memories to prompt via new `formatMemories()` method
   - Parse `observation` field from LLM response
   - Call `facade.ingest()` with observation + source refs
   - Submit async importance scoring via `llmQueue.submit(new LlmRequest(prompt, LOW, metadata, responseHandler))`
   - On heartbeat: check `reflectionTrigger.shouldReflect()`, invoke `reflectionOrchestrator.reflect()` if true
4. Update `buildUserPrompt()` to accept `List<Memory>` and format "What I remember" section
5. Update `parseResponse()` to extract `observation` field

- [ ] **Step 6: Run all ChatAgencyLoop tests**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=ChatAgencyLoopTest -q`
Expected: PASS — both old (backward-compatible) and new tests

- [ ] **Step 7: Commit**

```bash
git add quarkmind-chat/
git commit -m "feat(#280): ChatAgencyLoop memory integration — recall, ingest, scoring Refs #280"
```

---

## Batch 3: Reflection synthesizer + end-to-end test

### Task 5: LlmReflectionSynthesizer

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/LlmReflectionSynthesizer.java`
- Create: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/LlmReflectionSynthesizerTest.java`

**Interfaces:**
- Consumes: `ReflectionSynthesizer.synthesize()` SPI, `ChatAgencyLoop.LlmInvoker`, `Memory`, `ReflectionEvent`
- Produces: `LlmReflectionSynthesizer` — `@Alternative @Priority(1)` CDI bean implementing `ReflectionSynthesizer`

- [ ] **Step 1: Write LlmReflectionSynthesizerTest**

```java
package io.quarkmind.chat.agent;

import io.casehub.neocortex.memory.*;
import io.casehub.neocortex.memory.experience.ExperienceEvents;
import io.casehub.neocortex.memory.reflection.ReflectionEvent;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class LlmReflectionSynthesizerTest {

    @Test
    void synthesizesReflectionFromSourceMemories() {
        var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) ->
                "[{\"insight\":\"Bob is deeply interested in NLP and transformer architectures\"}]";

        var synthesizer = new LlmReflectionSynthesizer(llm, "agent-1");
        var sources = List.of(
                new Memory("m1", "agent-1", ExperienceEvents.DOMAIN, "t1", null,
                        "Talked to Bob about transformers", Map.of(),
                        Instant.now().minusSeconds(3600), 0.8),
                new Memory("m2", "agent-1", ExperienceEvents.DOMAIN, "t1", null,
                        "Bob asked about attention mechanisms", Map.of(),
                        Instant.now().minusSeconds(1800), 0.7));

        var events = synthesizer.synthesize("agent-1", "t1", sources, 1);
        assertFalse(events.isEmpty());
        assertEquals("agent-1", events.get(0).agentId());
        assertEquals("t1", events.get(0).tenantId());
        assertFalse(events.get(0).insight().isBlank());
        assertEquals(1, events.get(0).level());
    }

    @Test
    void returnsEmptyOnLlmFailure() {
        var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) -> {
            throw new RuntimeException("LLM unavailable");
        };

        var synthesizer = new LlmReflectionSynthesizer(llm, "agent-1");
        var sources = List.of(
                new Memory("m1", "agent-1", ExperienceEvents.DOMAIN, "t1", null,
                        "Something happened", Map.of(), Instant.now(), 0.5));

        var events = synthesizer.synthesize("agent-1", "t1", sources, 1);
        assertTrue(events.isEmpty());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=LlmReflectionSynthesizerTest -q`
Expected: FAIL — class not found

- [ ] **Step 3: Implement LlmReflectionSynthesizer**

Create `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/LlmReflectionSynthesizer.java`:

```java
package io.quarkmind.chat.agent;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.neocortex.memory.Memory;
import io.casehub.neocortex.memory.reflection.ReflectionEvent;
import io.casehub.neocortex.memory.reflection.ReflectionSynthesizer;
import org.jboss.logging.Logger;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class LlmReflectionSynthesizer implements ReflectionSynthesizer {

    private static final Logger LOG = Logger.getLogger(LlmReflectionSynthesizer.class);
    private static final ObjectMapper MAPPER = new ObjectMapper();

    private final ChatAgencyLoop.LlmInvoker llmInvoker;
    private final String agentId;

    public LlmReflectionSynthesizer(ChatAgencyLoop.LlmInvoker llmInvoker, String agentId) {
        this.llmInvoker = llmInvoker;
        this.agentId = agentId;
    }

    @Override
    public List<ReflectionEvent> synthesize(String agentId, String tenantId,
                                            List<Memory> sources, int targetLevel) {
        try {
            String sourceText = sources.stream()
                    .map(m -> "- " + m.text())
                    .collect(Collectors.joining("\n"));

            String prompt = "Given these recent experiences:\n" + sourceText +
                    "\n\nWhat patterns or insights do you notice? " +
                    "Produce 1-3 generalized reflections as a JSON array: " +
                    "[{\"insight\":\"...\"}]";

            String response = llmInvoker.invoke("You are a reflective observer.", prompt, agentId);
            JsonNode root = MAPPER.readTree(response);

            var events = new ArrayList<ReflectionEvent>();
            var sourceIds = sources.stream().map(Memory::memoryId).toList();

            if (root.isArray()) {
                for (JsonNode node : root) {
                    String insight = node.has("insight") ? node.get("insight").asText() : null;
                    if (insight != null && !insight.isBlank()) {
                        events.add(new ReflectionEvent(agentId, tenantId, null,
                                insight, targetLevel, sourceIds, null, Map.of()));
                    }
                }
            }
            return events;
        } catch (Exception e) {
            LOG.warn("Reflection synthesis failed", e);
            return List.of();
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=LlmReflectionSynthesizerTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add quarkmind-chat/
git commit -m "feat(#280): LlmReflectionSynthesizer — LLM-based memory consolidation Refs #280"
```

### Task 6: End-to-end test + CLAUDE.md update

**Files:**
- Modify: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatAgentEndToEndTest.java`
- Modify: `CLAUDE.md` (add new test listings)

**Interfaces:**
- Consumes: All components from Tasks 1–5

- [ ] **Step 1: Add memory integration end-to-end test**

Add to `ChatAgentEndToEndTest.java`:

```java
@Test
void memoryIntegrationEndToEnd() {
    var store = new ChatMemoryFacadeTest.RecordingMemoryStore();
    var facade = new ChatMemoryFacade(store, store, false);
    var reflectionTrigger = new IdleReflectionTrigger(3.0, 5);
    var scoringRequests = new ArrayList<LlmRequest>();
    var queue = new LlmRequestQueue() {
        @Override public void submit(LlmRequest r) { scoringRequests.add(r); }
        @Override public int pendingCount() { return 0; }
        @Override public boolean hasCapacity() { return true; }
    };

    var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) ->
            "{\"action\":\"SEND\",\"channel\":\"ch-1\",\"text\":\"I remember!\","
            + "\"observation\":\"Bob asked about ML. I helped.\"}";

    var loop = new ChatAgencyLoop(llm, stubDetector(), queue, new ObjectMapper(),
            new DefaultChatPerceptionBridge(new ChatObservationRenderer(10)),
            facade, reflectionTrigger, null);
    loop.setSystemPrompt("You are a friendly bot.");
    loop.setAgentId("agent-1");

    // Tick 1: first conversation
    loop.tick(contextWith(perceptionWithMessage("tell me about ML", "ch-1")));

    // Verify observation was stored
    assertEquals(1, store.stored.size());
    assertTrue(store.stored.get(0).text().contains("Bob asked about ML"));
    assertNull(store.stored.get(0).importance());

    // Verify async scoring was submitted
    assertTrue(scoringRequests.stream()
            .anyMatch(r -> r.priority() == LlmPriority.LOW && r.responseHandler() != null));

    // Simulate scoring callback
    var scoringReq = scoringRequests.stream()
            .filter(r -> r.responseHandler() != null).findFirst().orElseThrow();
    scoringReq.responseHandler().accept("0.7");
    assertEquals("mem-1", store.lastUpdatedMemoryId);
    assertEquals(0.7, store.lastUpdatedImportance, 0.001);

    // Tick 2: memory should be recalled
    store.queryResults = List.of(
            new Memory("mem-1", "agent-1", ExperienceEvents.DOMAIN, "t1", null,
                    "Bob asked about ML. I helped.", Map.of(), Instant.now(), 0.7));

    loop.tick(contextWith(perceptionWithMessage("what did we talk about?", "ch-1")));
    assertNotNull(store.lastQuery);
}
```

- [ ] **Step 2: Run end-to-end test**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=ChatAgentEndToEndTest -q`
Expected: PASS

- [ ] **Step 3: Run full quarkmind-chat test suite**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-protocol,quarkmind-chat/quarkmind-chat-agent -q`
Expected: PASS — all tests including new memory tests

- [ ] **Step 4: Run full quarkmind test suite**

Run: `mvn test -q`
Expected: PASS

- [ ] **Step 5: Update CLAUDE.md with new test listings**

Add to the chat agent test listings in CLAUDE.md:
- `ChatMemoryFacadeTest`
- `LlmReflectionSynthesizerTest`

Add `IdleReflectionTriggerTest` to the quarkmind-core test listings.

- [ ] **Step 6: Commit**

```bash
git add quarkmind-chat/ quarkmind-core/ CLAUDE.md
git commit -m "feat(#280): memory integration end-to-end test + CLAUDE.md update Refs #280"
```

---

## References

- `specs/issue-279-quarkmind-discord/2026-08-20-memory-integration-design.md` — design spec
- `casehub-neocortex/memory-api/src/main/java/io/casehub/neocortex/memory/CaseMemoryStore.java` — memory SPI
- `casehub-neocortex/memory-api/src/main/java/io/casehub/neocortex/memory/GraphCaseMemoryStore.java` — graph memory SPI
- `casehub-neocortex/memory-api/src/main/java/io/casehub/neocortex/memory/experience/ExperienceQuery.java` — retrieval factory
- `casehub-neocortex/memory-api/src/main/java/io/casehub/neocortex/memory/personality/PersonalityWeightedRetrieval.java` — re-ranker
- `casehub-neocortex/memory/src/main/java/io/casehub/neocortex/memory/reflection/runtime/ReflectionService.java` — reflection orchestrator
- `quarkmind-core/src/main/java/io/quarkmind/agency/llm/LlmRequest.java` — LLM queue request
- `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatAgencyLoop.java` — agency loop
- GE-20260630-815259 — cross-repo SPI displacement (CbrCaseMemoryStore standalone pattern)
- GE-20260811-aa9a26 — blocks dependency transitively requires neocortex config properties
- GitHub #280 — focal issue
