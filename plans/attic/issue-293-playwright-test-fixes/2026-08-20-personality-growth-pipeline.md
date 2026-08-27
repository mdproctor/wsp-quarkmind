# Personality Growth Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #281 — quarkmind-chat — personality growth pipeline (D6 Layer 3)
**Issue group:** #279, #280, #281, #282, #283

**Goal:** Wire the reflection → disposition activation → personality evolution pipeline so the chat character's personality evolves over time based on accumulated experiences.

**Architecture:** Four new classes across quarkmind-core and quarkmind-chat-agent. `ReflectionDispositionActivator` (SPI in quarkmind-core) classifies which disposition function term a reflection activates. `PersonalityEvolutionPipeline` (quarkmind-core) orchestrates the health check → evolution → decay cycle. `LlmReflectionDispositionActivator` (quarkmind-chat-agent) provides the LLM-based classification implementation. `DispositionAwareReflectionSynthesizer` (quarkmind-chat-agent) decorates `ReflectionSynthesizer` to feed insights to the activator at synthesis time — no store round-trip needed.

**Tech Stack:** Java 21, JUnit 5, casehub-eidos-api (DispositionSignalStore, DispositionHealth, DispositionEvolution), casehub-neocortex memory-api (ReflectionSynthesizer, ReflectionEvent)

## Global Constraints

- No new Maven dependencies — eidos-api already on quarkmind-core and quarkmind-chat-agent classpath
- All new classes are plain Java (no CDI, no Quarkus imports) — matches quarkmind-core and quarkmind-chat-agent conventions
- DispositionSignalStore.recordActivation() is the only mutation — async via LlmRequestQueue at LOW priority
- LLM classification is best-effort — failure means no activation recorded, not a system error
- Test style: plain JUnit 5 with stubs, no mocks library, no @QuarkusTest

---

## Batch 1: Foundation — SPI and Evolution Pipeline

### Task 1: ReflectionDispositionActivator SPI + PersonalityEvolutionPipeline

**Files:**
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/personality/ReflectionDispositionActivator.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/personality/PersonalityEvolutionPipeline.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/personality/PersonalityEvolutionPipelineTest.java`

**Interfaces:**
- Consumes: `DispositionHealth.probe(AgentDescriptor, ProbeContext)`, `DispositionEvolution.evaluate(AgentDescriptor, EvolutionPending)`, `DispositionSignalStore.decay(agentId, tenancyId, decayFactor)`
- Produces: `ReflectionDispositionActivator` interface — `void onReflection(String agentId, String tenantId, String insight)`; `PersonalityEvolutionPipeline` — `Optional<DispositionEvolution.EvolutionResult> checkEvolution(AgentDescriptor descriptor)`

- [ ] **Step 1: Write the ReflectionDispositionActivator SPI**

```java
package io.quarkmind.agency.personality;

public interface ReflectionDispositionActivator {
    void onReflection(String agentId, String tenantId, String insight);
}
```

Use `ide_create_file` for `quarkmind-core/src/main/java/io/quarkmind/agency/personality/ReflectionDispositionActivator.java`.

- [ ] **Step 2: Write failing tests for PersonalityEvolutionPipeline**

```java
package io.quarkmind.agency.personality;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.eidos.api.DispositionHealth.DispositionStatus;
import io.casehub.eidos.api.DispositionEvolution.EvolutionResult;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class PersonalityEvolutionPipelineTest {

    @Test
    void returnsEmptyWhenAligned() {
        var store = new StubSignalStore();
        var pipeline = new PersonalityEvolutionPipeline(
                (desc, ctx) -> new DispositionStatus.Aligned(Map.of()),
                (desc, pending) -> { throw new AssertionError("should not be called"); },
                store);
        var result = pipeline.checkEvolution(testDescriptor());
        assertTrue(result.isEmpty());
    }

    @Test
    void returnsEmptyWhenDrifted() {
        var store = new StubSignalStore();
        var pipeline = new PersonalityEvolutionPipeline(
                (desc, ctx) -> new DispositionStatus.Drifted(Map.of(), "empathetic", 0.1),
                (desc, pending) -> { throw new AssertionError("should not be called"); },
                store);
        var result = pipeline.checkEvolution(testDescriptor());
        assertTrue(result.isEmpty());
    }

    @Test
    void returnsEvolvedAndDoesNotDecay() {
        var store = new StubSignalStore();
        var newProfile = List.of(
                new DispositionValue("empathetic", 0.35),
                new DispositionValue("analytical", 0.20),
                new DispositionValue("playful", 0.45));
        var pipeline = new PersonalityEvolutionPipeline(
                (desc, ctx) -> new DispositionStatus.EvolutionPending(
                        () -> "DOMINANT_AUXILIARY_SWAP", "empathetic", Map.of()),
                (desc, pending) -> new EvolutionResult.Evolved(newProfile, "ANALYTICAL-EMPATHETIC", "EMPATHETIC-PLAYFUL"),
                store);
        var result = pipeline.checkEvolution(testDescriptor());
        assertTrue(result.isPresent());
        assertInstanceOf(EvolutionResult.Evolved.class, result.get());
        assertFalse(store.decayCalled);
    }

    @Test
    void returnsDampenedAndDecays() {
        var store = new StubSignalStore();
        var pipeline = new PersonalityEvolutionPipeline(
                (desc, ctx) -> new DispositionStatus.EvolutionPending(
                        () -> "STRUCTURAL_REORGANIZATION", "curious", Map.of()),
                (desc, pending) -> new EvolutionResult.Dampened(0.2),
                store);
        var result = pipeline.checkEvolution(testDescriptor());
        assertTrue(result.isPresent());
        assertInstanceOf(EvolutionResult.Dampened.class, result.get());
        assertTrue(store.decayCalled);
        assertEquals(0.2, store.lastDecayFactor, 0.001);
        assertEquals("agent-1", store.lastDecayAgentId);
    }

    private AgentDescriptor testDescriptor() {
        return AgentDescriptor.builder()
                .agentId("agent-1").name("Test").slot("chat").tenancyId("t1")
                .disposition(AgentDisposition.builder()
                        .dispositionProfile(
                                new DispositionValue("analytical", 0.35),
                                new DispositionValue("empathetic", 0.20),
                                new DispositionValue("playful", 0.45))
                        .build())
                .build();
    }

    static class StubSignalStore implements DispositionSignalStore {
        boolean decayCalled = false;
        double lastDecayFactor;
        String lastDecayAgentId;

        @Override public void recordActivation(String agentId, String tenancyId, String functionTerm) {}
        @Override public Map<String, Integer> activationCounts(String agentId, String tenancyId) { return Map.of(); }
        @Override public void decay(String agentId, String tenancyId, double decayFactor) {
            decayCalled = true;
            lastDecayFactor = decayFactor;
            lastDecayAgentId = agentId;
        }
        @Override public void clear(String agentId, String tenancyId) {}
    }
}
```

Use `ide_create_file` for the test file.

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-core -Dtest=PersonalityEvolutionPipelineTest -q`
Expected: FAIL with "cannot find symbol: class PersonalityEvolutionPipeline"

