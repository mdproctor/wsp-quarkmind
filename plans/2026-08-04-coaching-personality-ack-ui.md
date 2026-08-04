# Coaching Personality Model & Acknowledgment UI — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #251 — epic: coaching personality model
**Issue group:** #251, #246, #247, #248

**Goal:** Expand coaching from two fixed personalities to a 2-axis disposition model with urgency-driven intra-game switching, and add human Accept/Dismiss acknowledgment to the workbench coaching UI.

**Architecture:** Bottom-up build order: data model enrichment → CoachingStyle enum + prompt refactoring → broker/evaluator pipeline enrichment → acknowledgment handler + WebSocket → visualizer.js UI. Two decoupled concerns (personality model and acknowledgment UI) sharing a common data model foundation.

**Tech Stack:** Quarkus CDI, casehub-qhorus MessageService, casehub-eidos AgentDescriptor/DispositionAxis, Quarkus WebSockets Next, vanilla JS (visualizer.js), Playwright for render tests.

## Global Constraints

- ConscientiousnessTerm vocabulary values: RISK_APPETITE uses `BOLD`/"bold" and `CONSERVATIVE`/"conservative". SOCIAL_ORIENTATION uses `INDEPENDENT`/"independent" (tell/assertive) and `COLLABORATIVE`/"collaborative" (ask/Socratic). There is NO "assertive" term — the spec's conceptual "assertive" maps to `INDEPENDENT`.
- `@UnlessBuildProfile("prod")` on all workbench/QA components.
- Domain model (`domain/`) stays plain Java — no CDI, no Quarkus imports.
- OpenCommitment constructor call sites exist in: `CoachingChannelBrokerTest` (6 sites), `CoachingComplianceEvaluatorTest` (7 sites), `CoachingIntegrationIT`, `WorkbenchSocketIT`.
- `CoachingEffectivenessTrustRecorder.record()` is called by: `CoachingComplianceEvaluator.evaluate()` (3 sites), `CoachingComplianceEvaluator.withdrawAll()` (1 site).
- Use `ide_insert_member` / `ide_replace_member` for code edits. Use `ide_refactor_rename` for renames. No bash file operations on source files.

---

### Task 1: Data Model Enrichment

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/coaching/OpenCommitment.java`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingComplianceResolved.java`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingAdvicePublished.java`
- Modify: `src/main/java/io/quarkmind/qa/workbench/CoachingPayload.java`
- Modify: `src/main/java/io/quarkmind/qa/workbench/CoachingCompliancePayload.java`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingEffectivenessTrustRecorder.java`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluator.java`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingChannelBroker.java`
- Modify: `src/main/java/io/quarkmind/qa/workbench/WorkbenchEnricher.java`
- Test: `src/test/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluatorTest.java`
- Test: `src/test/java/io/quarkmind/plugin/coaching/CoachingChannelBrokerTest.java`
- Test: `src/test/java/io/quarkmind/qa/workbench/WorkbenchEventTest.java`

**Interfaces:**
- Produces: `OpenCommitment(String correlationId, String agentId, CoachingAdvice advice, long issuedAtFrame)`
- Produces: `CoachingComplianceResolved(long gameFrame, CoachingDomain domain, String status, String correlationId)`
- Produces: `CoachingAdvicePublished(CoachingAdvice advice, CoachingUrgencyTier urgencyTier, long gameFrame, String correlationId)`
- Produces: `CoachingPayload(String advice, CoachingDomain domain, CoachingUrgencyTier urgency, long gameFrame, String correlationId)`
- Produces: `CoachingCompliancePayload(long gameFrame, CoachingDomain domain, String status, String correlationId)`
- Produces: `CoachingEffectivenessTrustRecorder.record(String correlationId, String agentId, String outcome, CoachingAdvice advice)`

- [ ] **Step 1: Update all 5 records — add agentId/correlationId fields**

`OpenCommitment.java`:
```java
public record OpenCommitment(String correlationId, String agentId, CoachingAdvice advice, long issuedAtFrame) {}
```

`CoachingComplianceResolved.java`:
```java
public record CoachingComplianceResolved(long gameFrame, CoachingDomain domain, String status, String correlationId) {}
```

`CoachingAdvicePublished.java`:
```java
public record CoachingAdvicePublished(CoachingAdvice advice, CoachingUrgencyTier urgencyTier, long gameFrame, String correlationId) {}
```

`CoachingPayload.java`:
```java
public record CoachingPayload(String advice, CoachingDomain domain, CoachingUrgencyTier urgency, long gameFrame, String correlationId) implements WorkbenchPayload {}
```

`CoachingCompliancePayload.java`:
```java
public record CoachingCompliancePayload(long gameFrame, CoachingDomain domain, String status, String correlationId) implements WorkbenchPayload {}
```

- [ ] **Step 2: Update CoachingEffectivenessTrustRecorder — add agentId parameter**

```java
public void record(String correlationId, String agentId, String outcome, CoachingAdvice advice) {
    log.infof("[COACHING-TRUST] correlationId=%s agentId=%s outcome=%s domain=%s advice=%s",
        correlationId, agentId, outcome, advice.domainTag(), advice.advice());
}
```

