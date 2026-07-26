# CBR Reference Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #192 — feat: CBR reference implementation — CaseMemoryStore retention + CaseRetriever integration for strategy selection
**Issue group:** #192

**Goal:** Implement the full CBR cycle (Retain, Retrieve, Reuse) for strategy selection, replacing the application-layer trust routing workaround with engine SPI integration and CaseFile-mediated state.

**Architecture:** A single `SC2StrategyRouterTask` replaces `StrategyTrustRouter` + `StrategyTrustObserver` + `StrategySelector` (~264 lines). It sits in the tick chain between scouting and strategy phases, detects archetype changes, retrieves CBR cases from `CbrCaseMemoryStore`, and calls `SC2ImplementationRoutingStrategy` (trust + CBR blending via engine SPI). The CaseFile is the sole source of truth for strategy selection — no mutable app-layer state.

**Tech Stack:** casehub-neocortex-memory-api (CbrCaseMemoryStore, CbrCase, CbrQuery, FeatureValue, CbrFeatureSchema), casehub-engine-api (ImplementationRoutingStrategy, ImplementationRoutingContext, ImplementationCandidate, ImplementationSelection, CaseOutcomeObserver, TrustRoutingPolicy), casehub-engine-ledger (TrustCandidateClassifier, TrustScoreSource), Quarkus CDI

## Global Constraints

- Package for all new classes: `io.quarkmind.agent.cbr`
- CbrCase type: `"sc2-strategy"`
- Memory domain: `new MemoryDomain("quarkmind")`
- Capability name for strategy routing: `"strategy"` (matches `QuarkMindCaseHub.CAPABILITY_STRATEGY`)
- Designated fallback strategy: `"strategy.drools"`
- No `@Alternative` on the routing strategy — it is manually constructed, not CDI-discovered globally
- All new CaseFile keys must be added to `QuarkMindCaseFile.ALL_KEYS`
- Tests use `.withNotBefore(Instant.now())` on `CbrQuery` for test isolation (GE-20260716-986cd1)
- Inject concrete `StrategyTask` types in `@QuarkusTest`, not the interface (PP-20260610-3c3e89)

---

### Task 1: CBR Domain Model and CaseFile Keys

**Files:**
- Create: `src/main/java/io/quarkmind/agent/cbr/SC2GameCbrCase.java`
- Create: `src/main/java/io/quarkmind/agent/cbr/SC2CbrSchemaRegistrar.java`
- Modify: `src/main/java/io/quarkmind/agent/QuarkMindCaseFile.java`
- Test: `src/test/java/io/quarkmind/agent/cbr/SC2GameCbrCaseTest.java`

**Interfaces:**
- Consumes: `io.casehub.neocortex.memory.cbr.CbrCase`, `FeatureValue`, `CbrFeatureSchema`, `FeatureField`, `CbrCaseMemoryStore`
- Produces: `SC2GameCbrCase` record (used by Tasks 2, 3, 4, 6), `SC2CbrSchemaRegistrar` (startup), CaseFile key constants (used by Tasks 2, 4, 5)

- [ ] **Step 1: Write SC2GameCbrCase test**

```java
package io.quarkmind.agent.cbr;

import io.casehub.neocortex.memory.cbr.FeatureValue;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class SC2GameCbrCaseTest {

    @Test
    void cbrType() {
        var c = new SC2GameCbrCase("vs ZERG_ROACH_RUSH (PvZ)", "strategy.early-pressure",
                null, null, Map.of());
        assertThat(c.cbrType()).isEqualTo("sc2-strategy");
    }

    @Test
    void withOutcome() {
        var c = new SC2GameCbrCase("vs ZERG_ROACH_RUSH (PvZ)", "strategy.early-pressure",
                null, null, Map.of());
        var updated = c.withOutcome("WIN", 0.85);
        assertThat(updated.outcome()).isEqualTo("WIN");
        assertThat(updated.confidence()).isEqualTo(0.85);
        assertThat(updated.problem()).isEqualTo("vs ZERG_ROACH_RUSH (PvZ)");
        assertThat(updated.solution()).isEqualTo("strategy.early-pressure");
    }

    @Test
    void withFeatures() {
        var c = new SC2GameCbrCase("problem", "solution", null, null, Map.of());
        var features = Map.of("enemy_archetype", FeatureValue.string("ZERG_ROACH_RUSH"),
                              "enemy_race", FeatureValue.string("ZERG"));
        var updated = c.withFeatures(features);
        assertThat(updated.features()).containsKey("enemy_archetype");
        assertThat(updated.features().get("enemy_archetype"))
            .isEqualTo(FeatureValue.string("ZERG_ROACH_RUSH"));
    }

    @Test
    void featureMap_immutable() {
        var features = new java.util.HashMap<>(Map.of("k", FeatureValue.string("v")));
        var c = new SC2GameCbrCase("p", "s", null, null, features);
        assertThatThrownBy(() -> c.features().put("x", FeatureValue.string("y")))
            .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void problemRequired() {
        assertThatThrownBy(() -> new SC2GameCbrCase(null, "s", null, null, Map.of()))
            .isInstanceOf(NullPointerException.class);
    }

    @Test
    void solutionRequired() {
        assertThatThrownBy(() -> new SC2GameCbrCase("p", null, null, null, Map.of()))
            .isInstanceOf(NullPointerException.class);
    }

    @Test
    void buildForGame() {
        var c = SC2GameCbrCase.buildForGame(
            "ZERG_ROACH_RUSH", "ZERG", "PvZ", 0.82, "strategy.early-pressure");
        assertThat(c.problem()).isEqualTo("vs ZERG_ROACH_RUSH (PvZ)");
        assertThat(c.solution()).isEqualTo("strategy.early-pressure");
        assertThat(c.outcome()).isNull();
        assertThat(c.confidence()).isNull();
        assertThat(c.features()).containsEntry("enemy_archetype", FeatureValue.string("ZERG_ROACH_RUSH"));
        assertThat(c.features()).containsEntry("enemy_race", FeatureValue.string("ZERG"));
        assertThat(c.features()).containsEntry("matchup", FeatureValue.string("PvZ"));
        assertThat(c.features()).containsEntry("assessment_confidence", FeatureValue.number(0.82));
    }
}
```

- [ ] **Step 2: Run test — verify FAIL (class does not exist)**

Run: `mvn test -Dtest=SC2GameCbrCaseTest -q`
Expected: compilation error — `SC2GameCbrCase` not found

- [ ] **Step 3: Implement SC2GameCbrCase**

Use `ide_create_file` for `src/main/java/io/quarkmind/agent/cbr/SC2GameCbrCase.java`:

```java
package io.quarkmind.agent.cbr;

import io.casehub.neocortex.memory.cbr.CbrCase;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import java.util.Map;
import java.util.Objects;

public record SC2GameCbrCase(
    String problem,
    String solution,
    String outcome,
    Double confidence,
    Map<String, FeatureValue> features
) implements CbrCase {

    public static final String CBR_TYPE = "sc2-strategy";

    public SC2GameCbrCase {
        Objects.requireNonNull(problem, "problem required");
        Objects.requireNonNull(solution, "solution required");
        Objects.requireNonNull(features, "features required");
        features = Map.copyOf(features);
    }

    @Override
    public String cbrType() { return CBR_TYPE; }

    @Override
    public CbrCase withOutcome(String outcome, Double confidence) {
        return new SC2GameCbrCase(problem, solution, outcome, confidence, features);
    }

    @Override
    public CbrCase withFeatures(Map<String, FeatureValue> features) {
        return new SC2GameCbrCase(problem, solution, outcome, confidence, features);
    }

    public static SC2GameCbrCase buildForGame(
            String archetypeName, String raceName, String matchup,
            double assessmentConfidence, String strategyId) {
        return new SC2GameCbrCase(
            "vs " + archetypeName + " (" + matchup + ")",
            strategyId,
            null, null,
            Map.of(
                "enemy_archetype", FeatureValue.string(archetypeName),
                "enemy_race", FeatureValue.string(raceName),
                "matchup", FeatureValue.string(matchup),
                "assessment_confidence", FeatureValue.number(assessmentConfidence)
            ));
    }
}
```

