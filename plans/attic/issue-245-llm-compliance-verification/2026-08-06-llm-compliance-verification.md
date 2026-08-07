# LLM Compliance Verification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #245 — feat: LLM-evaluated compliance verification for coaching
**Issue group:** #245

**Goal:** When coaching advice has no structured predicate (non-verifiable), dispatch an async LLM call comparing game state before/after to assess compliance, instead of auto-resolving as NEUTRAL.

**Architecture:** Separate evaluation path in `CoachingComplianceEvaluator` — when the verification window closes for non-verifiable advice, a `ComplianceWorkerDispatcher` submits an async LLM call via `CompletableFuture.supplyAsync()` on a managed executor. The worker compares baseline and current game state summaries. An in-flight tracking set guards against orphaned callbacks on game end or supersession.

**Tech Stack:** Quarkus CDI, LangChain4j ChatModel, Java CompletableFuture, ManagedExecutor

## Global Constraints

- Pre-release — breaking changes are free
- All new code in `io.quarkmind.plugin.coaching` package
- Plain JUnit for unit tests, `@QuarkusTest` for integration tests only
- No `@QuarkusTest` when plain JUnit suffices
- `OpenCommitment` is a Java record — adding a field is a breaking constructor change; all callers must be updated in the same task

---

### Task 1: ComplianceVerdict record + parsing

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/coaching/ComplianceVerdict.java`
- Test: `src/test/java/io/quarkmind/plugin/coaching/ComplianceVerdictTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `ComplianceVerdict(String verdict, double confidence, String reasoning)` record with `static ComplianceVerdict parse(String text)`. Verdict values: `"COMPLIED"`, `"PARTIALLY"`, `"IGNORED"`. Parse fallback: malformed → `new ComplianceVerdict("NEUTRAL", 0.0, "")`.

- [ ] **Step 1: Write the failing tests**

```java
package io.quarkmind.plugin.coaching;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ComplianceVerdictTest {

    @Test
    void parse_validJson_allFieldsExtracted() {
        var verdict = ComplianceVerdict.parse("""
            {"verdict": "COMPLIED", "confidence": 0.85, "reasoning": "Built stalkers as advised"}
            """);
        assertEquals("COMPLIED", verdict.verdict());
        assertEquals(0.85, verdict.confidence(), 0.001);
        assertEquals("Built stalkers as advised", verdict.reasoning());
    }

    @Test
    void parse_markdownFencedJson_stripped() {
        var verdict = ComplianceVerdict.parse("""
            ```json
            {"verdict": "IGNORED", "confidence": 0.9, "reasoning": "No new units"}
            ```
            """);
        assertEquals("IGNORED", verdict.verdict());
    }

    @Test
    void parse_malformedJson_returnsNeutral() {
        var verdict = ComplianceVerdict.parse("not json at all");
        assertEquals("NEUTRAL", verdict.verdict());
        assertEquals(0.0, verdict.confidence(), 0.001);
    }

    @Test
    void parse_missingFields_usesDefaults() {
        var verdict = ComplianceVerdict.parse("""
            {"verdict": "PARTIALLY"}
            """);
        assertEquals("PARTIALLY", verdict.verdict());
        assertEquals(0.5, verdict.confidence(), 0.001);
        assertEquals("", verdict.reasoning());
    }

    @Test
    void parse_nullInput_returnsNeutral() {
        var verdict = ComplianceVerdict.parse(null);
        assertEquals("NEUTRAL", verdict.verdict());
    }

    @Test
    void parse_emptyInput_returnsNeutral() {
        var verdict = ComplianceVerdict.parse("");
        assertEquals("NEUTRAL", verdict.verdict());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=ComplianceVerdictTest -q`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement ComplianceVerdict**

