# Bringing the Engine Home

QuarkMind has been living a double life. The game loop — scouting, strategy, tactics, economics, all ticking every 500ms — ran on `casehub-poc`, a retired proof-of-concept engine that predated everything the CaseHub platform became. The trust routing, the plugin dispatch, the strategy selection — all of it was wired through an engine we'd already replaced everywhere else.

I'd been putting off the migration. The poc worked. The game loop was stable. Why risk breaking 1059 tests for architectural purity?

The answer arrived when I started designing #180 — the LLM advisory team. Three advisory roles (crisis, strategic, economic), each backed by competing LLM configurations selected via trust-weighted routing. The design needed `AgentRoutingStrategy`, `TrustWeightedAgentStrategy`, eidos `AgentDescriptor` with disposition traits, the whole platform trust model. And all of it assumed the game was an engine case.

## The wrong first answer

I initially proposed building an `AdvisoryBroker` in `io.casehub.blocks.advisory` — a standalone advisory framework pre-positioned for extraction to casehub-blocks. It would have its own trust-weighted selector, its own invocation SPI, its own outcome evaluator. Clean, isolated, portable.

It was also wrong. PLATFORM.md is explicit: "Do not implement implementation-selection trust routing in application repos." The engine's `AgentRoutingStrategy` already does trust-weighted selection. Building a parallel framework was creating the exact duplication the platform coherence protocol exists to prevent.

The turning point was checking engine#490. I'd assumed the engine couldn't handle QuarkMind's synchronous, ordered, 500ms tick — Workers were async, right? Wrong. `signalAndAwaitSync()` provides synchronous bulk context update with settlement detection. `SequentialPlanningStrategy` (confusingly closed under the name "SequenceWorker" in engine#484, though no such class exists) provides ordered execution. `ImplementationRoutingStrategy` handles competing implementations. All four prerequisites had shipped.

## The migration

The port removed the poc entirely — about 5000 lines deleted. `QuarkMindCaseHub extends CaseHub` defines a programmatic `CaseDefinition`. `TickOrchestratorWorker` chains plugins via `WorkerRuntime.execute()` inside a single `WorkerFunction.Sync`. Each tick calls `signalAndAwaitSync()` to update game state and wait for all triggered Workers to settle.

The CDI wiring was the hard part. Adding the engine runtime to quarkmind's classpath brought 33 unsatisfied SPI dependencies. The fix required `casehub-engine-persistence-memory` at compile scope (not test — Quarkus augmentation runs before test classpath loads), `casehub-engine-scheduler-quartz` for the `JobScheduler` SPI, `casehub-eidos-memory` for in-memory agent registry stubs, and a careful `quarkus.arc.exclude-types` list for eidos JPA beans and qhorus reactive services that quarkmind doesn't need.

A subtler issue: the engine's `DefaultCaseDefinitionRegistry` runs `registerKnownDefinitions()` on a Vert.x IO thread. Any CDI bean transitively discovered during that callback that does JPA work in `@PostConstruct` — like `ScoutingIntelBroker` creating its Qhorus channel — hits `BlockingNotAllowedException`. The fix was deferring channel creation to lazy init with double-checked locking.

## Advisory architecture

With the engine port done, the advisory design fell into place naturally. LLM advisors are engine Workers — each has an eidos `AgentDescriptor` with model family, disposition traits, and capability declarations. The engine's `AgentRoutingStrategy` selects among them using `TrustWeightedAgentStrategy` (trust scores) composed with a quarkmind-specific `DispositionAwareRoutingStrategy` (game-context-dependent multipliers: prefer conservative advisors against aggressive opponents, bold advisors against economic ones).

The timing problem — LLM calls take 1-5 seconds, ticks are 500ms — is solved by a two-signal pattern. The tick orchestrator settles synchronously via `signalAndAwaitSync()`. A subsequent fire-and-forget `signal()` propagates advisory triggers. Advisory Workers run asynchronously; results appear in `CaseContext` whenever the LLM responds. Strategy plugins read them on the next tick if available. Advisory is additive signal — the game loop never blocks on it.

Trust scoring uses four dimensions: response latency (immediate), recommendation quality (deferred evaluation against game state delta), and game outcome (end-of-game attribution). Each maps to a `CAPABILITY_DIMENSION` score in casehub-ledger. Quality floors exclude advisors that are fast but wrong, or right but slow.

## What's left

The advisory Workers themselves — wiring them into the `CaseDefinition`, the trigger dispatch, the completion lifecycle, the Qhorus advisory channel, the in-loop consumption by `DroolsStrategyTask`. Eight tasks remain of fifteen. The foundation is in place; what's left is plumbing.

The more interesting question is what this enables. The commentator (#181) and the enemy strategy classifier (#183) can now be Workers in the same case, triggered by the same event hierarchy, scored by the same trust model. The game becomes a live A/B testing environment for LLM configurations — something the regulated-domain harnesses (AML, clinical) can't offer at the same cadence.