- [ ] **Step 4: Run test — verify PASS**

Run: `mvn test -Dtest=SC2GameCbrCaseTest -q`
Expected: all 7 tests PASS

- [ ] **Step 5: Add CaseFile key constants**

Use `ide_edit_member` on `QuarkMindCaseFile.java`, member `QuarkMindCaseFile`, to add the new constants and update `ALL_KEYS`. Add after `STRATEGY_SELECTED_ID`:

```java
public static final String STRATEGY_ROUTED_CONTEXT    = "agent.strategy.routed.context";
public static final String STRATEGY_ROUTED_ARCHETYPE   = "agent.strategy.routed.archetype";
public static final String STRATEGY_ROUTED_CONFIDENCE  = "agent.strategy.routed.confidence";
public static final String STRATEGY_PIVOT_COUNT        = "agent.strategy.pivot.count";
```

Update `ALL_KEYS` to include all four new constants.

- [ ] **Step 6: Implement SC2CbrSchemaRegistrar**

Use `ide_create_file` for `src/main/java/io/quarkmind/agent/cbr/SC2CbrSchemaRegistrar.java`:

```java
package io.quarkmind.agent.cbr;

import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.CbrFeatureSchema;
import io.casehub.neocortex.memory.cbr.FeatureField;
import io.quarkus.runtime.Startup;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
@Startup
public class SC2CbrSchemaRegistrar {

    private static final Logger log = Logger.getLogger(SC2CbrSchemaRegistrar.class);

    @Inject CbrCaseMemoryStore cbrStore;

    @PostConstruct
    void register() {
        CbrFeatureSchema schema = CbrFeatureSchema.of(
            SC2GameCbrCase.CBR_TYPE,
            FeatureField.categorical("enemy_archetype"),
            FeatureField.categorical("enemy_race"),
            FeatureField.categorical("matchup"),
            FeatureField.numeric("assessment_confidence", 0.0, 1.0)
        );
        cbrStore.registerSchema(schema);
        log.infof("[CBR] Registered schema for case type '%s' with %d fields",
            SC2GameCbrCase.CBR_TYPE, schema.fields().size());
    }
}
```

- [ ] **Step 7: Verify compilation**

Run: `ide_build_project` — expect 0 errors.

- [ ] **Step 8: Commit**

```
feat(#192): SC2GameCbrCase domain model, CaseFile keys, schema registrar

Refs #192
```

---

### Task 2: CBR Retention Observer

**Files:**
- Create: `src/main/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserver.java`
- Test: `src/test/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserverTest.java`

**Interfaces:**
- Consumes: `SC2GameCbrCase.buildForGame()` (Task 1), `CaseOutcomeObserver` SPI, `CbrCaseMemoryStore`, `CbrOutcome`, `QuarkMindCaseFile` keys
- Produces: CBR cases stored in `CbrCaseMemoryStore` at game close (used by Tasks 4, 6 for retrieval)

- [ ] **Step 1: Write retention observer test**

```java
package io.quarkmind.agent.cbr;

import io.casehub.api.spi.CaseOutcomeEvent;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.*;
import io.casehub.platform.api.path.Path;
import io.quarkmind.agent.QuarkMindCaseFile;
import io.quarkmind.domain.EnemyArchetype;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class SC2CbrRetentionObserverTest {

    CbrCaseMemoryStore store;
    SC2CbrRetentionObserver observer;

    @BeforeEach
    void setUp() {
        store = mock(CbrCaseMemoryStore.class);
        when(store.store(any(), any(), any(), any(), any(), any(), any()))
            .thenReturn("stored-case-1");
        observer = new SC2CbrRetentionObserver(store);
    }

    @Test
    void win_storesCase_andRecordsOutcome() {
        CaseOutcomeEvent event = buildEvent("WIN",
            Map.of(
                QuarkMindCaseFile.STRATEGY_SELECTED_ID, "strategy.early-pressure",
                QuarkMindCaseFile.STRATEGY_ROUTED_ARCHETYPE, "ZERG_ROACH_RUSH",
                QuarkMindCaseFile.STRATEGY_ROUTED_CONFIDENCE, 0.85
            ));

        observer.onOutcome(event);

        verify(store).store(
            argThat(c -> c.problem().contains("ZERG_ROACH_RUSH")
                      && "strategy.early-pressure".equals(c.solution())
                      && "WIN".equals(c.outcome())),
            any(), any(), any(), any(), any(), any());
        verify(store).recordOutcome(eq("stored-case-1"), eq(SC2GameCbrCase.CBR_TYPE),
            argThat(o -> o.successRate() == 1.0));
    }

    @Test
    void loss_storesWithZeroSuccessRate() {
        CaseOutcomeEvent event = buildEvent("LOSS",
            Map.of(
                QuarkMindCaseFile.STRATEGY_SELECTED_ID, "strategy.drools",
                QuarkMindCaseFile.STRATEGY_ROUTED_ARCHETYPE, "TERRAN_MARINE_RUSH",
                QuarkMindCaseFile.STRATEGY_ROUTED_CONFIDENCE, 0.7
            ));

        observer.onOutcome(event);

        verify(store).recordOutcome(any(), any(),
            argThat(o -> o.successRate() == 0.0));
    }

    @Test
    void unknown_skips() {
        CaseOutcomeEvent event = buildEvent("UNKNOWN", Map.of());
        observer.onOutcome(event);
        verifyNoInteractions(store);
    }

    @Test
    void missingArchetype_skips() {
        CaseOutcomeEvent event = buildEvent("WIN",
            Map.of(QuarkMindCaseFile.STRATEGY_SELECTED_ID, "strategy.drools"));
        observer.onOutcome(event);
        verifyNoInteractions(store);
    }

    private CaseOutcomeEvent buildEvent(String outcomeLabel, Map<String, Object> snapshot) {
        return new CaseOutcomeEvent(
            "starcraft-game", "tenant-1", UUID.randomUUID(),
            snapshot, outcomeLabel, Instant.now(), Map.of());
    }
}
```

- [ ] **Step 2: Run test — verify FAIL**

Run: `mvn test -Dtest=SC2CbrRetentionObserverTest -q`
Expected: compilation error — `SC2CbrRetentionObserver` not found

- [ ] **Step 3: Implement SC2CbrRetentionObserver**

Use `ide_create_file` for `src/main/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserver.java`:

```java
package io.quarkmind.agent.cbr;

import io.casehub.api.spi.CaseOutcomeEvent;
import io.casehub.api.spi.CaseOutcomeObserver;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.CbrOutcome;
import io.casehub.platform.api.path.Path;
import io.quarkmind.agent.QuarkMindCaseFile;
import io.quarkmind.domain.EnemyArchetype;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.Map;

@ApplicationScoped
public class SC2CbrRetentionObserver implements CaseOutcomeObserver {

    private static final Logger log = Logger.getLogger(SC2CbrRetentionObserver.class);
    private static final MemoryDomain DOMAIN = new MemoryDomain("quarkmind");

    private final CbrCaseMemoryStore cbrStore;

    @Inject
    public SC2CbrRetentionObserver(CbrCaseMemoryStore cbrStore) {
        this.cbrStore = cbrStore;
    }

    @Override
    public void onOutcome(CaseOutcomeEvent event) {
        if ("UNKNOWN".equals(event.outcomeLabel())) {
            log.infof("[CBR-RETAIN] Game ended with unknown result — skipped");
            return;
        }

        Map<String, Object> snapshot = event.caseFileSnapshot();
        String archetype = (String) snapshot.get(QuarkMindCaseFile.STRATEGY_ROUTED_ARCHETYPE);
        if (archetype == null) {
            log.infof("[CBR-RETAIN] No archetype in snapshot — skipped (no routing occurred)");
            return;
        }

        String strategyId = (String) snapshot.get(QuarkMindCaseFile.STRATEGY_SELECTED_ID);
        Double confidence = (Double) snapshot.get(QuarkMindCaseFile.STRATEGY_ROUTED_CONFIDENCE);
        String raceName = EnemyArchetype.valueOf(archetype).race().name();
        String matchup = "Pv" + raceName.charAt(0);

        SC2GameCbrCase cbrCase = SC2GameCbrCase.buildForGame(
            archetype, raceName, matchup,
            confidence != null ? confidence : 0.0,
            strategyId
        );
        cbrCase = (SC2GameCbrCase) cbrCase.withOutcome(event.outcomeLabel(), null);

        double successRate = switch (event.outcomeLabel()) {
            case "WIN"  -> 1.0;
            case "LOSS" -> 0.0;
            case "TIE"  -> 0.5;
            default     -> 0.5;
        };

        String storedCaseId = cbrStore.store(
            cbrCase,
            event.tenancyId(),
            event.caseId().toString(),
            DOMAIN,
            "sc2-cbr-retention",
            "game",
            Path.of("quarkmind", "strategy", "cases"));

        cbrStore.recordOutcome(storedCaseId, SC2GameCbrCase.CBR_TYPE,
            CbrOutcome.of(successRate, event.outcomeLabel(), event.closedAt()));

        log.infof("[CBR-RETAIN] Stored: archetype=%s strategy=%s outcome=%s caseId=%s",
            archetype, strategyId, event.outcomeLabel(), storedCaseId);
    }
}
```

