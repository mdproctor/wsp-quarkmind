# Human Feedback Trust Dimensions — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #231 — Human feedback trust dimensions — timing quality and accuracy for commentary
**Issue group:** #231

**Goal:** Add per-event human feedback buttons to workbench commentary, recording `timing-quality` and `accuracy` trust dimensions via OutcomeRecorder.

**Architecture:** CommentaryFeedbackHandler (new CDI bean, mirrors CoachingAcknowledgmentHandler) receives feedback from the workbench WebSocket and records OutcomeRecords. WorkbenchSocket parses `commentary_feedback` messages and delegates. QuarkMindTrustRoutingPolicyProvider gains quality floors for the two new dimensions.

**Tech Stack:** Java 21, Quarkus, Lit/TypeScript (webui), WebSocket, Ledger OutcomeRecorder SPI

## Global Constraints

- QA classes carry `@UnlessBuildProfile("prod")` — feedback handler is QA-only
- Domain model (`domain/`) must remain plain Java
- WebSocket message format: `{ type: "commentary_feedback", workerId: string, dimension: string, positive: boolean }`

---

## Batch 1: Backend — feedback handler + trust policy + WebSocket wiring

### Task 1: CommentaryFeedbackHandler + trust policy update

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/qa/workbench/CommentaryFeedbackHandler.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/QuarkMindTrustRoutingPolicyProvider.java:99-114`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/qa/workbench/CommentaryFeedbackHandlerTest.java` (new)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/agent/QuarkMindTrustRoutingPolicyProviderTest.java` (modify)

**Interfaces:**
- Consumes: `OutcomeRecorder` SPI (from casehub-ledger), `GameSession` (for session ID)
- Produces: `recordFeedback(String workerId, String dimension, boolean positive)` — consumed by WorkbenchSocket in Task 2

- [ ] **Step 1: Write the failing test — positive feedback records ENDORSED**

```java
package io.quarkmind.qa.workbench;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.OutcomeRecorder;
import io.quarkmind.agent.GameSession;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class CommentaryFeedbackHandlerTest {

    private TestOutcomeRecorder outcomeRecorder;
    private CommentaryFeedbackHandler handler;
    private UUID gameSessionId;

    @BeforeEach
    void setUp() {
        outcomeRecorder = new TestOutcomeRecorder();
        gameSessionId = UUID.randomUUID();
        GameSession session = new GameSession();
        session.setCaseId(gameSessionId);
        handler = new CommentaryFeedbackHandler();
        handler.outcomeRecorder = outcomeRecorder;
        handler.gameSession = session;
    }

    @Test
    void positiveFeedback_recordsEndorsedWithFullConfidence() {
        handler.recordFeedback("claude:narrator-reactive@v1", "timing-quality", true);

        assertThat(outcomeRecorder.records).hasSize(1);
        OutcomeRecord record = outcomeRecorder.records.get(0);
        assertThat(record.actorId()).isEqualTo("claude:narrator-reactive@v1");
        assertThat(record.subjectId()).isEqualTo(gameSessionId);
        assertThat(record.capabilityTag()).isEqualTo("timing-quality");
        assertThat(record.verdict()).isEqualTo(AttestationVerdict.ENDORSED);
        assertThat(record.confidence()).isEqualTo(1.0);
    }

    @Test
    void negativeFeedback_recordsChallengedWithZeroConfidence() {
        handler.recordFeedback("claude:narrator-reactive@v1", "accuracy", false);

        assertThat(outcomeRecorder.records).hasSize(1);
        OutcomeRecord record = outcomeRecorder.records.get(0);
        assertThat(record.capabilityTag()).isEqualTo("accuracy");
        assertThat(record.verdict()).isEqualTo(AttestationVerdict.CHALLENGED);
        assertThat(record.confidence()).isEqualTo(0.0);
    }

    @Test
    void bothDimensions_recordedIndependently() {
        handler.recordFeedback("claude:narrator-reactive@v1", "timing-quality", true);
        handler.recordFeedback("claude:narrator-reactive@v1", "accuracy", false);

        assertThat(outcomeRecorder.records).hasSize(2);
        assertThat(outcomeRecorder.records.get(0).capabilityTag()).isEqualTo("timing-quality");
        assertThat(outcomeRecorder.records.get(1).capabilityTag()).isEqualTo("accuracy");
    }

    static class TestOutcomeRecorder implements OutcomeRecorder {
        final List<OutcomeRecord> records = new ArrayList<>();

        @Override
        public UUID record(OutcomeRecord record) {
            records.add(record);
            return UUID.randomUUID();
        }

        @Override
        public UUID record(OutcomeRecord record, String source) {
            return record(record);
        }

        @Override
        public void addAttestation(UUID id, AttestationVerdict verdict, double confidence, String dimension) {}
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test-compile -pl quarkmind-sc2 -q`
Expected: FAIL — `CommentaryFeedbackHandler` does not exist

- [ ] **Step 3: Write CommentaryFeedbackHandler**

Create `quarkmind-sc2/src/main/java/io/quarkmind/qa/workbench/CommentaryFeedbackHandler.java`:

```java
package io.quarkmind.qa.workbench;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.OutcomeRecorder;
import io.quarkmind.agent.GameSession;
import io.quarkus.arc.profile.UnlessBuildProfile;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@UnlessBuildProfile("prod")
@ApplicationScoped
public class CommentaryFeedbackHandler {

    @Inject OutcomeRecorder outcomeRecorder;
    @Inject GameSession gameSession;

    public void recordFeedback(String workerId, String dimension, boolean positive) {
        double score = positive ? 1.0 : 0.0;
        AttestationVerdict verdict = positive ? AttestationVerdict.ENDORSED : AttestationVerdict.CHALLENGED;
        outcomeRecorder.record(new OutcomeRecord(
            workerId,
            gameSession.id(),
            dimension,
            verdict,
            score
        ));
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=CommentaryFeedbackHandlerTest -q`
Expected: ALL PASS (3 tests)

- [ ] **Step 5: Update QuarkMindTrustRoutingPolicyProvider — add quality floors**

In `buildCommentaryPolicy()` (line ~99-114), update the quality floors map:

```java
// Before:
Map<String, Double> qualityFloors = Map.of(
    "response-latency", latencyFloor
);

// After:
Map<String, Double> qualityFloors = Map.of(
    "response-latency", latencyFloor,
    "timing-quality", 0.3,
    "accuracy", 0.4
);
```

- [ ] **Step 6: Update QuarkMindTrustRoutingPolicyProviderTest**

Add assertions for the new quality floors in the commentary policy test:

```java
// In existing commentary policy test method:
assertThat(policy.qualityFloors())
    .containsKeys("response-latency", "timing-quality", "accuracy");
assertThat(policy.qualityFloors().get("timing-quality")).isEqualTo(0.3);
assertThat(policy.qualityFloors().get("accuracy")).isEqualTo(0.4);
```

- [ ] **Step 7: Run policy provider tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=QuarkMindTrustRoutingPolicyProviderTest -q`
Expected: ALL PASS

- [ ] **Step 8: Wire into WorkbenchSocket.onMessage()**

Add `@Inject CommentaryFeedbackHandler feedbackHandler;` field.

In `onMessage()`, add handling after the existing `coaching_response` block:

```java
if ("commentary_feedback".equals(type)) {
    String workerId = node.path("workerId").asText(null);
    String dimension = node.path("dimension").asText(null);
    boolean positive = node.path("positive").asBoolean();
    if (workerId != null && dimension != null) {
        feedbackHandler.recordFeedback(workerId, dimension, positive);
    }
    return;
}
```

- [ ] **Step 9: Run full test suite for regression**

Run: `mvn test -pl quarkmind-sc2 -Dtest=CommentaryFeedbackHandlerTest,QuarkMindTrustRoutingPolicyProviderTest,WorkbenchSocketIT -q`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/qa/workbench/CommentaryFeedbackHandler.java
git add quarkmind-sc2/src/main/java/io/quarkmind/qa/workbench/WorkbenchSocket.java
git add quarkmind-sc2/src/main/java/io/quarkmind/agent/QuarkMindTrustRoutingPolicyProvider.java
git add quarkmind-sc2/src/test/java/io/quarkmind/qa/workbench/CommentaryFeedbackHandlerTest.java
git add quarkmind-sc2/src/test/java/io/quarkmind/agent/QuarkMindTrustRoutingPolicyProviderTest.java
git commit -m "feat: commentary feedback handler + trust policy quality floors

Adds CommentaryFeedbackHandler for per-event timing-quality and accuracy
trust dimensions. Wires into WorkbenchSocket via commentary_feedback
message type. Quality floors: timing-quality=0.3, accuracy=0.4.

Refs #231"
```

---

## Batch 2: Frontend — feedback buttons on commentary entries

### Task 2: Add feedback buttons to qm-commentary-page

**Files:**
- Modify: `quarkmind-sc2/src/main/webui/workbench/qm-commentary-page.ts`

**Interfaces:**
- Consumes: WebSocket connection (sends `commentary_feedback` messages)
- Produces: user-facing feedback buttons on each commentary entry

**Note:** The commentary page currently delegates all rendering to the `blocks-channel-feed` web component. Feedback buttons need to be added either as a custom renderer for the feed, or as an overlay on commentary entries. The approach depends on whether `blocks-channel-feed` supports action slots.

- [ ] **Step 1: Check blocks-channel-feed API for action slot support**

Inspect the `blocks-channel-feed` component to determine if it supports per-message action slots or custom message templates. If yes, use the slot. If no, add a custom message list that renders commentary entries with buttons instead of using `blocks-channel-feed`.

- [ ] **Step 2: Add feedback button rendering**

Add a `_sendFeedback` method that sends the WebSocket message:

```typescript
private _sendFeedback(workerId: string, dimension: string, positive: boolean) {
  const ws = (this.closest('qm-workbench-shell') as any)?._ws;
  if (!ws) return;
  ws.send(JSON.stringify({
    type: 'commentary_feedback',
    workerId,
    dimension,
    positive
  }));
}
```

The exact WebSocket reference depends on how `qm-workbench-shell` exposes the connection. Check the shell component for the WebSocket property name.

Add per-entry state tracking for feedback-given flags and render buttons inline with each commentary entry. Button labels: "👍 Timing", "👎 Timing", "✓ Accurate", "✗ Wrong". Disable the pair after a click.

- [ ] **Step 3: Build webui**

Run: `cd quarkmind-sc2/src/main/webui && npm run build`
Expected: BUILD SUCCESS

- [ ] **Step 4: Manual verification (if dev server running)**

Run: `mvn quarkus:dev -pl quarkmind-sc2`
Open workbench, verify feedback buttons appear on commentary entries. Click a button, verify the WebSocket message is sent (check browser dev tools Network tab).

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/main/webui/workbench/qm-commentary-page.ts
git commit -m "feat: commentary feedback buttons in workbench UI

Per-event timing-quality and accuracy feedback buttons on each
commentary entry. Sends commentary_feedback WebSocket message on click.

Closes #231"
```

---

## References

- [2026-09-12-human-feedback-trust-design.md] — design spec
- [CoachingAcknowledgmentHandler.java] — coaching feedback pattern (template)
- [WorkbenchSocket.java:41-56] — WebSocket message handling
- [QuarkMindTrustRoutingPolicyProvider.java:99-114] — commentary trust policy
- [LlmWorkerLatencyRecorderTest.java:187-206] — TestOutcomeRecorder pattern
- [CommentaryPayload.java] — workerId field available in payload
- [CommentaryCompleted.java] — commentary event record
- [qm-commentary-page.ts] — workbench commentary Lit component
- Protocol `trust-routing-config-flags-required.md`
- GitHub #231, #181