- [ ] **Step 4: Implement PersonalityEvolutionPipeline**

```java
package io.quarkmind.agency.personality;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.eidos.api.DispositionEvolution;
import io.casehub.eidos.api.DispositionEvolution.EvolutionResult;
import io.casehub.eidos.api.DispositionHealth;
import io.casehub.eidos.api.DispositionHealth.DispositionStatus;
import io.casehub.eidos.api.DispositionSignalStore;

import java.util.Optional;

public class PersonalityEvolutionPipeline {

    private final DispositionHealth health;
    private final DispositionEvolution evolution;
    private final DispositionSignalStore signalStore;

    public PersonalityEvolutionPipeline(DispositionHealth health,
                                        DispositionEvolution evolution,
                                        DispositionSignalStore signalStore) {
        this.health = health;
        this.evolution = evolution;
        this.signalStore = signalStore;
    }

    public Optional<EvolutionResult> checkEvolution(AgentDescriptor descriptor) {
        var status = health.probe(descriptor, ProbeContext.of("chat"));
        if (status instanceof DispositionStatus.EvolutionPending pending) {
            var result = evolution.evaluate(descriptor, pending);
            if (result instanceof EvolutionResult.Dampened dampened) {
                signalStore.decay(descriptor.agentId(), descriptor.tenancyId(),
                        dampened.decayFactor());
            }
            return Optional.of(result);
        }
        return Optional.empty();
    }
}
```

Use `ide_create_file` for the implementation file.

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-core -Dtest=PersonalityEvolutionPipelineTest -q`
Expected: PASS (4 tests)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/129/quarkmind add quarkmind-core/src/main/java/io/quarkmind/agency/personality/ReflectionDispositionActivator.java quarkmind-core/src/main/java/io/quarkmind/agency/personality/PersonalityEvolutionPipeline.java quarkmind-core/src/test/java/io/quarkmind/agency/personality/PersonalityEvolutionPipelineTest.java
git -C /Users/mdproctor/claude/casehub/slots/129/quarkmind commit -m "feat(#281): ReflectionDispositionActivator SPI + PersonalityEvolutionPipeline

Refs #281"
```

---

## Batch 2: LLM Activator and Decorator

### Task 2: LlmReflectionDispositionActivator

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/LlmReflectionDispositionActivator.java`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/LlmReflectionDispositionActivatorTest.java`

**Interfaces:**
- Consumes: `ReflectionDispositionActivator` SPI (from Task 1), `LlmRequestQueue.submit(LlmRequest)`, `DispositionSignalStore.recordActivation(agentId, tenancyId, functionTerm)`
- Produces: `LlmReflectionDispositionActivator` — constructor takes `(LlmRequestQueue, DispositionSignalStore, List<DispositionValue> initialProfile)`, `updateProfile(List<DispositionValue>)` for post-evolution updates

- [ ] **Step 1: Write failing tests**