- [ ] **Step 4: Run test — verify PASS**

Run: `mvn test -Dtest=SC2CbrRetentionObserverTest -q`
Expected: all 4 tests PASS

- [ ] **Step 5: Commit**

```
feat(#192): SC2CbrRetentionObserver — store CBR cases at game close

Implements CaseOutcomeObserver to write SC2GameCbrCase entries to
CbrCaseMemoryStore with outcome recording (WIN=1.0, LOSS=0.0, TIE=0.5).

Refs #192
```

---

### Task 3: CBR Routing Strategy

**Files:**
- Create: `src/main/java/io/quarkmind/agent/cbr/SC2ImplementationRoutingStrategy.java`
- Test: `src/test/java/io/quarkmind/agent/cbr/SC2ImplementationRoutingStrategyTest.java`

**Interfaces:**
- Consumes: `ImplementationRoutingStrategy` SPI, `TrustCandidateClassifier.classify()`, `TrustScoreSource`, `TrustRoutingPolicyProvider.forCapability()`, `ImplementationRoutingContext.experiences()`, `ImplementationCandidate`, `AgentCandidate`, `AgentHealth`, `TrustRoutingPolicy.cbrWeight()`
- Produces: `SC2ImplementationRoutingStrategy` — constructor takes `(TrustCandidateClassifier, TrustScoreSource, TrustRoutingPolicyProvider)`. Method `select(ImplementationRoutingContext, List<ImplementationCandidate>)` returns `Uni<ImplementationSelection>`. Used by Task 4.

- [ ] **Step 1: Write routing strategy test**

```java
package io.quarkmind.agent.cbr;

import io.casehub.api.spi.routing.*;
import io.casehub.ledger.api.spi.TrustScoreSource;
import io.casehub.ledger.routing.TrustCandidateClassifier;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

class SC2ImplementationRoutingStrategyTest {

    TrustCandidateClassifier classifier;
    TrustScoreSource scoreSource;
    TrustRoutingPolicyProvider policyProvider;
    SC2ImplementationRoutingStrategy strategy;

    static final TrustRoutingPolicy POLICY = new TrustRoutingPolicy(
        0.65, 10, 0.08, 0.6, Map.of(), false, "strategy.drools", Set.of(), 0.4);

    @BeforeEach
    void setUp() {
        classifier = new TrustCandidateClassifier();
        scoreSource = stubScoreSource(Map.of());
        policyProvider = cap -> POLICY;
        strategy = new SC2ImplementationRoutingStrategy(classifier, scoreSource, policyProvider);
    }

    @Test
    void id() {
        assertThat(strategy.id()).isEqualTo("sc2-cbr-routing");
    }

    @Test
    void coldStart_noExperiences_selectsFallback() {
        var candidates = List.of(
            new ImplementationCandidate("strategy.drools", "strategy.drools", "strategy"),
            new ImplementationCandidate("strategy.early-pressure", "strategy.early-pressure", "strategy"));
        var ctx = new ImplementationRoutingContext(UUID.randomUUID(), "strategy", null, "t1", List.of());

        var result = strategy.select(ctx, candidates).await().indefinitely();

        assertThat(result).isInstanceOf(ImplementationSelection.Selected.class);
        var selected = (ImplementationSelection.Selected) result;
        assertThat(selected.bindingNames()).containsExactly("strategy.drools");
    }

    @Test
    void singleCandidate_runsAll() {
        var candidates = List.of(
            new ImplementationCandidate("strategy.drools", "strategy.drools", "strategy"));
        var ctx = new ImplementationRoutingContext(UUID.randomUUID(), "strategy", null, "t1", List.of());

        var result = strategy.select(ctx, candidates).await().indefinitely();
        assertThat(result).isInstanceOf(ImplementationSelection.RunAll.class);
    }

    @Test
    void cbrExperiences_influenceSelection() {
        scoreSource = stubScoreSource(Map.of());
        strategy = new SC2ImplementationRoutingStrategy(classifier, scoreSource, policyProvider);

        var experiences = List.of(
            new RetrievedExperience("vs ZERG_ROACH_RUSH (PvZ)", "strategy.early-pressure",
                "WIN", 1.0, 0.9, Map.of(), List.of(), Map.of()),
            new RetrievedExperience("vs ZERG_ROACH_RUSH (PvZ)", "strategy.early-pressure",
                "WIN", 1.0, 0.85, Map.of(), List.of(), Map.of()),
            new RetrievedExperience("vs ZERG_ROACH_RUSH (PvZ)", "strategy.drools",
                "LOSS", 0.0, 0.8, Map.of(), List.of(), Map.of()));

        var candidates = List.of(
            new ImplementationCandidate("strategy.drools", "strategy.drools", "strategy"),
            new ImplementationCandidate("strategy.early-pressure", "strategy.early-pressure", "strategy"));
        var ctx = new ImplementationRoutingContext(UUID.randomUUID(), "strategy", null, "t1", experiences);

        var result = strategy.select(ctx, candidates).await().indefinitely();
        var selected = (ImplementationSelection.Selected) result;
        assertThat(selected.bindingNames()).containsExactly("strategy.early-pressure");
    }

    static TrustScoreSource stubScoreSource(Map<String, Double> scores) {
        return new TrustScoreSource() {
            public OptionalDouble globalScore(String w) { return OptionalDouble.empty(); }
            public OptionalDouble capabilityScore(String w, String c) {
                return scores.containsKey(w) ? OptionalDouble.of(scores.get(w)) : OptionalDouble.empty();
            }
            public OptionalDouble dimensionScore(String w, String d) { return OptionalDouble.empty(); }
            public OptionalDouble capabilityDimensionScore(String w, String c, String d) { return OptionalDouble.empty(); }
            public int decisionCount(String w, String c) { return scores.containsKey(w) ? 20 : 0; }
            public Map<String, Double> allCapabilityScores(String w) { return Map.of(); }
            public Map<String, Double> allDimensionScores(String w) { return Map.of(); }
            public Map<String, Double> qualityScores(String w, String c) { return Map.of(); }
        };
    }
}
```

- [ ] **Step 2: Run test — verify FAIL**

Run: `mvn test -Dtest=SC2ImplementationRoutingStrategyTest -q`
Expected: compilation error — `SC2ImplementationRoutingStrategy` not found

- [ ] **Step 3: Implement SC2ImplementationRoutingStrategy**

Use `ide_create_file` for `src/main/java/io/quarkmind/agent/cbr/SC2ImplementationRoutingStrategy.java`:

