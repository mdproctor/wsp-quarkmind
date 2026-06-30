# LLM Advisory Team + Engine Port — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Port QuarkMind to the casehub-engine Worker model, then build LLM advisory Workers with trust-weighted routing.

**Architecture:** Three sequential phases: (1) TrustWeightedImplementationRoutingStrategy in engine-ledger, (2) QuarkMindCaseHub + signalAndAwaitSync migration, (3) LLM advisory Workers with eidos integration and multi-dimensional trust scoring.

**Tech Stack:** Java 21, Quarkus 3.x, casehub-engine, casehub-eidos, casehub-ledger, LangChain4j ChatModel, Drools CEP

## Global Constraints

- **Design spec:** `docs/superpowers/specs/2026-06-30-llm-advisory-team-design.md`
- **Migration spec:** `docs/superpowers/specs/2026-06-13-casehub-engine-migration-design.md`
- **TDD:** Red → Green → Refactor for every task. Test first, always.
- **IntelliJ MCP first:** Use `mcp__intellij-index__*` for all class lookups, find-references, rename, move. Never bash grep for classes.
- **PLATFORM.md coherence:** Read `casehub-parent/docs/PLATFORM.md` before implementing. Consult garden protocols in `casehub/garden/docs/protocols/` throughout.
- **No commit attribution:** Do not add `Co-Authored-By` trailers.
- **Issue refs on all commits:** `Refs #N` or `Closes #N`.
- **Agent identity format:** `{model-family}:{persona}@{major}` (e.g. `claude:crisis-aggressive@v1`)
- **Breaking changes are free:** No backwards-compatibility shims. Fix the design.
- **Cross-repo work:** Phase 1 targets `casehub-engine` repo. Phases 2-3 target `quarkmind` repo.

---

## Phase 1: engine#625 — TrustWeightedImplementationRoutingStrategy

**Repo:** `/Users/mdproctor/claude/casehub/engine`
**Module:** `ledger/`

### Task 1: TrustWeightedImplementationRoutingStrategy

**Files:**
- Create: `ledger/src/main/java/io/casehub/ledger/routing/TrustWeightedImplementationRoutingStrategy.java`
- Create: `ledger/src/test/java/io/casehub/ledger/routing/TrustWeightedImplementationRoutingStrategyTest.java`

**Interfaces:**
- Consumes: `ImplementationRoutingStrategy` SPI (`io.casehub.api.spi.routing`), `TrustCandidateClassifier`, `TrustScoreSource`, `TrustRoutingPolicyProvider`
- Produces: `@Alternative @Priority(1)` CDI bean that displaces `NoOpImplementationRoutingStrategy` when engine-ledger is on classpath