```java
package io.quarkmind.chat.agent;

import io.casehub.eidos.api.DispositionSignalStore;
import io.casehub.eidos.api.DispositionValue;
import io.quarkmind.agency.llm.LlmPriority;
import io.quarkmind.agency.llm.LlmRequest;
import io.quarkmind.agency.llm.LlmRequestQueue;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class LlmReflectionDispositionActivatorTest {

    @Test
    void submitsLowPriorityClassificationRequest() {
        var submitted = new ArrayList<LlmRequest>();
        var queue = stubQueue(submitted);
        var store = new RecordingSignalStore();
        var profile = List.of(
                new DispositionValue("empathetic", 0.4),
                new DispositionValue("analytical", 0.3),
                new DispositionValue("playful", 0.3));

        var activator = new LlmReflectionDispositionActivator(queue, store, profile);
        activator.onReflection("agent-1", "t1", "Users tend to be emotionally expressive");

        assertEquals(1, submitted.size());
        assertEquals(LlmPriority.LOW, submitted.get(0).priority());
        assertTrue(submitted.get(0).prompt().contains("empathetic"));
        assertTrue(submitted.get(0).prompt().contains("analytical"));
        assertTrue(submitted.get(0).prompt().contains("playful"));
    }

    @Test
    void recordsActivationWhenLlmReturnsValidTerm() {
        var submitted = new ArrayList<LlmRequest>();
        var queue = stubQueue(submitted);
        var store = new RecordingSignalStore();
        var profile = List.of(
                new DispositionValue("empathetic", 0.4),
                new DispositionValue("analytical", 0.3));

        var activator = new LlmReflectionDispositionActivator(queue, store, profile);
        activator.onReflection("agent-1", "t1", "People open up emotionally");

        submitted.get(0).responseHandler().accept("empathetic");

        assertEquals(1, store.activations.size());
        assertEquals("empathetic", store.activations.get(0).term);
        assertEquals("agent-1", store.activations.get(0).agentId);
        assertEquals("t1", store.activations.get(0).tenantId);
    }

    @Test
    void ignoresInvalidTermFromLlm() {
        var submitted = new ArrayList<LlmRequest>();
        var queue = stubQueue(submitted);
        var store = new RecordingSignalStore();
        var profile = List.of(new DispositionValue("empathetic", 0.5));

        var activator = new LlmReflectionDispositionActivator(queue, store, profile);
        activator.onReflection("agent-1", "t1", "Some reflection");

        submitted.get(0).responseHandler().accept("nonexistent-term");

        assertTrue(store.activations.isEmpty());
    }

    @Test
    void handlesNoneResponse() {
        var submitted = new ArrayList<LlmRequest>();
        var queue = stubQueue(submitted);
        var store = new RecordingSignalStore();
        var profile = List.of(new DispositionValue("empathetic", 0.5));

        var activator = new LlmReflectionDispositionActivator(queue, store, profile);
        activator.onReflection("agent-1", "t1", "Mundane observation");

        submitted.get(0).responseHandler().accept("none");

        assertTrue(store.activations.isEmpty());
    }

    @Test
    void skipsWhenProfileIsEmpty() {
        var submitted = new ArrayList<LlmRequest>();
        var queue = stubQueue(submitted);
        var store = new RecordingSignalStore();

        var activator = new LlmReflectionDispositionActivator(queue, store, List.of());
        activator.onReflection("agent-1", "t1", "Something happened");

        assertTrue(submitted.isEmpty());
    }

    @Test
    void matchesTermCaseInsensitively() {
        var submitted = new ArrayList<LlmRequest>();
        var queue = stubQueue(submitted);
        var store = new RecordingSignalStore();
        var profile = List.of(new DispositionValue("empathetic", 0.5));

        var activator = new LlmReflectionDispositionActivator(queue, store, profile);
        activator.onReflection("agent-1", "t1", "Reflection");

        submitted.get(0).responseHandler().accept("  Empathetic  ");

        assertEquals(1, store.activations.size());
        assertEquals("empathetic", store.activations.get(0).term);
    }

    @Test
    void updateProfileChangesTermsForSubsequentCalls() {
        var submitted = new ArrayList<LlmRequest>();
        var queue = stubQueue(submitted);
        var store = new RecordingSignalStore();
        var profile = List.of(new DispositionValue("empathetic", 0.5));

        var activator = new LlmReflectionDispositionActivator(queue, store, profile);
        activator.updateProfile(List.of(new DispositionValue("curious", 0.5)));
        activator.onReflection("agent-1", "t1", "Something");

        assertTrue(submitted.get(0).prompt().contains("curious"));
        assertFalse(submitted.get(0).prompt().contains("empathetic"));
    }

    private LlmRequestQueue stubQueue(List<LlmRequest> sink) {
        return new LlmRequestQueue() {
            @Override public void submit(LlmRequest r) { sink.add(r); }
            @Override public int pendingCount() { return 0; }
            @Override public boolean hasCapacity() { return true; }
        };
    }

    record Activation(String agentId, String tenantId, String term) {}

    static class RecordingSignalStore implements DispositionSignalStore {
        final List<Activation> activations = new ArrayList<>();
        @Override public void recordActivation(String agentId, String tenancyId, String functionTerm) {
            activations.add(new Activation(agentId, tenancyId, functionTerm));
        }
        @Override public Map<String, Integer> activationCounts(String agentId, String tenancyId) { return Map.of(); }
        @Override public void decay(String agentId, String tenancyId, double decayFactor) {}
        @Override public void clear(String agentId, String tenancyId) {}
    }
}
```