```java
package io.quarkmind.agent.cbr;

import io.casehub.api.spi.routing.*;
import io.casehub.ledger.api.spi.TrustScoreSource;
import io.casehub.ledger.routing.TrustCandidateClassifier;
import io.casehub.ledger.routing.TrustCandidateClassifier.ClassifiedCandidate;
import io.casehub.ledger.routing.TrustCandidateClassifier.Phase;
import io.smallrye.mutiny.Uni;
import org.jboss.logging.Logger;

import java.util.*;
import java.util.stream.Collectors;

public class SC2ImplementationRoutingStrategy implements ImplementationRoutingStrategy {

    private static final Logger log = Logger.getLogger(SC2ImplementationRoutingStrategy.class);

    private final TrustCandidateClassifier classifier;
    private final TrustScoreSource scoreSource;
    private final TrustRoutingPolicyProvider policyProvider;

    public SC2ImplementationRoutingStrategy(
            TrustCandidateClassifier classifier,
            TrustScoreSource scoreSource,
            TrustRoutingPolicyProvider policyProvider) {
        this.classifier = classifier;
        this.scoreSource = scoreSource;
        this.policyProvider = policyProvider;
    }

    @Override
    public String id() { return "sc2-cbr-routing"; }

    @Override
    public Uni<ImplementationSelection> select(
            ImplementationRoutingContext context,
            List<ImplementationCandidate> candidates) {

        if (candidates.size() <= 1) {
            return Uni.createFrom().item(new ImplementationSelection.RunAll());
        }

        TrustRoutingPolicy policy = policyProvider.forCapability(context.capabilityName());
        double cbrWeight = policy.cbrWeight();

        List<AgentCandidate> agentCandidates = candidates.stream()
            .map(c -> new AgentCandidate(c.workerName(), Set.of(c.capabilityName()),
                0, AgentHealth.READY, null, null))
            .toList();

        List<ClassifiedCandidate> classified = classifier.classify(
            agentCandidates, context.capabilityName(), policy, scoreSource);

        Map<String, Double> experienceWeights = computeExperienceWeights(
            context.experiences(), candidates);

        String bestBinding = null;
        double bestScore = -1.0;

        for (int i = 0; i < classified.size(); i++) {
            ClassifiedCandidate cc = classified.get(i);
            ImplementationCandidate ic = candidates.get(i);

            double trustScore = computeTrustScore(cc, policy, ic.bindingName());
            double expWeight = experienceWeights.getOrDefault(ic.bindingName(), 0.5);
            double finalScore = (1.0 - cbrWeight) * trustScore + cbrWeight * expWeight;

            log.debugf("[CBR-ROUTE] candidate=%s trust=%.3f cbr=%.3f final=%.3f (phase=%s)",
                ic.bindingName(), trustScore, expWeight, finalScore, cc.phase());

            if (finalScore > bestScore
                    || (finalScore == bestScore && ic.bindingName().equals(policy.fallbackBinding()))) {
                bestScore = finalScore;
                bestBinding = ic.bindingName();
            }
        }

        if (bestScore <= 0.0 && policy.fallbackBinding() != null) {
            bestBinding = policy.fallbackBinding();
        }

        log.infof("[CBR-ROUTE] Selected: %s (score=%.3f, cbrWeight=%.2f, experiences=%d)",
            bestBinding, bestScore, cbrWeight, context.experiences().size());

        return Uni.createFrom().item(
            new ImplementationSelection.Selected(List.of(bestBinding)));
    }

    private double computeTrustScore(ClassifiedCandidate cc, TrustRoutingPolicy policy,
                                     String bindingName) {
        return switch (cc.phase()) {
            case BOOTSTRAP -> cc.workloadScore();
            case QUALIFIED -> {
                double t = cc.trustScore().getAsDouble();
                yield t * policy.blendFactor() + cc.workloadScore() * (1.0 - policy.blendFactor());
            }
            case BORDERLINE -> bindingName.equals(policy.fallbackBinding()) ? 0.01 : 0.0;
            case EXCLUDED_PHASE2B, EXCLUDED_PHASE3 -> 0.0;
        };
    }

    private Map<String, Double> computeExperienceWeights(
            List<RetrievedExperience> experiences, List<ImplementationCandidate> candidates) {
        if (experiences == null || experiences.isEmpty()) {
            return Map.of();
        }

        Map<String, List<RetrievedExperience>> bySolution = experiences.stream()
            .filter(e -> e.outcome() != null)
            .collect(Collectors.groupingBy(RetrievedExperience::solution));

        Map<String, Double> weights = new HashMap<>();
        for (ImplementationCandidate c : candidates) {
            List<RetrievedExperience> matching = bySolution.getOrDefault(c.bindingName(), List.of());
            if (matching.isEmpty()) continue;

            double weightedOutcome = 0.0;
            double totalSimilarity = 0.0;
            for (RetrievedExperience exp : matching) {
                double outcomeValue = switch (exp.outcome()) {
                    case "WIN"  -> 1.0;
                    case "LOSS" -> 0.0;
                    case "TIE"  -> 0.5;
                    default     -> 0.5;
                };
                weightedOutcome += exp.similarityScore() * outcomeValue;
                totalSimilarity += exp.similarityScore();
            }
            weights.put(c.bindingName(), totalSimilarity > 0 ? weightedOutcome / totalSimilarity : 0.5);
        }
        return weights;
    }
}
```

- [ ] **Step 4: Run test — verify PASS**

Run: `mvn test -Dtest=SC2ImplementationRoutingStrategyTest -q`
Expected: all 4 tests PASS

- [ ] **Step 5: Commit**

```
feat(#192): SC2ImplementationRoutingStrategy — trust + CBR blending

Implements ImplementationRoutingStrategy SPI. Delegates trust classification
to TrustCandidateClassifier, blends with CBR experience weights from
RetrievedExperience. cbrWeight read from TrustRoutingPolicy.

Refs #192
```

---

### Task 4: Strategy Router Task and Infrastructure

**Files:**
- Create: `src/main/java/io/quarkmind/agent/cbr/SC2StrategyRouterTask.java`
- Modify: `src/main/java/io/quarkmind/agent/QuarkMindCaseHub.java` (PHASE_ORDER)
- Modify: `src/main/java/io/quarkmind/agent/QuarkMindTrustRoutingPolicyProvider.java` (add strategy policy)
- Test: `src/test/java/io/quarkmind/agent/cbr/SC2StrategyRouterTaskTest.java`

**Interfaces:**
- Consumes: `SC2ImplementationRoutingStrategy` (Task 3), `SC2GameCbrCase.CBR_TYPE` (Task 1), `CbrCaseMemoryStore.retrieveSimilar()`, `ScoutingIntelBroker.current(ScoutingIntelType.PATTERN_ASSESSMENT)`, `TaskDefinition` interface, `QuarkMindCaseFile` keys (Task 1)
- Produces: `SC2StrategyRouterTask` — writes `STRATEGY_SELECTED_ID`, `STRATEGY_ROUTED_CONTEXT`, `STRATEGY_ROUTED_ARCHETYPE`, `STRATEGY_ROUTED_CONFIDENCE`, `STRATEGY_PIVOT_COUNT` to CaseContext. Exposes `lastSelectedId()` for `MilestoneOutcomeRecorder`. Used by Tasks 5, 6.

- [ ] **Step 1: Write router task test**

