---
layout: post
title: "Advisors on the Wire"
date: 2026-07-02
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [llm, advisory, trust-routing, eidos, disposition, casehub-engine]
series: issue-180-llm-advisory-scouting
---

The engine port landed. QuarkMind now runs on casehub-engine — signalAndAwaitSync per tick, TickOrchestratorWorker chaining plugins via WorkerFunction, the poc dependency gone. That was the prerequisite. The point was what comes after: LLM advisors that the engine dispatches, scores, and replaces based on trust.

Six advisor configurations sit in eidos — two per role (crisis, strategic, economic), each with different disposition traits. An aggressive crisis responder is bold and flexible; a conservative one is measured and strict. The engine selects which advisor handles each request via DispositionAwareRoutingStrategy, which composes trust classification with game-context disposition scoring. When the enemy is aggressive, the conservative advisor scores higher — the system learns to match personality to situation.

The two-signal pattern keeps the 500ms tick budget. The synchronous tick settles plugins (scouting, strategy, tactics, economics). Then a second fire-and-forget signal triggers advisory bindings. Advisory Workers run asynchronously on virtual threads. Their responses arrive whenever the LLM finishes — potentially multiple ticks later. The strategy plugin reads advisory output from CaseContext on the next tick, checks staleness, and factors it into Drools decisions alongside existing Level 1 intel.

Trust scoring runs across three dimensions. Latency is scored immediately on response — normalised against per-role thresholds (crisis has 2 seconds, strategic gets 5). Recommendation quality is deferred by 200 frames: did the game state improve after the advisor's recommendation? Game outcome attribution writes a flat attestation per invoked advisor at game end. An advisor that's fast but gives bad recommendations fails the quality floor and enters EXCLUDED phase — the system finds the configuration that's both fast enough and smart enough.

The final whole-branch review caught an infinite recursion: AdvisoryCompletionObserver was re-firing the same CDI event it observes. The other three observers (latency, deferred evaluation, channel broker) already receive the event from the original fire — the re-fire was unnecessary and would have caused a StackOverflowError on first advisory completion. Also caught: snapshot() returning a live view instead of a copy, and an unsynchronized ArrayList in the deferred evaluator.

What's left is runtime verification. No LLM runs in test profiles — the integration test proves the wiring is correct up to the ChatModel boundary. The first real advisory dispatch happens when someone adds a LangChain4j Anthropic provider to the classpath and configures an API key. The trust model starts in BOOTSTRAP phase and converges from there.