Use `ide_create_file` for the test file.

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=LlmReflectionDispositionActivatorTest -q`
Expected: FAIL with "cannot find symbol: class LlmReflectionDispositionActivator"

- [ ] **Step 3: Implement LlmReflectionDispositionActivator**

```java
package io.quarkmind.chat.agent;

import io.casehub.eidos.api.DispositionSignalStore;
import io.casehub.eidos.api.DispositionValue;
import io.quarkmind.agency.llm.LlmPriority;
import io.quarkmind.agency.llm.LlmRequest;
import io.quarkmind.agency.llm.LlmRequestQueue;
import io.quarkmind.agency.personality.ReflectionDispositionActivator;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class LlmReflectionDispositionActivator implements ReflectionDispositionActivator {

    private static final Logger LOG = Logger.getLogger(LlmReflectionDispositionActivator.class);

    private final LlmRequestQueue llmQueue;
    private final DispositionSignalStore signalStore;
    private volatile List<DispositionValue> dispositionProfile;

    public LlmReflectionDispositionActivator(LlmRequestQueue llmQueue,
                                             DispositionSignalStore signalStore,
                                             List<DispositionValue> initialProfile) {
        this.llmQueue = llmQueue;
        this.signalStore = signalStore;
        this.dispositionProfile = List.copyOf(initialProfile);
    }

    public void updateProfile(List<DispositionValue> newProfile) {
        this.dispositionProfile = List.copyOf(newProfile);
    }

    @Override
    public void onReflection(String agentId, String tenantId, String insight) {
        var profile = this.dispositionProfile;
        if (profile.isEmpty()) return;

        String terms = profile.stream()
                .map(DispositionValue::term)
                .collect(Collectors.joining(", "));

        String prompt = "Given this reflection insight and these personality function terms, " +
                "which term does this reflection most strongly activate? " +
                "Respond with a single function term from the list, or \"none\" if no term applies.\n\n" +
                "Reflection: " + insight + "\n\n" +
                "Function terms: " + terms;

        llmQueue.submit(new LlmRequest(prompt, LlmPriority.LOW, Map.of(), response -> {
            try {
                String term = response.trim();
                if ("none".equalsIgnoreCase(term)) return;
                String matched = profile.stream()
                        .map(DispositionValue::term)
                        .filter(t -> t.equalsIgnoreCase(term))
                        .findFirst().orElse(null);
                if (matched != null) {
                    signalStore.recordActivation(agentId, tenantId, matched);
                }
            } catch (Exception e) {
                LOG.debug("Disposition activation classification failed", e);
            }
        }));
    }
}
```

Use `ide_create_file` for the implementation file.

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=LlmReflectionDispositionActivatorTest -q`
Expected: PASS (7 tests)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/129/quarkmind add quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/LlmReflectionDispositionActivator.java quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/LlmReflectionDispositionActivatorTest.java
git -C /Users/mdproctor/claude/casehub/slots/129/quarkmind commit -m "feat(#281): LlmReflectionDispositionActivator — async LLM classification

Refs #281"
```

### Task 3: DispositionAwareReflectionSynthesizer

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/DispositionAwareReflectionSynthesizer.java`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/DispositionAwareReflectionSynthesizerTest.java`

**Interfaces:**
- Consumes: `ReflectionSynthesizer` (delegate), `ReflectionDispositionActivator.onReflection(agentId, tenantId, insight)`
- Produces: `DispositionAwareReflectionSynthesizer implements ReflectionSynthesizer` — decorator that intercepts insights at synthesis time

- [ ] **Step 1: Write failing tests**

```java
package io.quarkmind.chat.agent;