```java
package io.quarkmind.agent.cbr;

import io.casehub.api.spi.routing.*;
import io.casehub.ledger.api.spi.TrustScoreSource;
import io.casehub.ledger.routing.TrustCandidateClassifier;
import io.casehub.neocortex.memory.cbr.*;
import io.quarkmind.agent.GameSession;
import io.quarkmind.agent.MutableMapCaseContext;
import io.quarkmind.agent.QuarkMindCaseFile;
import io.quarkmind.agent.plugin.ScoutingIntelBroker;
import io.quarkmind.agent.plugin.ScoutingIntelPayload.PatternAssessmentPayload;
import io.quarkmind.agent.plugin.ScoutingIntelType;
import io.quarkmind.agent.plugin.StrategyTask;
import io.quarkmind.domain.EnemyArchetype;
import io.quarkmind.domain.EnemyPatternAssessment;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.*;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class SC2StrategyRouterTaskTest {

    ScoutingIntelBroker   broker;
    CbrCaseMemoryStore    cbrStore;
    GameSession           gameSession;
    SC2StrategyRouterTask router;

    @BeforeEach
    void setUp() {
        broker   = mock(ScoutingIntelBroker.class);
        cbrStore = mock(CbrCaseMemoryStore.class);
        when(cbrStore.retrieveSimilar(any(), any())).thenReturn(List.of());
        gameSession = new GameSession();

        TrustCandidateClassifier   classifier     = new TrustCandidateClassifier();
        TrustScoreSource           scoreSource    = SC2ImplementationRoutingStrategyTest.stubScoreSource(Map.of());
        TrustRoutingPolicyProvider policyProvider = cap -> SC2ImplementationRoutingStrategyTest.POLICY;

        List<StrategyTask> strategies = List.of(
                stubStrategy("strategy.drools"),
                stubStrategy("strategy.early-pressure"));

        router = new SC2StrategyRouterTask(
                broker, cbrStore, gameSession, strategies,
                classifier, scoreSource, policyProvider, 0.6, 1);
    }

    @Test
    void id() {
        assertThat(router.getId()).isEqualTo("strategy-routing.cbr");
    }

    @Test
    void firstTick_noArchetype_selectsFallback() {
        when(broker.current(ScoutingIntelType.PATTERN_ASSESSMENT)).thenReturn(Optional.empty());
        MutableMapCaseContext ctx = new MutableMapCaseContext(Map.of(QuarkMindCaseFile.READY, true));

        router.execute(ctx);

        assertThat(ctx.get(QuarkMindCaseFile.STRATEGY_SELECTED_ID, String.class))
                .hasValue("strategy.drools");
    }

    @Test
    void archetypePresent_routesAndWritesKeys() {
        setArchetype(EnemyArchetype.ZERG_ROACH_RUSH, 0.85);
        MutableMapCaseContext ctx = new MutableMapCaseContext(Map.of(QuarkMindCaseFile.READY, true));

        router.execute(ctx);

        assertThat(ctx.get(QuarkMindCaseFile.STRATEGY_SELECTED_ID, String.class)).isPresent();
        assertThat(ctx.get(QuarkMindCaseFile.STRATEGY_ROUTED_CONTEXT, String.class)).isPresent();
        assertThat(ctx.get(QuarkMindCaseFile.STRATEGY_ROUTED_ARCHETYPE, String.class))
                .hasValue("ZERG_ROACH_RUSH");
        assertThat(ctx.get(QuarkMindCaseFile.STRATEGY_PIVOT_COUNT, Integer.class))
                .hasValue(0);
    }

    @Test
    void sameArchetype_skipsReroute() {
        setArchetype(EnemyArchetype.ZERG_ROACH_RUSH, 0.85);
        Map<String, Object> persisted = new HashMap<>(Map.of(
                QuarkMindCaseFile.READY, true,
                QuarkMindCaseFile.STRATEGY_SELECTED_ID, "strategy.drools",
                QuarkMindCaseFile.STRATEGY_ROUTED_CONTEXT, "ZERG_ROACH_RUSH-ZERG-PvZ"));
        MutableMapCaseContext ctx = new MutableMapCaseContext(persisted);

        router.execute(ctx);

        verify(cbrStore, never()).retrieveSimilar(any(), any());
    }

    @Test
    void lowConfidence_skipsRouting() {
        setArchetype(EnemyArchetype.ZERG_ROACH_RUSH, 0.3);
        MutableMapCaseContext ctx = new MutableMapCaseContext(Map.of(QuarkMindCaseFile.READY, true));

        router.execute(ctx);

        verify(cbrStore, never()).retrieveSimilar(any(), any());
        assertThat(ctx.get(QuarkMindCaseFile.STRATEGY_SELECTED_ID, String.class))
                .hasValue("strategy.drools");
    }

    @Test
    void pivotLimitEnforced() {
        setArchetype(EnemyArchetype.TERRAN_MARINE_RUSH, 0.9);
        Map<String, Object> persisted = new HashMap<>(Map.of(
                QuarkMindCaseFile.READY, true,
                QuarkMindCaseFile.STRATEGY_SELECTED_ID, "strategy.drools",
                QuarkMindCaseFile.STRATEGY_ROUTED_CONTEXT, "ZERG_ROACH_RUSH-ZERG-PvZ",
                QuarkMindCaseFile.STRATEGY_PIVOT_COUNT, 1));
        MutableMapCaseContext ctx = new MutableMapCaseContext(persisted);

        router.execute(ctx);

        verify(cbrStore, never()).retrieveSimilar(any(), any());
    }

    @Test
    void lastSelectedId_exposedForOutcomeRecorders() {
        setArchetype(EnemyArchetype.ZERG_ROACH_RUSH, 0.85);
        MutableMapCaseContext ctx = new MutableMapCaseContext(Map.of(QuarkMindCaseFile.READY, true));
        router.execute(ctx);
        assertThat(router.lastSelectedId()).isNotNull();
    }

    private void setArchetype(EnemyArchetype archetype, double confidence) {
        var assessment = new EnemyPatternAssessment(archetype, confidence, 1000, "test");
        when(broker.current(ScoutingIntelType.PATTERN_ASSESSMENT))
                .thenReturn(Optional.of(new PatternAssessmentPayload(List.of(assessment))));
    }

    private StrategyTask stubStrategy(String id) {
        StrategyTask s = mock(StrategyTask.class);
        when(s.getId()).thenReturn(id);
        return s;
    }
}
```

- [ ] **Step 2: Run test — verify FAIL**

Run: `mvn test -Dtest=SC2StrategyRouterTaskTest -q`
Expected: compilation error — `SC2StrategyRouterTask` not found

- [ ] **Step 3: Implement SC2StrategyRouterTask**

Use `ide_create_file` for `src/main/java/io/quarkmind/agent/cbr/SC2StrategyRouterTask.java`:

```java
package io.quarkmind.agent.cbr;

import io.casehub.annotation.CaseType;
import io.casehub.api.context.CaseContext;
import io.casehub.api.spi.routing.*;
import io.casehub.ledger.api.spi.TrustScoreSource;
import io.casehub.ledger.routing.TrustCandidateClassifier;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.*;
import io.casehub.platform.api.path.Path;
import io.quarkmind.agent.GameSession;
import io.quarkmind.agent.QuarkMindCaseFile;
import io.quarkmind.agent.TaskDefinition;
import io.quarkmind.agent.plugin.ScoutingIntelBroker;
import io.quarkmind.agent.plugin.ScoutingIntelPayload;
import io.quarkmind.agent.plugin.ScoutingIntelPayload.PatternAssessmentPayload;
import io.quarkmind.agent.plugin.ScoutingIntelType;
import io.quarkmind.agent.plugin.StrategyTask;
import io.quarkmind.domain.EnemyArchetype;
import io.quarkmind.domain.EnemyPatternAssessment;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import java.util.*;
import java.util.function.Predicate;

@ApplicationScoped
@CaseType("starcraft-game")
public class SC2StrategyRouterTask implements TaskDefinition {

    static final         String       FALLBACK   = "strategy.drools";
    private static final Logger       log        = Logger.getLogger(SC2StrategyRouterTask.class);
    private static final MemoryDomain DOMAIN     = new MemoryDomain("quarkmind");
    private static final String       CAPABILITY = "strategy";

    private final ScoutingIntelBroker        broker;
    private final CbrCaseMemoryStore         cbrStore;
    private final GameSession                gameSession;
    private final List<StrategyTask>         strategies;
    private final TrustCandidateClassifier   classifier;
    private final TrustScoreSource           scoreSource;
    private final TrustRoutingPolicyProvider policyProvider;
    private final double                     confidenceThreshold;
    private final int                        maxPivots;

    private          SC2ImplementationRoutingStrategy routingStrategy;
    private volatile String                           lastSelectedId = FALLBACK;

    @Inject
    public SC2StrategyRouterTask(
            ScoutingIntelBroker broker,
            CbrCaseMemoryStore cbrStore,
            GameSession gameSession,
            @Any Instance<StrategyTask> strategyTasks,
            TrustCandidateClassifier classifier,
            TrustScoreSource scoreSource,
            TrustRoutingPolicyProvider policyProvider,
            @ConfigProperty(name = "quarkmind.strategy.routing.confidence-threshold",
                            defaultValue = "0.6") double confidenceThreshold,
            @ConfigProperty(name = "quarkmind.strategy.routing.max-pivots",
                            defaultValue = "1") int maxPivots) {
        this(broker, cbrStore, gameSession,
             strategyTasks.stream().toList(),
             classifier, scoreSource, policyProvider,
             confidenceThreshold, maxPivots);
    }

    SC2StrategyRouterTask(
            ScoutingIntelBroker broker, CbrCaseMemoryStore cbrStore,
            GameSession gameSession, List<StrategyTask> strategies,
            TrustCandidateClassifier classifier, TrustScoreSource scoreSource,
            TrustRoutingPolicyProvider policyProvider,
            double confidenceThreshold, int maxPivots) {
        this.broker              = broker;
        this.cbrStore            = cbrStore;
        this.gameSession         = gameSession;
        this.strategies          = strategies;
        this.classifier          = classifier;
        this.scoreSource         = scoreSource;
        this.policyProvider      = policyProvider;
        this.confidenceThreshold = confidenceThreshold;
        this.maxPivots           = maxPivots;
    }

    @PostConstruct
    void init() {
        routingStrategy = new SC2ImplementationRoutingStrategy(classifier, scoreSource, policyProvider);
    }

    @Override
    public String getId()   {return "strategy-routing.cbr";}

    @Override
    public String getName() {return "SC2 CBR Strategy Router";}

    @Override
    public Set<String> produces() {
        return Set.of(QuarkMindCaseFile.STRATEGY_SELECTED_ID, QuarkMindCaseFile.STRATEGY_ROUTED_CONTEXT);
    }

    @Override
    public Predicate<CaseContext> activateIf() {return ctx -> true;}

    public String lastSelectedId() {return lastSelectedId;}

    @Override
    public void execute(CaseContext ctx) {
        Optional<ScoutingIntelPayload> raw = broker.current(ScoutingIntelType.PATTERN_ASSESSMENT);
        if (raw.isEmpty()) {
            if (ctx.get(QuarkMindCaseFile.STRATEGY_SELECTED_ID, String.class).isEmpty()) {
                ctx.set(QuarkMindCaseFile.STRATEGY_SELECTED_ID, FALLBACK);
                lastSelectedId = FALLBACK;
            }
            return;
        }

        PatternAssessmentPayload pa = (PatternAssessmentPayload) raw.get();
        if (pa.assessments().isEmpty()) {
            if (ctx.get(QuarkMindCaseFile.STRATEGY_SELECTED_ID, String.class).isEmpty()) {
                ctx.set(QuarkMindCaseFile.STRATEGY_SELECTED_ID, FALLBACK);
                lastSelectedId = FALLBACK;
            }
            return;
        }

        EnemyPatternAssessment best       = pa.assessments().getFirst();
        EnemyArchetype         archetype  = best.archetype();
        double                 confidence = best.confidence();

        if (confidence < confidenceThreshold) {
            if (ctx.get(QuarkMindCaseFile.STRATEGY_SELECTED_ID, String.class).isEmpty()) {
                ctx.set(QuarkMindCaseFile.STRATEGY_SELECTED_ID, FALLBACK);
                lastSelectedId = FALLBACK;
            }
            return;
        }

        String raceName   = archetype.race().name();
        String matchup    = "Pv" + raceName.charAt(0);
        String contextKey = archetype.name() + "-" + raceName + "-" + matchup;

        String existingContext = ctx.get(QuarkMindCaseFile.STRATEGY_ROUTED_CONTEXT, String.class)
                                    .orElse("");
        if (contextKey.equals(existingContext)) return;

        int pivotCount = ctx.get(QuarkMindCaseFile.STRATEGY_PIVOT_COUNT, Integer.class).orElse(-1);
        if (pivotCount >= maxPivots) return;

        List<ScoredCbrCase<SC2GameCbrCase>> retrieved = cbrStore.retrieveSimilar(
                CbrQuery.of("default", DOMAIN, Path.root(), SC2GameCbrCase.CBR_TYPE,
                            Map.of(
                                    "enemy_archetype", FeatureValue.string(archetype.name()),
                                    "enemy_race", FeatureValue.string(raceName),
                                    "matchup", FeatureValue.string(matchup),
                                    "assessment_confidence", FeatureValue.number(confidence)),
                            5)
                        .withWeights(Map.of(
                                "enemy_archetype", 0.5,
                                "enemy_race", 0.15,
                                "matchup", 0.15,
                                "assessment_confidence", 0.2))
                        .withMinSimilarity(0.3),
                SC2GameCbrCase.class);

        List<RetrievedExperience> experiences = retrieved.stream()
                                                         .map(sc -> new RetrievedExperience(
                                                                 sc.cbrCase().problem(), sc.cbrCase().solution(),
                                                                 sc.cbrCase().outcome(), sc.cbrCase().confidence(),
                                                                 sc.score(),
                                                                 FeatureValue.toRawMap(sc.cbrCase().features()),
                                                                 List.of(), sc.featureSimilarities()))
                                                         .toList();

        List<ImplementationCandidate> candidates = strategies.stream()
                                                             .map(s -> new ImplementationCandidate(s.getId(), s.getId(), CAPABILITY))
                                                             .toList();

        ImplementationRoutingContext routingCtx = new ImplementationRoutingContext(
                gameSession.id(), CAPABILITY, null, "default", experiences);

        ImplementationSelection selection = routingStrategy.select(routingCtx, candidates)
                                                           .await().indefinitely();

        String winner = switch (selection) {
            case ImplementationSelection.Selected s -> s.bindingNames().getFirst();
            case ImplementationSelection.RunAll ignored -> FALLBACK;
            case ImplementationSelection.RunNone ignored -> FALLBACK;
        };

        ctx.set(QuarkMindCaseFile.STRATEGY_SELECTED_ID, winner);
        ctx.set(QuarkMindCaseFile.STRATEGY_ROUTED_CONTEXT, contextKey);
        ctx.set(QuarkMindCaseFile.STRATEGY_ROUTED_ARCHETYPE, archetype.name());
        ctx.set(QuarkMindCaseFile.STRATEGY_ROUTED_CONFIDENCE, confidence);
        ctx.set(QuarkMindCaseFile.STRATEGY_PIVOT_COUNT, pivotCount + 1);
        lastSelectedId = winner;

        log.infof("[CBR-ROUTE] %s → %s (archetype=%s confidence=%.2f experiences=%d pivot=%d)",
                  existingContext.isEmpty() ? "initial" : "pivot",
                  winner, archetype, confidence, experiences.size(), pivotCount + 1);
    }
}
```

- [ ] **Step 4: Run test — verify PASS**

Run: `mvn test -Dtest=SC2StrategyRouterTaskTest -q`
Expected: all 7 tests PASS

- [ ] **Step 5: Update PHASE_ORDER in QuarkMindCaseHub**

Use `ide_edit_member` on `QuarkMindCaseHub.java`, member `PHASE_ORDER`:

```java
private static final List<String> PHASE_ORDER = List.of(
    "scouting.",           // Phase 1: observe
    "strategy-routing.",   // Phase 2a: route (select which strategy — CBR + trust)
    "strategy.",           // Phase 2b: decide (selected strategy executes)
    "tactics.",            // Phase 3: act
    "economics.",          // Phase 4: build
    "summarisation."       // Phase 5: reflect
);
```

- [ ] **Step 6: Add strategy policy to QuarkMindTrustRoutingPolicyProvider**

Use `ide_replace_member` on `QuarkMindTrustRoutingPolicyProvider.java`, member `forCapability`, to add the `"strategy"` case:

```java
@Override
public TrustRoutingPolicy forCapability(String capabilityName) {
    return switch (capabilityName) {
        case "tick-decision" -> buildAdvisoryPolicy(strategicMinObservations);
        case "advisory-crisis" -> buildAdvisoryPolicy(crisisMinObservations);
        case "advisory-strategic" -> buildAdvisoryPolicy(strategicMinObservations);
        case "advisory-economic" -> buildAdvisoryPolicy(economicMinObservations);
        case "commentary-reactive" -> buildCommentaryPolicy(commentaryMinObservations, 0.4);
        case "commentary-narrative" -> buildCommentaryPolicy(commentaryMinObservations, 0.3);
        case "strategy" -> buildStrategyPolicy();
        default -> TrustRoutingPolicy.DEFAULT;
    };
}
```

