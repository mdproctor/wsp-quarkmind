# Commentator LLM (Observer Mode) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #181 — feat: Commentator/Coach LLM — real-time game narration and human coaching via oversight channel
**Issue group:** #181

**Goal:** Add a human-facing LLM commentator that narrates SC2 games in real time using two complementary patterns — reactive (Pattern A: Worker dispatch for immediate moment reactions) and narrative (Pattern B: EventAccumulator + Worker dispatch for periodic contextual summaries).

**Architecture:** Both patterns feed a single `quarkmind-commentary` Qhorus channel. Pattern A uses `CommentaryTriggerBuilder` (CDI bean with cooldown) to map L2 moments to CaseFile signal keys and dispatch reactive Workers. Pattern B uses `CommentaryAccumulator` wrapping `EventAccumulator<GameMoment>` + `WindowPolicy` (~45s/4 moments) to accumulate context, snapshot L3/L4 state from `NarrativeContextHolder`, and dispatch narrative Workers. Advisory infrastructure is partially generalized: `AdvisoryLatencyRecorder` → `LlmWorkerLatencyRecorder` (observes new shared `LlmWorkerCompleted` event), `QuarkMindAdvisorRegistrar` → `QuarkMindAgentRegistrar`.

**Tech Stack:** Java 21, Quarkus, CDI, Drools rule units, LangChain4j ChatModel, casehub-qhorus, casehub-eidos, casehub-ledger trust scoring

**Spec:** `docs/superpowers/specs/2026-07-06-commentator-observer-llm-design.md`

## Global Constraints

- All new classes follow existing package conventions: `io.quarkmind.agent` for orchestration/trust, `io.quarkmind.plugin.commentary` for commentary-specific implementations
- Plain JUnit for unit tests (no `@QuarkusTest` unless CDI context is needed)
- TDD: failing test → implement → verify → commit
- No `@QuarkusTest` for tests that can be plain JUnit
- IntelliJ MCP for all renames and reference updates
- Commit messages reference `#181`

---

## Task Dependency Graph

```
Task 1 (GameMomentTypes)  ──┐
Task 2 (LlmWorkerCompleted) ├── independent, can run in parallel
Task 3 (Data model+channel) ├──
Task 4 (Eidos+trust)       ──┘
                              │
Task 5 (Pattern A)  ──────── depends on 3
Task 6 (Pattern B)  ──────── depends on 3, 5
Task 7 (Integration) ─────── depends on all
```

Tasks 1-4 are independent and can run as parallel subagents. Tasks 5-7 are sequential.

---