import io.casehub.neocortex.memory.Memory;
import io.casehub.neocortex.memory.experience.ExperienceEvents;
import io.casehub.neocortex.memory.reflection.ReflectionEvent;
import io.casehub.neocortex.memory.reflection.ReflectionSynthesizer;
import io.quarkmind.agency.personality.ReflectionDispositionActivator;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class DispositionAwareReflectionSynthesizerTest {

    @Test
    void delegatesToUnderlyingSynthesizer() {
        var delegateCalled = new java.util.concurrent.atomic.AtomicBoolean(false);
        ReflectionSynthesizer delegate = (agentId, tenantId, sources, level) -> {
            delegateCalled.set(true);
            return List.of(new ReflectionEvent(agentId, tenantId, null,
                    "An insight", 1, List.of("m1"), 0.7, Map.of()));
        };
        var activations = new ArrayList<String>();
        ReflectionDispositionActivator activator = (aid, tid, insight) ->
                activations.add(insight);

        var synthesizer = new DispositionAwareReflectionSynthesizer(delegate, activator);
        var sources = List.of(testMemory("m1"));
        var events = synthesizer.synthesize("agent-1", "t1", sources, 1);

        assertTrue(delegateCalled.get());
        assertEquals(1, events.size());
        assertEquals("An insight", events.get(0).insight());
    }

    @Test
    void feedsEachInsightToActivator() {
        ReflectionSynthesizer delegate = (agentId, tenantId, sources, level) -> List.of(
                new ReflectionEvent(agentId, tenantId, null, "Insight A", 1, List.of("m1"), 0.6, Map.of()),
                new ReflectionEvent(agentId, tenantId, null, "Insight B", 1, List.of("m1"), 0.7, Map.of()));
        var activations = new ArrayList<String>();
        ReflectionDispositionActivator activator = (aid, tid, insight) ->
                activations.add(insight);

        var synthesizer = new DispositionAwareReflectionSynthesizer(delegate, activator);
        synthesizer.synthesize("agent-1", "t1", List.of(testMemory("m1")), 1);

        assertEquals(2, activations.size());
        assertEquals("Insight A", activations.get(0));
        assertEquals("Insight B", activations.get(1));
    }

    @Test
    void passesCorrectAgentAndTenantToActivator() {
        ReflectionSynthesizer delegate = (agentId, tenantId, sources, level) -> List.of(
                new ReflectionEvent(agentId, tenantId, null, "Insight", 1, List.of("m1"), 0.5, Map.of()));
        var agentIds = new ArrayList<String>();
        var tenantIds = new ArrayList<String>();
        ReflectionDispositionActivator activator = (aid, tid, insight) -> {
            agentIds.add(aid);
            tenantIds.add(tid);
        };

        var synthesizer = new DispositionAwareReflectionSynthesizer(delegate, activator);
        synthesizer.synthesize("bot-42", "server-7", List.of(testMemory("m1")), 1);

        assertEquals("bot-42", agentIds.get(0));
        assertEquals("server-7", tenantIds.get(0));
    }

    @Test
    void returnsEmptyWhenDelegateReturnsEmpty() {
        ReflectionSynthesizer delegate = (agentId, tenantId, sources, level) -> List.of();
        var activations = new ArrayList<String>();
        ReflectionDispositionActivator activator = (aid, tid, insight) ->
                activations.add(insight);

        var synthesizer = new DispositionAwareReflectionSynthesizer(delegate, activator);
        var events = synthesizer.synthesize("agent-1", "t1", List.of(testMemory("m1")), 1);

        assertTrue(events.isEmpty());
        assertTrue(activations.isEmpty());
    }

    @Test
    void activatorFailureDoesNotBreakSynthesis() {
        ReflectionSynthesizer delegate = (agentId, tenantId, sources, level) -> List.of(
                new ReflectionEvent(agentId, tenantId, null, "Insight", 1, List.of("m1"), 0.5, Map.of()));
        ReflectionDispositionActivator activator = (aid, tid, insight) -> {
            throw new RuntimeException("activator failed");
        };

        var synthesizer = new DispositionAwareReflectionSynthesizer(delegate, activator);
        var events = synthesizer.synthesize("agent-1", "t1", List.of(testMemory("m1")), 1);

        assertEquals(1, events.size());
    }

    private Memory testMemory(String id) {
        return new Memory(id, "agent-1", ExperienceEvents.DOMAIN, "t1", null,
                "Some experience", Map.of(), Instant.now(), 0.5);
    }
}
```

Use `ide_create_file` for the test file.

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=DispositionAwareReflectionSynthesizerTest -q`
Expected: FAIL with "cannot find symbol: class DispositionAwareReflectionSynthesizer"

- [ ] **Step 3: Implement DispositionAwareReflectionSynthesizer**

```java
package io.quarkmind.chat.agent;

import io.casehub.neocortex.memory.Memory;
import io.casehub.neocortex.memory.reflection.ReflectionEvent;
import io.casehub.neocortex.memory.reflection.ReflectionSynthesizer;
import io.quarkmind.agency.personality.ReflectionDispositionActivator;
import org.jboss.logging.Logger;

import java.util.List;

public class DispositionAwareReflectionSynthesizer implements ReflectionSynthesizer {

    private static final Logger LOG = Logger.getLogger(DispositionAwareReflectionSynthesizer.class);

    private final ReflectionSynthesizer delegate;
    private final ReflectionDispositionActivator activator;

    public DispositionAwareReflectionSynthesizer(ReflectionSynthesizer delegate,
                                                 ReflectionDispositionActivator activator) {
        this.delegate = delegate;
        this.activator = activator;
    }

    @Override
    public List<ReflectionEvent> synthesize(String agentId, String tenantId,
                                            List<Memory> sources, int targetLevel) {
        var events = delegate.synthesize(agentId, tenantId, sources, targetLevel);
        for (var event : events) {
            try {
                activator.onReflection(agentId, tenantId, event.insight());
            } catch (Exception e) {
                LOG.debug("Disposition activation failed for reflection", e);
            }
        }
        return events;
    }
}
```

Use `ide_create_file` for the implementation file.

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=DispositionAwareReflectionSynthesizerTest -q`
Expected: PASS (5 tests)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/129/quarkmind add quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/DispositionAwareReflectionSynthesizer.java quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/DispositionAwareReflectionSynthesizerTest.java
git -C /Users/mdproctor/claude/casehub/slots/129/quarkmind commit -m "feat(#281): DispositionAwareReflectionSynthesizer — intercepts insights for activation

Refs #281"
```