**Design decisions:**
- `TrustCandidateClassifier.classify()` takes `List<AgentCandidate>`. Adapt `ImplementationCandidate` → `AgentCandidate` with `workerId=workerName`, `runningJobs=0` (in-process, no workload), `health=AVAILABLE`, `agentDescriptor=null`.
- `workloadScore` is always `1.0` (since `runningJobs=0`). Blending is trust-only, which is correct for implementation routing — it's about quality, not load.
- `TrustCandidateClassifier.decide()` returns `AgentAssignment`. Map: `Assigned(workerId)` → find the corresponding `ImplementationCandidate` by workerName → `Selected(List.of(bindingName))`. `Unresolvable` → `Selected(List.of(firstCandidate.bindingName()))` (backstop — always run something). `EscalateToOversight` → `Selected(List.of(firstCandidate.bindingName()))` (same backstop — implementations can't escalate to humans).
- First candidate in the list is the backstop. Order is binding declaration order from the CaseDefinition.

- [ ] **Step 1: Write failing test** — Test that with three candidates and seeded trust scores, the highest-scoring candidate is selected.

```java
@QuarkusTest
class TrustWeightedImplementationRoutingStrategyTest {

    @Inject TrustWeightedImplementationRoutingStrategy strategy;
    @Inject ActorTrustScoreRepository trustRepo;

    @Test
    void select_qualifiedCandidates_returnsHighestTrustScore() {
        // Seed trust scores: workerA=0.9, workerB=0.6, workerC=0.4
        TrustTestUtils.seedCapabilityScore(trustRepo, "workerA", "strategy", 0.9, 15);
        TrustTestUtils.seedCapabilityScore(trustRepo, "workerB", "strategy", 0.6, 15);
        TrustTestUtils.seedCapabilityScore(trustRepo, "workerC", "strategy", 0.4, 15);

        var candidates = List.of(
            new ImplementationCandidate("binding-a", "workerA", "strategy"),
            new ImplementationCandidate("binding-b", "workerB", "strategy"),
            new ImplementationCandidate("binding-c", "workerC", "strategy"));

        var ctx = new ImplementationRoutingContext(UUID.randomUUID(), "strategy", null);
        var result = strategy.select(ctx, candidates).await().indefinitely();

        assertThat(result).isInstanceOf(ImplementationSelection.Selected.class);
        var selected = (ImplementationSelection.Selected) result;
        assertThat(selected.bindingNames()).containsExactly("binding-a");
    }
}
```

- [ ] **Step 2: Run test to verify it fails** — `mvn test -pl ledger -Dtest=TrustWeightedImplementationRoutingStrategyTest -q`. Expected: FAIL (class not found).

- [ ] **Step 3: Implement TrustWeightedImplementationRoutingStrategy**

```java
package io.casehub.ledger.routing;

@Alternative
@Priority(1)
@ApplicationScoped
public class TrustWeightedImplementationRoutingStrategy implements ImplementationRoutingStrategy {

    private final TrustCandidateClassifier classifier;
    private final TrustScoreSource source;
    private final TrustRoutingPolicyProvider policyProvider;

    @Inject
    public TrustWeightedImplementationRoutingStrategy(
            TrustCandidateClassifier classifier,
            TrustScoreSource source,
            TrustRoutingPolicyProvider policyProvider) {
        this.classifier = classifier;
        this.source = source;
        this.policyProvider = policyProvider;
    }

    @Override
    public Uni<ImplementationSelection> select(
            ImplementationRoutingContext context,
            List<ImplementationCandidate> candidates) {

        if (candidates.size() <= 1) {
            return Uni.createFrom().item(new ImplementationSelection.RunAll());
        }

        TrustRoutingPolicy policy = policyProvider.forCapability(context.capabilityName());
        Map<String, ImplementationCandidate> byWorker = candidates.stream()
            .collect(Collectors.toMap(ImplementationCandidate::workerName, c -> c, (a, b) -> a));

        List<AgentCandidate> agentCandidates = candidates.stream()
            .map(c -> new AgentCandidate(c.workerName(), Set.of(c.capabilityName()),
                0, AgentHealth.AVAILABLE, null))
            .toList();

        List<TrustCandidateClassifier.ClassifiedCandidate> classified =
            classifier.classify(agentCandidates, context.capabilityName(), policy, source);

        List<TrustCandidateClassifier.ScoredCandidate> scored = classified.stream()
            .filter(cc -> !cc.isExcluded())
            .map(cc -> new TrustCandidateClassifier.ScoredCandidate(cc, computeScore(cc, policy)))
            .sorted(Comparator.comparingDouble(
                TrustCandidateClassifier.ScoredCandidate::finalScore).reversed())
            .toList();

        AgentAssignment assignment = classifier.decide(classified, scored, context.capabilityName());

        ImplementationSelection selection = switch (assignment) {
            case AgentAssignment.Assigned a -> {
                ImplementationCandidate winner = byWorker.get(a.workerId());
                yield new ImplementationSelection.Selected(List.of(winner.bindingName()));
            }
            case AgentAssignment.Unresolvable ignored ->
                new ImplementationSelection.Selected(List.of(candidates.get(0).bindingName()));
            case AgentAssignment.EscalateToOversight ignored ->
                new ImplementationSelection.Selected(List.of(candidates.get(0).bindingName()));
        };

        return Uni.createFrom().item(selection);
    }

    private double computeScore(TrustCandidateClassifier.ClassifiedCandidate cc,
                                TrustRoutingPolicy policy) {
        return switch (cc.phase()) {
            case BOOTSTRAP -> cc.workloadScore();
            case QUALIFIED -> cc.trustScore().orElse(0.5) * policy.blendFactor()
                + cc.workloadScore() * (1.0 - policy.blendFactor());
            default -> 0.0;
        };
    }
}
```

- [ ] **Step 4: Run test to verify it passes** — `mvn test -pl ledger -Dtest=TrustWeightedImplementationRoutingStrategyTest -q`. Expected: PASS.

- [ ] **Step 5: Add edge case tests** — BOOTSTRAP (no history), BORDERLINE, all-excluded backstop, single candidate passthrough.

- [ ] **Step 6: Commit** — `git commit -m "feat(#625): TrustWeightedImplementationRoutingStrategy — trust-based selection among competing implementations"`

---

## Phase 2: quarkmind#207 — Engine Migration Phase 2

**Repo:** `/Users/mdproctor/claude/casehub/quarkmind`

### Task 2: QuarkMindCaseHub

**Files:**
- Create: `src/main/java/io/quarkmind/agent/QuarkMindCaseHub.java`
- Create: `src/test/java/io/quarkmind/agent/QuarkMindCaseHubTest.java`

**Interfaces:**
- Consumes: `CaseHub` (casehub-engine-api), all `TaskDefinition` implementations via `@Any Instance<TaskDefinition>`
- Produces: `CaseDefinition` with capabilities, workers, and bindings for the starcraft-game case type

**Key logic:**
- Extends `CaseHub`, overrides `getDefinition()`
- Discovers all `TaskDefinition` implementations via CDI `Instance<TaskDefinition>`
- Creates a `tick-decision` Capability with a single `TickOrchestratorWorker`
- Creates a `ContextChangeTrigger` on `.working["game.frame"] | . != null`
- Strategy implementations registered under `"strategy"` capability for ImplementationRoutingStrategy

- [ ] **Step 1: Write failing test** — Test that `getDefinition()` returns a CaseDefinition with the tick-decision capability and all plugin Workers.

- [ ] **Step 2: Implement QuarkMindCaseHub** — `@ApplicationScoped`, extends `CaseHub`, discovers plugins, builds CaseDefinition.

- [ ] **Step 3: Run tests, verify pass**

- [ ] **Step 4: Commit** — `Refs #207`

### Task 3: TickOrchestratorWorker

**Files:**
- Create: `src/main/java/io/quarkmind/agent/TickOrchestratorWorker.java`
- Create: `src/test/java/io/quarkmind/agent/TickOrchestratorWorkerTest.java`

**Interfaces:**
- Consumes: All `TaskDefinition` implementations, `WorkerRuntime` for chaining
- Produces: `WorkerFunction.Sync` that chains scouting → strategy → tactics → economics

**Key logic:**
- Wraps the sequential plugin chain as a single `WorkerFunction.Sync`
- Inside the function: calls `WorkerRuntime.execute()` for each plugin in order
- Each plugin's `activateIf()` is checked before execution (skip if false)
- `requires()` keys verified in CaseContext before execution

- [ ] **Step 1: Write failing test** — Verify plugins execute in order with mock CaseContext.

- [ ] **Step 2: Implement TickOrchestratorWorker**

- [ ] **Step 3: Run tests, verify pass**

- [ ] **Step 4: Commit** — `Refs #207`

### Task 4: Migrate GameTickExecutor to signalAndAwaitSync

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/AgentOrchestrator.java`
- Modify: `src/main/java/io/quarkmind/agent/GameTickExecutor.java`
- Create: `src/test/java/io/quarkmind/agent/GameTickExecutorMigrationTest.java`

**Key changes:**
- `startGame()`: call `quarkMindCaseHub.startCase(initialData)` → store `gameSessionId`
- `gameTick()`: replace `caseEngine.createAndSolve()` with `caseHub.signalAndAwaitSync(gameSessionId, translator.toMap(), Duration.ofSeconds(5))`
- `stopGame()`: call `caseHub.cancelCase(gameSessionId)`
- Remove `CaseEngine` injection and poc-based dispatch

- [ ] **Step 1: Write failing test** — Test tick flow with mock CaseHubRuntime.

- [ ] **Step 2: Migrate GameTickExecutor**

- [ ] **Step 3: Run full test suite** — `mvn test -q`. Fix any broken tests from the API change.

- [ ] **Step 4: Commit** — `Refs #207`

### Task 5: Delete poc dependency and bridge code

**Files:**
- Delete: `StrategyTrustRouter.java`, `StrategySelector.java`, `StrategyTrustObserver.java`, `QuarkMindTaskRegistrar.java`
- Modify: `pom.xml` — remove `casehub-core`, `casehub-persistence-memory` dependencies
- Modify: All `@CaseType` injection points per the table in the migration spec
- Modify: All test classes that use poc `CaseFile` — switch to `CaseContext` only

**Key logic:**
- Remove the bridge `canActivate(CaseFile)` and `execute(CaseFile)` methods from all `TaskDefinition` implementations
- Remove `io.casehub.core.TaskDefinition` extends from all seam interfaces
- Update all test construction from `InMemoryCaseFileRepository` to `CaseContextImpl`

- [ ] **Step 1: Delete StrategyTrustRouter, StrategySelector, StrategyTrustObserver, QuarkMindTaskRegistrar**

- [ ] **Step 2: Remove poc dependencies from pom.xml**

- [ ] **Step 3: Remove bridge methods from all TaskDefinition implementations** — Use IntelliJ find-references on `CaseFile` to locate all call sites.

- [ ] **Step 4: Update all @CaseType injection points** — per the table in migration spec §3.

- [ ] **Step 5: Run full test suite** — `mvn test -q`. Fix all compilation errors from removed types.

- [ ] **Step 6: Commit** — `Closes #207`

---

## Phase 3: quarkmind#180 — LLM Advisory Workers

**Repo:** `/Users/mdproctor/claude/casehub/quarkmind`

### Task 6: AdvisorDescriptorRegistrar (eidos integration)

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/advisory/QuarkMindAdvisorRegistrar.java`
- Create: `src/test/java/io/quarkmind/plugin/advisory/QuarkMindAdvisorRegistrarTest.java`

**Interfaces:**
- Consumes: `AgentDescriptorRegistrar` SPI (casehub-eidos-api)
- Produces: List of `AgentDescriptor` for each advisory configuration (2 per role × 3 roles = 6 advisors)

**Key logic:**
- Each descriptor has: agentId (format: `claude:{persona}@v1`), slot, disposition (ConscientiousnessTerm values), capability matching an advisory role
- Crisis: aggressive + conservative configurations
- Strategic: bold + measured configurations  
- Economic: expansion + defensive configurations

- [ ] **Step 1: Write test** — Verify registrar returns 6 descriptors with correct capability tags and disposition traits.

- [ ] **Step 2: Implement QuarkMindAdvisorRegistrar**

- [ ] **Step 3: Add eidos dependencies to pom.xml** — `casehub-eidos-api` (compile), `casehub-eidos-vocab` (compile), `casehub-eidos` (runtime)

- [ ] **Step 4: Commit** — `Refs #180`

### Task 7: DispositionAwareRoutingStrategy

**Files:**
- Create: `src/main/java/io/quarkmind/agent/DispositionAwareRoutingStrategy.java`
- Create: `src/main/java/io/quarkmind/agent/DispositionPreference.java`
- Create: `src/test/java/io/quarkmind/agent/DispositionAwareRoutingStrategyTest.java`

**Interfaces:**
- Consumes: `AgentRoutingStrategy` SPI, `TrustCandidateClassifier`, `TrustScoreSource`, `TrustRoutingPolicyProvider`
- Produces: `@ApplicationScoped` quarkmind-specific routing strategy that composes trust scores with disposition fit

**Key logic:**
- Implements `AgentRoutingStrategy`
- Classifies candidates via `TrustCandidateClassifier` (reuses four-phase model)
- Computes disposition multiplier (0.8–1.2) based on game state from `AgentRoutingContext.caseContext()` (JSON node)
- Soft preference — disposition never hard-excludes; it only adjusts score
- `DispositionPreference` record: maps game context (enemy posture, game phase) to preferred disposition axis values

- [ ] **Step 1: Write test** — With two advisors (bold + conservative) and enemy AGGRESSIVE context, verify conservative advisor scores higher.

- [ ] **Step 2: Implement DispositionAwareRoutingStrategy and DispositionPreference**

- [ ] **Step 3: Commit** — `Refs #180`

### Task 8: Advisory Worker definitions in CaseDefinition

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/QuarkMindCaseHub.java` — add advisory capabilities, bindings, and workers
- Create: `src/main/java/io/quarkmind/plugin/advisory/AdvisoryWorkerFactory.java`
- Create: `src/test/java/io/quarkmind/plugin/advisory/AdvisoryWorkerFactoryTest.java`

**Interfaces:**
- Consumes: `Agent.builder()`, `AgentWorkerFunction`, `SystemPromptRenderer` (eidos), `ChatModel` (LangChain4j)
- Produces: Advisory Workers wrapping `AgentWorkerFunction(Agent)` for each advisor configuration

**Key logic:**
- `AdvisoryWorkerFactory` builds `Worker` instances from `AgentDescriptor` configurations
- Each Worker's `AgentWorkerFunction` wraps an `Agent` with: system prompt from `SystemPromptRenderer`, user message template with game state, `outputTransformer` for key-prefixing (`agent.advisory.{role}.*`), `responseSchema` for structured LLM output
- Workers registered in CaseDefinition under three advisory capabilities
- Advisory bindings use `ContextChangeTrigger` on `game.advisory.trigger.{role}` keys

- [ ] **Step 1: Write test** — Verify factory creates Workers with correct capability names and AgentWorkerFunction type.

- [ ] **Step 2: Implement AdvisoryWorkerFactory**

- [ ] **Step 3: Wire advisory into QuarkMindCaseHub.getDefinition()** — add capabilities, workers, bindings.

- [ ] **Step 4: Commit** — `Refs #180`

### Task 9: Two-signal advisory trigger dispatch

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/GameTickExecutor.java` — add post-tick advisory signal
- Create: `src/main/java/io/quarkmind/agent/AdvisoryTriggerBuilder.java`
- Create: `src/test/java/io/quarkmind/agent/AdvisoryTriggerBuilderTest.java`

**Key logic:**
- After `signalAndAwaitSync()` returns, check CaseContext for detected L2 moments and L3 phase transitions
- Build `Map<String, Object>` of advisory trigger keys (selective — only for detected events)
- Fire `caseHub.signal(gameSessionId, advisoryTriggers)` (fire-and-forget with `exceptionally()` logging)
- Each trigger payload includes `gameFrame` as monotonic component for ContextChangeTrigger detection

- [ ] **Step 1: Write test** — Verify trigger builder maps L2 moments to correct trigger keys.

- [ ] **Step 2: Implement AdvisoryTriggerBuilder**

- [ ] **Step 3: Wire into GameTickExecutor post-tick flow**

- [ ] **Step 4: Commit** — `Refs #180`

### Task 10: Advisory completion lifecycle

**Files:**
- Create: `src/main/java/io/quarkmind/agent/AdvisoryCompletionObserver.java`
- Create: `src/main/java/io/quarkmind/agent/AdvisoryCompleted.java`
- Create: `src/main/java/io/quarkmind/agent/AdvisoryInvocationCounter.java`
- Create: `src/test/java/io/quarkmind/agent/AdvisoryCompletionObserverTest.java`
- Create: `src/test/java/io/quarkmind/agent/AdvisoryInvocationCounterTest.java`

**Key logic:**
- `AdvisoryCompletionObserver`: `@ObservesAsync WorkerDecisionEvent`, filters by `advisory-` capability prefix, reads advisory output from CaseContext, fires `AdvisoryCompleted` CDI event
- `AdvisoryCompleted` record: advisorId, capability, gameFrame, recommendation, confidence, latencyMs
- `AdvisoryInvocationCounter`: tracks invoked advisors per game (ConcurrentHashMap.newKeySet()), clears on `@Observes GameStarted`
- Latency measured via `System.nanoTime()` stored in CaseContext by the advisory Worker before/after LLM call

- [ ] **Step 1: Write tests** for observer filtering and invocation counter lifecycle.

- [ ] **Step 2: Implement all three classes**

- [ ] **Step 3: Commit** — `Refs #180`

### Task 11: Trust scoring — latency and deferred evaluation

**Files:**
- Create: `src/main/java/io/quarkmind/agent/AdvisoryLatencyRecorder.java`
- Create: `src/main/java/io/quarkmind/agent/DeferredAdvisoryEvaluator.java`
- Create: `src/main/java/io/quarkmind/agent/PendingEvaluation.java`
- Create: `src/test/java/io/quarkmind/agent/AdvisoryLatencyRecorderTest.java`
- Create: `src/test/java/io/quarkmind/agent/DeferredAdvisoryEvaluatorTest.java`

**Key logic:**
- `AdvisoryLatencyRecorder`: `@Observes AdvisoryCompleted`, computes normalised latency score (`1.0 - actualMs/maxMs`), writes `OutcomeRecord` with dimension key `response-latency`
- `DeferredAdvisoryEvaluator`: called post-tick, checks pending evaluations in CaseContext, compares game state snapshots at advisory time vs current, writes `OutcomeRecord` with dimension key `recommendation-quality`
- `PendingEvaluation` record: advisorId, capability, advisoryFrame, recommendation, confidence, gameStateSnapshot (Map<String, Double>)
- `EVALUATION_DELAY_FRAMES = 200` (~17 seconds)

- [ ] **Step 1: Write tests** — Verify latency scoring formula and deferred delta computation.

- [ ] **Step 2: Implement all classes**

- [ ] **Step 3: Commit** — `Refs #180`

### Task 12: Game-outcome recording and advisor phase observer

**Files:**
- Create: `src/main/java/io/quarkmind/agent/AdvisoryGameOutcomeRecorder.java`
- Create: `src/main/java/io/quarkmind/agent/AdvisorTrustPhaseObserver.java`
- Create: `src/main/java/io/quarkmind/agent/AdvisorExcludedEvent.java`
- Create: `src/test/java/io/quarkmind/agent/AdvisoryGameOutcomeRecorderTest.java`

**Key logic:**
- `AdvisoryGameOutcomeRecorder`: `@Observes GameStopped`, reads `AdvisoryInvocationCounter.snapshot()`, writes one `OutcomeRecord` per invoked advisor with flat weight 1.0 and dimension key `game-outcome`
- `AdvisorTrustPhaseObserver`: fires `AdvisorExcludedEvent` CDI event when an advisor transitions to EXCLUDED phase (for observability/monitoring)

- [ ] **Step 1: Write tests**

- [ ] **Step 2: Implement**

- [ ] **Step 3: Commit** — `Refs #180`

### Task 13: Qhorus advisory channel + in-loop consumption

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/advisory/AdvisoryChannelBroker.java`
- Modify: `src/main/java/io/quarkmind/plugin/DroolsStrategyTask.java` — read advisory output from CaseContext
- Create: `src/test/java/io/quarkmind/plugin/advisory/AdvisoryChannelBrokerTest.java`

**Key logic:**
- `AdvisoryChannelBroker`: creates/finds `quarkmind-advisory` Qhorus channel at `@PostConstruct`, `@Observes AdvisoryCompleted` dispatches STATUS message with advisory JSON payload
- `DroolsStrategyTask`: adds advisory fact reading from CaseContext keys (`agent.advisory.{role}.*`), checks staleness threshold, adds `AdvisoryFact` to Drools rule unit store
- Channel: `ChannelSemantic.APPEND`, `Set.of(MessageType.STATUS)`

- [ ] **Step 1: Write tests** — Channel creation, message dispatch, DroolsStrategyTask advisory consumption.

- [ ] **Step 2: Implement AdvisoryChannelBroker**

- [ ] **Step 3: Wire advisory reading into DroolsStrategyTask**

- [ ] **Step 4: Commit** — `Refs #180`

### Task 14: TrustRoutingPolicyProvider + configuration

**Files:**
- Create: `src/main/java/io/quarkmind/agent/QuarkMindTrustRoutingPolicyProvider.java`
- Modify: `src/main/resources/application.properties` — advisory trust configuration
- Create: `src/test/java/io/quarkmind/agent/QuarkMindTrustRoutingPolicyProviderTest.java`

**Key logic:**
- `@ApplicationScoped @Alternative @Priority(1)` — displaces engine's default provider
- Returns per-capability policies: `tick-decision` (strategy), `advisory-crisis`, `advisory-strategic`, `advisory-economic`
- Quality floors from `@ConfigProperty` per the spec: `response-latency=0.3`, `recommendation-quality=0.2`, `game-outcome=0.2`
- Different `minimumObservations` per role: crisis=5 (fast convergence), strategic/economic=10

- [ ] **Step 1: Write tests**

- [ ] **Step 2: Implement**

- [ ] **Step 3: Add configuration to application.properties**

- [ ] **Step 4: Commit** — `Refs #180`

### Task 15: HIL coaching ChannelBackend + integration test

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/advisory/AdvisoryChannelBackend.java`
- Create: `src/test/java/io/quarkmind/plugin/advisory/AdvisoryChannelBackendTest.java`
- Create: `src/test/java/io/quarkmind/plugin/advisory/AdvisoryIntegrationIT.java`

**Key logic:**
- `AdvisoryChannelBackend implements ChannelBackend`: registers on `quarkmind-advisory`, forwards `post()` to WebSocket `/ws/advisory` endpoint
- Integration test: full flow from tick → moment detection → advisory trigger → Worker dispatch (mock ChatModel) → CaseContext write → strategy consumption → Qhorus channel persistence

- [ ] **Step 1: Implement AdvisoryChannelBackend**

- [ ] **Step 2: Write integration test** — end-to-end flow with mock LLM responses.

- [ ] **Step 3: Run full test suite** — `mvn test -q`. All 1059+ tests must pass.

- [ ] **Step 4: Commit** — `Closes #180`

---

## Execution Notes

### Cross-repo workflow
- Phase 1 (engine#625): work in `/Users/mdproctor/claude/casehub/engine`, branch `issue-625-trust-impl-routing`
- After Phase 1: `mvn install -DskipTests` the engine to update quarkmind's classpath
- Phases 2-3 (quarkmind#207, #180): work in `/Users/mdproctor/claude/casehub/quarkmind`, branch `issue-180-llm-advisory-scouting` (already created)

### Dependency order
engine#625 → `mvn install` → quarkmind#207 → quarkmind#180

### Key protocol references
- `PP-20260603-cefed9`: requires()/activateIf() split — retained on TaskDefinition during Phase 2
- `PP-20260608-8584ab`: scouting consumer @PostConstruct — advisory consumers follow this
- `PP-20260610-88dbbd`: game lifecycle observer synchrony — GameStarted/GameStopped observers use @Observes (sync)
- `PP-20260521-1ca0c8`: trust maturity model — four-phase, never block
- `PP-20260522-3b1ccd`: no workarounds — fix the design