```java
package io.quarkmind.plugin.coaching;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

public record ComplianceVerdict(String verdict, double confidence, String reasoning) {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final ComplianceVerdict NEUTRAL = new ComplianceVerdict("NEUTRAL", 0.0, "");

    public static ComplianceVerdict parse(String text) {
        if (text == null || text.isBlank()) return NEUTRAL;
        try {
            String json = text.strip();
            if (json.startsWith("```")) {
                json = json.replaceAll("^```[a-z]*\\n?", "").replaceAll("\\n?```$", "").strip();
            }
            JsonNode node = MAPPER.readTree(json);
            String verdict = node.path("verdict").asText("IGNORED");
            double confidence = node.path("confidence").asDouble(0.5);
            String reasoning = node.path("reasoning").asText("");
            return new ComplianceVerdict(verdict, confidence, reasoning);
        } catch (Exception e) {
            return NEUTRAL;
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -Dtest=ComplianceVerdictTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/plugin/coaching/ComplianceVerdict.java src/test/java/io/quarkmind/plugin/coaching/ComplianceVerdictTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#245): ComplianceVerdict record with JSON parsing and fallback"
```

---

### Task 2: LlmComplianceWorkerFactory — summarisation + prompt + worker

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/coaching/LlmComplianceWorkerFactory.java`
- Test: `src/test/java/io/quarkmind/plugin/coaching/LlmComplianceWorkerFactoryTest.java`

**Interfaces:**
- Consumes: `ComplianceVerdict.parse()` from Task 1, `GameState` record, `CoachingAdvice` record, `Worker`/`WorkerFunction`/`WorkerResult` from `io.casehub.worker.api`, `ChatModel` from LangChain4j
- Produces: `static Worker createWorker(ChatModel chatModel, ComplianceWorkerDispatcher.Callback onCompletion)`, `static String summariseForCompliance(GameState baseline, GameState current, String adviceText)`, `static String buildSystemPrompt()`

- [ ] **Step 1: Write the failing tests**

```java
package io.quarkmind.plugin.coaching;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class LlmComplianceWorkerFactoryTest {

    @Test
    void summarise_unitAddsAndRemovals_renderedCorrectly() {
        var baseline = new GameState(450, 200, 46, 38,
            List.of(unit(UnitType.STALKER), unit(UnitType.STALKER), unit(UnitType.ZEALOT)),
            List.of(building(BuildingType.NEXUS), building(BuildingType.GATEWAY)),
            List.of(), List.of(), List.of(), List.of(), List.of(), 1200, null);
        var current = new GameState(280, 150, 62, 52,
            List.of(unit(UnitType.STALKER), unit(UnitType.STALKER), unit(UnitType.STALKER),
                    unit(UnitType.STALKER), unit(UnitType.ZEALOT), unit(UnitType.SENTRY)),
            List.of(building(BuildingType.NEXUS), building(BuildingType.NEXUS), building(BuildingType.GATEWAY)),
            List.of(), List.of(), List.of(), List.of(), List.of(), 1650, null);

        String result = LlmComplianceWorkerFactory.summariseForCompliance(baseline, current, "Build more Stalkers");

        assertTrue(result.contains("ADVICE: \"Build more Stalkers\""));
        assertTrue(result.contains("BEFORE"));
        assertTrue(result.contains("AFTER"));
        assertTrue(result.contains("CHANGES"));
        assertTrue(result.contains("+2x STALKER"));
        assertTrue(result.contains("+1x SENTRY"));
        assertTrue(result.contains("+1x NEXUS"));
    }

    @Test
    void summarise_noChanges_changesShowsNone() {
        var state = new GameState(400, 200, 46, 38,
            List.of(unit(UnitType.STALKER)),
            List.of(building(BuildingType.NEXUS)),
            List.of(), List.of(), List.of(), List.of(), List.of(), 1200, null);

        String result = LlmComplianceWorkerFactory.summariseForCompliance(state, state, "Do something");

        assertTrue(result.contains("CHANGES"));
        assertTrue(result.contains("No unit changes") || result.contains("(none)"));
    }

    @Test
    void summarise_emptyArmy_handledGracefully() {
        var baseline = new GameState(400, 200, 46, 38,
            List.of(), List.of(), List.of(), List.of(), List.of(), List.of(), List.of(), 1200, null);
        var current = new GameState(300, 150, 54, 44,
            List.of(unit(UnitType.STALKER), unit(UnitType.STALKER)),
            List.of(), List.of(), List.of(), List.of(), List.of(), List.of(), 1650, null);

        String result = LlmComplianceWorkerFactory.summariseForCompliance(baseline, current, "Build units");

        assertTrue(result.contains("+2x STALKER"));
    }

    @Test
    void buildSystemPrompt_containsVerdictVocabulary() {
        String prompt = LlmComplianceWorkerFactory.buildSystemPrompt();

        assertTrue(prompt.contains("COMPLIED"));
        assertTrue(prompt.contains("PARTIALLY"));
        assertTrue(prompt.contains("IGNORED"));
        assertTrue(prompt.contains("verdict"));
        assertTrue(prompt.contains("confidence"));
        assertTrue(prompt.contains("reasoning"));
    }

    private static Unit unit(UnitType type) {
        return new Unit("tag-" + type.name(), type, new Point2d(0, 0));
    }

    private static Building building(BuildingType type) {
        return new Building("tag-" + type.name(), type, new Point2d(0, 0), 1.0f);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=LlmComplianceWorkerFactoryTest -q`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement LlmComplianceWorkerFactory**

```java
package io.quarkmind.plugin.coaching;

import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import io.quarkmind.domain.*;
import org.jboss.logging.Logger;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public final class LlmComplianceWorkerFactory {

    static final String WORKER_ID = "llm-compliance-evaluator";
    static final String CAPABILITY_NAME = "coaching-llm-compliance";

    private static final Logger log = Logger.getLogger(LlmComplianceWorkerFactory.class);

    private LlmComplianceWorkerFactory() {}

    public static Worker createWorker(ChatModel chatModel, ComplianceWorkerDispatcher.Callback onCompletion) {
        return Worker.builder()
            .name(WORKER_ID)
            .capabilityName(CAPABILITY_NAME)
            .function(new WorkerFunction.Sync<>(Map.class, Map.class, (input, scope) ->
                executeCompliance(chatModel, input, onCompletion)))
            .description("LLM compliance evaluator for non-verifiable coaching advice")
            .build();
    }

    @SuppressWarnings("unchecked")
    private static WorkerResult executeCompliance(ChatModel chatModel, Map<String, Object> input,
                                                   ComplianceWorkerDispatcher.Callback onCompletion) {
        try {
            String correlationId = (String) input.get("correlationId");
            String agentId = (String) input.get("agentId");
            long gameFrame = ((Number) input.get("gameFrame")).longValue();
            String summary = (String) input.get("summary");
            CoachingAdvice advice = (CoachingAdvice) input.get("advice");

            ChatRequest request = ChatRequest.builder()
                .messages(new SystemMessage(buildSystemPrompt()), new UserMessage(summary))
                .build();

            ChatResponse response = chatModel.chat(request);
            ComplianceVerdict verdict = ComplianceVerdict.parse(response.aiMessage().text());

            onCompletion.onCompleted(correlationId, agentId, verdict, advice, gameFrame);

            return WorkerResult.of(Map.of("verdict", verdict.verdict(), "confidence", String.valueOf(verdict.confidence())));
        } catch (Exception e) {
            log.warnf(e, "[LLM-COMPLIANCE] Evaluation failed: %s", e.getMessage());
            return WorkerResult.failed("LLM compliance evaluation failed: " + e.getMessage());
        }
    }

    static String buildSystemPrompt() {
        return """
            You are a StarCraft II coaching compliance evaluator. You will receive:
            1. The coaching ADVICE that was given to the human player
            2. A BEFORE snapshot of the game state when the advice was given
            3. An AFTER snapshot of the game state after the verification window
            4. A CHANGES summary showing what happened

            Assess whether the human player's actions reflect compliance with the advice.

            Respond with JSON in this exact format:
            {
              "verdict": "<COMPLIED | PARTIALLY | IGNORED>",
              "confidence": <0.0 to 1.0>,
              "reasoning": "<one sentence explaining your assessment>"
            }

            Verdict definitions:
            - COMPLIED: Clear evidence the player followed the advice
            - PARTIALLY: Some relevant actions but incomplete or mixed execution
            - IGNORED: No evidence of following the advice, or actions contradict it

            Confidence reflects how clear the signal is:
            - High (0.8-1.0): Obvious compliance or obvious disregard
            - Medium (0.5-0.8): Some ambiguity but a clear lean
            - Low (0.0-0.5): Very unclear whether the player attempted compliance
            """;
    }

    static String summariseForCompliance(GameState baseline, GameState current, String adviceText) {
        StringBuilder sb = new StringBuilder();
        sb.append("ADVICE: \"").append(adviceText).append("\"\n\n");

        appendSnapshot(sb, "BEFORE", baseline);
        sb.append("\n");
        appendSnapshot(sb, "AFTER", current);
        sb.append("\n");
        appendChanges(sb, baseline, current);

        return sb.toString();
    }

    private static void appendSnapshot(StringBuilder sb, String label, GameState state) {
        int minutes = (int) (state.gameFrame() / 12.0 / 60);
        int seconds = (int) (state.gameFrame() / 12.0 % 60);
        sb.append(String.format("%s (frame %d, %d:%02d):\n", label, state.gameFrame(), minutes, seconds));
        sb.append(String.format("Resources: %d minerals, %d vespene, %d/%d supply\n",
            state.minerals(), state.vespene(), state.supplyUsed(), state.supply()));

        Map<UnitType, Long> unitCounts = state.myUnits().stream()
            .collect(Collectors.groupingBy(Unit::type, LinkedHashMap::new, Collectors.counting()));
        if (unitCounts.isEmpty()) {
            sb.append("Army: (none)\n");
        } else {
            sb.append("Army: ");
            sb.append(unitCounts.entrySet().stream()
                .map(e -> e.getValue() + "x " + e.getKey().name())
                .collect(Collectors.joining(", ")));
            sb.append(String.format(" (%d units)\n", state.myUnits().size()));
        }

        Map<BuildingType, Long> buildingCounts = state.myBuildings().stream()
            .collect(Collectors.groupingBy(Building::type, LinkedHashMap::new, Collectors.counting()));
        if (!buildingCounts.isEmpty()) {
            sb.append("Buildings: ");
            sb.append(buildingCounts.entrySet().stream()
                .map(e -> e.getValue() > 1 ? e.getValue() + "x " + e.getKey().name() : e.getKey().name())
                .collect(Collectors.joining(", ")));
            sb.append("\n");
        }
    }

    private static void appendChanges(StringBuilder sb, GameState baseline, GameState current) {
        sb.append("CHANGES:\n");

        Map<UnitType, Long> beforeUnits = baseline.myUnits().stream()
            .collect(Collectors.groupingBy(Unit::type, Collectors.counting()));
        Map<UnitType, Long> afterUnits = current.myUnits().stream()
            .collect(Collectors.groupingBy(Unit::type, Collectors.counting()));
        boolean anyUnitChange = appendDelta(sb, beforeUnits, afterUnits);

        Map<BuildingType, Long> beforeBuildings = baseline.myBuildings().stream()
            .collect(Collectors.groupingBy(Building::type, Collectors.counting()));
        Map<BuildingType, Long> afterBuildings = current.myBuildings().stream()
            .collect(Collectors.groupingBy(Building::type, Collectors.counting()));
        boolean anyBuildingChange = appendDelta(sb, beforeBuildings, afterBuildings);

        if (!anyUnitChange && !anyBuildingChange) {
            sb.append("No unit or building changes\n");
        }

        sb.append(String.format("Minerals: %+d, Vespene: %+d, Supply: %+d/%+d\n",
            current.minerals() - baseline.minerals(),
            current.vespene() - baseline.vespene(),
            current.supplyUsed() - baseline.supplyUsed(),
            current.supply() - baseline.supply()));
    }

    private static <T extends Enum<T>> boolean appendDelta(StringBuilder sb,
            Map<T, Long> before, Map<T, Long> after) {
        boolean any = false;
        var allKeys = new java.util.LinkedHashSet<>(before.keySet());
        allKeys.addAll(after.keySet());
        for (T key : allKeys) {
            long diff = after.getOrDefault(key, 0L) - before.getOrDefault(key, 0L);
            if (diff != 0) {
                sb.append(String.format("%+dx %s\n", diff, key.name()));
                any = true;
            }
        }
        return any;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -Dtest=LlmComplianceWorkerFactoryTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/plugin/coaching/LlmComplianceWorkerFactory.java src/test/java/io/quarkmind/plugin/coaching/LlmComplianceWorkerFactoryTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#245): LlmComplianceWorkerFactory — summarisation, prompt, worker creation"
```

---

### Task 3: OpenCommitment + CoachingChannelBroker — baseline state capture

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/coaching/OpenCommitment.java`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingChannelBroker.java`
- Modify: `src/test/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluatorTest.java` — update all `new OpenCommitment(...)` calls
- Modify: `src/test/java/io/quarkmind/qa/workbench/CoachingAcknowledgmentHandlerTest.java` — update all `new OpenCommitment(...)` calls
- Modify: `src/test/java/io/quarkmind/plugin/coaching/CoachingIntegrationIT.java` — update `new OpenCommitment(...)` call

**Interfaces:**
- Consumes: `GameState` record
- Produces: `OpenCommitment(String correlationId, String agentId, CoachingAdvice advice, long issuedAtFrame, GameState baselineState)` — `baselineState` is non-null for non-verifiable advice, null for verifiable advice

- [ ] **Step 1: Add `baselineState` to OpenCommitment**

Change `OpenCommitment.java` to:
```java
package io.quarkmind.plugin.coaching;

public record OpenCommitment(
        String correlationId,
        String agentId,
        CoachingAdvice advice,
        long issuedAtFrame,
        io.quarkmind.domain.GameState baselineState
) {}
```

- [ ] **Step 2: Update CoachingChannelBroker to populate baselineState**

In `CoachingChannelBroker.onCoachingCompleted()`, change the `commitments.put()` call at line 91:

Before:
```java
OpenCommitment old = commitments.put(domain, new OpenCommitment(correlationId, event.workerId(), adviceWithBaseline, gameFrame));
```

After:
```java
var baselineForLlm = !adviceWithBaseline.isVerifiable() ? event.triggerState() : null;
OpenCommitment old = commitments.put(domain, new OpenCommitment(correlationId, event.workerId(), adviceWithBaseline, gameFrame, baselineForLlm));
```

- [ ] **Step 3: Update all test call sites**

Every `new OpenCommitment("corr-N", "worker-1", advice, NNN)` becomes `new OpenCommitment("corr-N", "worker-1", advice, NNN, null)`.

Files to update:
- `CoachingComplianceEvaluatorTest.java` — 10 call sites
- `CoachingAcknowledgmentHandlerTest.java` — 3 call sites
- `CoachingIntegrationIT.java` — 1 call site

- [ ] **Step 4: Run all coaching tests to verify**

Run: `mvn test -Dtest=CoachingComplianceEvaluatorTest,CoachingAcknowledgmentHandlerTest -q`
Expected: PASS — all existing tests pass with the new null parameter

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/plugin/coaching/OpenCommitment.java src/main/java/io/quarkmind/plugin/coaching/CoachingChannelBroker.java src/test/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluatorTest.java src/test/java/io/quarkmind/qa/workbench/CoachingAcknowledgmentHandlerTest.java src/test/java/io/quarkmind/plugin/coaching/CoachingIntegrationIT.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#245): OpenCommitment gains baselineState for LLM compliance evaluation"
```

---

### Task 4: ComplianceWorkerDispatcher — async dispatch with in-flight tracking

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/coaching/ComplianceWorkerDispatcher.java`
- Test: `src/test/java/io/quarkmind/plugin/coaching/ComplianceWorkerDispatcherTest.java`

**Interfaces:**
- Consumes: `LlmComplianceWorkerFactory.summariseForCompliance()` from Task 2, `ComplianceVerdict` from Task 1, `OpenCommitment` from Task 3, `ChatModel`, `ManagedExecutor`, `CoachingEffectivenessTrustRecorder.record()`, `Event<CoachingComplianceResolved>`
- Produces: `void dispatch(OpenCommitment commitment, GameState currentState)`, `boolean isAvailable()`, `void cancelAll()`, nested `Callback` interface

- [ ] **Step 1: Write the failing tests**

```java
package io.quarkmind.plugin.coaching;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.concurrent.atomic.AtomicReference;
import static org.junit.jupiter.api.Assertions.*;

class ComplianceWorkerDispatcherTest {

    @Test
    void verdictMapping_complied_mapsToEndorsed() {
        var result = ComplianceWorkerDispatcher.mapVerdictToOutcome("COMPLIED");
        assertEquals("ENDORSED", result);
    }

    @Test
    void verdictMapping_partially_mapsToPartial() {
        var result = ComplianceWorkerDispatcher.mapVerdictToOutcome("PARTIALLY");
        assertEquals("PARTIAL", result);
    }

    @Test
    void verdictMapping_ignored_mapsToChallenged() {
        var result = ComplianceWorkerDispatcher.mapVerdictToOutcome("IGNORED");
        assertEquals("CHALLENGED", result);
    }

    @Test
    void verdictMapping_neutral_mapsToNeutral() {
        var result = ComplianceWorkerDispatcher.mapVerdictToOutcome("NEUTRAL");
        assertEquals("NEUTRAL", result);
    }

    @Test
    void verdictMapping_unknown_mapsToNeutral() {
        var result = ComplianceWorkerDispatcher.mapVerdictToOutcome("SOMETHING_UNEXPECTED");
        assertEquals("NEUTRAL", result);
    }

    @Test
    void cancelAll_discardsInFlightEntries() {
        var dispatcher = new ComplianceWorkerDispatcher();
        dispatcher.trackInFlight("corr-1");
        dispatcher.trackInFlight("corr-2");
        assertTrue(dispatcher.isInFlight("corr-1"));

        dispatcher.cancelAll();

        assertFalse(dispatcher.isInFlight("corr-1"));
        assertFalse(dispatcher.isInFlight("corr-2"));
    }

    @Test
    void removeFromFlight_returnsTrueIfPresent() {
        var dispatcher = new ComplianceWorkerDispatcher();
        dispatcher.trackInFlight("corr-1");
        assertTrue(dispatcher.removeFromFlight("corr-1"));
        assertFalse(dispatcher.removeFromFlight("corr-1"));
    }

    @Test
    void removeFromFlight_returnsFalseIfAbsent() {
        var dispatcher = new ComplianceWorkerDispatcher();
        assertFalse(dispatcher.removeFromFlight("corr-unknown"));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=ComplianceWorkerDispatcherTest -q`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement ComplianceWorkerDispatcher**

```java
package io.quarkmind.plugin.coaching;

import dev.langchain4j.model.chat.ChatModel;
import io.quarkmind.domain.GameState;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.eclipse.microprofile.context.ManagedExecutor;
import org.jboss.logging.Logger;

import java.util.Map;
import java.util.Set;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;

@ApplicationScoped
public class ComplianceWorkerDispatcher {

    private static final Logger log = Logger.getLogger(ComplianceWorkerDispatcher.class);

    @Inject
    Instance<ChatModel> chatModel;
    @Inject
    CoachingEffectivenessTrustRecorder recorder;
    @Inject
    jakarta.enterprise.event.Event<CoachingComplianceResolved> complianceResolvedEvent;
    @Inject
    ManagedExecutor executor;
    @ConfigProperty(name = "quarkmind.coaching.compliance.llm-timeout-seconds", defaultValue = "10")
    int timeoutSeconds;

    private final Set<String> inFlight = ConcurrentHashMap.newKeySet();

    ComplianceWorkerDispatcher() {}

    public boolean isAvailable() {
        return chatModel != null && chatModel.isResolvable();
    }

    public void dispatch(OpenCommitment commitment, GameState currentState) {
        String correlationId = commitment.correlationId();
        trackInFlight(correlationId);

        String summary = LlmComplianceWorkerFactory.summariseForCompliance(
            commitment.baselineState(), currentState, commitment.advice().advice());

        CompletableFuture.supplyAsync(() -> {
            var request = dev.langchain4j.model.chat.request.ChatRequest.builder()
                .messages(
                    new dev.langchain4j.data.message.SystemMessage(LlmComplianceWorkerFactory.buildSystemPrompt()),
                    new dev.langchain4j.data.message.UserMessage(summary))
                .build();
            return chatModel.get().chat(request);
        }, executor)
        .orTimeout(timeoutSeconds, TimeUnit.SECONDS)
        .whenComplete((response, throwable) -> {
            if (!removeFromFlight(correlationId)) {
                log.debugf("[LLM-COMPLIANCE] Discarding result for %s — no longer in flight", correlationId);
                return;
            }
            if (throwable != null) {
                log.warnf("[LLM-COMPLIANCE] Evaluation failed for %s: %s", correlationId, throwable.getMessage());
                recorder.record(correlationId, commitment.agentId(), "NEUTRAL", commitment.advice());
                fireResolved(commitment, "NEUTRAL");
                return;
            }
            ComplianceVerdict verdict = ComplianceVerdict.parse(response.aiMessage().text());
            String outcome = mapVerdictToOutcome(verdict.verdict());
            log.infof("[LLM-COMPLIANCE] %s → %s (confidence=%.2f, reason=%s)",
                correlationId, outcome, verdict.confidence(), verdict.reasoning());
            recorder.record(correlationId, commitment.agentId(), outcome, commitment.advice());
            fireResolved(commitment, outcome);
        });
    }

    public void cancelAll() {
        inFlight.clear();
    }

    void trackInFlight(String correlationId) {
        inFlight.add(correlationId);
    }

    boolean removeFromFlight(String correlationId) {
        return inFlight.remove(correlationId);
    }

    boolean isInFlight(String correlationId) {
        return inFlight.contains(correlationId);
    }

    static String mapVerdictToOutcome(String verdict) {
        return switch (verdict) {
            case "COMPLIED" -> "ENDORSED";
            case "PARTIALLY" -> "PARTIAL";
            case "IGNORED" -> "CHALLENGED";
            default -> "NEUTRAL";
        };
    }

    private void fireResolved(OpenCommitment commitment, String status) {
        if (complianceResolvedEvent != null) {
            complianceResolvedEvent.fireAsync(new CoachingComplianceResolved(
                commitment.issuedAtFrame(), commitment.advice().domainTag(), status, commitment.correlationId()));
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -Dtest=ComplianceWorkerDispatcherTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/plugin/coaching/ComplianceWorkerDispatcher.java src/test/java/io/quarkmind/plugin/coaching/ComplianceWorkerDispatcherTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#245): ComplianceWorkerDispatcher — async LLM dispatch with in-flight tracking"
```

---

### Task 5: CoachingComplianceEvaluator — extend for LLM dispatch

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluator.java`
- Modify: `src/test/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluatorTest.java`

**Interfaces:**
- Consumes: `ComplianceWorkerDispatcher.dispatch()`, `ComplianceWorkerDispatcher.isAvailable()`, `ComplianceWorkerDispatcher.cancelAll()` from Task 4, `OpenCommitment.baselineState()` from Task 3
- Produces: Extended `evaluate()` — non-verifiable advice with baseline dispatches to LLM instead of auto-NEUTRAL. Extended `withdrawAll()` — calls `cancelAll()` on dispatcher.

- [ ] **Step 1: Write the failing tests for LLM dispatch path**

Add these tests to `CoachingComplianceEvaluatorTest.java`:

```java
@Test
void nonVerifiable_withBaselineAndDispatcher_dispatchesLlm() {
    var commitments = new ConcurrentHashMap<CoachingDomain, OpenCommitment>();
    var recorder = mock(CoachingEffectivenessTrustRecorder.class);
    var locationResolver = new LocationResolver();
    var dispatcher = mock(ComplianceWorkerDispatcher.class);
    when(dispatcher.isAvailable()).thenReturn(true);
    var evaluator = new CoachingComplianceEvaluator(commitments, recorder, locationResolver, dispatcher);

    var advice = new CoachingAdvice("Improve your macro", CoachingDomain.BUILD, null, 200);
    var baselineState = new GameState(400, 200, 46, 38, List.of(), List.of(),
        List.of(), List.of(), List.of(), List.of(), List.of(), 100, null);
    commitments.put(CoachingDomain.BUILD, new OpenCommitment("corr-1", "worker-1", advice, 100, baselineState));

    var currentState = new GameState(300, 150, 54, 44, List.of(), List.of(),
        List.of(), List.of(), List.of(), List.of(), List.of(), 400, null);
    evaluator.evaluate(currentState, 400);

    verify(dispatcher).dispatch(any(), eq(currentState));
    verify(recorder, never()).record(any(), any(), any(), any());
    assertTrue(commitments.isEmpty());
}

@Test
void nonVerifiable_withoutBaseline_degradesToNeutral() {
    var commitments = new ConcurrentHashMap<CoachingDomain, OpenCommitment>();
    var recorder = mock(CoachingEffectivenessTrustRecorder.class);
    var locationResolver = new LocationResolver();
    var dispatcher = mock(ComplianceWorkerDispatcher.class);
    when(dispatcher.isAvailable()).thenReturn(true);
    var evaluator = new CoachingComplianceEvaluator(commitments, recorder, locationResolver, dispatcher);

    var advice = new CoachingAdvice("Improve your macro", CoachingDomain.BUILD, null, 200);
    commitments.put(CoachingDomain.BUILD, new OpenCommitment("corr-1", "worker-1", advice, 100, null));

    var currentState = new GameState(300, 150, 54, 44, List.of(), List.of(),
        List.of(), List.of(), List.of(), List.of(), List.of(), 400, null);
    evaluator.evaluate(currentState, 400);

    verify(dispatcher, never()).dispatch(any(), any());
    verify(recorder).record("corr-1", "worker-1", "NEUTRAL", advice);
}

@Test
void nonVerifiable_withoutDispatcher_degradesToNeutral() {
    var commitments = new ConcurrentHashMap<CoachingDomain, OpenCommitment>();
    var recorder = mock(CoachingEffectivenessTrustRecorder.class);
    var locationResolver = new LocationResolver();
    var evaluator = new CoachingComplianceEvaluator(commitments, recorder, locationResolver);

    var advice = new CoachingAdvice("Improve your macro", CoachingDomain.BUILD, null, 200);
    var baselineState = new GameState(400, 200, 46, 38, List.of(), List.of(),
        List.of(), List.of(), List.of(), List.of(), List.of(), 100, null);
    commitments.put(CoachingDomain.BUILD, new OpenCommitment("corr-1", "worker-1", advice, 100, baselineState));

    var currentState = new GameState(300, 150, 54, 44, List.of(), List.of(),
        List.of(), List.of(), List.of(), List.of(), List.of(), 400, null);
    evaluator.evaluate(currentState, 400);

    verify(recorder).record("corr-1", "worker-1", "NEUTRAL", advice);
}

@Test
void withdrawAll_callsCancelAllOnDispatcher() {
    var commitments = new ConcurrentHashMap<CoachingDomain, OpenCommitment>();
    var recorder = mock(CoachingEffectivenessTrustRecorder.class);
    var locationResolver = new LocationResolver();
    var dispatcher = mock(ComplianceWorkerDispatcher.class);
    var evaluator = new CoachingComplianceEvaluator(commitments, recorder, locationResolver, dispatcher);

    evaluator.withdrawAll();

    verify(dispatcher).cancelAll();
}
```

- [ ] **Step 2: Run new tests to verify they fail**

Run: `mvn test -Dtest=CoachingComplianceEvaluatorTest -q`
Expected: FAIL — new constructor and dispatch logic do not exist

- [ ] **Step 3: Modify CoachingComplianceEvaluator**

Changes:
1. Add `ComplianceWorkerDispatcher dispatcher` field (nullable)
2. Add constructor overload accepting dispatcher
3. Modify CDI constructor to inject `Instance<ComplianceWorkerDispatcher>`
4. Modify `evaluate()` — the `!advice.isVerifiable()` branch checks for baseline + dispatcher
5. Modify `withdrawAll()` — calls `dispatcher.cancelAll()` when present

- [ ] **Step 4: Run all coaching tests**

Run: `mvn test -Dtest=CoachingComplianceEvaluatorTest,CoachingAcknowledgmentHandlerTest -q`
Expected: PASS — all existing tests pass, new tests pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluator.java src/test/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluatorTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#245): extend CoachingComplianceEvaluator with LLM dispatch path"
```

---

### Task 6: Integration test — full cycle

**Files:**
- Create: `src/test/java/io/quarkmind/plugin/coaching/LlmComplianceIT.java`

**Interfaces:**
- Consumes: All components from Tasks 1-5, mock `ChatModel`

- [ ] **Step 1: Write the integration test**

```java
package io.quarkmind.plugin.coaching;

import io.quarkmind.domain.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.concurrent.ConcurrentHashMap;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class LlmComplianceIT {

    @Inject
    CoachingComplianceEvaluator evaluator;
    @Inject
    CoachingChannelBroker broker;

    @Test
    void fullCycle_nonVerifiableAdvice_llmEvaluatesCompliance() throws InterruptedException {
        var advice = new CoachingAdvice("Improve your macro", CoachingDomain.BUILD, null, 200);
        var triggerState = new GameState(400, 200, 46, 38, List.of(), List.of(),
            List.of(), List.of(), List.of(), List.of(), List.of(), 100, null);

        broker.onCoachingCompleted(new CoachingCompleted(
            "test-worker", "coaching", 100, advice,
            CoachingUrgencyTier.ECONOMIC, 50, triggerState));

        var commitments = broker.commitments();
        assertEquals(1, commitments.size());
        var commitment = commitments.values().iterator().next();
        assertNotNull(commitment.baselineState());

        var currentState = new GameState(300, 150, 54, 44,
            List.of(new Unit("u1", UnitType.STALKER, new Point2d(10, 10))),
            List.of(), List.of(), List.of(), List.of(), List.of(), List.of(), 400, null);

        evaluator.evaluate(currentState, 400);

        assertTrue(commitments.isEmpty());

        Thread.sleep(300);
    }
}
```

- [ ] **Step 2: Run the integration test**

Run: `mvn test -Dtest=LlmComplianceIT -q`
Expected: PASS if ChatModel is available in test profile, otherwise the test verifies the NEUTRAL degradation path

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/test/java/io/quarkmind/plugin/coaching/LlmComplianceIT.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "test(#245): LlmComplianceIT — full cycle integration test"
```

---

### Task 7: Full test suite verification

- [ ] **Step 1: Run all tests**

Run: `mvn test -q`
Expected: PASS — no regressions

- [ ] **Step 2: Commit any remaining fixes**

If any tests failed due to the `OpenCommitment` change being missed in a call site, fix and commit.
