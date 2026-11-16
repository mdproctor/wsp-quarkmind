---
layout: post
title: "Two Delivery Paths"
date: 2026-06-08
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [layer3, qhorus, architecture, scouting, llm-advisory]
---

Layer 3 shipped a Qhorus channel for scouting intel delivery, and it was the right idea pointed in the wrong direction.

The channel was designed so `DroolsTacticsTask` could receive typed threat intel — position, posture, timing — each tick. But Qhorus isn't a game-speed transport. It's a cross-process, async, ledger-audited message bus for agent communication. Every intel update meant a JPA write, a JSON serialisation round-trip, and a mandatory 1-tick delivery lag. The machinery was architecturally sound; the application of it was not.

The redesign separates two concerns that were being conflated.

**For plugins:** `ScoutingIntelBroker` becomes a typed in-memory store. `DroolsScoutingTask` calls `broker.update(new ThreatPosition(nearest))` on the same game-loop thread, in-process. Tactics, strategy, and economics read from it in their `execute()` — synchronous, zero lag, no serialisation. The broker is the fast path.

**For observers:** the Qhorus channel stays. LLM advisors — a crisis-response model that fires on timing alerts, an economics optimizer watching army size — will subscribe as `MessageObserver` implementations. The advisory layer operates at a different timescale from the game loop; async delivery fits. Trust-weighted selection among LLM configurations, scored by response latency and advice quality over time, connects directly to the routing work in Layer 6.

I hadn't planned the LLM advisory layer at the start of this session. It surfaced as a consequence of thinking through why Qhorus was wrong for one use case but right for another. A sports commentator was the trigger: an LLM narrating what the agent is doing, coaching a human watching the game, personality-aware enough that different commentator configurations compete on engagement quality. That led to the question of how trust-weighted routing handles latency as a selection criterion. The game loop is an unusually clean test environment for all of this — outcomes are measurable, fast, and objective in a way that clinical or AML applications aren't.

That branch went into four separate issues (#180–#183) rather than implementation. What actually shipped: the broker migration for Tactics, new `ScoutingIntelConsumer` implementations on Strategy and Economics, `NEAREST_THREAT` removed from the CaseFile entirely, and two hot-reload endpoints for subscription and threshold refresh without restart.

Two small details that bit during implementation. `activeTypes` on the broker must be initialised to `Set.of()` at field declaration, not in `@PostConstruct` — otherwise `new ScoutingIntelBroker()` in plain JUnit tests gets a null field and `isSubscribed()` throws. And `clearLatest()` had to be public: a CDI proxy doesn't magically expose package-private methods to test classes in different Java packages. Both compile cleanly until you try to write the test; neither has a warning sign beforehand.