### Task 1: New GameMomentTypes + Drools Rules

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/summarisation/GameMomentType.java`
- Modify: `src/main/java/io/quarkmind/plugin/summarisation/MomentDetectionRuleUnit.java`
- Modify: `src/main/java/io/quarkmind/plugin/summarisation/MomentDetectionTask.java`
- Modify: `src/main/resources/io/quarkmind/plugin/summarisation/MomentDetectionTask.drl`
- Modify: `src/test/java/io/quarkmind/plugin/summarisation/MomentDetectionTaskTest.java`

**Interfaces:**
- Consumes: `ScoutingIntelPayload` (sealed interface with `PostureUpdate`, `ArmySize` variants)
- Produces: `GameMomentType.ARMY_SHIFT`, `GameMomentType.POSTURE_CHANGE`, `GameMomentType.GAME_ENDING`

**What to build:**
1. Add three enum values to `GameMomentType`: `ARMY_SHIFT`, `POSTURE_CHANGE`, `GAME_ENDING`
2. Add `previousArmyValue` (int) and `previousPosture` (String) fields to `MomentDetectionRuleUnit` with getters/setters
3. Add Drools rules for the three new types:
   - `ARMY_SHIFT`: when `ArmySize` intel fires and `|newValue - previousArmyValue| / max(previousArmyValue, 1) > 0.3`
   - `POSTURE_CHANGE`: when `PostureUpdate` intel fires and posture differs from `previousPosture`
   - `GAME_ENDING`: when enemy building count = 0 OR own building count = 0 (check via CaseContext keys)
4. Update `MomentDetectionTask.fireRules()` to save `previousArmyValue` and `previousPosture` after each invocation
5. Clear `previousArmyValue` and `previousPosture` in `onGameStarted()`

**TDD steps:**
- [ ] Write failing test: `ARMY_SHIFT` detected when army value changes >30%
- [ ] Write Drools rule + RuleUnit field, verify test passes
- [ ] Write failing test: `POSTURE_CHANGE` detected when posture changes
- [ ] Write Drools rule + RuleUnit field, verify test passes
- [ ] Write failing test: `GAME_ENDING` detected when enemy buildings = 0
- [ ] Write Drools rule, verify test passes
- [ ] Write failing test: previous state cleared on `GameStarted`
- [ ] Implement reset, verify passes
- [ ] `mvn test -Dtest=MomentDetectionTaskTest -q` — all pass
- [ ] Commit: `feat(#181): add ARMY_SHIFT, POSTURE_CHANGE, GAME_ENDING moment types`

---

### Task 2: Three-Event Completion Model + Latency Recorder Migration

**Files:**
- Create: `src/main/java/io/quarkmind/agent/LlmWorkerCompleted.java`
- Rename: `AdvisoryLatencyRecorder.java` → `LlmWorkerLatencyRecorder.java` (IntelliJ refactor)
- Modify: `src/main/java/io/quarkmind/agent/QuarkMindCaseHub.java` (advisory callback fires both events)
- Rename: `AdvisoryLatencyRecorderTest.java` → `LlmWorkerLatencyRecorderTest.java` (IntelliJ refactor)
- Modify: test to use `LlmWorkerCompleted` event

**Interfaces:**
- Produces: `LlmWorkerCompleted` record — `(String workerId, String capability, long gameFrame, long latencyMs)`
- Produces: `LlmWorkerLatencyRecorder` — observes `LlmWorkerCompleted`, records `response-latency` dimension

**What to build:**
1. Create `LlmWorkerCompleted` record with shared fields only: `workerId`, `capability`, `gameFrame`, `latencyMs`
2. Rename `AdvisoryLatencyRecorder` → `LlmWorkerLatencyRecorder` via IntelliJ
3. Change `@Observes AdvisoryCompleted` → `@Observes LlmWorkerCompleted` in the recorder
4. Add `commentary-reactive: 2000L` and `commentary-narrative: 5000L` to `MAX_LATENCY_MS` map
5. Update `QuarkMindCaseHub`: inject `Event<LlmWorkerCompleted>`, update advisory completion callback to fire both `AdvisoryCompleted` (unchanged) and `LlmWorkerCompleted` (new, shared fields only)
6. Update test to construct and fire `LlmWorkerCompleted` events

**TDD steps:**
- [ ] Write failing test: `LlmWorkerLatencyRecorder` records latency from `LlmWorkerCompleted` event
- [ ] Create `LlmWorkerCompleted` record
- [ ] Rename `AdvisoryLatencyRecorder` → `LlmWorkerLatencyRecorder` via `ide_refactor_rename`
- [ ] Change observed event type, add commentary entries to MAX_LATENCY_MS map
- [ ] Verify test passes
- [ ] Write test: advisory latency thresholds still work (crisis=2000, strategic=5000, economic=4000)
- [ ] Write test: commentary latency thresholds (reactive=2000, narrative=5000)
- [ ] Update `QuarkMindCaseHub.wireAdvisory()` callback to fire both events
- [ ] `mvn test -q` — all existing tests pass (critical: `AdvisoryIntegrationIT`, `AdvisoryWorkerFactoryTest`)
- [ ] Commit: `refactor(#181): three-event completion model — LlmWorkerCompleted + latency recorder migration`

---

### Task 3: Commentary Data Model + Channel Infrastructure

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/commentary/CommentaryType.java`
- Create: `src/main/java/io/quarkmind/plugin/commentary/Commentary.java`
- Create: `src/main/java/io/quarkmind/plugin/commentary/CommentaryCompleted.java`
- Create: `src/main/java/io/quarkmind/plugin/commentary/CommentaryCompletionCallback.java`
- Create: `src/main/java/io/quarkmind/plugin/commentary/CommentaryChannelBroker.java`
- Create: `src/main/java/io/quarkmind/plugin/commentary/CommentaryChannelBackend.java`
- Create: `src/test/java/io/quarkmind/plugin/commentary/CommentaryChannelBrokerTest.java`
- Create: `src/test/java/io/quarkmind/plugin/commentary/CommentaryChannelBackendTest.java`
- Modify: `src/main/java/io/quarkmind/agent/QuarkMindCaseFile.java` (add commentary key constants)

**Interfaces:**
- Produces: `CommentaryType` enum — `REACTIVE`, `NARRATIVE`
- Produces: `Commentary` record — `(String text, long gameFrame, CommentaryType type)`
- Produces: `CommentaryCompleted` record — `(String workerId, String capability, long gameFrame, String text, CommentaryType commentaryType, long latencyMs)`
- Produces: `CommentaryCompletionCallback` — `@FunctionalInterface` with commentary-specific parameters
- Produces: `CommentaryChannelBroker` — owns `quarkmind-commentary` channel, dispatches STATUS messages
- Produces: `CommentaryChannelBackend` — `HumanObserverChannelBackend` implementation

**Pattern to follow:** `AdvisoryChannelBroker` and `AdvisoryChannelBackend` — identical structure, different channel name and event type. `CommentaryChannelBroker` observes `CommentaryCompleted` events and dispatches to `quarkmind-commentary` channel. `CommentaryChannelBackend` stores latest message.

**CaseFile key constants to add:**
```java
public static final String COMMENTARY_TRIGGER = "game.commentary.trigger";
public static final String COMMENTARY_NARRATIVE_TRIGGER = "game.commentary.narrative.trigger";
```

**TDD steps:**
- [ ] Write `CommentaryChannelBackendTest` — test backendId, actorType, message storage, counter
- [ ] Create `CommentaryType`, `Commentary`, `CommentaryCompleted`, `CommentaryCompletionCallback`
- [ ] Create `CommentaryChannelBackend`, verify test passes
- [ ] Write `CommentaryChannelBrokerTest` — mock `MessageService`, verify STATUS dispatch on `CommentaryCompleted` event
- [ ] Create `CommentaryChannelBroker`, verify test passes
- [ ] Add CaseFile key constants
- [ ] Commit: `feat(#181): commentary data model + channel infrastructure`

---

### Task 4: Eidos Registration — Vocabulary, Descriptors, Trust Routing

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/commentary/CommentaryDispositionTerm.java`
- Create: `src/main/java/io/quarkmind/plugin/commentary/CommentaryDispositionRegistrar.java`
- Rename: `QuarkMindAdvisorRegistrar.java` → `QuarkMindAgentRegistrar.java` (IntelliJ refactor)
- Modify: `QuarkMindAgentRegistrar.java` — add 4 commentator descriptors
- Modify: `src/main/java/io/quarkmind/agent/QuarkMindTrustRoutingPolicyProvider.java` — add commentary policies, split buildPolicy()
- Modify: `src/main/java/io/quarkmind/agent/DispositionAwareRoutingStrategy.java` — add commentary disposition preferences
- Create: `src/test/java/io/quarkmind/plugin/commentary/CommentaryDispositionTermTest.java`
- Modify: `src/test/java/io/quarkmind/plugin/advisory/AdvisoryIntegrationIT.java` — verify 10 total descriptors (6 advisory + 4 commentary)

**Interfaces:**
- Produces: `CommentaryDispositionTerm` enum implementing `VocabularyTerm` — `ENERGETIC`, `ANALYTICAL`, `DRAMATIC`, `TACTICAL`
- Produces: 4 commentator `AgentDescriptor` instances with `commentary-reactive` and `commentary-narrative` capabilities

**Disposition axis mapping (as per spec):**
| Term | AgentDisposition field | Value |
|---|---|---|
| ENERGETIC | riskAppetite | bold |
| ANALYTICAL | riskAppetite | conservative |
| DRAMATIC | ruleFollowing | flexible |
| TACTICAL | ruleFollowing | strict |

**Trust routing policy additions:**
- `commentary-reactive`: minObs=5, qualityFloors={response-latency: 0.4}
- `commentary-narrative`: minObs=5, qualityFloors={response-latency: 0.3}
- `buildPolicy()` split into `buildAdvisoryPolicy(int minObs)` and `buildCommentaryPolicy(int minObs)`

**Disposition preference additions to `resolvePreference()`:**
Commentary capabilities use different preference mappings than advisory:
- High combat → riskAppetite:bold (ENERGETIC)
- Economic → riskAppetite:conservative (ANALYTICAL)
- Phase transitions → ruleFollowing:flexible (DRAMATIC)
- Stable → ruleFollowing:strict (TACTICAL)
These map to the same axes but the spec says `resolvePreference()` needs "commentary-specific game-context mappings." Since both advisory and commentary use the same two axes and the existing mappings already cover these game states, verify the existing logic works — if it does, no change needed.

**TDD steps:**
- [ ] Write `CommentaryDispositionTermTest` — verify 4 terms, vocabulary URI
- [ ] Create `CommentaryDispositionTerm` + `CommentaryDispositionRegistrar`
- [ ] Rename `QuarkMindAdvisorRegistrar` → `QuarkMindAgentRegistrar` via `ide_refactor_rename`
- [ ] Write failing test: registrar produces 10 descriptors (6 advisory + 4 commentary)
- [ ] Add 4 commentator descriptor builder methods, verify passes
- [ ] Write failing test: `forCapability("commentary-reactive")` returns policy with minObs=5, floor={response-latency: 0.4}
- [ ] Refactor `buildPolicy()` → `buildAdvisoryPolicy()` + `buildCommentaryPolicy()`, add commentary cases
- [ ] Verify existing advisory policy tests still pass
- [ ] `mvn test -q` — all pass
- [ ] Commit: `feat(#181): eidos registration — commentary vocabulary, descriptors, trust routing`

---

### Task 5: Pattern A — CommentaryTriggerBuilder + CommentaryWorkerFactory

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/commentary/CommentaryTriggerBuilder.java`
- Create: `src/main/java/io/quarkmind/plugin/commentary/CommentaryWorkerFactory.java`
- Create: `src/test/java/io/quarkmind/plugin/commentary/CommentaryTriggerBuilderTest.java`
- Create: `src/test/java/io/quarkmind/plugin/commentary/CommentaryWorkerFactoryTest.java`

**Interfaces:**
- Consumes: `QuarkMindCaseFile.MOMENTS_LATEST` (List<GameMoment>), `GameMomentType`, `CommentaryCompletionCallback`
- Produces: `CommentaryTriggerBuilder.build(CaseContext, long)` → `Map<String, Object>` (trigger map or empty)
- Produces: `CommentaryWorkerFactory.createReactiveWorkers(descriptors, chatModel, callback)` → `List<Worker>`

**CommentaryTriggerBuilder (@ApplicationScoped):**
- CDI bean (not static) — cooldown requires instance state
- `build(CaseContext ctx, long gameFrame)`: reads MOMENTS_LATEST, returns trigger map if moments present and cooldown elapsed
- Cooldown: `lastFiredFrame` field, 110-frame gap (~5s)
- Maps ALL moment types to single `game.commentary.trigger` key
- Payload: `Map.of("gameFrame", frame, "momentTypes", typeNames, "minerals", ..., "supply", ..., "army", ...)`
- `onGameStarted(@Observes GameStarted)`: resets cooldown

**CommentaryWorkerFactory (static utility):**
- `createReactiveWorkers(descriptors, chatModel, callback)` → filters descriptors by `commentary-reactive` capability
- `createNarrativeWorkers(descriptors, chatModel, callback)` → filters by `commentary-narrative` capability
- `executeReactiveCommentary(descriptor, chatModel, input, callback)` → WorkerResult
  - System prompt: "You are a StarCraft II commentator. React to the moment described. Be [energetic/analytical] based on disposition."
  - Response format: plain text (no structured sections — commentary is natural language)
- `executeNarrativeCommentary(descriptor, chatModel, input, callback)` → WorkerResult
  - System prompt: "You are a StarCraft II narrator. Summarize the strategic arc. Do NOT repeat specific moments just announced."
  - Reads accumulated moments + L3 phase + L4 arc from input map

**TDD steps:**
- [ ] Write `CommentaryTriggerBuilderTest`: moments present → trigger map returned
- [ ] Write test: empty moments → empty map
- [ ] Write test: cooldown enforcement — second call within 110 frames returns empty
- [ ] Write test: cooldown reset on GameStarted
- [ ] Implement `CommentaryTriggerBuilder`, verify all pass
- [ ] Write `CommentaryWorkerFactoryTest`: reactive worker creates correct output keys
- [ ] Write test: system prompt includes disposition traits
- [ ] Write test: error handling returns failed WorkerResult
- [ ] Implement `CommentaryWorkerFactory`, verify all pass
- [ ] Commit: `feat(#181): Pattern A — reactive commentary trigger builder + worker factory`

---

### Task 6: Pattern B — NarrativeContextHolder + CommentaryAccumulator

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/commentary/NarrativeContextHolder.java`
- Create: `src/main/java/io/quarkmind/plugin/commentary/CommentaryAccumulator.java`
- Create: `src/test/java/io/quarkmind/plugin/commentary/NarrativeContextHolderTest.java`
- Create: `src/test/java/io/quarkmind/plugin/commentary/CommentaryAccumulatorTest.java`

**Interfaces:**
- Consumes: `SummarisationLifecycle.phaseBus()`, `SummarisationLifecycle.arcBus()`, `MomentBroker.momentBus()`
- Produces: `NarrativeContextHolder.latestPhase()` → `GamePhase`, `NarrativeContextHolder.latestArc()` → `GameArc`
- Produces: `CommentaryAccumulator.tick(long)` → `Map<String, Object>` (trigger map or empty)

**NarrativeContextHolder (@ApplicationScoped):**
- Subscribes to L3 phaseBus and L4 arcBus at `@PostConstruct`
- `volatile GamePhase latestPhase`, `volatile GameArc latestArc`
- `snapshot()` → `Map<String, String>` with phase name, rationale, arc narrative (for CaseFile serialization)
- `onGameStarted(@Observes GameStarted)` — clears both fields

**CommentaryAccumulator (@ApplicationScoped):**
- Wraps `EventAccumulator<GameMoment>` with `WindowPolicy(1000, 4)` (~45s or 4 moments)
- Subscribes to L2 moment bus at `@PostConstruct`
- `tick(long now)`: if shouldEmit AND minimum time floor (672 frames since last emit), drains batch, calls `contextHolder.snapshot()`, returns trigger map with serialized batch + context. Else returns empty map.
- `lastEmitFrame` field for minimum time floor enforcement
- `onGameStarted(@Observes GameStarted)` — clears accumulator + lastEmitFrame

**TDD steps:**
- [ ] Write `NarrativeContextHolderTest`: publishes to phase bus → latestPhase updated
- [ ] Write test: publishes to arc bus → latestArc updated
- [ ] Write test: GameStarted clears both
- [ ] Write test: snapshot() returns map with phase/arc data
- [ ] Implement `NarrativeContextHolder`, verify all pass
- [ ] Write `CommentaryAccumulatorTest`: 4 moments accumulated → tick returns trigger map
- [ ] Write test: window time elapsed → tick returns trigger map
- [ ] Write test: minimum time floor — emits blocked if <672 frames since last emit
- [ ] Write test: GameStarted clears accumulator
- [ ] Implement `CommentaryAccumulator`, verify all pass
- [ ] Commit: `feat(#181): Pattern B — narrative context holder + commentary accumulator`

---

### Task 7: Game Loop Integration + End-to-End Test

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/QuarkMindCaseHub.java` — add `wireCommentary()`, register capabilities + bindings + workers
- Modify: `src/main/java/io/quarkmind/agent/GameTickExecutor.java` — inject `CommentaryTriggerBuilder` + `CommentaryAccumulator`, add steps 4b/7/8/8b
- Create: `src/main/java/io/quarkmind/plugin/commentary/CommentaryCompletionObserver.java` — observes `CommentaryCompleted`, fires `LlmWorkerCompleted`
- Modify: `src/main/resources/application.properties` — add commentary config
- Create: `src/test/java/io/quarkmind/plugin/commentary/CommentaryIntegrationIT.java`

**Interfaces:**
- Consumes: All previous tasks' outputs
- Produces: Fully wired commentary pipeline end-to-end

**QuarkMindCaseHub.wireCommentary():**
- Same pattern as `wireAdvisory()`: check chatModel availability, create 2 capabilities (`commentary-reactive`, `commentary-narrative`), 2 bindings with ContextChangeTriggers on `COMMENTARY_TRIGGER` and `COMMENTARY_NARRATIVE_TRIGGER` keys, commentary completion callback that fires both `CommentaryCompleted` and `LlmWorkerCompleted`, call `CommentaryWorkerFactory.createReactiveWorkers()` + `createNarrativeWorkers()`

**GameTickExecutor changes:**
- Inject `CommentaryTriggerBuilder` and `CommentaryAccumulator`
- After `summarisationLifecycle.tick()` (line 44): call `commentaryAccumulator.tick(gameFrame)` → store result
- After `deferredAdvisoryEvaluator.evaluate()`: call `commentaryTriggerBuilder.build(ctx, gameFrame)` → store result
- Signal both reactive and narrative triggers via `caseHub.signal()` (before advisory signal)

**CommentaryCompletionObserver (@ApplicationScoped):**
- `@Observes CommentaryCompleted` → fires `LlmWorkerCompleted` (bridges commentary completion to shared latency recording)

**Configuration (application.properties):**
```properties
quarkmind.commentary.reactive.cooldown-frames=110
quarkmind.commentary.narrative.window-frames=1000
quarkmind.commentary.narrative.window-count=4
quarkmind.commentary.narrative.min-emit-frames=672
quarkmind.commentary.trust.min-observations=5
```

**TDD steps:**
- [ ] Write `CommentaryIntegrationIT`: inject channel broker, verify channel created on startup
- [ ] Write test: inject registrar, verify 10 descriptors (6 advisory + 4 commentary)
- [ ] Write test: inject trust policy provider, verify commentary capabilities return valid policies
- [ ] Write test: inject commentary trigger builder, verify build() returns empty map with no moments
- [ ] Create `CommentaryCompletionObserver`
- [ ] Wire `QuarkMindCaseHub.wireCommentary()` — call from `getDefinition()`
- [ ] Update `GameTickExecutor.execute()` — add commentary steps
- [ ] Add configuration properties
- [ ] `mvn test -q` — ALL tests pass (critical: existing advisory, strategy, and summarisation tests unchanged)
- [ ] Commit: `feat(#181): game loop integration — commentary pipeline end-to-end`

---

## Verification

After all tasks complete:

1. **Unit tests:** `mvn test -q` — all pass, no regressions
2. **Integration test:** `CommentaryIntegrationIT` verifies full CDI wiring
3. **Run emulated mode:** `mvn quarkus:dev -Dquarkus.profile=emulated` — verify commentary channel appears in QA endpoints, no errors in game loop
4. **Existing advisory pipeline:** `AdvisoryIntegrationIT` passes unchanged — advisory flow unbroken
5. **Latency recording:** `LlmWorkerLatencyRecorderTest` verifies both advisory and commentary latency recording via shared `LlmWorkerCompleted` event