- [ ] **Step 3: Update CoachingChannelBroker.onCoachingCompleted() — pass agentId and correlationId**

```java
// Pass event.workerId() as agentId to OpenCommitment:
commitments.put(domain, new OpenCommitment(correlationId, event.workerId(), adviceWithBaseline, gameFrame));

// Pass correlationId to CoachingAdvicePublished:
coachingAdviceEvent.fire(new CoachingAdvicePublished(event.advice(), event.urgencyTier(), gameFrame, correlationId));
```

- [ ] **Step 4: Update CoachingComplianceEvaluator — agentId to recorder, correlationId to events**

In `evaluate()`, update all 3 `recorder.record()` calls to pass `commitment.agentId()`:
```java
recorder.record(commitment.correlationId(), commitment.agentId(), "NEUTRAL", advice);
recorder.record(commitment.correlationId(), commitment.agentId(), "ENDORSED", advice);
recorder.record(commitment.correlationId(), commitment.agentId(), "CHALLENGED", advice);
```

Update all `fireComplianceResolved()` calls to pass correlationId:
```java
fireComplianceResolved(currentFrame, domain, "NEUTRAL", commitment.correlationId());
fireComplianceResolved(currentFrame, domain, "ENDORSED", commitment.correlationId());
fireComplianceResolved(currentFrame, domain, "CHALLENGED", commitment.correlationId());
```

Update `withdrawAll()`:
```java
commitments.forEach((domain, commitment) ->
    recorder.record(commitment.correlationId(), commitment.agentId(), "NEUTRAL", commitment.advice()));
```

Update `fireComplianceResolved` signature:
```java
private void fireComplianceResolved(long frame, CoachingDomain domain, String status, String correlationId) {
    if (complianceResolvedEvent != null) {
        complianceResolvedEvent.fire(new CoachingComplianceResolved(frame, domain, status, correlationId));
    }
}
```

- [ ] **Step 5: Update WorkbenchEnricher — pass correlationId through both handlers**

```java
void onCoachingAdvice(@Observes CoachingAdvicePublished event) {
    broadcaster.broadcast(new WorkbenchEvent("coaching",
        new CoachingPayload(event.advice().advice(), event.advice().domainTag(),
            event.urgencyTier(), event.gameFrame(), event.correlationId())));
}

void onCoachingCompliance(@Observes CoachingComplianceResolved event) {
    broadcaster.broadcast(new WorkbenchEvent("coaching_compliance",
        new CoachingCompliancePayload(event.gameFrame(), event.domain(), event.status(), event.correlationId())));
}
```

- [ ] **Step 6: Fix test compilation — all OpenCommitment constructors**

Every `new OpenCommitment("corr-N", advice, frame)` becomes `new OpenCommitment("corr-N", "worker-1", advice, frame)`.

`CoachingComplianceEvaluatorTest`: 7 sites.
`CoachingChannelBrokerTest`: no direct construction (broker creates them).

Update `TestTrustRecorder`:
```java
static class TestTrustRecorder extends CoachingEffectivenessTrustRecorder {
    String lastOutcome;
    String lastAgentId;
    Map<String, String> outcomes = new LinkedHashMap<>();
    @Override
    public void record(String correlationId, String agentId, String outcome, CoachingAdvice advice) {
        this.lastOutcome = outcome;
        this.lastAgentId = agentId;
        this.outcomes.put(correlationId, outcome);
    }
}
```

- [ ] **Step 7: Fix WorkbenchEventTest — add correlationId to assertions**

Update coaching and compliance test methods to include correlationId parameter.

- [ ] **Step 8: Add test — agentId flows through broker to commitment**

```java
@Test
void onCoachingCompleted_storesAgentId() {
    var advice = new CoachingAdvice("build stalkers", CoachingDomain.MILITARY,
                                    new CountDelta(UnitType.STALKER, null, 3, 0), 450);
    var event = new CoachingCompleted("worker-1", "coaching", 100,
                                      advice, CoachingUrgencyTier.STRATEGIC, 500, null);
    broker.onCoachingCompleted(event);
    assertThat(broker.commitments().get(CoachingDomain.MILITARY).agentId()).isEqualTo("worker-1");
}
```

- [ ] **Step 9: Run all tests**

Run: `mvn test -q`
Expected: All tests pass.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#251): enrich data model — agentId on OpenCommitment, correlationId on compliance/advice events

Add agentId to OpenCommitment for future per-personality trust attribution.
Add correlationId to CoachingComplianceResolved, CoachingAdvicePublished,
CoachingPayload, and CoachingCompliancePayload for end-to-end correlation.
Update CoachingEffectivenessTrustRecorder.record() signature with agentId.
Refs #251"
```

---

### Task 2: CoachingStyle Enum + Prompt Refactoring (#247, #246)

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/coaching/CoachingStyle.java`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingWorkerFactory.java`
- Modify: `src/main/java/io/quarkmind/plugin/advisory/QuarkMindAgentRegistrar.java`
- Create: `src/test/java/io/quarkmind/plugin/coaching/CoachingStyleTest.java`
- Modify: `src/test/java/io/quarkmind/plugin/coaching/CoachingWorkerFactoryTest.java`

**Interfaces:**
- Consumes: `CoachingUrgencyTier` (existing enum: CRISIS, STRATEGIC, ECONOMIC)
- Consumes: `AgentDisposition.primaryTerm(DispositionAxis)` → `String` or null
- Produces: `CoachingStyle.resolve(AgentDisposition, CoachingUrgencyTier)` → `CoachingStyle`
- Produces: `CoachingWorkerFactory.buildSystemPrompt(AgentDescriptor, CoachingUrgencyTier)` → `String`

- [ ] **Step 1: Write CoachingStyleTest — 15 tests covering all quadrant × tier combinations + null**

```java
package io.quarkmind.plugin.coaching;