---

## Batch 3: Wiring and End-to-End Test

### Task 4: Wire ChatAgencyLoop for personality evolution

**Files:**
- Modify: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatAgencyLoop.java`
- Modify: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatAgencyLoopTest.java`

**Interfaces:**
- Consumes: `PersonalityEvolutionPipeline.checkEvolution(AgentDescriptor)`, `LlmReflectionDispositionActivator.updateProfile(List<DispositionValue>)`
- Produces: Updated `ChatAgencyLoop` — new constructor accepts `PersonalityEvolutionPipeline` and `AgentDescriptor` supplier; checks evolution periodically during idle ticks; logs personality shifts

- [ ] **Step 1: Write failing test for evolution check during idle**

Add to `ChatAgencyLoopTest.java`:

```java
@Test
void idleTickChecksPersonalityEvolution() {
    var store = new ChatMemoryFacadeTest.RecordingMemoryStore();
    var facade = new ChatMemoryFacade(store, store, false);
    var trigger = new IdleReflectionTrigger(100.0, 100);

    var evolutionChecked = new AtomicBoolean(false);
    var pipeline = new PersonalityEvolutionPipeline(
            (desc, ctx) -> {
                evolutionChecked.set(true);
                return new DispositionHealth.DispositionStatus.Aligned(Map.of());
            },
            (desc, pending) -> { throw new AssertionError("should not be called"); },
            new PersonalityEvolutionPipelineTest.StubSignalStore());

    var descriptor = AgentDescriptor.builder()
            .agentId("agent-1").name("Test").slot("chat").tenancyId("t1")
            .disposition(AgentDisposition.builder()
                    .dispositionProfile(new DispositionValue("empathetic", 0.5))
                    .build())
            .build();

    var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) ->
            "{\"action\":\"WAIT\",\"observation\":\"idle\"}";
    var loop = new ChatAgencyLoop(llm, detector, llmQueue, mapper,
            new DefaultChatPerceptionBridge(new ChatObservationRenderer(10)),
            facade, trigger, null, pipeline, () -> descriptor);

    var heartbeat = new ChatPerception(Map.of(), Map.of(), WakeReason.HEARTBEAT);
    loop.tick(contextWith(heartbeat));

    assertTrue(evolutionChecked.get());
}

@Test
void evolvedResultUpdatesActivatorProfile() {
    var store = new ChatMemoryFacadeTest.RecordingMemoryStore();
    var facade = new ChatMemoryFacade(store, store, false);
    var trigger = new IdleReflectionTrigger(100.0, 100);
    var signalStore = new PersonalityEvolutionPipelineTest.StubSignalStore();

    var newProfile = List.of(
            new DispositionValue("curious", 0.6),
            new DispositionValue("empathetic", 0.4));

    var pipeline = new PersonalityEvolutionPipeline(
            (desc, ctx) -> new DispositionHealth.DispositionStatus.EvolutionPending(
                    () -> "DOMINANT_AUXILIARY_SWAP", "curious", Map.of()),
            (desc, pending) -> new DispositionEvolution.EvolutionResult.Evolved(
                    newProfile, "EMPATHETIC-CURIOUS", "CURIOUS-EMPATHETIC"),
            signalStore);

    var submitted = new ArrayList<LlmRequest>();
    var activator = new LlmReflectionDispositionActivator(
            stubQueue(submitted), signalStore,
            List.of(new DispositionValue("empathetic", 0.6)));

    var descriptor = AgentDescriptor.builder()
            .agentId("agent-1").name("Test").slot("chat").tenancyId("t1")
            .disposition(AgentDisposition.builder()
                    .dispositionProfile(new DispositionValue("empathetic", 0.6),
                            new DispositionValue("curious", 0.4))
                    .build())
            .build();

    var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) ->
            "{\"action\":\"WAIT\",\"observation\":\"idle\"}";
    var loop = new ChatAgencyLoop(llm, detector, llmQueue, mapper,
            new DefaultChatPerceptionBridge(new ChatObservationRenderer(10)),
            facade, trigger, null, pipeline, () -> descriptor);
    loop.setDispositionActivator(activator);

    var heartbeat = new ChatPerception(Map.of(), Map.of(), WakeReason.HEARTBEAT);
    loop.tick(contextWith(heartbeat));

    activator.onReflection("agent-1", "t1", "test insight");
    assertTrue(submitted.get(0).prompt().contains("curious"));
}

private LlmRequestQueue stubQueue(List<LlmRequest> sink) {
    return new LlmRequestQueue() {
        @Override public void submit(LlmRequest r) { sink.add(r); }
        @Override public int pendingCount() { return 0; }
        @Override public boolean hasCapacity() { return true; }
    };
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=ChatAgencyLoopTest#idleTickChecksPersonalityEvolution -q`
Expected: FAIL — constructor signature doesn't match

- [ ] **Step 3: Modify ChatAgencyLoop to accept evolution dependencies**

Add new fields and constructor to `ChatAgencyLoop`:
- `PersonalityEvolutionPipeline evolutionPipeline` (nullable)
- `Supplier<AgentDescriptor> descriptorSupplier` (nullable)
- `LlmReflectionDispositionActivator dispositionActivator` (nullable, set via setter)

