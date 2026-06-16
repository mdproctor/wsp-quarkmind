---
layout: post
title: "Chapter Six Closes"
date: 2026-06-16
type: phase-update
entry_type: note
subtype: diary
projects: [QuarkMind]
tags: [casehub, benchmark, arc42, layer7]
---

Chapter 6 of the Game Agent Coordination journey is closed. L7 wasn't a new capability — it was the question I'd been putting off since the first layer: what does the harness actually cost, and does it close what L1 couldn't?

The answer is yes on both counts, and the numbers are more interesting than I expected.

**The harness cost is negligible**

I brought Claude in to measure it properly. With MockEngine (the `%test` profile, dispatch only), we measured: mean 0ms, P99 2ms, max 2ms. The harness adds essentially nothing. `createAndSolve()` dispatching four plugins against the CaseFile blackboard costs 2ms at worst. The quality requirement is P99 < 400ms — there's 398ms of headroom. The aspirational question — could the harness run at native SC2 physics rate (22Hz, 45ms/tick) — is also answered: yes, with margin.

**EmulatedGame is much faster than I expected**

The second benchmark was harder to set up. My first attempt — adding a measurement to `GameLoopBenchmarkTest` — reported all zeros and Infinity ticks/sec. Claude flagged it: `GameLoopBenchmarkTest` is `@QuarkusTest`, which forces `%test` profile, which loads MockEngine. EmulatedEngine is `@IfBuildProfile("emulated")` and never activates there. Every "EmulatedGame" measurement was measuring nothing.

The fix was a separate plain JUnit class in `io.quarkmind.sc2.emulated` — that package for package-private access to `RaceModelFactory.forRace()` and `EmulatedGame.enemy`. The class mirrors `EmulatedEngine.joinGame()` exactly: terrain, pathfinding, Protoss race model, PROTOSS_4GATE enemy AI. 120 warmup ticks past the first attack trigger, 50 measured ticks in steady-state combat.

Result: 15,571 ticks/sec. EmulatedGame runs at roughly 750× the 22Hz ceiling of a real SC2 game. The worst-case tick was 138µs — 0.3% of the 45ms per-tick envelope. Physics isn't a constraint.

(One more thing worth noting: I initially measured in milliseconds. Since EmulatedGame ticks complete in ~60µs, every sample was 0ms. Copying benchmark templates from non-physics code will silently give wrong results. Microseconds are the right unit here.)

**The capability table revealed a gap**

The §9.4 comparison table — L1 naive, uncoordinated SC2 API bot, QuarkMind L7 — was supposed to document what the harness delivers. Claude caught a problem in my first draft: the "Formal out-of-scope signal" row claimed "Structured DECLINE speech act (L3)" as a delivered QuarkMind capability. But `MessageType.DECLINE` doesn't appear anywhere in the production game-loop code. QuarkMind uses `MessageType.STATUS` for scouting intel. DECLINE is defined in casehub-qhorus; it isn't wired in the plugin dispatch path.

A capability the platform provides and the application could use is not the same as a capability the application delivers. The corrected row names the distinction explicitly: typed inter-plugin channels are delivered (L3, STATUS messages); DECLINE is platform-defined, not yet wired.

The same issue applied to the inter-plugin state row. "Shared CaseFile read/write within tick" implied symmetric access that doesn't exist. What actually happens: `GameStateTranslator` pre-populates the CaseFile before `createAndSolve()`, `DroolsStrategyTask` writes one decision key back, downstream plugins read from the pre-populated state. One writer. The row was overstating.

Neither is a bug. They're gaps between what the platform offers and what this application has chosen to wire.

**Win-rate comparison deferred**

The §9.4 stub had committed to documenting win-rate delta alongside latency delta. I dropped it, and said why explicitly rather than leaving a silent gap. L1 was never deployed — it's a conceptual baseline for naming accountability gaps, not a runnable harness to benchmark against. Comparing to a different SC2 bot would measure strategy quality, not harness contribution. And the trust routing needs a meaningful game corpus for Bayesian Beta scores to converge; that corpus doesn't exist yet.

The comparison that matters is answered. Win-rate waits for real games.

**The chapter closes**

Six chapters, L1 through L7, from conceptual baseline through blackboard dispatch, typed messaging, outcome tracking, adaptive selection, trust routing, and now the validation close. The C2–C5 chapter narrative entries in ARC42STORIES.MD are still stubs — the layers are complete, the entries aren't written ([#198](https://github.com/casehubio/quarkmind/issues/198)). That's next.

Phase 2 of the casehub-engine migration remains blocked on engine#483 and engine#484. When those land the benchmark numbers will shift — the Phase 1 bridge adds indirection that Phase 2 removes. Worth re-running.