import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.vocab.ConscientiousnessTerm;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class CoachingStyleTest {

    private AgentDisposition boldIndependent() {
        return AgentDisposition.builder()
            .riskAppetite(ConscientiousnessTerm.BOLD.value())
            .socialOrient(ConscientiousnessTerm.INDEPENDENT.value())
            .build();
    }

    private AgentDisposition boldCollaborative() {
        return AgentDisposition.builder()
            .riskAppetite(ConscientiousnessTerm.BOLD.value())
            .socialOrient(ConscientiousnessTerm.COLLABORATIVE.value())
            .build();
    }

    private AgentDisposition cautiousIndependent() {
        return AgentDisposition.builder()
            .riskAppetite(ConscientiousnessTerm.CONSERVATIVE.value())
            .socialOrient(ConscientiousnessTerm.INDEPENDENT.value())
            .build();
    }

    private AgentDisposition cautiousCollaborative() {
        return AgentDisposition.builder()
            .riskAppetite(ConscientiousnessTerm.CONSERVATIVE.value())
            .socialOrient(ConscientiousnessTerm.COLLABORATIVE.value())
            .build();
    }

    @Test void crisis_boldIndependent_commander()      { assertThat(CoachingStyle.resolve(boldIndependent(), CoachingUrgencyTier.CRISIS)).isEqualTo(CoachingStyle.COMMANDER); }
    @Test void crisis_boldCollaborative_commander()     { assertThat(CoachingStyle.resolve(boldCollaborative(), CoachingUrgencyTier.CRISIS)).isEqualTo(CoachingStyle.COMMANDER); }
    @Test void crisis_cautiousIndependent_commander()   { assertThat(CoachingStyle.resolve(cautiousIndependent(), CoachingUrgencyTier.CRISIS)).isEqualTo(CoachingStyle.COMMANDER); }
    @Test void crisis_cautiousCollaborative_commander() { assertThat(CoachingStyle.resolve(cautiousCollaborative(), CoachingUrgencyTier.CRISIS)).isEqualTo(CoachingStyle.COMMANDER); }

    @Test void strategic_boldIndependent_commander()      { assertThat(CoachingStyle.resolve(boldIndependent(), CoachingUrgencyTier.STRATEGIC)).isEqualTo(CoachingStyle.COMMANDER); }
    @Test void strategic_boldCollaborative_rally()         { assertThat(CoachingStyle.resolve(boldCollaborative(), CoachingUrgencyTier.STRATEGIC)).isEqualTo(CoachingStyle.RALLY); }
    @Test void strategic_cautiousIndependent_commander()   { assertThat(CoachingStyle.resolve(cautiousIndependent(), CoachingUrgencyTier.STRATEGIC)).isEqualTo(CoachingStyle.COMMANDER); }
    @Test void strategic_cautiousCollaborative_rally()     { assertThat(CoachingStyle.resolve(cautiousCollaborative(), CoachingUrgencyTier.STRATEGIC)).isEqualTo(CoachingStyle.RALLY); }

    @Test void economic_boldIndependent_commander()      { assertThat(CoachingStyle.resolve(boldIndependent(), CoachingUrgencyTier.ECONOMIC)).isEqualTo(CoachingStyle.COMMANDER); }
    @Test void economic_boldCollaborative_rally()         { assertThat(CoachingStyle.resolve(boldCollaborative(), CoachingUrgencyTier.ECONOMIC)).isEqualTo(CoachingStyle.RALLY); }
    @Test void economic_cautiousIndependent_instructor()  { assertThat(CoachingStyle.resolve(cautiousIndependent(), CoachingUrgencyTier.ECONOMIC)).isEqualTo(CoachingStyle.INSTRUCTOR); }
    @Test void economic_cautiousCollaborative_mentor()    { assertThat(CoachingStyle.resolve(cautiousCollaborative(), CoachingUrgencyTier.ECONOMIC)).isEqualTo(CoachingStyle.MENTOR); }

    @Test void nullDisposition_economic_mentor()    { assertThat(CoachingStyle.resolve(null, CoachingUrgencyTier.ECONOMIC)).isEqualTo(CoachingStyle.MENTOR); }
    @Test void nullDisposition_crisis_commander()   { assertThat(CoachingStyle.resolve(null, CoachingUrgencyTier.CRISIS)).isEqualTo(CoachingStyle.COMMANDER); }
    @Test void nullDisposition_strategic_rally()    { assertThat(CoachingStyle.resolve(null, CoachingUrgencyTier.STRATEGIC)).isEqualTo(CoachingStyle.RALLY); }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=CoachingStyleTest -q`
Expected: FAIL — `CoachingStyle` class does not exist.

- [ ] **Step 3: Create CoachingStyle enum**

```java
package io.quarkmind.plugin.coaching;

import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.api.DispositionAxis;

public enum CoachingStyle {
    COMMANDER,
    RALLY,
    INSTRUCTOR,
    MENTOR;

    public static CoachingStyle resolve(AgentDisposition disposition, CoachingUrgencyTier tier) {
        boolean bold = tier == CoachingUrgencyTier.CRISIS
                    || tier == CoachingUrgencyTier.STRATEGIC
                    || (disposition != null && "bold".equals(disposition.primaryTerm(DispositionAxis.RISK_APPETITE)));
        boolean independent = tier == CoachingUrgencyTier.CRISIS
                           || (disposition != null && "independent".equals(disposition.primaryTerm(DispositionAxis.SOCIAL_ORIENTATION)));

        if (bold && independent) return COMMANDER;
        if (bold)                return RALLY;
        if (independent)         return INSTRUCTOR;
        return MENTOR;
    }
}
```

- [ ] **Step 4: Run CoachingStyleTest — verify all 15 pass**

Run: `mvn test -Dtest=CoachingStyleTest -q`
Expected: All 15 tests PASS.

- [ ] **Step 5: Refactor buildSystemPrompt — tier-based with 4 quadrant templates**

Change signature from `buildSystemPrompt(AgentDescriptor descriptor, boolean crisisOverride)` to `buildSystemPrompt(AgentDescriptor descriptor, CoachingUrgencyTier tier)`.

Replace the binary directive/Socratic block:
```java
static String buildSystemPrompt(AgentDescriptor descriptor, CoachingUrgencyTier tier) {
    AgentDisposition disposition = descriptor.disposition();
    CoachingStyle style = CoachingStyle.resolve(disposition, tier);

    StringBuilder sb = new StringBuilder();
    sb.append("You are a StarCraft II coach providing real-time advice to a human player.\n\n");

    switch (style) {
        case COMMANDER -> {
            sb.append("Style: Give direct, actionable instructions. Use imperative voice. Be urgent.\n");
            sb.append("Example: \"Build 3 Stalkers now and move your army to their natural.\"\n\n");
        }
        case RALLY -> {
            sb.append("Style: Ask urgent guiding questions to help the player act quickly.\n");
            sb.append("Example: \"Your base is under pressure — what could you warp in to defend?\"\n\n");
        }
        case INSTRUCTOR -> {
            sb.append("Style: Give clear, calm instructions. Be direct but measured.\n");
            sb.append("Example: \"You should start +1 weapons from the Forge.\"\n\n");
        }
        case MENTOR -> {
            sb.append("Style: Ask guiding questions to help the player discover the right action.\n");
            sb.append("Example: \"What tech path do you think would counter those Roaches?\"\n\n");
        }
    }

    // Behavioural disposition and rest of prompt unchanged
```

- [ ] **Step 6: Update executeCoaching — pass tier instead of boolean**

```java
CoachingUrgencyTier tier = resolveUrgencyTier(input);
SystemMessage systemMessage = new SystemMessage(buildSystemPrompt(descriptor, tier));
```

- [ ] **Step 7: Update directive agent descriptor — socialOrient to INDEPENDENT**

In `QuarkMindAgentRegistrar.buildCoachDirective()`, change:
```java
.socialOrient(ConscientiousnessTerm.INDEPENDENT.value())
```

- [ ] **Step 8: Update CoachingWorkerFactoryTest — new signature + quadrant tests**

Replace `buildSystemPrompt(descriptor, false)` → `buildSystemPrompt(descriptor, CoachingUrgencyTier.ECONOMIC)`.
Replace `buildSystemPrompt(descriptor, true)` → `buildSystemPrompt(descriptor, CoachingUrgencyTier.CRISIS)`.

Update `directiveDescriptor()` socialOrient to `INDEPENDENT`.

Add quadrant differentiation tests:
```java
@Test
void buildSystemPrompt_socratic_strategic_usesRallyStyle() {
    var prompt = CoachingWorkerFactory.buildSystemPrompt(socraticDescriptor(), CoachingUrgencyTier.STRATEGIC);
    assertThat(prompt).contains("urgent guiding questions");
}

@Test
void buildSystemPrompt_socratic_economic_usesMentorStyle() {
    var prompt = CoachingWorkerFactory.buildSystemPrompt(socraticDescriptor(), CoachingUrgencyTier.ECONOMIC);
    assertThat(prompt).contains("discover the right action");
}

@Test
void buildSystemPrompt_directive_economic_usesCommanderStyle() {
    var prompt = CoachingWorkerFactory.buildSystemPrompt(directiveDescriptor(), CoachingUrgencyTier.ECONOMIC);
    assertThat(prompt).contains("direct, actionable instructions");
}
```

- [ ] **Step 9: Run all tests**

Run: `mvn test -q`
Expected: All tests pass.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#247,#246): 2-axis disposition model with urgency-aware prompt switching

Add CoachingStyle enum — 4 derived quadrants (Commander, Rally, Instructor,
Mentor) resolved from RISK_APPETITE × SOCIAL_ORIENTATION × urgency tier.
Refactor buildSystemPrompt to accept CoachingUrgencyTier instead of boolean.
CRISIS forces Commander, STRATEGIC forces bold, ECONOMIC uses defaults.
Update directive agent to use INDEPENDENT socialOrient. Refs #251"
```

---

### Task 3: Broker Supersession + resolveHuman()

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingChannelBroker.java`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluator.java`
- Test: `src/test/java/io/quarkmind/plugin/coaching/CoachingChannelBrokerTest.java`
- Test: `src/test/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluatorTest.java`

**Interfaces:**
- Consumes: `OpenCommitment(correlationId, agentId, advice, issuedAtFrame)` from Task 1
- Consumes: `CoachingComplianceResolved(gameFrame, domain, status, correlationId)` from Task 1
- Produces: `CoachingComplianceEvaluator.resolveHuman(String correlationId, boolean accepted)` → `boolean`

- [ ] **Step 1: Write failing test — resolveHuman DONE**

```java
@Test
void resolveHuman_done_endorsedAndRemoved() {
    var commitments = new ConcurrentHashMap<CoachingDomain, OpenCommitment>();
    var recorder = new TestTrustRecorder();
    var evaluator = new CoachingComplianceEvaluator(commitments, recorder, new LocationResolver());

    var advice = new CoachingAdvice("build stalkers", CoachingDomain.MILITARY,
        new CountDelta(UnitType.STALKER, null, 3, 0), 450);
    commitments.put(CoachingDomain.MILITARY, new OpenCommitment("corr-1", "worker-1", advice, 100));

    boolean result = evaluator.resolveHuman("corr-1", true);

    assertThat(result).isTrue();
    assertThat(commitments).isEmpty();
    assertThat(recorder.lastOutcome).isEqualTo("ENDORSED");
    assertThat(recorder.lastAgentId).isEqualTo("worker-1");
}
```

- [ ] **Step 2: Write failing tests — resolveHuman DECLINE and unknown correlationId**

```java
@Test
void resolveHuman_decline_challengedAndRemoved() {
    var commitments = new ConcurrentHashMap<CoachingDomain, OpenCommitment>();
    var recorder = new TestTrustRecorder();
    var evaluator = new CoachingComplianceEvaluator(commitments, recorder, new LocationResolver());

    var advice = new CoachingAdvice("build stalkers", CoachingDomain.MILITARY,
        new CountDelta(UnitType.STALKER, null, 3, 0), 450);
    commitments.put(CoachingDomain.MILITARY, new OpenCommitment("corr-1", "worker-1", advice, 100));

    assertThat(evaluator.resolveHuman("corr-1", false)).isTrue();
    assertThat(recorder.lastOutcome).isEqualTo("CHALLENGED");
}

@Test
void resolveHuman_unknownCorrelationId_returnsFalse() {
    var commitments = new ConcurrentHashMap<CoachingDomain, OpenCommitment>();
    var recorder = new TestTrustRecorder();
    var evaluator = new CoachingComplianceEvaluator(commitments, recorder, new LocationResolver());

    assertThat(evaluator.resolveHuman("nonexistent", true)).isFalse();
    assertThat(recorder.lastOutcome).isNull();
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -Dtest=CoachingComplianceEvaluatorTest -q`
Expected: 3 new tests FAIL — `resolveHuman` does not exist.

- [ ] **Step 4: Implement resolveHuman()**

```java
public boolean resolveHuman(String correlationId, boolean accepted) {
    var iterator = commitments.entrySet().iterator();
    while (iterator.hasNext()) {
        var entry = iterator.next();
        var commitment = entry.getValue();
        if (correlationId.equals(commitment.correlationId())) {
            String status = accepted ? "ENDORSED" : "CHALLENGED";
            recorder.record(commitment.correlationId(), commitment.agentId(), status, commitment.advice());
            fireComplianceResolved(commitment.issuedAtFrame(), entry.getKey(), status, commitment.correlationId());
            iterator.remove();
            return true;
        }
    }
    return false;
}
```

- [ ] **Step 5: Run tests — verify all pass**

Run: `mvn test -Dtest=CoachingComplianceEvaluatorTest -q`
Expected: All tests PASS.

- [ ] **Step 6: Implement supersession broadcast in CoachingChannelBroker**

In `onCoachingCompleted()`, before `commitments.put()`, fire SUPERSEDED for old commitment:
```java
OpenCommitment old = commitments.get(domain);
if (old != null && complianceResolvedEvent != null) {
    complianceResolvedEvent.fire(new CoachingComplianceResolved(
        old.issuedAtFrame(), domain, "SUPERSEDED", old.correlationId()));
}
```

Add CDI event injection if not present:
```java
@Inject
jakarta.enterprise.event.Event<CoachingComplianceResolved> complianceResolvedEvent;
```

- [ ] **Step 7: Write test — supersession fires SUPERSEDED**

The unit test `CoachingChannelBrokerTest` doesn't have CDI, so it can't capture CDI events. Verify supersession indirectly: after supersession, the old correlationId is no longer in commitments.

```java
@Test
void supersession_oldCommitmentRemoved() {
    var advice1 = new CoachingAdvice("build stalkers", CoachingDomain.MILITARY,
                                     new CountDelta(UnitType.STALKER, null, 3, 0), 450);
    broker.onCoachingCompleted(new CoachingCompleted("w1", "coaching", 100,
                                                     advice1, CoachingUrgencyTier.STRATEGIC, 500, null));
    String oldCorrId = broker.commitments().get(CoachingDomain.MILITARY).correlationId();

    var advice2 = new CoachingAdvice("build zealots", CoachingDomain.MILITARY,
                                     new CountDelta(UnitType.ZEALOT, null, 4, 0), 450);
    broker.onCoachingCompleted(new CoachingCompleted("w1", "coaching", 200,
                                                     advice2, CoachingUrgencyTier.STRATEGIC, 500, null));

    assertThat(broker.commitments().get(CoachingDomain.MILITARY).correlationId()).isNotEqualTo(oldCorrId);
}
```

- [ ] **Step 8: Run all tests**

Run: `mvn test -q`
Expected: All tests pass.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#248): resolveHuman() and broker supersession broadcast

Add CoachingComplianceEvaluator.resolveHuman() — human DONE records ENDORSED,
DECLINE records CHALLENGED, removes commitment. Broker fires SUPERSEDED
CoachingComplianceResolved when a new commitment replaces an existing one
for the same domain. Refs #251"
```

---

### Task 4: Acknowledgment Handler + WebSocket Bidirectional (#248 Backend)

**Files:**
- Create: `src/main/java/io/quarkmind/qa/workbench/CoachingAcknowledgmentHandler.java`
- Modify: `src/main/java/io/quarkmind/qa/workbench/WorkbenchSocket.java`
- Modify: `src/main/java/io/quarkmind/qa/workbench/WorkbenchBroadcaster.java`
- Create: `src/test/java/io/quarkmind/qa/workbench/CoachingAcknowledgmentHandlerTest.java`

**Interfaces:**
- Consumes: `CoachingComplianceEvaluator.resolveHuman(String, boolean)` → `boolean` from Task 3
- Consumes: `CoachingChannelBroker.channelId()` → `UUID`
- Consumes: `MessageService.dispatch(MessageDispatch)` from casehub-qhorus
- Produces: `CoachingAcknowledgmentHandler.acknowledge(String correlationId, boolean accepted)` → `boolean`

- [ ] **Step 1: Write failing test — acknowledge resolves and returns true**

```java
package io.quarkmind.qa.workbench;

import io.quarkmind.domain.UnitType;
import io.quarkmind.plugin.coaching.*;
import org.junit.jupiter.api.Test;
import java.util.concurrent.ConcurrentHashMap;
import static org.assertj.core.api.Assertions.assertThat;

class CoachingAcknowledgmentHandlerTest {

    @Test
    void acknowledge_done_resolvesAndReturnsTrue() {
        var commitments = new ConcurrentHashMap<CoachingDomain, OpenCommitment>();
        var recorder = new CoachingComplianceEvaluatorTest.TestTrustRecorder();
        var evaluator = new CoachingComplianceEvaluator(commitments, recorder, new LocationResolver());

        var advice = new CoachingAdvice("build stalkers", CoachingDomain.MILITARY,
            new CountDelta(UnitType.STALKER, null, 3, 0), 450);
        commitments.put(CoachingDomain.MILITARY, new OpenCommitment("corr-1", "worker-1", advice, 100));

        var handler = new CoachingAcknowledgmentHandler(evaluator, null, null);

        assertThat(handler.acknowledge("corr-1", true)).isTrue();
        assertThat(commitments).isEmpty();
        assertThat(recorder.lastOutcome).isEqualTo("ENDORSED");
    }

    @Test
    void acknowledge_decline_challengedAndReturnsTrue() {
        var commitments = new ConcurrentHashMap<CoachingDomain, OpenCommitment>();
        var recorder = new CoachingComplianceEvaluatorTest.TestTrustRecorder();
        var evaluator = new CoachingComplianceEvaluator(commitments, recorder, new LocationResolver());

        var advice = new CoachingAdvice("build stalkers", CoachingDomain.MILITARY,
            new CountDelta(UnitType.STALKER, null, 3, 0), 450);
        commitments.put(CoachingDomain.MILITARY, new OpenCommitment("corr-1", "worker-1", advice, 100));

        var handler = new CoachingAcknowledgmentHandler(evaluator, null, null);

        assertThat(handler.acknowledge("corr-1", false)).isTrue();
        assertThat(recorder.lastOutcome).isEqualTo("CHALLENGED");
    }

    @Test
    void acknowledge_unknownCorrelationId_returnsFalseNoDispatch() {
        var commitments = new ConcurrentHashMap<CoachingDomain, OpenCommitment>();
        var recorder = new CoachingComplianceEvaluatorTest.TestTrustRecorder();
        var evaluator = new CoachingComplianceEvaluator(commitments, recorder, new LocationResolver());

        var handler = new CoachingAcknowledgmentHandler(evaluator, null, null);

        assertThat(handler.acknowledge("nonexistent", true)).isFalse();
        assertThat(recorder.lastOutcome).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=CoachingAcknowledgmentHandlerTest -q`
Expected: FAIL — class does not exist.

- [ ] **Step 3: Implement CoachingAcknowledgmentHandler**

```java
package io.quarkmind.qa.workbench;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.message.MessageService;
import io.quarkmind.plugin.coaching.CoachingChannelBroker;
import io.quarkmind.plugin.coaching.CoachingComplianceEvaluator;
import io.quarkus.arc.profile.UnlessBuildProfile;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@UnlessBuildProfile("prod")
@ApplicationScoped
public class CoachingAcknowledgmentHandler {

    private static final Logger log = Logger.getLogger(CoachingAcknowledgmentHandler.class);

    private final CoachingComplianceEvaluator evaluator;
    private final CoachingChannelBroker broker;
    private final MessageService messageService;

    @Inject
    CoachingAcknowledgmentHandler(CoachingComplianceEvaluator evaluator,
                                   CoachingChannelBroker broker,
                                   MessageService messageService) {
        this.evaluator = evaluator;
        this.broker = broker;
        this.messageService = messageService;
    }

    public boolean acknowledge(String correlationId, boolean accepted) {
        boolean resolved = evaluator.resolveHuman(correlationId, accepted);
        if (!resolved) {
            log.debugf("[COACHING-ACK] correlationId=%s not found", correlationId);
            return false;
        }

        if (messageService != null && broker != null && broker.channelId() != null) {
            try {
                messageService.dispatch(
                    MessageDispatch.builder()
                        .channelId(broker.channelId())
                        .sender("coaching.acknowledgment-handler")
                        .type(accepted ? MessageType.DONE : MessageType.DECLINE)
                        .content(accepted ? "Human accepted coaching advice" : "Human declined coaching advice")
                        .correlationId(correlationId)
                        .inReplyTo(correlationId)
                        .actorType(ActorType.HUMAN)
                        .build());
            } catch (Exception e) {
                log.warnf(e, "[COACHING-ACK] MessageService dispatch failed: %s", e.getMessage());
            }
        }

        return true;
    }
}
```

- [ ] **Step 4: Run tests — verify all 3 pass**

Run: `mvn test -Dtest=CoachingAcknowledgmentHandlerTest -q`
Expected: All 3 tests PASS.

- [ ] **Step 5: Add @OnTextMessage to WorkbenchSocket**

```java
@Inject CoachingAcknowledgmentHandler handler;

@OnTextMessage
public void onMessage(String message, WebSocketConnection connection) {
    try {
        var node = new com.fasterxml.jackson.databind.ObjectMapper().readTree(message);
        String type = node.path("type").asText(null);
        if (!"coaching_response".equals(type)) return;

        String correlationId = node.path("correlationId").asText(null);
        String response = node.path("response").asText(null);
        if (correlationId == null || correlationId.isBlank()) return;
        if (!"DONE".equals(response) && !"DECLINE".equals(response)) return;

        handler.acknowledge(correlationId, "DONE".equals(response));
    } catch (Exception e) {
        // malformed message — ignore
    }
}
```

- [ ] **Step 6: Update WorkbenchBroadcaster — compliance snapshot for reconnection**

```java
private final java.util.concurrent.ConcurrentHashMap<String, WorkbenchEvent> complianceSnapshots = new java.util.concurrent.ConcurrentHashMap<>();

// In updateSnapshot(), add:
case "coaching_compliance" -> {
    if (event.payload() instanceof CoachingCompliancePayload p) {
        complianceSnapshots.put(p.correlationId(), event);
    }
}

// In pushSnapshot(), after coaching snapshot:
complianceSnapshots.values().forEach(e -> {
    try { sendOne(connection, e); } catch (Exception ex) { /* ignore */ }
});

// In onGameStarted (add observer or method):
complianceSnapshots.clear();
```

- [ ] **Step 7: Run all tests**

Run: `mvn test -q`
Expected: All tests pass.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#248): coaching acknowledgment handler with WebSocket bidirectional

Add CoachingAcknowledgmentHandler — resolves commitment first, then dispatches
DONE/DECLINE to qhorus MessageService. Add @OnTextMessage to WorkbenchSocket.
Add compliance snapshot to WorkbenchBroadcaster for reconnection consistency.
Refs #251"
```

---

### Task 5: Visualizer.js Coaching Acknowledgment UI (#248 Frontend)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/workbench/WorkbenchRenderTest.java` (Playwright)

**Interfaces:**
- Consumes: WebSocket `coaching` event with `correlationId` in payload
- Consumes: WebSocket `coaching_compliance` event with `correlationId` in payload
- Produces: WebSocket outbound `{type: "coaching_response", correlationId: "...", response: "DONE"|"DECLINE"}`

- [ ] **Step 1: Update applyComplianceUpdate — match by correlationId**

```javascript
function applyComplianceUpdate(payload) {
    var label = payload.status === 'ENDORSED' ? '✅ Endorsed' :
                payload.status === 'CHALLENGED' ? '❌ Challenged' :
                payload.status === 'SUPERSEDED' ? '⏭ Superseded' : '⏸ Neutral';
    var entry = workbenchState.coaching.find(function(c) {
        return c.correlationId === payload.correlationId;
    });
    if (entry) { entry.complianceStatus = label; renderCoachingPage(); }
}
```

- [ ] **Step 2: Update renderCoachingPage — add Accept/Dismiss buttons**

```javascript
function renderCoachingPage() {
    var el = document.getElementById('page-coaching');
    if (workbenchState.coaching.length === 0) { el.innerHTML = 'No coaching advice yet'; return; }

    el.innerHTML = workbenchState.coaching.map(function(c) {
        var time = Math.floor(c.gameFrame / 22.4) + 's';
        var status = c.complianceStatus || '⏳ Pending';
        var isPending = !c.complianceStatus;
        var buttons = isPending
            ? '<button class="coaching-btn coaching-accept" data-cid="' + c.correlationId + '">✓ Accept</button>' +
              '<button class="coaching-btn coaching-dismiss" data-cid="' + c.correlationId + '">✗ Dismiss</button>'
            : '';
        return '<div class="coaching-item">' +
            '<div class="coaching-header">' + time + ' [' + c.domain + '] ' + (c.urgency || '') + '</div>' +
            '<div class="coaching-advice">' + c.advice + '</div>' +
            '<div class="coaching-controls">' + buttons + '<span class="coaching-status">' + status + '</span></div>' +
            '</div>';
    }).join('');

    el.querySelectorAll('.coaching-accept').forEach(function(btn) {
        btn.addEventListener('click', function() { sendCoachingResponse(btn.dataset.cid, 'DONE'); });
    });
    el.querySelectorAll('.coaching-dismiss').forEach(function(btn) {
        btn.addEventListener('click', function() { sendCoachingResponse(btn.dataset.cid, 'DECLINE'); });
    });
}
```

- [ ] **Step 3: Add sendCoachingResponse function and store WebSocket reference**

```javascript
function sendCoachingResponse(correlationId, response) {
    if (window.__workbenchWs && window.__workbenchWs.readyState === 1) {
        window.__workbenchWs.send(JSON.stringify({
            type: 'coaching_response',
            correlationId: correlationId,
            response: response
        }));
    }
}
```

Store the WebSocket reference where the connection is created:
```javascript
window.__workbenchWs = ws;
```

- [ ] **Step 4: Add CSS for coaching buttons**

```css
.coaching-btn { cursor: pointer; padding: 2px 8px; margin-right: 4px; border: 1px solid #555; background: #222; color: #ccc; border-radius: 3px; font-size: 12px; }
.coaching-accept:hover { background: #1a3a1a; border-color: #4a4; }
.coaching-dismiss:hover { background: #3a1a1a; border-color: #a44; }
.coaching-controls { display: flex; align-items: center; gap: 4px; margin-top: 4px; }
```

- [ ] **Step 5: Run existing Playwright tests — verify no regression**

Run: `mvn test -Pplaywright -q`
Expected: All existing render tests pass.

- [ ] **Step 6: Add Playwright test — coaching buttons render and respond**

In `WorkbenchRenderTest`, add a test that:
1. Fires a `CoachingCompleted` event through the engine to produce coaching advice
2. Navigates to the coaching tab
3. Asserts `coaching-accept` and `coaching-dismiss` buttons are visible
4. Clicks the Accept button
5. Asserts buttons disappear and the status shows "✅ Endorsed"

- [ ] **Step 7: Run all tests including Playwright**

Run: `mvn test -Pplaywright -q`
Expected: All tests pass.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#248): coaching acknowledgment UI — Accept/Dismiss buttons in workbench

Add Accept/Dismiss buttons per coaching advice item in visualizer.js.
Clicking sends coaching_response via WebSocket with correlationId.
Match compliance updates by correlationId instead of gameFrame+domain.
Superseded advice shows ⏭ badge. Closes #248, Refs #251"
```

---

## Self-Review Checklist

**Spec coverage:**
- ✅ OpenCommitment agentId (Task 1)
- ✅ CoachingComplianceResolved correlationId (Task 1)
- ✅ CoachingAdvicePublished correlationId + explicit pass-through chain (Task 1)
- ✅ CoachingPayload/CoachingCompliancePayload correlationId (Task 1)
- ✅ CoachingEffectivenessTrustRecorder agentId param (Task 1)
- ✅ WorkbenchEnricher correlationId pass-through (Task 1)
- ✅ withdrawAll() agentId pass (Task 1)
- ✅ CoachingStyle enum with 4 quadrants (Task 2)
- ✅ buildSystemPrompt tier-based with 4 templates (Task 2)
- ✅ Directive agent INDEPENDENT socialOrient (Task 2)
- ✅ resolveHuman() with issuedAtFrame for compliance event (Task 3)
- ✅ Broker supersession SUPERSEDED broadcast (Task 3)
- ✅ CoachingAcknowledgmentHandler resolve-before-dispatch (Task 4)
- ✅ WorkbenchSocket @OnTextMessage with validation (Task 4)
- ✅ Reconnection compliance snapshot (Task 4)
- ✅ Visualizer.js Accept/Dismiss buttons (Task 5)
- ✅ correlationId-based UI matching (Task 5)
- ✅ Supersession ⏭ badge (Task 5)
- ✅ CoachingDispositionTerm unchanged — 2 base agents, 4 runtime styles (documented in spec, no code change needed)

**Placeholder scan:** None found.
**Type consistency:** Verified — record signatures, method names, and vocabulary term values match across all tasks.
**Tooling safety:** No bash file operations on source files.