Add new constructor:
```java
public ChatAgencyLoop(LlmInvoker llmInvoker, BotIdentityDetector identityDetector,
                      LlmRequestQueue llmQueue, ObjectMapper mapper,
                      ChatPerceptionBridge perceptionBridge,
                      ChatMemoryFacade memoryFacade,
                      IdleReflectionTrigger reflectionTrigger,
                      ReflectionOrchestrator reflectionOrchestrator,
                      PersonalityEvolutionPipeline evolutionPipeline,
                      Supplier<AgentDescriptor> descriptorSupplier) {
    // ... delegates to existing constructor, stores pipeline + supplier
}
```

Add `setDispositionActivator(LlmReflectionDispositionActivator activator)` setter.

Modify heartbeat idle path in `tick()` to call `checkEvolution()` after `checkReflection()`:
```java
if (perception.reason() == WakeReason.HEARTBEAT && !perception.hasActivity()) {
    consecutiveIdleTicks++;
    checkReflection();
    checkEvolution();
    context.put("intents", List.of());
    return;
}
```

Add `checkEvolution()`:
```java
private void checkEvolution() {
    if (evolutionPipeline == null || descriptorSupplier == null) return;
    try {
        var descriptor = descriptorSupplier.get();
        var result = evolutionPipeline.checkEvolution(descriptor);
        result.ifPresent(r -> {
            if (r instanceof DispositionEvolution.EvolutionResult.Evolved evolved) {
                LOG.infof("Personality evolved: %s → %s", evolved.previousTypeLabel(), evolved.newTypeLabel());
                if (dispositionActivator != null) {
                    dispositionActivator.updateProfile(evolved.newProfile());
                }
            } else if (r instanceof DispositionEvolution.EvolutionResult.Dampened dampened) {
                LOG.infof("Personality evolution dampened (decay=%.2f)", dampened.decayFactor());
            }
        });
    } catch (Exception e) {
        LOG.warn("Evolution check failed", e);
    }
}
```

Use `ide_edit_member` and `ide_insert_member` for the modifications.

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=ChatAgencyLoopTest -q`
Expected: PASS (all existing + 2 new tests)

- [ ] **Step 5: Run all chat module tests to check for regressions**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-protocol,quarkmind-chat/quarkmind-chat-agent -q`
Expected: PASS — no regressions

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/129/quarkmind add quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatAgencyLoop.java quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatAgencyLoopTest.java
git -C /Users/mdproctor/claude/casehub/slots/129/quarkmind commit -m "feat(#281): wire personality evolution into ChatAgencyLoop idle path

Refs #281"
```

### Task 5: Personality evolution end-to-end test + CLAUDE.md

**Files:**
- Modify: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatAgentEndToEndTest.java`
- Modify: `CLAUDE.md` (add new test classes to test registry)

**Interfaces:**
- Consumes: All classes from Tasks 1-4
- Produces: End-to-end test that exercises: reflection synthesis → disposition activation → signal recording → health probe → evolution result

- [ ] **Step 1: Write the end-to-end personality evolution test**

Add to `ChatAgentEndToEndTest.java`:

