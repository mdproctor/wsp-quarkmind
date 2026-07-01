# QuarkMind Handover — 2026-07-01

## Last Session

Designed, reviewed, and partially implemented #180 (LLM advisory team) with full engine port (#207) as prerequisite. The session covered: brainstorming → adversarial design review (6 rounds, $20) → implementation plan → SDD execution of Tasks 1-7/15. Phase 1 (engine#625) and Phase 2 (quarkmind#207 engine migration) are complete. Phase 3 (advisory Workers) is 2/10 tasks in.

## Immediate Next Step

Resume SDD execution at **Task 8** (Advisory Worker definitions in CaseDefinition). Run `resume handover`, then dispatch Task 8 per the plan. The SDD progress ledger at `.superpowers/sdd/progress.md` carries task-level state. Use `superpowers:subagent-driven-development` to continue dispatching tasks.

## Execution State — SDD Mid-Run

**Progress ledger:** `.superpowers/sdd/progress.md` — read this FIRST. It lists every completed task with commit SHAs.

**Plan:** `plans/2026-06-30-llm-advisory-engine-port.md` (workspace repo)

**Spec:** `docs/superpowers/specs/2026-06-30-llm-advisory-team-design.md` (project repo — updated by design review)

**Branches:**
- quarkmind: `issue-180-llm-advisory-scouting`
- workspace: `issue-180-llm-advisory-scouting`
- engine: `issue-625-trust-impl-routing` (Task 1 committed here)

**Test state:** 1110 passing, 1 pre-existing error (`DroolsScoutingRulesTest` — qhorus datasource not configured for its test profile). All @QuarkusTest tests pass after CDI wiring fixes.

### Tasks Complete (7/15)

| # | Task | Commit | Phase |
|---|------|--------|-------|
| 1 | TrustWeightedImplementationRoutingStrategy | `def6c2c5` (engine) | Phase 1 |
| 2 | QuarkMindCaseHub | `9305a96` | Phase 2 |
| 3 | TickOrchestratorWorker | `476b393` | Phase 2 |
| 4 | GameTickExecutor migration | `50f2def` | Phase 2 |
| 5 | Delete poc + bridge code | `b1252c2` | Phase 2 |
| 6 | AdvisorDescriptorRegistrar | `c25ed33` | Phase 3 |
| 7 | DispositionAwareRoutingStrategy | `fe44227` | Phase 3 |

### Tasks Remaining (8/15)

| # | Task | What it does |
|---|------|-------------|
| 8 | Advisory Worker defs in CaseDefinition | AdvisoryWorkerFactory + wire into QuarkMindCaseHub |
| 9 | Two-signal advisory trigger | Post-tick fire-and-forget signal() for advisory |
| 10 | Advisory completion lifecycle | AdvisoryCompletionObserver, AdvisoryCompleted event, invocation counter |
| 11 | Trust scoring — latency + deferred | AdvisoryLatencyRecorder, DeferredAdvisoryEvaluator, PendingEvaluation |
| 12 | Game-outcome recording | AdvisoryGameOutcomeRecorder, AdvisorExcludedEvent |
| 13 | Qhorus channel + in-loop consumption | AdvisoryChannelBroker, DroolsStrategyTask advisory reading |
| 14 | TrustRoutingPolicyProvider | Per-capability trust policies with quality floors |
| 15 | HIL coaching + integration test | AdvisoryChannelBackend, end-to-end test |

## Key Architecture Decisions Made

1. **Full engine port** (not a blocks advisory framework) — quarkmind now uses `signalAndAwaitSync()` per tick via `QuarkMindCaseHub extends CaseHub`, with `TickOrchestratorWorker` chaining plugins via `WorkerRuntime.execute()`
2. **Two-signal pattern** — tick settles synchronously, advisory triggers fire asynchronously via separate `signal()` call to avoid LLM latency blocking the 500ms tick
3. **`DispositionAwareRoutingStrategy`** replaces `TrustWeightedAgentStrategy` for advisory routing — composes trust classification with game-context disposition scoring (soft multipliers 0.8-1.2)
4. **`ImplementationRoutingStrategy`** (engine#625) replaces `StrategyTrustRouter` for competing strategy implementations — but StrategyTrustRouter/Selector/Observer NOT deleted yet (still actively used by the tick chain until full ImplementationRoutingStrategy integration)

## CDI Wiring — Hard-Won Configuration

The engine migration required extensive CDI wiring. Key non-obvious configuration in `application.properties`:
- `casehub-engine-persistence-memory` at **compile** scope (not test) — garden entry GE-20260701-4ee3fd
- `casehub-engine-scheduler-quartz` added — provides JobScheduler SPI
- `casehub-eidos-memory` added — in-memory stubs for AgentRegistry, AgentStateStore
- `quarkus.arc.exclude-types` for: JPA eidos beans, reactive qhorus beans, NoOpCapabilityHealth
- `quarkus.arc.selected-alternatives` for all engine + eidos in-memory impls
- `%test.quarkus.scheduler.start-mode=halted` (not `enabled=false`) — garden entry GE-20260701-a73f76

## Cross-Module

**engine#625** (TrustWeightedImplementationRoutingStrategy) committed on engine branch `issue-625-trust-impl-routing` — needs to be merged to engine main and published to ~/.m2 before quarkmind's ImplementationRoutingStrategy integration is complete.

## Issues Filed This Session

- **engine#625** — TrustWeightedImplementationRoutingStrategy (implemented, on branch)
- **quarkmind#207** — Engine migration Phase 2 (functionally complete, on branch)

## Garden Entries (4 submitted)

- GE-20260701-4ee3fd — engine-persistence-memory compile scope
- GE-20260701-ee19df — @PostConstruct JPA on Vert.x IO thread
- GE-20260701-82fc11 — SequenceWorker doesn't exist (it's SequentialPlanningStrategy)
- GE-20260701-a73f76 — scheduler start-mode=halted not enabled=false
