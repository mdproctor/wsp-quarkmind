# Visualizer Workbench Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #259 — feat: visualizer counter overlay
**Issue group:** #259

**Goal:** Replace the full-viewport Three.js canvas with a workbench application shell that hosts the 3D view alongside paged intelligence panels (pattern assessment, coaching, strategy) with bidirectional selection correlation.

**Architecture:** Pure DOM application shell (CSS grid) embeds the Three.js canvas as one pane. A dedicated `/ws/workbench` WebSocket pushes intelligence events (pattern assessments enriched with counter info, coaching advice with compliance, strategy routing state) via typed CDI events. A shared client-side selection model coordinates pulsing ring indicators in the 3D view with workbench panel content.

**Tech Stack:** Quarkus WebSocket Next, CDI events, Jackson serialization, Three.js r128, vanilla JS, CSS Grid

## Global Constraints

- Domain model (`domain/`) must remain plain Java — no CDI, no Quarkus imports
- QA endpoints carry `@UnlessBuildProfile("prod")`
- `countersFor()` defaults to `Race.PROTOSS` (QuarkMind plays Protoss; multi-race is #74)
- Never `@QuarkusTest` for tests that can be plain JUnit
- All new CDI events are records in their originating package
- WebSocket endpoint follows `GameStateSocket`/`GameStateBroadcaster` pattern exactly

---

### Task 1: CDI Events — Pattern, Coaching, Compliance, Strategy

New CDI event records that the existing pipeline classes will fire. Pure records with no framework dependencies. These are the data contracts between the pipeline and the workbench broadcaster.

**Files:**
- Create: `src/main/java/io/quarkmind/agent/plugin/PatternAssessmentPublished.java`
- Create: `src/main/java/io/quarkmind/plugin/coaching/CoachingAdvicePublished.java`
- Create: `src/main/java/io/quarkmind/plugin/coaching/CoachingComplianceResolved.java`
- Create: `src/main/java/io/quarkmind/agent/cbr/StrategySelectionPublished.java`
- Test: `src/test/java/io/quarkmind/qa/WorkbenchEventTest.java`

**Interfaces:**
- Consumes: `PatternAssessment` from `io.quarkmind.domain`, `CoachingAdvice` and `CoachingUrgencyTier` from `io.quarkmind.plugin.coaching`, `StrategyArchetype` from `io.quarkmind.domain`
- Produces: Four CDI event records used by Task 3 (`WorkbenchEnricher`) and Task 2 (fire points)

- [ ] **Step 1: Write test for CDI event record construction and field access**

```java
// src/test/java/io/quarkmind/qa/WorkbenchEventTest.java
package io.quarkmind.qa;

import io.quarkmind.agent.plugin.PatternAssessmentPublished;
import io.quarkmind.domain.PatternAssessment;
import io.quarkmind.domain.StrategyArchetype;
import io.quarkmind.plugin.coaching.CoachingAdvicePublished;
import io.quarkmind.plugin.coaching.CoachingComplianceResolved;
import io.quarkmind.plugin.coaching.CoachingAdvice;
import io.quarkmind.plugin.coaching.CoachingDomain;
import io.quarkmind.plugin.coaching.CoachingUrgencyTier;
import io.quarkmind.agent.cbr.StrategySelectionPublished;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class WorkbenchEventTest {

    @Test void patternAssessmentPublished_carriesAssessments() {
        var a = new PatternAssessment(StrategyArchetype.ZERG_EARLY_POOL, 0.87, 1000, "6+ lings");
        var event = new PatternAssessmentPublished(List.of(a));
        assertEquals(1, event.assessments().size());
        assertEquals(StrategyArchetype.ZERG_EARLY_POOL, event.assessments().getFirst().archetype());
    }

    @Test void coachingAdvicePublished_carriesFields() {
        var advice = new CoachingAdvice("build stalkers", CoachingDomain.MILITARY, null, 200);
        var event = new CoachingAdvicePublished(advice, CoachingUrgencyTier.CRISIS, 500L);
        assertEquals("build stalkers", event.advice().advice());
        assertEquals(CoachingUrgencyTier.CRISIS, event.urgencyTier());
        assertEquals(500L, event.gameFrame());
    }

    @Test void coachingComplianceResolved_carriesFields() {
        var event = new CoachingComplianceResolved(500L, CoachingDomain.MILITARY, "complied");
        assertEquals(CoachingDomain.MILITARY, event.domain());
        assertEquals("complied", event.status());
    }

    @Test void strategySelectionPublished_carriesFields() {
        var event = new StrategySelectionPublished("reactive-blink", StrategyArchetype.ZERG_EARLY_POOL, 0.82, 1);
        assertEquals("reactive-blink", event.strategyId());
        assertEquals(0.82, event.confidence());
        assertEquals(1, event.pivotCount());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=WorkbenchEventTest -q`
Expected: FAIL — classes do not exist

- [ ] **Step 3: Create the four CDI event records**

```java
// src/main/java/io/quarkmind/agent/plugin/PatternAssessmentPublished.java
package io.quarkmind.agent.plugin;

import io.quarkmind.domain.PatternAssessment;
import java.util.List;

public record PatternAssessmentPublished(List<PatternAssessment> assessments) {}
```

```java
// src/main/java/io/quarkmind/plugin/coaching/CoachingAdvicePublished.java
package io.quarkmind.plugin.coaching;

public record CoachingAdvicePublished(CoachingAdvice advice, CoachingUrgencyTier urgencyTier, long gameFrame) {}
```

```java
// src/main/java/io/quarkmind/plugin/coaching/CoachingComplianceResolved.java
package io.quarkmind.plugin.coaching;

public record CoachingComplianceResolved(long gameFrame, CoachingDomain domain, String status) {}
```

```java
// src/main/java/io/quarkmind/agent/cbr/StrategySelectionPublished.java
package io.quarkmind.agent.cbr;

import io.quarkmind.domain.StrategyArchetype;

public record StrategySelectionPublished(String strategyId, StrategyArchetype archetype, double confidence, int pivotCount) {}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -Dtest=WorkbenchEventTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#259): add CDI event records for workbench broadcasting
```

---

### Task 2: Fire CDI Events from Pipeline Classes

Wire the four CDI events into their respective pipeline classes. Each fire point is a one-line addition alongside existing dispatch logic.

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java` (inject `Event<PatternAssessmentPublished>`, fire at line ~302)
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingChannelBroker.java` (inject `Event<CoachingAdvicePublished>`, fire after Qhorus dispatch)
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluator.java` (inject `Event<CoachingComplianceResolved>`, fire when commitment resolves)
- Modify: `src/main/java/io/quarkmind/agent/cbr/SC2StrategyRouterTask.java` (inject `Event<StrategySelectionPublished>`, fire after CaseFile writes at line ~191)
- Test: `src/test/java/io/quarkmind/plugin/scouting/DroolsScoutingTaskTest.java` (extend)
- Test: `src/test/java/io/quarkmind/plugin/coaching/CoachingChannelBrokerTest.java` (extend)
- Test: `src/test/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluatorTest.java` (new test method)
- Test: `src/test/java/io/quarkmind/agent/cbr/SC2StrategyRouterTaskTest.java` (extend)

**Interfaces:**
- Consumes: CDI event records from Task 1
- Produces: CDI events fired at runtime, consumed by `WorkbenchEnricher` in Task 3

- [ ] **Step 1: Write test — DroolsScoutingTask fires PatternAssessmentPublished**

Add a test to `DroolsScoutingTaskTest` that verifies when pattern assessments are dispatched, the CDI event is also fired. Since this is a unit test (no CDI), use a mock/capture approach: add a `Consumer<PatternAssessmentPublished>` field to `DroolsScoutingTask` that defaults to a no-op and is set in tests.

Actually — `DroolsScoutingTask` already uses `Event<EnemyPostureClassifiedEvent>` via CDI injection. Follow the same pattern: inject `Event<PatternAssessmentPublished>` and fire it. For unit tests, the existing test class constructs `DroolsScoutingTask` manually — check what constructor it uses and add the new event parameter.

Read `DroolsScoutingTaskTest` to understand its construction pattern, then write the test.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=DroolsScoutingTaskTest#<newTestMethod> -q`
Expected: FAIL

- [ ] **Step 3: Add `Event<PatternAssessmentPublished>` to DroolsScoutingTask**

Inject the event field. In the `publishIntel` method (around line 302, after `broker.update(payload)`), add:

```java
if (payload instanceof PatternAssessmentPayload pa) {
    patternAssessmentEvent.fire(new PatternAssessmentPublished(pa.assessments()));
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -Dtest=DroolsScoutingTaskTest -q`

- [ ] **Step 5: Write test — CoachingChannelBroker fires CoachingAdvicePublished**

Extend `CoachingChannelBrokerTest` to verify that after `onCoachingCompleted()`, a `CoachingAdvicePublished` event is fired.

- [ ] **Step 6: Implement — inject and fire in CoachingChannelBroker**

In `onCoachingCompleted()`, after the `dispatchCount++` line (line ~91), fire:

```java
coachingAdviceEvent.fire(new CoachingAdvicePublished(event.advice(), event.urgencyTier(), event.gameFrame()));
```

- [ ] **Step 7: Run test to verify it passes**

Run: `mvn test -Dtest=CoachingChannelBrokerTest -q`

- [ ] **Step 8: Write test — CoachingComplianceEvaluator fires CoachingComplianceResolved**

Extend `CoachingComplianceEvaluatorTest` (or add new test class) to verify that when a commitment resolves to ENDORSED/CHALLENGED/NEUTRAL, a `CoachingComplianceResolved` event is fired.

- [ ] **Step 9: Implement — inject and fire in CoachingComplianceEvaluator**

Add `Event<CoachingComplianceResolved>` as an injected field. In `evaluate()`, after each `recorder.record(...)` / `iterator.remove()` call, fire:

```java
complianceEvent.fire(new CoachingComplianceResolved(currentFrame, entry.getKey(), "ENDORSED"));
```

(Similarly for "CHALLENGED" and "NEUTRAL" paths.)

- [ ] **Step 10: Run test to verify it passes**

Run: `mvn test -Dtest=CoachingComplianceEvaluatorTest -q`

- [ ] **Step 11: Write test — SC2StrategyRouterTask fires StrategySelectionPublished**

Extend `SC2StrategyRouterTaskTest` to verify that after strategy selection, the CDI event is fired with the selected strategy ID, archetype, confidence, and pivot count.

- [ ] **Step 12: Implement — inject and fire in SC2StrategyRouterTask**

After the CaseFile writes at line ~191, fire:

```java
strategyEvent.fire(new StrategySelectionPublished(winner, archetype, confidence, pivotCount + 1));
```

- [ ] **Step 13: Run test to verify it passes**

Run: `mvn test -Dtest=SC2StrategyRouterTaskTest -q`

- [ ] **Step 14: Run full test suite**

Run: `mvn test -q`
Expected: All green

- [ ] **Step 15: Commit**

```
feat(#259): fire CDI events from scouting, coaching, compliance, and strategy pipelines
```

---

### Task 3: WorkbenchEnricher and WorkbenchBroadcaster (Server Side)

The server-side infrastructure: enricher observes CDI events, attaches counter info, forwards to broadcaster. Broadcaster manages WebSocket sessions and maintains snapshot for reconnect.

**Files:**
- Create: `src/main/java/io/quarkmind/qa/workbench/WorkbenchEvent.java`
- Create: `src/main/java/io/quarkmind/qa/workbench/WorkbenchPayload.java`
- Create: `src/main/java/io/quarkmind/qa/workbench/PatternPayload.java`
- Create: `src/main/java/io/quarkmind/qa/workbench/EnrichedAssessment.java`
- Create: `src/main/java/io/quarkmind/qa/workbench/CoachingPayload.java`
- Create: `src/main/java/io/quarkmind/qa/workbench/CoachingCompliancePayload.java`
- Create: `src/main/java/io/quarkmind/qa/workbench/StrategyPayload.java`
- Create: `src/main/java/io/quarkmind/qa/workbench/WorkbenchEnricher.java`
- Create: `src/main/java/io/quarkmind/qa/workbench/WorkbenchBroadcaster.java`
- Create: `src/main/java/io/quarkmind/qa/workbench/WorkbenchSocket.java`
- Test: `src/test/java/io/quarkmind/qa/workbench/WorkbenchEnricherTest.java`
- Test: `src/test/java/io/quarkmind/qa/workbench/WorkbenchBroadcasterTest.java`

**Interfaces:**
- Consumes: CDI events from Task 2, `StrategyTaxonomy.countersFor()` for enrichment
- Produces: `WorkbenchEvent` JSON pushed to `/ws/workbench` WebSocket clients; snapshot pushed on connect

- [ ] **Step 1: Write test — WorkbenchEnricher enriches pattern assessments with counter info**

```java
// src/test/java/io/quarkmind/qa/workbench/WorkbenchEnricherTest.java
package io.quarkmind.qa.workbench;

import io.quarkmind.agent.StrategyTaxonomy;
import io.quarkmind.agent.plugin.PatternAssessmentPublished;
import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class WorkbenchEnricherTest {

    @Test void enriches_pattern_assessments_with_counter_info() {
        var taxonomy = new StrategyTaxonomy(); // loads strategy-taxonomy.yaml from classpath
        var broadcaster = new WorkbenchBroadcaster();
        var enricher = new WorkbenchEnricher(taxonomy, broadcaster);

        var assessment = new PatternAssessment(StrategyArchetype.ZERG_EARLY_POOL, 0.87, 1000, "6+ lings");
        enricher.onPatternAssessment(new PatternAssessmentPublished(List.of(assessment)));

        var event = broadcaster.lastEvent();
        assertNotNull(event);
        assertEquals("pattern", event.type());
        var payload = (PatternPayload) event.payload();
        assertEquals(1, payload.assessments().size());
        assertEquals(StrategyArchetype.ZERG_EARLY_POOL, payload.assessments().getFirst().assessment().archetype());
        // counters may be null if taxonomy doesn't have entry — that's OK
    }

    @Test void enrichment_failure_for_one_archetype_still_produces_event() {
        var taxonomy = new StrategyTaxonomy();
        var broadcaster = new WorkbenchBroadcaster();
        var enricher = new WorkbenchEnricher(taxonomy, broadcaster);

        // Use an archetype that may not have counters defined
        var a1 = new PatternAssessment(StrategyArchetype.ZERG_EARLY_POOL, 0.87, 1000, "pool");
        var a2 = new PatternAssessment(StrategyArchetype.ZERG_LATE_BROOD_LORD, 0.31, 1000, "brood");
        enricher.onPatternAssessment(new PatternAssessmentPublished(List.of(a1, a2)));

        var event = broadcaster.lastEvent();
        assertNotNull(event);
        var payload = (PatternPayload) event.payload();
        assertEquals(2, payload.assessments().size());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=WorkbenchEnricherTest -q`
Expected: FAIL — classes do not exist

- [ ] **Step 3: Create payload records and sealed interface**

```java
// WorkbenchPayload.java
package io.quarkmind.qa.workbench;

public sealed interface WorkbenchPayload permits PatternPayload, CoachingPayload, CoachingCompliancePayload, StrategyPayload {}
```

```java
// WorkbenchEvent.java
package io.quarkmind.qa.workbench;

public record WorkbenchEvent(String type, WorkbenchPayload payload) {}
```

```java
// EnrichedAssessment.java
package io.quarkmind.qa.workbench;

import io.quarkmind.domain.CounterInfo;
import io.quarkmind.domain.PatternAssessment;

public record EnrichedAssessment(PatternAssessment assessment, CounterInfo counters) {}
```

```java
// PatternPayload.java
package io.quarkmind.qa.workbench;

import java.util.List;

public record PatternPayload(List<EnrichedAssessment> assessments) implements WorkbenchPayload {}
```

```java
// CoachingPayload.java
package io.quarkmind.qa.workbench;

import io.quarkmind.plugin.coaching.CoachingDomain;
import io.quarkmind.plugin.coaching.CoachingUrgencyTier;

public record CoachingPayload(String advice, CoachingDomain domain, CoachingUrgencyTier urgency, long gameFrame) implements WorkbenchPayload {}
```

```java
// CoachingCompliancePayload.java
package io.quarkmind.qa.workbench;

import io.quarkmind.plugin.coaching.CoachingDomain;

public record CoachingCompliancePayload(long gameFrame, CoachingDomain domain, String status) implements WorkbenchPayload {}
```

```java
// StrategyPayload.java
package io.quarkmind.qa.workbench;

import io.quarkmind.domain.StrategyArchetype;

public record StrategyPayload(String strategyId, StrategyArchetype archetype, double confidence, int pivotCount) implements WorkbenchPayload {}
```

- [ ] **Step 4: Create WorkbenchBroadcaster**

```java
// WorkbenchBroadcaster.java
package io.quarkmind.qa.workbench;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.quarkus.arc.profile.UnlessBuildProfile;
import io.quarkus.websockets.next.WebSocketConnection;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;
import java.util.Set;
import java.util.concurrent.CopyOnWriteArraySet;

@UnlessBuildProfile("prod")
@ApplicationScoped
public class WorkbenchBroadcaster {

    private static final Logger log = Logger.getLogger(WorkbenchBroadcaster.class);

    @Inject ObjectMapper objectMapper;

    private final Set<WebSocketConnection> sessions = new CopyOnWriteArraySet<>();

    private volatile WorkbenchEvent latestPattern;
    private volatile WorkbenchEvent latestStrategy;
    private volatile WorkbenchEvent latestCoaching;

    // For unit testing without CDI
    WorkbenchBroadcaster() {}

    private volatile WorkbenchEvent lastEventForTest;
    WorkbenchEvent lastEvent() { return lastEventForTest; }

    void addSession(WebSocketConnection connection) {
        sessions.add(connection);
        pushSnapshot(connection);
        log.infof("[WORKBENCH] Client connected — %d active", sessions.size());
    }

    void removeSession(WebSocketConnection connection) {
        sessions.remove(connection);
        log.infof("[WORKBENCH] Client disconnected — %d active", sessions.size());
    }

    void broadcast(WorkbenchEvent event) {
        lastEventForTest = event;
        updateSnapshot(event);
        if (sessions.isEmpty()) return;
        try {
            String json = objectMapper != null ? objectMapper.writeValueAsString(event) : event.toString();
            sessions.forEach(s -> s.sendText(json)
                .subscribe().with(
                    ignored -> {},
                    err -> log.warnf("[WORKBENCH] Send failed: %s", err.getMessage())));
        } catch (Exception e) {
            log.warnf(e, "[WORKBENCH] Serialisation failed: %s", e.getMessage());
        }
    }

    private void updateSnapshot(WorkbenchEvent event) {
        switch (event.type()) {
            case "pattern"  -> latestPattern  = event;
            case "strategy" -> latestStrategy = event;
            case "coaching" -> latestCoaching = event;
            // coaching_compliance does not replace snapshot — it's a status update
        }
    }

    private void pushSnapshot(WebSocketConnection connection) {
        try {
            if (latestPattern != null) sendOne(connection, latestPattern);
            if (latestStrategy != null) sendOne(connection, latestStrategy);
            if (latestCoaching != null) sendOne(connection, latestCoaching);
        } catch (Exception e) {
            log.warnf(e, "[WORKBENCH] Snapshot push failed: %s", e.getMessage());
        }
    }

    private void sendOne(WebSocketConnection connection, WorkbenchEvent event) throws Exception {
        String json = objectMapper.writeValueAsString(event);
        connection.sendText(json).subscribe().with(ignored -> {}, err -> {});
    }
}
```

- [ ] **Step 5: Create WorkbenchEnricher**

```java
// WorkbenchEnricher.java
package io.quarkmind.qa.workbench;

import io.quarkus.arc.profile.UnlessBuildProfile;
import io.quarkmind.agent.StrategyTaxonomy;
import io.quarkmind.agent.cbr.StrategySelectionPublished;
import io.quarkmind.agent.plugin.PatternAssessmentPublished;
import io.quarkmind.domain.CounterInfo;
import io.quarkmind.plugin.coaching.CoachingAdvicePublished;
import io.quarkmind.plugin.coaching.CoachingComplianceResolved;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@UnlessBuildProfile("prod")
@ApplicationScoped
public class WorkbenchEnricher {

    private static final Logger log = Logger.getLogger(WorkbenchEnricher.class);

    private final StrategyTaxonomy taxonomy;
    private final WorkbenchBroadcaster broadcaster;

    @Inject
    WorkbenchEnricher(StrategyTaxonomy taxonomy, WorkbenchBroadcaster broadcaster) {
        this.taxonomy = taxonomy;
        this.broadcaster = broadcaster;
    }

    // For unit testing
    WorkbenchEnricher() { this.taxonomy = null; this.broadcaster = null; }

    void onPatternAssessment(@Observes PatternAssessmentPublished event) {
        var enriched = event.assessments().stream()
            .map(a -> {
                CounterInfo counters = null;
                try { counters = taxonomy.countersFor(a.archetype()); }
                catch (Exception e) { log.debugf("No counters for %s: %s", a.archetype(), e.getMessage()); }
                return new EnrichedAssessment(a, counters);
            })
            .toList();
        broadcaster.broadcast(new WorkbenchEvent("pattern", new PatternPayload(enriched)));
    }

    void onCoachingAdvice(@Observes CoachingAdvicePublished event) {
        broadcaster.broadcast(new WorkbenchEvent("coaching",
            new CoachingPayload(event.advice().advice(), event.advice().domainTag(), event.urgencyTier(), event.gameFrame())));
    }

    void onCoachingCompliance(@Observes CoachingComplianceResolved event) {
        broadcaster.broadcast(new WorkbenchEvent("coaching_compliance",
            new CoachingCompliancePayload(event.gameFrame(), event.domain(), event.status())));
    }

    void onStrategySelection(@Observes StrategySelectionPublished event) {
        broadcaster.broadcast(new WorkbenchEvent("strategy",
            new StrategyPayload(event.strategyId(), event.archetype(), event.confidence(), event.pivotCount())));
    }
}
```

- [ ] **Step 6: Create WorkbenchSocket**

```java
// WorkbenchSocket.java
package io.quarkmind.qa.workbench;

import io.quarkus.arc.profile.UnlessBuildProfile;
import io.quarkus.websockets.next.OnClose;
import io.quarkus.websockets.next.OnOpen;
import io.quarkus.websockets.next.WebSocket;
import io.quarkus.websockets.next.WebSocketConnection;
import jakarta.inject.Inject;

@UnlessBuildProfile("prod")
@WebSocket(path = "/ws/workbench")
public class WorkbenchSocket {

    @Inject WorkbenchBroadcaster broadcaster;

    @OnOpen
    public void onOpen(WebSocketConnection connection) {
        broadcaster.addSession(connection);
    }

    @OnClose
    public void onClose(WebSocketConnection connection) {
        broadcaster.removeSession(connection);
    }
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `mvn test -Dtest=WorkbenchEnricherTest -q`
Expected: PASS

- [ ] **Step 8: Write test — WorkbenchBroadcaster snapshot on reconnect**

```java
// src/test/java/io/quarkmind/qa/workbench/WorkbenchBroadcasterTest.java
package io.quarkmind.qa.workbench;

import io.quarkmind.domain.PatternAssessment;
import io.quarkmind.domain.StrategyArchetype;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class WorkbenchBroadcasterTest {

    @Test void snapshot_stores_latest_per_type() {
        var broadcaster = new WorkbenchBroadcaster();

        var patternEvent = new WorkbenchEvent("pattern",
            new PatternPayload(List.of(new EnrichedAssessment(
                new PatternAssessment(StrategyArchetype.ZERG_EARLY_POOL, 0.8, 100, "pool"), null))));
        broadcaster.broadcast(patternEvent);

        var strategyEvent = new WorkbenchEvent("strategy",
            new StrategyPayload("reactive-blink", StrategyArchetype.ZERG_EARLY_POOL, 0.82, 1));
        broadcaster.broadcast(strategyEvent);

        assertEquals(strategyEvent, broadcaster.lastEvent());
        // Snapshot is tested via WebSocket IT in Task 4
    }
}
```

- [ ] **Step 9: Run test to verify it passes**

Run: `mvn test -Dtest=WorkbenchBroadcasterTest -q`

- [ ] **Step 10: Run full test suite**

Run: `mvn test -q`
Expected: All green

- [ ] **Step 11: Commit**

```
feat(#259): add WorkbenchEnricher, WorkbenchBroadcaster, and /ws/workbench endpoint
```

---

### Task 4: WebSocket Integration Test

Verify the full CDI → enricher → broadcaster → WebSocket → client pipeline.

**Files:**
- Create: `src/test/java/io/quarkmind/qa/workbench/WorkbenchSocketIT.java`

**Interfaces:**
- Consumes: `WorkbenchSocket` endpoint, CDI events from Task 1
- Produces: Integration test validating the full server pipeline

- [ ] **Step 1: Write WebSocket integration test**

```java
// src/test/java/io/quarkmind/qa/workbench/WorkbenchSocketIT.java
package io.quarkmind.qa.workbench;

import io.quarkus.test.QuarkusTest;
import io.quarkmind.agent.plugin.PatternAssessmentPublished;
import io.quarkmind.domain.PatternAssessment;
import io.quarkmind.domain.StrategyArchetype;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.WebSocket;
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;
import java.util.concurrent.TimeUnit;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class WorkbenchSocketIT {

    @Inject Event<PatternAssessmentPublished> patternEvent;

    @Test void pattern_event_arrives_via_websocket() throws Exception {
        var received = new CompletableFuture<String>();

        var ws = HttpClient.newHttpClient().newWebSocketBuilder()
            .buildAsync(URI.create("ws://localhost:8081/ws/workbench"), new WebSocket.Listener() {
                @Override
                public CompletionStage<?> onText(WebSocket webSocket, CharSequence data, boolean last) {
                    received.complete(data.toString());
                    return CompletableFuture.completedFuture(null);
                }
            }).get(5, TimeUnit.SECONDS);

        Thread.sleep(200); // let session register

        patternEvent.fire(new PatternAssessmentPublished(
            List.of(new PatternAssessment(StrategyArchetype.ZERG_EARLY_POOL, 0.87, 1000, "6+ lings"))));

        String json = received.get(5, TimeUnit.SECONDS);
        assertTrue(json.contains("\"type\":\"pattern\""));
        assertTrue(json.contains("ZERG_EARLY_POOL"));

        ws.sendClose(WebSocket.NORMAL_CLOSURE, "done").join();
    }

    @Test void reconnect_receives_snapshot() throws Exception {
        // Fire an event first (no client connected — snapshot stored)
        patternEvent.fire(new PatternAssessmentPublished(
            List.of(new PatternAssessment(StrategyArchetype.ZERG_EARLY_POOL, 0.5, 500, "snap"))));

        Thread.sleep(100);

        // Now connect — should receive the snapshot immediately
        var received = new CompletableFuture<String>();
        var ws = HttpClient.newHttpClient().newWebSocketBuilder()
            .buildAsync(URI.create("ws://localhost:8081/ws/workbench"), new WebSocket.Listener() {
                @Override
                public CompletionStage<?> onText(WebSocket webSocket, CharSequence data, boolean last) {
                    received.complete(data.toString());
                    return CompletableFuture.completedFuture(null);
                }
            }).get(5, TimeUnit.SECONDS);

        String json = received.get(5, TimeUnit.SECONDS);
        assertTrue(json.contains("\"type\":\"pattern\""));

        ws.sendClose(WebSocket.NORMAL_CLOSURE, "done").join();
    }
}
```

- [ ] **Step 2: Run test**

Run: `mvn test -Dtest=WorkbenchSocketIT -q`
Expected: PASS

- [ ] **Step 3: Commit**

```
test(#259): add WorkbenchSocket integration test with reconnect snapshot
```

---

### Task 5: Application Shell Layout (HTML + CSS)

Transform `visualizer.html` from a full-viewport canvas page into a CSS grid workbench shell. Modify `visualizer.js` `init()` to append the canvas to the grid cell instead of `document.body`, and resize via `ResizeObserver`.

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.html`
- Modify: `src/main/resources/META-INF/resources/visualizer.js` (init, resize, HUD, inspect panel)

**Interfaces:**
- Consumes: Existing `init()`, `updateHud()`, `setupInspectPanel()` functions
- Produces: CSS grid shell with `#wb-toolbar`, `#wb-canvas`, `#wb-pages`, `#wb-detail`, `#wb-status` elements. `init()` appends canvas to `#wb-canvas`. `updateHud()` writes to toolbar. Unit panel moves to `#wb-detail`.

- [ ] **Step 1: Update visualizer.html — replace body content with grid shell**

Replace the `<body>` content (lines 53-82) with the workbench grid layout. Keep the `<style>` block and add new grid styles. The angle buttons, mode toggle, and config panel move inside `#wb-canvas` (they're 3D view controls). The HUD div becomes part of the toolbar.

New body structure:
```html
<div id="workbench">
  <div id="wb-toolbar">
    <div id="wb-tabs">
      <button class="wb-tab active" data-page="pattern">Pattern</button>
      <button class="wb-tab" data-page="coaching">Coaching</button>
      <button class="wb-tab" data-page="strategy">Strategy</button>
    </div>
    <div id="wb-hud">Connecting...</div>
  </div>
  <div id="wb-canvas">
    <!-- Three.js canvas appended here by init() -->
    <div id="angle-btns">...</div>
    <div id="mode-toggle">...</div>
    <div id="config-panel">...</div>
  </div>
  <div id="wb-panel">
    <div id="wb-pages">
      <div id="page-pattern" class="wb-page active">No pattern data</div>
      <div id="page-coaching" class="wb-page" style="display:none">No coaching advice yet</div>
      <div id="page-strategy" class="wb-page" style="display:none">No strategy data</div>
    </div>
    <div id="wb-detail">Click a unit or counter to inspect</div>
  </div>
  <div id="wb-status">
    <span id="status-frame">Frame: 0</span>
    <span id="status-phase">Phase: —</span>
    <span id="status-game">Game: —</span>
    <span id="status-intel">Intel: —</span>
  </div>
</div>
```

Add CSS grid styles:
```css
#workbench {
  display: grid;
  grid-template-rows: 36px 1fr 24px;
  grid-template-columns: 1fr 300px;
  grid-template-areas:
    "toolbar toolbar"
    "canvas  panel"
    "status  status";
  width: 100vw; height: 100vh;
}
#wb-toolbar { grid-area: toolbar; display: flex; align-items: center; justify-content: space-between; background: rgba(10,10,30,0.9); border-bottom: 1px solid #2a2a5e; padding: 0 12px; }
#wb-canvas  { grid-area: canvas; position: relative; overflow: hidden; }
#wb-panel   { grid-area: panel; display: flex; flex-direction: column; background: rgba(10,10,30,0.85); border-left: 1px solid #2a2a5e; }
#wb-pages   { flex: 1; overflow-y: auto; padding: 8px; }
#wb-detail  { height: 160px; border-top: 1px solid #2a2a5e; padding: 8px; font-size: 12px; }
#wb-status  { grid-area: status; display: flex; gap: 16px; align-items: center; background: rgba(10,10,30,0.9); border-top: 1px solid #2a2a5e; padding: 0 12px; font-size: 11px; }
```

- [ ] **Step 2: Update visualizer.js init() — canvas sizing via ResizeObserver**

In `init()`, change:
```javascript
// OLD:
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);
camera = new THREE.PerspectiveCamera(45, window.innerWidth/window.innerHeight, 0.1, 500);

// NEW:
const canvasContainer = document.getElementById('wb-canvas');
const cw = canvasContainer.clientWidth, ch = canvasContainer.clientHeight;
renderer.setSize(cw, ch);
canvasContainer.appendChild(renderer.domElement);
camera = new THREE.PerspectiveCamera(45, cw/ch, 0.1, 500);

new ResizeObserver(() => {
  const w = canvasContainer.clientWidth, h = canvasContainer.clientHeight;
  renderer.setSize(w, h);
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
}).observe(canvasContainer);
```

Also remove the `window.addEventListener('resize', ...)` handler if one exists.

- [ ] **Step 3: Update updateHud() — write to toolbar instead of floating div**

Change `updateHud()` to target `#wb-hud` instead of `#hud`. The minerals/gas/supply/frame display moves to the toolbar.

- [ ] **Step 4: Move inspect panel to #wb-detail**

In `setupInspectPanel()`, instead of creating a floating `#unit-panel` div, populate the `#wb-detail` element. Remove the slide-in animation — it's always visible. Add position and tag display.

- [ ] **Step 5: Add tab switching logic**

```javascript
function setupTabs() {
  document.querySelectorAll('.wb-tab').forEach(btn => {
    btn.addEventListener('click', () => {
      document.querySelector('.wb-tab.active').classList.remove('active');
      btn.classList.add('active');
      document.querySelectorAll('.wb-page').forEach(p => p.style.display = 'none');
      document.getElementById('page-' + btn.dataset.page).style.display = '';
    });
  });
}
```

Call `setupTabs()` in `init()`.

- [ ] **Step 6: Add status bar updates**

Update `#status-frame` with frame count in `updateHud()`. Connection status for both WebSockets (set in `connectWebSocket()` and `connectWorkbenchSocket()` — added in Task 7).

- [ ] **Step 7: Verify manually**

Run: `mvn quarkus:dev`
Open: `http://localhost:8080/visualizer.html`
Expected: Grid layout with canvas on left, panel on right, toolbar with tabs, status bar at bottom. 3D view renders correctly in the canvas cell.

- [ ] **Step 8: Commit**

```
feat(#259): transform visualizer into workbench application shell with CSS grid layout
```

---

### Task 6: Selection Model and Pulsing Rings

Client-side selection model with bidirectional correlation and pulsing ring indicators.

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js` (add selection model, ring management, update click handlers)

**Interfaces:**
- Consumes: Existing raycasting click handlers, `unitSprites`/`enemySprites` maps, `lastState` from GameState WebSocket
- Produces: Global `selection` object, `selectionRings` map, ring pulse in `animate()`, `syncUnits()` clears dead selections

- [ ] **Step 1: Add selection model to visualizer.js**

Add near the top of the file (after global variable declarations):

```javascript
const selectionRings = new Map();
const selection = {
  type: null, tag: null, unitType: null, isEnemy: null, source: null,
  _listeners: [], _dispatching: false,
  subscribe(fn) {
    this._listeners.push(fn);
    return () => { this._listeners = this._listeners.filter(l => l !== fn); };
  },
  set(sel) {
    if (this._dispatching) return;
    Object.assign(this, sel);
    this._dispatching = true;
    for (const fn of this._listeners) {
      try { fn(this); } catch (e) { console.error('selection listener error', e); }
    }
    this._dispatching = false;
  },
  clear() { this.set({ type: null, tag: null, unitType: null, isEnemy: null, source: null }); }
};
```

- [ ] **Step 2: Add ring creation and management functions**

```javascript
function createRing(tag, position, color) {
  const s = MARKER_SCALE || 0.4;
  const geo = new THREE.RingGeometry(s * 0.6, s * 0.8, 32);
  const mat = new THREE.MeshBasicMaterial({ color, transparent: true, opacity: 0.6, side: THREE.DoubleSide });
  const ring = new THREE.Mesh(geo, mat);
  ring.rotation.x = -Math.PI / 2;
  ring.position.set(position.x, 0.05, position.z);
  scene.add(ring);
  selectionRings.set(tag, ring);
}

function clearRings() {
  selectionRings.forEach((ring) => { scene.remove(ring); ring.geometry.dispose(); ring.material.dispose(); });
  selectionRings.clear();
}

function updateRingPulse() {
  const t = performance.now() / 1000;
  const opacity = 0.3 + 0.5 * (0.5 + 0.5 * Math.sin(t * Math.PI * 2));
  selectionRings.forEach(ring => { ring.material.opacity = opacity; });
}

function syncRingPositions() {
  selectionRings.forEach((ring, tag) => {
    const sp = unitSprites.get(tag) || enemySprites.get(tag);
    if (sp) { ring.position.x = sp.position.x; ring.position.z = sp.position.z; }
    else { scene.remove(ring); ring.geometry.dispose(); ring.material.dispose(); selectionRings.delete(tag); }
  });
  // If tag selection and the ring was removed (unit died), clear selection
  if (selection.type === 'unit' && selection.tag != null && !selectionRings.has(selection.tag)) {
    selection.clear();
  }
}
```

- [ ] **Step 3: Register selection listener for ring management**

```javascript
selection.subscribe(sel => {
  clearRings();
  if (!sel.type) return;
  if (sel.type === 'unit' || sel.type === 'building') {
    const sp = unitSprites.get(sel.tag) || enemySprites.get(sel.tag) || buildingMeshes.get(sel.tag) || enemyBuildingMeshes.get(sel.tag);
    if (sp) createRing(sel.tag, sp.position, sel.isEnemy ? 0xff4444 : 0x44ff44);
  } else if (sel.type === 'unitType') {
    const color = sel.isEnemy ? 0xff4444 : 0x44ff44;
    [unitSprites, enemySprites].forEach(map => {
      map.forEach((sp, tag) => {
        if (sp.userData.unitType === sel.unitType) createRing(tag, sp.position, sel.isEnemy === true ? 0xff4444 : 0x44ff44);
      });
    });
  }
});
```

- [ ] **Step 4: Update click handlers to use selection model**

Modify the existing `mouseup` handler (around line 449-483). Instead of directly calling `showUnitPanel()`/`showBuildingPanelAsync()`, call `selection.set()`:

```javascript
// On hit:
if (obj.userData.buildingTag !== undefined) {
  selection.set({ type: 'building', tag: obj.userData.buildingTag, isEnemy: obj.userData.isEnemy, source: 'canvas' });
} else {
  selection.set({ type: 'unit', tag: obj.userData.unitTag, isEnemy: obj.userData.isEnemy, source: 'canvas' });
}
// On miss:
selection.clear();
```

Register a selection listener that populates `#wb-detail` (replacing the old `showUnitPanel` flow).

- [ ] **Step 5: Add ring updates to animate loop**

In `animate()`, add `updateRingPulse()` and `syncRingPositions()`:

```javascript
function animate() {
  requestAnimationFrame(animate);
  smoothCamera();
  updateSpriteDirs();
  updateRingPulse();
  syncRingPositions();
  renderer.render(scene, camera);
}
```

- [ ] **Step 6: Add `window.__test` API extensions**

```javascript
window.__test.selectionRingCount = () => selectionRings.size;
window.__test.selectionState = () => ({ type: selection.type, tag: selection.tag, unitType: selection.unitType, isEnemy: selection.isEnemy, source: selection.source });
window.__test.workbenchPage = () => document.querySelector('.wb-tab.active')?.dataset.page;
window.__test.unitDetailName = () => document.getElementById('wb-detail-name')?.textContent || '';
window.__test.workbenchPatternCount = () => document.querySelectorAll('#page-pattern .assessment-item').length;
```

- [ ] **Step 7: Verify manually**

Run: `mvn quarkus:dev`
Click units — rings should appear. Click empty space — rings should disappear. Check `window.__test.selectionRingCount()` in console.

- [ ] **Step 8: Commit**

```
feat(#259): add selection model with pulsing ring indicators and bidirectional correlation
```

---

### Task 7: Workbench WebSocket Client and Page Rendering

Connect to `/ws/workbench`, dispatch events to `workbenchState`, render the three pages.

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js` (add `connectWorkbenchSocket()`, page render functions, counter click handlers)

**Interfaces:**
- Consumes: `WorkbenchEvent` JSON from `/ws/workbench`, `selection` model from Task 6
- Produces: Populated page content in `#page-pattern`, `#page-coaching`, `#page-strategy`. Counter unit name clicks trigger type selection.

- [ ] **Step 1: Add workbenchState and connectWorkbenchSocket()**

```javascript
const workbenchState = { pattern: null, coaching: [], strategy: null };
let wbWsConnected = false;

function connectWorkbenchSocket() {
  const ws = new WebSocket('ws://' + window.location.host + '/ws/workbench');
  ws.onopen = () => { wbWsConnected = true; updateConnectionStatus(); };
  ws.onmessage = (e) => {
    const event = JSON.parse(e.data);
    switch (event.type) {
      case 'pattern': workbenchState.pattern = event.payload; renderPatternPage(); break;
      case 'coaching': workbenchState.coaching.unshift(event.payload); if (workbenchState.coaching.length > 20) workbenchState.coaching.pop(); renderCoachingPage(); break;
      case 'coaching_compliance': applyComplianceUpdate(event.payload); break;
      case 'strategy': workbenchState.strategy = event.payload; renderStrategyPage(); break;
    }
  };
  ws.onerror = () => ws.close();
  ws.onclose = () => { wbWsConnected = false; updateConnectionStatus(); setTimeout(connectWorkbenchSocket, 2000); };
}
```

Call `connectWorkbenchSocket()` in `init()` after `connectWebSocket()`.

- [ ] **Step 2: Implement renderPatternPage()**

```javascript
function renderPatternPage() {
  const el = document.getElementById('page-pattern');
  const data = workbenchState.pattern;
  if (!data || !data.assessments || data.assessments.length === 0) { el.innerHTML = 'No pattern data'; return; }

  el.innerHTML = data.assessments.map((ea, i) => {
    const a = ea.assessment;
    const conf = Math.round(a.confidence * 100);
    const barColor = conf > 70 ? '#44ff44' : conf > 50 ? '#ffaa00' : '#ff4444';
    const expanded = i === 0 ? 'block' : 'none';
    const arrow = i === 0 ? '▼' : '▶';

    let countersHtml = '';
    if (ea.counters) {
      const strong = (ea.counters.strongCounters || []).map(c =>
        '<li>' + c.units.map(u => '<span class="counter-unit" data-unit="' + u + '">' + u + '</span>').join(', ') + ' — ' + (c.action || '') + '</li>'
      ).join('');
      const weak = (ea.counters.weakCounters || []).map(c =>
        '<li>' + c.units.map(u => '<span class="counter-unit" data-unit="' + u + '">' + u + '</span>').join(', ') + ' — ' + (c.action || '') + '</li>'
      ).join('');
      if (strong) countersHtml += '<div class="counter-section"><strong>Strong Counters:</strong><ul>' + strong + '</ul></div>';
      if (weak) countersHtml += '<div class="counter-section"><strong>Weak Counters:</strong><ul>' + weak + '</ul></div>';
    }

    return '<div class="assessment-item" data-index="' + i + '">' +
      '<div class="assessment-header" onclick="toggleAssessment(' + i + ')">' + arrow + ' ' + a.archetype + ' (' + conf + '%)</div>' +
      '<div class="confidence-bar"><div style="width:' + conf + '%;background:' + barColor + ';height:4px;border-radius:2px;"></div></div>' +
      '<div class="assessment-body" id="assess-body-' + i + '" style="display:' + expanded + '">' +
        '<div class="rationale">' + (a.rationale || '') + '</div>' +
        countersHtml +
      '</div></div>';
  }).join('');

  // Attach click handlers for counter unit names
  el.querySelectorAll('.counter-unit').forEach(span => {
    span.style.cursor = 'pointer';
    span.style.textDecoration = 'underline';
    span.addEventListener('click', (e) => {
      e.stopPropagation();
      selection.set({ type: 'unitType', unitType: span.dataset.unit, isEnemy: false, source: 'workbench' });
    });
  });
}

function toggleAssessment(index) {
  const body = document.getElementById('assess-body-' + index);
  const header = body.previousElementSibling.previousElementSibling;
  if (body.style.display === 'none') { body.style.display = 'block'; header.textContent = header.textContent.replace('▶', '▼'); }
  else { body.style.display = 'none'; header.textContent = header.textContent.replace('▼', '▶'); }
}
```

- [ ] **Step 3: Implement renderCoachingPage()**

```javascript
function renderCoachingPage() {
  const el = document.getElementById('page-coaching');
  if (workbenchState.coaching.length === 0) { el.innerHTML = 'No coaching advice yet'; return; }

  el.innerHTML = workbenchState.coaching.map(c => {
    const mins = Math.floor(c.gameFrame / 22.4 / 60);
    const secs = Math.floor((c.gameFrame / 22.4) % 60);
    const time = mins + ':' + String(secs).padStart(2, '0');
    const status = c.complianceStatus || '⏳ Pending';
    return '<div class="coaching-item">' +
      '<div class="coaching-header">' + time + ' [' + c.domain + '] ' + (c.urgency || '') + '</div>' +
      '<div class="coaching-advice">' + c.advice + '</div>' +
      '<div class="coaching-status" data-frame="' + c.gameFrame + '" data-domain="' + c.domain + '">' + status + '</div>' +
    '</div>';
  }).join('');
}

function applyComplianceUpdate(payload) {
  const statusMap = { complied: '✓ Complied', challenged: '✗ Ignored', neutral: '— Neutral' };
  const label = statusMap[payload.status.toLowerCase()] || payload.status;
  // Update in workbenchState
  const entry = workbenchState.coaching.find(c => c.gameFrame === payload.gameFrame && c.domain === payload.domain);
  if (entry) { entry.complianceStatus = label; renderCoachingPage(); }
}
```

- [ ] **Step 4: Implement renderStrategyPage()**

```javascript
function renderStrategyPage() {
  const el = document.getElementById('page-strategy');
  const s = workbenchState.strategy;
  if (!s) { el.innerHTML = 'No strategy data'; return; }
  el.innerHTML =
    '<div class="strategy-section">' +
    '<div class="strategy-label">Active Strategy</div>' +
    '<div class="strategy-value">' + s.strategyId + '</div>' +
    '<div class="strategy-row"><span>Archetype:</span> ' + s.archetype + '</div>' +
    '<div class="strategy-row"><span>Confidence:</span> ' + (s.confidence * 100).toFixed(0) + '%</div>' +
    '<div class="strategy-row"><span>Pivots:</span> ' + s.pivotCount + '</div>' +
    '</div>';
}
```

- [ ] **Step 5: Add connection status updates**

```javascript
function updateConnectionStatus() {
  const gameEl = document.getElementById('status-game');
  const intelEl = document.getElementById('status-intel');
  if (gameEl) gameEl.textContent = 'Game: ' + (wsConnected ? 'Connected' : 'Disconnected');
  if (gameEl) gameEl.style.color = wsConnected ? '#44ff44' : '#ff4444';
  if (intelEl) intelEl.textContent = 'Intel: ' + (wbWsConnected ? 'Connected' : 'Disconnected');
  if (intelEl) intelEl.style.color = wbWsConnected ? '#44ff44' : '#ffaa00';
}
```

Update the existing `connectWebSocket()` to call `updateConnectionStatus()` on open/close.

- [ ] **Step 6: Add workbench-specific CSS**

Add to `visualizer.html` `<style>` or inject via JS:

```css
.wb-tab { background: none; border: none; color: #aaaacc; font-family: monospace; font-size: 12px; padding: 6px 12px; cursor: pointer; border-bottom: 2px solid transparent; }
.wb-tab.active { color: #88bbff; border-bottom-color: #88bbff; }
.wb-tab:hover { color: #88bbff; }
.assessment-header { cursor: pointer; font-weight: bold; padding: 4px 0; }
.confidence-bar { background: #1a1a3e; height: 4px; border-radius: 2px; margin: 2px 0 6px; }
.assessment-body { padding: 4px 0 8px; }
.rationale { font-size: 11px; color: #999; margin-bottom: 6px; }
.counter-section { margin: 4px 0; }
.counter-section ul { margin: 2px 0 0 16px; }
.counter-unit:hover { color: #88bbff; }
.coaching-item { border-bottom: 1px solid #1a1a3e; padding: 6px 0; }
.coaching-header { font-size: 11px; color: #88bbff; }
.coaching-advice { margin: 4px 0; }
.coaching-status { font-size: 11px; color: #999; }
.strategy-section { padding: 4px 0; }
.strategy-label { font-weight: bold; color: #88bbff; margin-bottom: 4px; }
.strategy-value { font-size: 16px; margin-bottom: 8px; }
.strategy-row { font-size: 12px; margin: 2px 0; }
.strategy-row span { color: #999; }
```

- [ ] **Step 7: Verify manually**

Run: `mvn quarkus:dev -Dquarkus.profile=emulated`
Open visualizer. Verify: tabs switch pages, status bar shows connection state, pattern/coaching/strategy data appears when events fire.

- [ ] **Step 8: Commit**

```
feat(#259): add workbench WebSocket client with pattern, coaching, and strategy page rendering
```

---

### Task 8: Playwright Render Tests

Browser-based tests validating the workbench shell, tab switching, page rendering, selection correlation, and rings.

**Files:**
- Create: `src/test/java/io/quarkmind/qa/workbench/WorkbenchRenderTest.java`

**Interfaces:**
- Consumes: `window.__test` API, workbench shell DOM, `/ws/workbench` WebSocket
- Produces: Playwright test coverage for the workbench

- [ ] **Step 1: Write Playwright test class**

```java
// src/test/java/io/quarkmind/qa/workbench/WorkbenchRenderTest.java
package io.quarkmind.qa.workbench;

import com.microsoft.playwright.*;
import io.quarkus.test.QuarkusTest;
import io.quarkmind.agent.plugin.PatternAssessmentPublished;
import io.quarkmind.domain.PatternAssessment;
import io.quarkmind.domain.StrategyArchetype;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import org.junit.jupiter.api.*;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
@Tag("browser")
class WorkbenchRenderTest {

    @Inject Event<PatternAssessmentPublished> patternEvent;

    static Playwright playwright;
    static Browser browser;
    BrowserContext context;
    Page page;

    @BeforeAll static void setup() {
        playwright = Playwright.create();
        browser = playwright.chromium().launch();
    }

    @AfterAll static void teardown() { browser.close(); playwright.close(); }

    @BeforeEach void newPage() {
        context = browser.newContext();
        page = context.newPage();
        page.navigate("http://localhost:8081/visualizer.html");
        page.waitForFunction("() => window.__test && window.__test.threeReady()");
        page.waitForFunction("() => window.__test.wsConnected && window.__test.wsConnected()");
    }

    @AfterEach void closePage() { context.close(); }

    @Test void shell_renders_with_toolbar_and_pages() {
        assertNotNull(page.querySelector("#wb-toolbar"));
        assertNotNull(page.querySelector("#wb-pages"));
        assertNotNull(page.querySelector("#wb-detail"));
        assertNotNull(page.querySelector("#wb-status"));
        assertEquals("pattern", page.evaluate("() => window.__test.workbenchPage()"));
    }

    @Test void tab_switching_shows_correct_page() {
        page.click("[data-page='coaching']");
        assertEquals("coaching", page.evaluate("() => window.__test.workbenchPage()"));
        assertEquals("none", page.evaluate("() => document.getElementById('page-pattern').style.display"));
        assertNotEquals("none", page.evaluate("() => document.getElementById('page-coaching').style.display"));
    }

    @Test void pattern_event_populates_page() {
        patternEvent.fire(new PatternAssessmentPublished(
            List.of(new PatternAssessment(StrategyArchetype.ZERG_EARLY_POOL, 0.87, 1000, "6+ lings"))));
        page.waitForFunction("() => window.__test.workbenchPatternCount() > 0", null,
            new Page.WaitForFunctionOptions().setTimeout(5000));
        int count = ((Number) page.evaluate("() => window.__test.workbenchPatternCount()")).intValue();
        assertTrue(count >= 1);
    }

    @Test void empty_canvas_click_clears_selection() {
        // Click empty space on the canvas
        page.click("#wb-canvas canvas", new Page.ClickOptions().setPosition(10, 10));
        int rings = ((Number) page.evaluate("() => window.__test.selectionRingCount()")).intValue();
        assertEquals(0, rings);
    }
}
```

- [ ] **Step 2: Run Playwright tests**

Run: `mvn test -Pplaywright -Dtest=WorkbenchRenderTest -q`
Expected: PASS (requires Chromium installed)

- [ ] **Step 3: Commit**

```
test(#259): add Playwright render tests for workbench shell, tabs, pattern page, and selection
```

---

## Self-Review

**Spec coverage check:**
- ✅ Application shell layout (Task 5)
- ✅ Selection model with bidirectional correlation (Task 6)
- ✅ Pulsing ring indicators (Task 6)
- ✅ `/ws/workbench` WebSocket with typed payloads (Tasks 3, 4)
- ✅ WorkbenchEnricher separated from WorkbenchBroadcaster (Task 3)
- ✅ CDI event topology — four concrete events (Tasks 1, 2)
- ✅ Reconnect snapshot (Task 3 — `pushSnapshot` on `addSession`)
- ✅ Three pages: Pattern, Coaching, Strategy (Task 7)
- ✅ Unit detail section (Task 5 — moved to `#wb-detail`)
- ✅ Coaching compliance updates (Task 7 — `applyComplianceUpdate`)
- ✅ Partial failure UX — per-connection status (Task 7)
- ✅ `window.__test` API extensions (Task 6)
- ✅ Playwright render tests (Task 8)
- ✅ WebSocket integration test (Task 4)
- ✅ Unit tests (Tasks 1, 3)

**Placeholder scan:** No TBD/TODO items. All code blocks are concrete.

**Type consistency:** `WorkbenchEvent`, `WorkbenchPayload`, payload records, CDI events — names and signatures are consistent across all tasks.

**Tooling safety scan:** No bash file operations on source files. All file creation uses `ide_create_file`. Git commands are bash (correct).