```java
@Test
void personalityEvolutionEndToEnd() {
    var mapper = new ObjectMapper();
    var store = new ChatMemoryFacadeTest.RecordingMemoryStore();
    var facade = new ChatMemoryFacade(store, store, false);
    var reflectionTrigger = new IdleReflectionTrigger(1.0, 2);
    reflectionTrigger.accumulate(1.5);

    var signalStore = new TrackingSignalStore();
    var scoringRequests = new ArrayList<LlmRequest>();
    var queue = new LlmRequestQueue() {
        @Override public void submit(LlmRequest r) { scoringRequests.add(r); }
        @Override public int pendingCount() { return 0; }
        @Override public boolean hasCapacity() { return true; }
    };

    var initialProfile = List.of(
            new DispositionValue("analytical", 0.35),
            new DispositionValue("empathetic", 0.20),
            new DispositionValue("playful", 0.45));
    var descriptor = AgentDescriptor.builder()
            .agentId("agent-1").name("TestBot").slot("chat").tenancyId("t1")
            .disposition(AgentDisposition.builder()
                    .dispositionProfile(initialProfile)
                    .build())
            .build();

    var activator = new LlmReflectionDispositionActivator(queue, signalStore, initialProfile);
    var baseSynthesizer = new LlmReflectionSynthesizer(
            (sys, usr, id) -> "[{\"insight\":\"Users consistently seek emotional support and empathetic engagement\"}]");
    var decoratedSynthesizer = new DispositionAwareReflectionSynthesizer(baseSynthesizer, activator);

    ReflectionOrchestrator orchestrator = (agentId, tenantId, since, max) -> {
        var sources = List.of(new Memory("m1", agentId, ExperienceEvents.DOMAIN, tenantId, null,
                "Helped someone with their feelings", Map.of(), Instant.now(), 0.8));
        var events = decoratedSynthesizer.synthesize(agentId, tenantId, sources, 1);
        return events.stream().map(e -> "ref-" + e.insight().hashCode()).toList();
    };

    var pipeline = new PersonalityEvolutionPipeline(
            (desc, ctx) -> {
                var counts = signalStore.activationCounts(desc.agentId(), desc.tenancyId());
                if (counts.getOrDefault("empathetic", 0) > 0) {
                    return new DispositionHealth.DispositionStatus.Drifted(
                            Map.of("empathetic", 0.4), "empathetic", 0.1);
                }
                return new DispositionHealth.DispositionStatus.Aligned(Map.of());
            },
            (desc, pending) -> new DispositionEvolution.EvolutionResult.Dampened(0.5),
            signalStore);

    var llm = (ChatAgencyLoop.LlmInvoker) (sys, usr, id) ->
            "{\"action\":\"WAIT\",\"observation\":\"idle\"}";
    var loop = new ChatAgencyLoop(llm, stubDetector(), queue, mapper,
            new DefaultChatPerceptionBridge(new ChatObservationRenderer(10)),
            facade, reflectionTrigger, orchestrator, pipeline, () -> descriptor);
    loop.setAgentId("agent-1");
    loop.setDispositionActivator(activator);

    var heartbeat = new ChatPerception(Map.of(), Map.of(), WakeReason.HEARTBEAT);

    // Tick 1 — not enough idle yet
    loop.tick(contextWith(heartbeat));
    // Tick 2 — not enough idle yet
    loop.tick(contextWith(heartbeat));
    // Tick 3 — threshold met, reflection fires
    loop.tick(contextWith(heartbeat));

    // Verify classification was submitted
    var classificationReqs = scoringRequests.stream()
            .filter(r -> r.prompt().contains("empathetic") && r.prompt().contains("analytical"))
            .toList();
    assertFalse(classificationReqs.isEmpty(), "Classification request should have been submitted");

    // Simulate LLM classification response
    classificationReqs.get(0).responseHandler().accept("empathetic");

    // Verify activation was recorded
    var counts = signalStore.activationCounts("agent-1", "t1");
    assertEquals(1, counts.getOrDefault("empathetic", 0));
}

private AgencyContext contextWith(ChatPerception perception) {
    var context = new AgencyContext(new NeedState());
    context.put("perception", perception);
    return context;
}

static class TrackingSignalStore implements DispositionSignalStore {
    private final java.util.concurrent.ConcurrentHashMap<String, Integer> counts = new java.util.concurrent.ConcurrentHashMap<>();
    @Override public void recordActivation(String agentId, String tenancyId, String functionTerm) {
        counts.merge(functionTerm, 1, Integer::sum);
    }
    @Override public Map<String, Integer> activationCounts(String agentId, String tenancyId) {
        return Map.copyOf(counts);
    }
    @Override public void decay(String agentId, String tenancyId, double decayFactor) {}
    @Override public void clear(String agentId, String tenancyId) { counts.clear(); }
}
```

- [ ] **Step 2: Run the test**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=ChatAgentEndToEndTest#personalityEvolutionEndToEnd -q`
Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-protocol,quarkmind-chat/quarkmind-chat-agent -q`
Expected: PASS — all tests green

- [ ] **Step 4: Update CLAUDE.md test registry**

Add to the unit test listing in CLAUDE.md:
- `PersonalityEvolutionPipelineTest` (quarkmind-core)
- `LlmReflectionDispositionActivatorTest` (quarkmind-chat-agent)
- `DispositionAwareReflectionSynthesizerTest` (quarkmind-chat-agent)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/129/quarkmind add quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatAgentEndToEndTest.java CLAUDE.md
git -C /Users/mdproctor/claude/casehub/slots/129/quarkmind commit -m "feat(#281): personality evolution end-to-end test + CLAUDE.md update

Refs #281"
```

## References

- [specs/issue-279-quarkmind-discord/2026-08-17-quarkmind-discord-design.md] — §Personality System Layer 3
- [specs/issue-279-quarkmind-discord/decisions.md#D6] — Personality: three-layer system, Layer 3 growth
- [specs/issue-279-quarkmind-discord/decisions.md#D19] — LlmReflectionSynthesizer placement
- [io/casehub/eidos/api/DispositionSignalStore.java] — recordActivation, activationCounts, decay APIs
- [io/casehub/eidos/api/DispositionHealth.java] — probe → Aligned | Drifted | EvolutionPending
- [io/casehub/eidos/api/DispositionEvolution.java] — evaluate → Evolved | Dampened
- [io/casehub/eidos/runtime/health/DefaultDispositionHealth.java] — effective weight computation, threshold crossing logic
- [io/casehub/eidos/runtime/health/DefaultDispositionEvolution.java] — swap/replacement/reorganization application
- [io/casehub/neocortex/memory/reflection/ReflectionService.java] — reflect → synthesize → store → fire CDI event
- [io/quarkmind/chat/agent/ChatAgencyLoop.java] — tick flow, checkReflection, submitImportanceScoring
- [io/quarkmind/chat/agent/LlmReflectionSynthesizer.java] — LLM-based synthesis implementation
- [GitHub #281] — personality growth pipeline issue