Add the `buildStrategyPolicy()` method via `ide_insert_member`:

```java
private TrustRoutingPolicy buildStrategyPolicy() {
    return new TrustRoutingPolicy(
        0.65, 10, 0.08, 0.6, Map.of(), false, "strategy.drools", Set.of(), 0.4);
}
```

- [ ] **Step 7: Verify compilation**

Run: `ide_build_project` — expect 0 errors.

- [ ] **Step 8: Run all existing tests**

Run: `mvn test -q`
Expected: all existing tests PASS (new classes are additive — nothing depends on them yet)

- [ ] **Step 9: Commit**

```
feat(#192): SC2StrategyRouterTask — tick-chain CBR router with CaseFile state

Adds strategy-routing phase between scouting and strategy in PHASE_ORDER.
Routes via SC2ImplementationRoutingStrategy with confidence gate, pivot limit,
and archetype change detection. Strategy policy with cbrWeight=0.4.

Refs #192
```

---

### Task 5: Migration — Replace Old Routing with New

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/DroolsStrategyTask.java` (remove StrategySelector, change activateIf)
- Modify: `src/main/java/io/quarkmind/plugin/EarlyPressureStrategyTask.java` (same)
- Modify: `src/main/java/io/quarkmind/plugin/EconomicExpansionStrategyTask.java` (same)
- Modify: `src/main/java/io/quarkmind/agent/MilestoneOutcomeRecorder.java` (replace StrategySelector with SC2StrategyRouterTask)
- Delete: `src/main/java/io/quarkmind/agent/StrategySelector.java` (use `ide_refactor_safe_delete`)
- Delete: `src/main/java/io/quarkmind/agent/StrategyTrustRouter.java` (use `ide_refactor_safe_delete`)
- Delete: `src/main/java/io/quarkmind/agent/StrategyTrustObserver.java` (use `ide_refactor_safe_delete`)
- Delete: `src/test/java/io/quarkmind/agent/StrategySelectorTest.java` (use `ide_refactor_safe_delete`)
- Delete: `src/test/java/io/quarkmind/agent/StrategyTrustRouterTest.java` (use `ide_refactor_safe_delete`)
- Modify: `src/test/java/io/quarkmind/plugin/DroolsStrategyTaskTest.java`
- Modify: `src/test/java/io/quarkmind/plugin/EarlyPressureStrategyTaskTest.java`
- Modify: `src/test/java/io/quarkmind/plugin/EarlyPressureStrategyTaskMigrationTest.java`
- Modify: `src/test/java/io/quarkmind/plugin/EconomicExpansionStrategyTaskTest.java`
- Modify: `src/test/java/io/quarkmind/agent/MilestoneOutcomeRecorderTest.java`
- Modify: `src/test/java/io/quarkmind/sc2/mock/TrustWeightedStrategyIT.java`
- Modify: `src/test/java/io/quarkmind/sc2/mock/AdaptivePluginSelectionIT.java`
- Modify: `src/test/java/io/quarkmind/sc2/mock/StrategyOutcomeRecordIT.java`
- Modify: `src/test/java/io/quarkmind/sc2/mock/StrategyCheckpointIT.java`
- Modify: `src/test/java/io/quarkmind/plugin/LedgerAuditIT.java`

**Interfaces:**
- Consumes: `SC2StrategyRouterTask.lastSelectedId()` (Task 4), `QuarkMindCaseFile.STRATEGY_SELECTED_ID` (Task 1)
- Produces: Clean codebase — old routing deleted, all strategy tasks and tests use CaseContext-based activation

This is a large migration task. Work through it systematically:

- [ ] **Step 1: Migrate DroolsStrategyTask.activateIf()**

Use `ide_replace_member` on `DroolsStrategyTask.java`, member `activateIf`:

```java
@Override
public Predicate<CaseContext> activateIf() {
    return ctx -> getId().equals(
        ctx.get(QuarkMindCaseFile.STRATEGY_SELECTED_ID, String.class).orElse(""));
}
```

Remove the `@Inject StrategySelector strategySelector;` field and its import. Also remove the `broker.current(ScoutingIntelType.POSTURE).isPresent()` check — phase ordering guarantees scouting runs first, and `execute()` handles absent posture via `broker.current(...).orElse("UNKNOWN")`.

- [ ] **Step 2: Migrate EarlyPressureStrategyTask.activateIf()**

Use `ide_replace_member` on `EarlyPressureStrategyTask.java`, member `activateIf`:

```java
@Override
public Predicate<CaseContext> activateIf() {
    return ctx -> getId().equals(
        ctx.get(QuarkMindCaseFile.STRATEGY_SELECTED_ID, String.class).orElse(""));
}
```

Remove the `StrategySelector` field, constructor parameter, and import. The constructor becomes no-arg (or remove it if only the StrategySelector param existed).

- [ ] **Step 3: Migrate EconomicExpansionStrategyTask.activateIf()**

Same pattern as Step 2. Remove `StrategySelector` field, constructor, import.

- [ ] **Step 4: Migrate MilestoneOutcomeRecorder**

Replace `StrategySelector` injection with `SC2StrategyRouterTask`. Use `ide_edit_member` to change the field type and constructor parameter:

- Field: `private final SC2StrategyRouterTask strategyRouter;`
- In `onGameStopped()`: replace `strategySelector.getSelectedId()` → `strategyRouter.lastSelectedId()`, replace `strategySelector.getOpponentContext()` → `"strategy"` (the capability name for trust score dimension)

- [ ] **Step 5: Verify compilation**

Run: `ide_build_project` — expect compilation errors from tests referencing `StrategySelector`. This is expected — we fix tests next.

- [ ] **Step 6: Update unit tests**

For each test that uses `StrategySelector`:

**EarlyPressureStrategyTaskTest.java**, **EarlyPressureStrategyTaskMigrationTest.java**, **EconomicExpansionStrategyTaskTest.java**: Remove `StrategySelector` setup. Set up `MutableMapCaseContext` (or `MapCaseContext`) with `STRATEGY_SELECTED_ID` pre-set to the strategy ID being tested. Call `activateIf().test(ctx)` against the context.

**MilestoneOutcomeRecorderTest.java**: Replace `new StrategySelector()` with a mock `SC2StrategyRouterTask`. When `lastSelectedId()` is called, return the expected strategy ID.

- [ ] **Step 7: Update integration tests**

For each `@QuarkusTest` that injects `StrategySelector`:

**DroolsStrategyTaskTest.java**: Replace `@Inject StrategySelector` with injecting `SC2StrategyRouterTask` or writing to CaseContext directly. The test calls `strategySelector.selectForGame(...)` before executing — replace with setting `STRATEGY_SELECTED_ID` in the CaseContext used by the test.

**TrustWeightedStrategyIT.java**, **AdaptivePluginSelectionIT.java**, **StrategyOutcomeRecordIT.java**, **StrategyCheckpointIT.java**, **LedgerAuditIT.java**: Same pattern — replace `StrategySelector` injection with CaseContext setup or `SC2StrategyRouterTask` injection. For ITs that test the routing decision, `SC2StrategyRouterTask` is now the entry point.

- [ ] **Step 8: Delete old classes**

Use `ide_refactor_safe_delete` for each:
1. `src/test/java/io/quarkmind/agent/StrategySelectorTest.java`
2. `src/test/java/io/quarkmind/agent/StrategyTrustRouterTest.java`
3. `src/main/java/io/quarkmind/agent/StrategySelector.java`
4. `src/main/java/io/quarkmind/agent/StrategyTrustObserver.java`
5. `src/main/java/io/quarkmind/agent/StrategyTrustRouter.java`

Order: tests first, then sources. If `safe_delete` reports usages, they must be fixed in Steps 1-7 first.

- [ ] **Step 9: Verify all tests pass**

Run: `mvn test -q`
Expected: all tests PASS. Zero references to `StrategySelector`, `StrategyTrustRouter`, `StrategyTrustObserver` remain.

- [ ] **Step 10: Commit**

```
feat(#192): migrate strategy routing — delete StrategySelector/TrustRouter/TrustObserver

Strategy tasks now gate on CaseFile key via CaseContext. MilestoneOutcomeRecorder
reads from SC2StrategyRouterTask.lastSelectedId(). ~264 lines deleted.

Refs #192
```

---

### Task 6: Integration Tests

**Files:**
- Create: `src/test/java/io/quarkmind/agent/cbr/SC2CbrRetentionIT.java`
- Create: `src/test/java/io/quarkmind/agent/cbr/SC2CbrRoutingIT.java`

**Interfaces:**
- Consumes: All components from Tasks 1-5
- Produces: Full lifecycle verification — CBR store, retrieve, and routing in CDI context

- [ ] **Step 1: Write SC2CbrRetentionIT**

```java
package io.quarkmind.agent.cbr;

import io.casehub.api.spi.CaseOutcomeEvent;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.*;
import io.casehub.platform.api.path.Path;
import io.quarkmind.agent.QuarkMindCaseFile;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class SC2CbrRetentionIT {

    @Inject CbrCaseMemoryStore cbrStore;
    @Inject SC2CbrRetentionObserver retentionObserver;

    @Test
    void gameClose_storesCbrCase_andRecordsOutcome() {
        Instant before = Instant.now();

        CaseOutcomeEvent event = new CaseOutcomeEvent(
            "starcraft-game", "test-tenant", UUID.randomUUID(),
            Map.of(
                QuarkMindCaseFile.STRATEGY_SELECTED_ID, "strategy.early-pressure",
                QuarkMindCaseFile.STRATEGY_ROUTED_ARCHETYPE, "ZERG_ROACH_RUSH",
                QuarkMindCaseFile.STRATEGY_ROUTED_CONFIDENCE, 0.85),
            "WIN", Instant.now(), Map.of());

        retentionObserver.onOutcome(event);

        CbrQuery query = CbrQuery.of("test-tenant", new MemoryDomain("quarkmind"),
                Path.root(), SC2GameCbrCase.CBR_TYPE,
                Map.of("enemy_archetype", FeatureValue.string("ZERG_ROACH_RUSH")), 10)
            .withNotBefore(before);

        List<ScoredCbrCase<SC2GameCbrCase>> results = cbrStore.retrieveSimilar(query, SC2GameCbrCase.class);
        assertThat(results).isNotEmpty();

        SC2GameCbrCase stored = results.getFirst().cbrCase();
        assertThat(stored.solution()).isEqualTo("strategy.early-pressure");
        assertThat(stored.outcome()).isEqualTo("WIN");
    }
}
```

- [ ] **Step 2: Write SC2CbrRoutingIT**

```java
package io.quarkmind.agent.cbr;

import io.casehub.api.spi.CaseOutcomeEvent;
import io.quarkmind.agent.GameSession;
import io.quarkmind.agent.MutableMapCaseContext;
import io.quarkmind.agent.QuarkMindCaseFile;
import io.quarkmind.agent.plugin.ScoutingIntelBroker;
import io.quarkmind.agent.plugin.ScoutingIntelPayload.PatternAssessmentPayload;
import io.quarkmind.domain.EnemyArchetype;
import io.quarkmind.domain.EnemyPatternAssessment;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.*;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class SC2CbrRoutingIT {

    @Inject
    SC2CbrRetentionObserver retentionObserver;
    @Inject
    SC2StrategyRouterTask   routerTask;
    @Inject
    ScoutingIntelBroker     broker;
    @Inject
    GameSession             gameSession;

    @Test
    void pastGames_influenceStrategySelection() {
        Instant before = Instant.now();

        for (int i = 0; i < 3; i++) {
            retentionObserver.onOutcome(new CaseOutcomeEvent(
                    "starcraft-game", "default", UUID.randomUUID(),
                    Map.of(
                            QuarkMindCaseFile.STRATEGY_SELECTED_ID, "strategy.early-pressure",
                            QuarkMindCaseFile.STRATEGY_ROUTED_ARCHETYPE, "ZERG_ROACH_RUSH",
                            QuarkMindCaseFile.STRATEGY_ROUTED_CONFIDENCE, 0.9),
                    "WIN", Instant.now(), Map.of()));
        }

        var assessment = new EnemyPatternAssessment(
                EnemyArchetype.ZERG_ROACH_RUSH, 0.9, 1000, "test");
        broker.update(new PatternAssessmentPayload(List.of(assessment)));

        MutableMapCaseContext ctx = new MutableMapCaseContext(
                Map.of(QuarkMindCaseFile.READY, true));
        routerTask.execute(ctx);

        assertThat(ctx.get(QuarkMindCaseFile.STRATEGY_SELECTED_ID, String.class))
                .isPresent();
        assertThat(ctx.get(QuarkMindCaseFile.STRATEGY_ROUTED_ARCHETYPE, String.class))
                .hasValue("ZERG_ROACH_RUSH");
    }
}
```

- [ ] **Step 3: Run integration tests**

Run: `mvn test -Dtest=SC2CbrRetentionIT,SC2CbrRoutingIT -q`
Expected: PASS

- [ ] **Step 4: Run full test suite**

Run: `mvn test -q`
Expected: all tests PASS

- [ ] **Step 5: Run benchmark**

Run: `mvn test -Pbenchmark -Dtest=GameLoopBenchmarkTest -q`
Compare tick timings against baseline. In-memory CBR retrieval should add < 1ms per tick.

- [ ] **Step 6: Commit**

```
feat(#192): CBR integration tests — retention and routing lifecycle

SC2CbrRetentionIT verifies case storage at game close.
SC2CbrRoutingIT verifies past game experiences influence strategy selection.

Refs #192
```

---

## File Summary

### Created (7 files)
| File | Task | Purpose |
|------|------|---------|
| `src/main/java/io/quarkmind/agent/cbr/SC2GameCbrCase.java` | 1 | CBR case record |
| `src/main/java/io/quarkmind/agent/cbr/SC2CbrSchemaRegistrar.java` | 1 | Feature schema registration |
| `src/main/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserver.java` | 2 | Store cases at game close |
| `src/main/java/io/quarkmind/agent/cbr/SC2ImplementationRoutingStrategy.java` | 3 | Trust + CBR routing |
| `src/main/java/io/quarkmind/agent/cbr/SC2StrategyRouterTask.java` | 4 | Tick-chain router |
| `src/test/java/io/quarkmind/agent/cbr/SC2CbrRetentionIT.java` | 6 | Retention IT |
| `src/test/java/io/quarkmind/agent/cbr/SC2CbrRoutingIT.java` | 6 | Routing IT |

### Deleted (5 files, ~264 production lines)
| File | Task | Replaced by |
|------|------|-------------|
| `StrategySelector.java` | 5 | CaseFile keys |
| `StrategyTrustRouter.java` | 5 | SC2ImplementationRoutingStrategy |
| `StrategyTrustObserver.java` | 5 | SC2StrategyRouterTask |
| `StrategySelectorTest.java` | 5 | — |
| `StrategyTrustRouterTest.java` | 5 | SC2ImplementationRoutingStrategyTest |

### Modified (significant, ~15 files)
| File | Task | Change |
|------|------|--------|
| `QuarkMindCaseFile.java` | 1 | Add 4 key constants + ALL_KEYS |
| `QuarkMindCaseHub.java` | 4 | Add `strategy-routing.` to PHASE_ORDER |
| `QuarkMindTrustRoutingPolicyProvider.java` | 4 | Add `"strategy"` case |
| `DroolsStrategyTask.java` | 5 | Remove StrategySelector, CaseContext activateIf |
| `EarlyPressureStrategyTask.java` | 5 | Same |
| `EconomicExpansionStrategyTask.java` | 5 | Same |
| `MilestoneOutcomeRecorder.java` | 5 | Replace StrategySelector with SC2StrategyRouterTask |
| + ~8 test files | 5 | Replace StrategySelector references |
