# Coaching Personality Model & Acknowledgment UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #251 — epic: coaching personality model — from two fixed to adaptive multi-axis
**Issue group:** #251, #246, #247, #248

**Goal:** Expand coaching from two fixed personalities to a 2-axis disposition model with urgency-driven switching, and add human acknowledgment buttons to the workbench UI.

**Architecture:** Bottom-up build: data model enrichment → CoachingStyle enum with 4-quadrant prompt templates → urgency-aware prompt switching → acknowledgment UI with WebSocket bidirectional messaging and human-wins compliance override.

**Tech Stack:** Java 21 (records, sealed interfaces), Quarkus CDI, casehub-qhorus (MessageService), casehub-eidos (DispositionAxis, AgentDescriptor), LangChain4j, Quarkus WebSocket Next, vanilla JS (visualizer.js)

## Global Constraints

- Domain model (`domain/`) must remain plain Java — no CDI, no Quarkus imports
- QA endpoints/workbench carry `@UnlessBuildProfile("prod")`
- Never `@QuarkusTest` for tests that can be plain JUnit
- All commits reference an issue
- Use IntelliJ MCP for code navigation and structural editing

---

### Task 1: Data model enrichment — agentId attribution chain

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/coaching/OpenCommitment.java`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingEffectivenessTrustRecorder.java`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluator.java`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingChannelBroker.java`
- Test: `src/test/java/io/quarkmind/plugin/coaching/CoachingChannelBrokerTest.java`
- Test: `src/test/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluatorTest.java`

**Interfaces:**
- Consumes: existing `CoachingCompleted.workerId()`, existing `OpenCommitment`, existing `CoachingEffectivenessTrustRecorder`
- Produces: `OpenCommitment(String correlationId, String agentId, CoachingAdvice advice, long issuedAtFrame)`, `CoachingEffectivenessTrustRecorder.record(String correlationId, String agentId, String outcome, CoachingAdvice advice)`

- [ ] **Step 1: Write failing test — OpenCommitment carries agentId**

In `CoachingChannelBrokerTest.java`, add a test that verifies the broker passes `workerId` from `CoachingCompleted` into the `OpenCommitment`:

```java
@Test
void onCoachingCompleted_storesAgentIdInCommitment() {
    var advice = new CoachingAdvice("Build Stalkers", CoachingDomain.BUILD, null, 450);
    var event = new CoachingCompleted("coach-directive", "coaching", 100L, advice,
                                      CoachingUrgencyTier.STRATEGIC, 50L, null);
    broker.onCoachingCompleted(event);

    var commitment = broker.commitments().get(CoachingDomain.BUILD);
    assertNotNull(commitment);
    assertEquals("coach-directive", commitment.agentId());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=CoachingChannelBrokerTest#onCoachingCompleted_storesAgentIdInCommitment -q`
Expected: FAIL — `OpenCommitment` has no `agentId()` method

- [ ] **Step 3: Add agentId to OpenCommitment**

Replace `OpenCommitment.java`:

```java
package io.quarkmind.plugin.coaching;

public record OpenCommitment(
        String correlationId,
        String agentId,
        CoachingAdvice advice,
        long issuedAtFrame
) {}
```

- [ ] **Step 4: Update CoachingChannelBroker to pass agentId**

In `CoachingChannelBroker.onCoachingCompleted()`, change the `OpenCommitment` constructor call:

```java
commitments.put(domain, new OpenCommitment(correlationId, event.workerId(), adviceWithBaseline, gameFrame));
```

- [ ] **Step 5: Update CoachingEffectivenessTrustRecorder signature**

```java
public void record(String correlationId, String agentId, String outcome, CoachingAdvice advice) {
    log.infof("[COACHING-TRUST] correlationId=%s agentId=%s outcome=%s domain=%s advice=%s",
        correlationId, agentId, outcome, advice.domainTag(), advice.advice());
}
```

- [ ] **Step 6: Update CoachingComplianceEvaluator — evaluate() and withdrawAll()**

In `evaluate()`, change all three `recorder.record()` calls to include `commitment.agentId()`:

```java
recorder.record(commitment.correlationId(), commitment.agentId(), "NEUTRAL", advice);
// ... and similarly for "ENDORSED" and "CHALLENGED"
```

In `withdrawAll()`:

```java
public void withdrawAll() {
    commitments.forEach((domain, commitment) ->
        recorder.record(commitment.correlationId(), commitment.agentId(), "NEUTRAL", commitment.advice()));
    commitments.clear();
}
```

- [ ] **Step 7: Fix all compilation — update test constructors**

Find all test files that construct `OpenCommitment` or call `recorder.record()` and update signatures. Use `ide_find_references` for `OpenCommitment` constructor and `CoachingEffectivenessTrustRecorder.record()`.

- [ ] **Step 8: Run tests to verify everything passes**

Run: `mvn test -Dtest=CoachingChannelBrokerTest,CoachingComplianceEvaluatorTest -q`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#251): add agentId to OpenCommitment for personality attribution

Refs #251"
```

---

### Task 2: Data model enrichment — correlationId pipeline + supersession

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingComplianceResolved.java`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingAdvicePublished.java`
- Modify: `src/main/java/io/quarkmind/qa/workbench/CoachingPayload.java`
- Modify: `src/main/java/io/quarkmind/qa/workbench/CoachingCompliancePayload.java`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingChannelBroker.java`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluator.java`
- Modify: `src/main/java/io/quarkmind/qa/workbench/WorkbenchEnricher.java`
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Test: `src/test/java/io/quarkmind/plugin/coaching/CoachingChannelBrokerTest.java`
- Test: `src/test/java/io/quarkmind/qa/workbench/WorkbenchEventTest.java`

**Interfaces:**
- Consumes: `OpenCommitment` (from Task 1)
- Produces: `CoachingAdvicePublished(..., String correlationId)`, `CoachingComplianceResolved(..., String correlationId)`, `CoachingPayload(..., String correlationId)`, `CoachingCompliancePayload(..., String correlationId)`

- [ ] **Step 1: Write failing test — supersession broadcasts SUPERSEDED**

```java
@Test
void onCoachingCompleted_supersededCommitment_broadcastsSuperseded() {
    var advice1 = new CoachingAdvice("Build Stalkers", CoachingDomain.BUILD, null, 450);
    var event1 = new CoachingCompleted("coach-directive", "coaching", 100L, advice1,
                                       CoachingUrgencyTier.STRATEGIC, 50L, null);
    broker.onCoachingCompleted(event1);
    String oldCorrelationId = broker.commitments().get(CoachingDomain.BUILD).correlationId();

    var advice2 = new CoachingAdvice("Build Zealots", CoachingDomain.BUILD, null, 450);
    var event2 = new CoachingCompleted("coach-directive", "coaching", 200L, advice2,
                                       CoachingUrgencyTier.ECONOMIC, 40L, null);
    broker.onCoachingCompleted(event2);

    // Verify old commitment was superseded — compliance event fired with SUPERSEDED status
    // (Verify via CDI event mock or captured events)
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=CoachingChannelBrokerTest#onCoachingCompleted_supersededCommitment_broadcastsSuperseded -q`
Expected: FAIL

- [ ] **Step 3: Add correlationId to records**

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

- [ ] **Step 4: Update CoachingComplianceEvaluator.fireComplianceResolved()**

```java
private void fireComplianceResolved(long frame, CoachingDomain domain, String status, String correlationId) {
    if (complianceResolvedEvent != null) {
        complianceResolvedEvent.fire(new CoachingComplianceResolved(frame, domain, status, correlationId));
    }
}
```

Update all call sites in `evaluate()` to pass `commitment.correlationId()`.

- [ ] **Step 5: Update CoachingChannelBroker — supersession + correlationId in AdvicePublished**

In `onCoachingCompleted()`, before `commitments.put(domain, ...)`:

```java
OpenCommitment oldCommitment = commitments.get(domain);
commitments.put(domain, new OpenCommitment(correlationId, event.workerId(), adviceWithBaseline, gameFrame));

if (oldCommitment != null) {
    // Fire supersession event for old commitment so UI hides its buttons
    if (complianceResolvedEvent != null) {
        complianceResolvedEvent.fire(new CoachingComplianceResolved(
            oldCommitment.issuedAtFrame(), domain, "SUPERSEDED", oldCommitment.correlationId()));
    }
}
```

Note: inject `Event<CoachingComplianceResolved>` into `CoachingChannelBroker` (currently only in the evaluator).

Update the `CoachingAdvicePublished` fire to include correlationId:

```java
coachingAdviceEvent.fire(new CoachingAdvicePublished(event.advice(), event.urgencyTier(), gameFrame, correlationId));
```

- [ ] **Step 6: Update WorkbenchEnricher — pass correlationId through**

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

- [ ] **Step 7: Update visualizer.js — match by correlationId**

In `renderCoachingPage()`, add `data-correlation-id` attribute and store correlationId:

```javascript
return '<div class="coaching-item">' +
  '<div class="coaching-header">' + time + ' [' + c.domain + '] ' + (c.urgency || '') + '</div>' +
  '<div class="coaching-advice">' + c.advice + '</div>' +
  '<div class="coaching-status" data-correlation-id="' + c.correlationId + '">' + status + '</div>' +
'</div>';
```

In `applyComplianceUpdate()`, match by correlationId:

```javascript
function applyComplianceUpdate(payload) {
  var statusMap = { ENDORSED: '✓ Complied', CHALLENGED: '✗ Ignored', NEUTRAL: '— Neutral', SUPERSEDED: '⤬ Superseded' };
  var label = statusMap[payload.status] || payload.status;
  var entry = workbenchState.coaching.find(function(c) { return c.correlationId === payload.correlationId; });
  if (entry) { entry.complianceStatus = label; renderCoachingPage(); }
}
```

- [ ] **Step 8: Fix all compilation — update constructor call sites**

Use `ide_find_references` for each modified record to find all test and production call sites. Update constructor arguments.

- [ ] **Step 9: Run tests**

Run: `mvn test -Dtest=CoachingChannelBrokerTest,CoachingComplianceEvaluatorTest,WorkbenchEventTest -q`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#251): add correlationId pipeline and supersession events

Refs #251, #248"
```

---

### Task 3: CoachingStyle enum + urgency-aware prompt switching (#247, #246)

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/coaching/CoachingStyle.java`
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingWorkerFactory.java`
- Test: `src/test/java/io/quarkmind/plugin/coaching/CoachingStyleTest.java`
- Test: `src/test/java/io/quarkmind/plugin/coaching/CoachingWorkerFactoryTest.java`

**Interfaces:**
- Consumes: `CoachingUrgencyTier`, `AgentDisposition`, `DispositionAxis`
- Produces: `CoachingStyle.resolve(AgentDisposition, CoachingUrgencyTier)`, `CoachingWorkerFactory.buildSystemPrompt(AgentDescriptor, CoachingUrgencyTier)`

- [ ] **Step 1: Write failing tests — CoachingStyle.resolve() all 12 combinations**

Create `CoachingStyleTest.java`:

```java
package io.quarkmind.plugin.coaching;

import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.api.DispositionAxis;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

import static org.junit.jupiter.api.Assertions.*;

class CoachingStyleTest {

    @ParameterizedTest
    @CsvSource({
        "bold, assertive, CRISIS, COMMANDER",
        "bold, assertive, STRATEGIC, COMMANDER",
        "bold, assertive, ECONOMIC, COMMANDER",
        "bold, collaborative, CRISIS, COMMANDER",
        "bold, collaborative, STRATEGIC, RALLY",
        "bold, collaborative, ECONOMIC, RALLY",
        "cautious, assertive, CRISIS, COMMANDER",
        "cautious, assertive, STRATEGIC, INSTRUCTOR",
        "cautious, assertive, ECONOMIC, INSTRUCTOR",
        "cautious, collaborative, CRISIS, COMMANDER",
        "cautious, collaborative, STRATEGIC, RALLY",
        "cautious, collaborative, ECONOMIC, MENTOR"
    })
    void resolve_allCombinations(String risk, String social, String tierName, String expected) {
        var disposition = AgentDisposition.builder()
            .withPrimaryTerm(DispositionAxis.RISK_APPETITE, risk)
            .withPrimaryTerm(DispositionAxis.SOCIAL_ORIENTATION, social)
            .build();
        var tier = CoachingUrgencyTier.valueOf(tierName);
        assertEquals(CoachingStyle.valueOf(expected), CoachingStyle.resolve(disposition, tier));
    }

    @Test
    void resolve_nullDisposition_defaultsToMentor() {
        assertEquals(CoachingStyle.MENTOR, CoachingStyle.resolve(null, CoachingUrgencyTier.ECONOMIC));
    }

    @Test
    void resolve_nullDisposition_crisisStillCommander() {
        assertEquals(CoachingStyle.COMMANDER, CoachingStyle.resolve(null, CoachingUrgencyTier.CRISIS));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=CoachingStyleTest -q`
Expected: FAIL — `CoachingStyle` class does not exist

- [ ] **Step 3: Create CoachingStyle enum**

Create `src/main/java/io/quarkmind/plugin/coaching/CoachingStyle.java`:

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
                    || (disposition != null
                        && "bold".equals(disposition.primaryTerm(DispositionAxis.RISK_APPETITE)));
        boolean assertive = tier == CoachingUrgencyTier.CRISIS
                         || (disposition != null
                             && "assertive".equals(disposition.primaryTerm(DispositionAxis.SOCIAL_ORIENTATION)));

        if (bold && assertive) return COMMANDER;
        if (bold)              return RALLY;
        if (assertive)         return INSTRUCTOR;
        return MENTOR;
    }
}
```

- [ ] **Step 4: Run CoachingStyleTest — verify passes**

Run: `mvn test -Dtest=CoachingStyleTest -q`
Expected: ALL PASS

- [ ] **Step 5: Write failing test — buildSystemPrompt with tier**

Update `CoachingWorkerFactoryTest.java` — replace existing `buildSystemPrompt` boolean tests with tier-based tests:

```java
@Test
void buildSystemPrompt_crisisTier_commanderStyle() {
    var descriptor = buildDescriptor("cautious", "collaborative");
    String prompt = CoachingWorkerFactory.buildSystemPrompt(descriptor, CoachingUrgencyTier.CRISIS);
    assertTrue(prompt.contains("direct, actionable instructions"));
    assertTrue(prompt.contains("imperative voice"));
}

@Test
void buildSystemPrompt_economicTier_mentorStyle() {
    var descriptor = buildDescriptor("cautious", "collaborative");
    String prompt = CoachingWorkerFactory.buildSystemPrompt(descriptor, CoachingUrgencyTier.ECONOMIC);
    assertTrue(prompt.contains("guiding questions"));
    assertTrue(prompt.contains("discover"));
}

@Test
void buildSystemPrompt_strategicTier_rallyStyle() {
    var descriptor = buildDescriptor("cautious", "collaborative");
    String prompt = CoachingWorkerFactory.buildSystemPrompt(descriptor, CoachingUrgencyTier.STRATEGIC);
    assertTrue(prompt.contains("urgent"));
    assertTrue(prompt.contains("guiding"));
}

@Test
void buildSystemPrompt_economicTier_instructorStyle() {
    var descriptor = buildDescriptor("cautious", "assertive");
    String prompt = CoachingWorkerFactory.buildSystemPrompt(descriptor, CoachingUrgencyTier.ECONOMIC);
    assertTrue(prompt.contains("clear, direct"));
    assertTrue(prompt.contains("measured"));
}
```

- [ ] **Step 6: Refactor buildSystemPrompt — accept CoachingUrgencyTier**

Change signature from `buildSystemPrompt(AgentDescriptor descriptor, boolean crisisOverride)` to `buildSystemPrompt(AgentDescriptor descriptor, CoachingUrgencyTier tier)`.

Replace the binary `isDirective` logic with `CoachingStyle.resolve()` and 4 quadrant-specific prompt blocks:

```java
static String buildSystemPrompt(AgentDescriptor descriptor, CoachingUrgencyTier tier) {
    CoachingStyle style = CoachingStyle.resolve(descriptor.disposition(), tier);

    StringBuilder sb = new StringBuilder();
    sb.append("You are a StarCraft II coach providing real-time advice to a human player.\n\n");

    switch (style) {
        case COMMANDER -> {
            sb.append("Style: Give direct, actionable instructions. Use imperative voice.\n");
            sb.append("Example: \"Build 3 Stalkers now and move your army to their natural.\"\n\n");
        }
        case RALLY -> {
            sb.append("Style: Convey urgency while asking guiding questions.\n");
            sb.append("Example: \"Your base is under pressure — what could you warp in to defend?\"\n\n");
        }
        case INSTRUCTOR -> {
            sb.append("Style: Give clear, direct advice in a measured tone.\n");
            sb.append("Example: \"You should start +1 weapons from the Forge.\"\n\n");
        }
        case MENTOR -> {
            sb.append("Style: Ask guiding questions to help the player discover the right action.\n");
            sb.append("Example: \"What tech path do you think would counter those Roaches?\"\n\n");
        }
    }

    // ... rest of prompt unchanged (behavioural disposition, JSON format, verification types)
```

- [ ] **Step 7: Update executeCoaching — pass tier instead of boolean**

In `executeCoaching()`:

```java
CoachingUrgencyTier tier = resolveUrgencyTier(input);
SystemMessage systemMessage = new SystemMessage(buildSystemPrompt(descriptor, tier));
```

Remove the `isCrisisTrigger()` boolean call. The `isCrisisTrigger()` private method can remain (used by tier logic internally) or be removed if `resolveUrgencyTier()` handles all cases.

- [ ] **Step 8: Run all coaching tests**

Run: `mvn test -Dtest=CoachingStyleTest,CoachingWorkerFactoryTest -q`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#247,#246): 2-axis coaching style with urgency-aware prompt switching

CoachingStyle enum derives 4 quadrants (Commander/Rally/Instructor/Mentor)
from RISK_APPETITE × SOCIAL_ORIENTATION axes. CRISIS forces Commander,
STRATEGIC forces bold, ECONOMIC uses agent defaults.

Closes #247, Closes #246"
```

---

### Task 4: resolveHuman() + CoachingAcknowledgmentHandler (#248 backend)

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluator.java`
- Create: `src/main/java/io/quarkmind/qa/workbench/CoachingAcknowledgmentHandler.java`
- Test: `src/test/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluatorTest.java`
- Test: `src/test/java/io/quarkmind/qa/workbench/CoachingAcknowledgmentHandlerTest.java`

**Interfaces:**
- Consumes: `OpenCommitment` (from Task 1), `CoachingComplianceResolved` with correlationId (from Task 2), `MessageService`, `CoachingChannelBroker`
- Produces: `CoachingComplianceEvaluator.resolveHuman(String correlationId, boolean accepted)` → `boolean`, `CoachingAcknowledgmentHandler.acknowledge(String correlationId, boolean accepted)` → `boolean`

- [ ] **Step 1: Write failing test — resolveHuman DONE → ENDORSED**

```java
@Test
void resolveHuman_done_recordsEndorsedAndRemoves() {
    var advice = new CoachingAdvice("Build Stalkers", CoachingDomain.BUILD, null, 450);
    commitments.put(CoachingDomain.BUILD, new OpenCommitment("corr-1", "agent-1", advice, 100L));

    boolean result = evaluator.resolveHuman("corr-1", true);

    assertTrue(result);
    assertTrue(commitments.isEmpty());
    // Verify recorder was called with ("corr-1", "agent-1", "ENDORSED", advice)
}
```

- [ ] **Step 2: Write failing test — resolveHuman DECLINE → CHALLENGED**

```java
@Test
void resolveHuman_decline_recordsChallengedAndRemoves() {
    var advice = new CoachingAdvice("Build Stalkers", CoachingDomain.BUILD, null, 450);
    commitments.put(CoachingDomain.BUILD, new OpenCommitment("corr-1", "agent-1", advice, 100L));

    boolean result = evaluator.resolveHuman("corr-1", false);

    assertTrue(result);
    assertTrue(commitments.isEmpty());
    // Verify recorder was called with ("corr-1", "agent-1", "CHALLENGED", advice)
}
```

- [ ] **Step 3: Write failing test — resolveHuman unknown correlationId**

```java
@Test
void resolveHuman_unknownCorrelationId_returnsFalse() {
    var advice = new CoachingAdvice("Build Stalkers", CoachingDomain.BUILD, null, 450);
    commitments.put(CoachingDomain.BUILD, new OpenCommitment("corr-1", "agent-1", advice, 100L));

    boolean result = evaluator.resolveHuman("unknown-id", true);

    assertFalse(result);
    assertEquals(1, commitments.size());
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `mvn test -Dtest=CoachingComplianceEvaluatorTest -q`
Expected: FAIL — `resolveHuman` method does not exist

- [ ] **Step 5: Implement resolveHuman()**

Add to `CoachingComplianceEvaluator`:

```java
public boolean resolveHuman(String correlationId, boolean accepted) {
    var iterator = commitments.entrySet().iterator();
    while (iterator.hasNext()) {
        var entry = iterator.next();
        var commitment = entry.getValue();
        if (correlationId.equals(commitment.correlationId())) {
            if (!commitments.remove(entry.getKey(), commitment)) {
                return false;
            }
            String status = accepted ? "ENDORSED" : "CHALLENGED";
            recorder.record(commitment.correlationId(), commitment.agentId(), status, commitment.advice());
            fireComplianceResolved(commitment.issuedAtFrame(), entry.getKey(), status, commitment.correlationId());
            return true;
        }
    }
    return false;
}
```

- [ ] **Step 6: Run resolveHuman tests — verify pass**

Run: `mvn test -Dtest=CoachingComplianceEvaluatorTest -q`
Expected: ALL PASS

- [ ] **Step 7: Write failing test — CoachingAcknowledgmentHandler**

Create `CoachingAcknowledgmentHandlerTest.java`:

```java
@Test
void acknowledge_done_dispatchesDoneAndResolves() {
    // Set up: put a commitment in the broker's map
    // Call: handler.acknowledge("corr-1", true)
    // Assert: evaluator.resolveHuman was called
    // Assert: messageService.dispatch was called with MessageType.DONE,
    //         correlationId="corr-1", inReplyTo="corr-1", actorType=HUMAN
    // Assert: broadcaster.broadcast was called with coaching_compliance ENDORSED
}

@Test
void acknowledge_decline_dispatchesDecline() {
    // Similar but MessageType.DECLINE and CHALLENGED status
}

@Test
void acknowledge_unknownCorrelationId_noDispatchNoBroadcast() {
    // evaluator.resolveHuman returns false
    // Assert: messageService.dispatch NOT called
    // Assert: broadcaster.broadcast NOT called
    // Assert: handler returns false
}
```

- [ ] **Step 8: Implement CoachingAcknowledgmentHandler**

Create `src/main/java/io/quarkmind/qa/workbench/CoachingAcknowledgmentHandler.java`:

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

    @Inject CoachingComplianceEvaluator evaluator;
    @Inject CoachingChannelBroker broker;
    @Inject MessageService messageService;
    @Inject WorkbenchBroadcaster broadcaster;

    public boolean acknowledge(String correlationId, boolean accepted) {
        if (!evaluator.resolveHuman(correlationId, accepted)) {
            log.debugf("[COACHING-ACK] correlationId=%s not found — already resolved or superseded", correlationId);
            return false;
        }

        try {
            messageService.dispatch(
                MessageDispatch.builder()
                    .channelId(broker.channelId())
                    .sender("coaching.human-ack")
                    .type(accepted ? MessageType.DONE : MessageType.DECLINE)
                    .content(accepted ? "Human accepted coaching advice" : "Human declined coaching advice")
                    .correlationId(correlationId)
                    .inReplyTo(correlationId)
                    .actorType(ActorType.HUMAN)
                    .build()
            );
        } catch (Exception e) {
            log.warnf(e, "[COACHING-ACK] MessageService dispatch failed for correlationId=%s: %s",
                       correlationId, e.getMessage());
        }

        return true;
    }
}
```

- [ ] **Step 9: Run all tests**

Run: `mvn test -Dtest=CoachingAcknowledgmentHandlerTest,CoachingComplianceEvaluatorTest -q`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#248): add resolveHuman() and CoachingAcknowledgmentHandler

Human DONE → ENDORSED, DECLINE → CHALLENGED, commitment removed.
Resolve-before-dispatch ordering. CAS-based removal prevents race
with auto-compliance evaluator.

Refs #248"
```

---

### Task 5: WorkbenchSocket bidirectional + visualizer.js buttons (#248 UI)

**Files:**
- Modify: `src/main/java/io/quarkmind/qa/workbench/WorkbenchSocket.java`
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Test: `src/test/java/io/quarkmind/qa/workbench/WorkbenchSocketIT.java` (if exists, else update existing WebSocket test)
- Test: `src/test/java/io/quarkmind/qa/workbench/WorkbenchRenderTest.java`

**Interfaces:**
- Consumes: `CoachingAcknowledgmentHandler.acknowledge()` (from Task 4), `CoachingPayload` with correlationId (from Task 2)
- Produces: `WorkbenchSocket.onMessage(String, WebSocketConnection)` — bidirectional WebSocket handler

- [ ] **Step 1: Write failing test — WorkbenchSocket receives coaching_response**

In `WorkbenchSocketIT.java` (or create if needed):

```java
@Test
void coachingResponse_sentViaWebSocket_resolvedByHandler() throws Exception {
    // Connect WebSocket client
    // Inject a coaching commitment into the broker
    // Send: {"type":"coaching_response","correlationId":"test-corr","response":"DONE"}
    // Assert: commitment removed from evaluator
    // Assert: coaching_compliance event received by client
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=WorkbenchSocketIT -q`
Expected: FAIL — no `@OnTextMessage` handler

- [ ] **Step 3: Add @OnTextMessage to WorkbenchSocket**

```java
@OnTextMessage
public void onMessage(String message, WebSocketConnection connection) {
    try {
        var node = objectMapper.readTree(message);
        String type = node.path("type").asText(null);
        if (!"coaching_response".equals(type)) {
            log.debugf("[WORKBENCH] Ignoring unknown message type: %s", type);
            return;
        }
        String correlationId = node.path("correlationId").asText(null);
        String response = node.path("response").asText(null);
        if (correlationId == null || correlationId.isBlank() || response == null) {
            log.debugf("[WORKBENCH] Malformed coaching_response — missing correlationId or response");
            return;
        }
        boolean accepted = "DONE".equals(response);
        handler.acknowledge(correlationId, accepted);
    } catch (Exception e) {
        log.debugf("[WORKBENCH] Failed to parse WebSocket message: %s", e.getMessage());
    }
}
```

Add injections to WorkbenchSocket:

```java
@Inject ObjectMapper objectMapper;
@Inject CoachingAcknowledgmentHandler handler;
```

- [ ] **Step 4: Update visualizer.js — add Accept/Dismiss buttons**

In `renderCoachingPage()`:

```javascript
function renderCoachingPage() {
  var el = document.getElementById('page-coaching');
  if (workbenchState.coaching.length === 0) { el.innerHTML = 'No coaching advice yet'; return; }

  el.innerHTML = workbenchState.coaching.map(function(c) {
    var secs = Math.floor(c.gameFrame / 22.4);
    var mins = Math.floor(secs / 60);
    var rem = secs % 60;
    var time = mins + ':' + String(rem).padStart(2, '0');
    var status = c.complianceStatus || '⏳ Pending';
    var buttons = '';
    if (!c.complianceStatus) {
      buttons = '<div class="coaching-buttons">' +
        '<button class="coaching-accept" onclick="sendCoachingResponse(\'' + c.correlationId + '\', \'DONE\')">✓ Accept</button>' +
        '<button class="coaching-dismiss" onclick="sendCoachingResponse(\'' + c.correlationId + '\', \'DECLINE\')">✗ Dismiss</button>' +
      '</div>';
    }
    return '<div class="coaching-item">' +
      '<div class="coaching-header">' + time + ' [' + c.domain + '] ' + (c.urgency || '') + '</div>' +
      '<div class="coaching-advice">' + c.advice + '</div>' +
      buttons +
      '<div class="coaching-status" data-correlation-id="' + c.correlationId + '">' + status + '</div>' +
    '</div>';
  }).join('');
}

function sendCoachingResponse(correlationId, response) {
  if (workbenchWs && workbenchWs.readyState === WebSocket.OPEN) {
    workbenchWs.send(JSON.stringify({type: 'coaching_response', correlationId: correlationId, response: response}));
  }
}
```

- [ ] **Step 5: Run WebSocket integration test**

Run: `mvn test -Dtest=WorkbenchSocketIT -q`
Expected: PASS

- [ ] **Step 6: Run Playwright render tests**

Run: `mvn test -Pplaywright`
Expected: ALL PASS (including new coaching button assertions in WorkbenchRenderTest)

- [ ] **Step 7: Run full test suite**

Run: `mvn test -q`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#248): coaching acknowledgment UI with WebSocket bidirectional messaging

Accept/Dismiss buttons on coaching page. Human responses dispatched
via MessageService with correct correlationId. Auto-compliance and
human responses use the same rendering path.

Closes #248"
```
