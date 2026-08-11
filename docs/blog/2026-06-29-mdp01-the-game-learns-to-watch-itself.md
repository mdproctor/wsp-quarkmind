---
layout: post
title: "The Game Learns to Watch Itself"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [temporal-abstraction, drools-cep, event-hierarchy, casehub-blocks]
---

Until now, QuarkMind's intelligence layer lived entirely at Level 1: Drools CEP fires on raw tick data, producing posture classifications and timing alerts. The strategy plugin reads those signals and decides DEFEND, ATTACK, or MACRO. It works — but it's the equivalent of a chess engine that can see individual pieces without understanding the position.

The hierarchy changes that. Four levels of temporal abstraction, each summarising the one below:

```
Level 0 — Raw ticks (22.4/sec)
Level 1 — Intel events (posture, timing, build order)
Level 2 — Moments (BATTLE_STARTED, NEXUS_UNDER_ATTACK, ECONOMIC_CRISIS)
Level 3 — Phases (EARLY_MACRO, MID_SKIRMISH, DEFENSIVE_HOLD)
Level 4 — Narrative arcs ("Economic advantage for 4 minutes, converting to army")
```

Level 2 moments are the interesting part. A `BATTLE_STARTED` isn't just "enemies near base" (that's Level 1's `TIMING_ALERT`). It's a pattern across multiple Level 1 signals — army size spiking, threat position closing, posture shifting — detected by a second Drools CEP rule unit firing on the Level 1 event stream. The strategy plugin can now reason about what's *happening* rather than what's *present*.

## The seam that matters

I wanted the generic temporal abstraction framework — `EventAccumulator`, `EventStreamBus`, `Summariser`, `SummarisationRunner` — to live in a package that would migrate cleanly to `casehub-blocks` later. So it sits in `io.casehub.blocks.summarisation` inside quarkmind, with zero CDI imports and zero SC2 knowledge. The SC2 application layer (`io.quarkmind.plugin.summarisation`) wires in Drools, the Qhorus channel, and the CDI consumer bridge. When the time comes, IntelliJ moves the package — imports don't change.

The design decision that took the most thought was where to draw the `Summariser` interface boundary. Levels 0→2 use Drools CEP — stateful, event-driven, temporal windowing. Levels 2→4 use batch summarisation — "here's a window of moments, classify the phase." These are genuinely different execution models. I tried to unify them under one `LevelTransition` interface and it leaked everywhere. The honest boundary is at Level 2: Drools publishes *into* the generic hierarchy at that point, but doesn't pretend to go through the generic interface.

## The bug the spec review couldn't predict

The adversarial design review caught 14 issues in the spec — missing plugin seam patterns, undefined tick loop integration, broken data flow assumptions. All real, all fixed before implementation. But the bug that actually bit was invisible to static review.

`EventStreamBus.clear()` removes all subscriptions. When called from a `GameStarted` handler to "reset state between games," it drops every callback registered in `@PostConstruct`. The L1→L2 pipeline silently dies after the first game restart — events published to an empty subscriber list, no error, no log. The mental model ("clear resets the component") is intuitive but wrong for pub/sub: a bus has no cached state to reset. Its state *is* its subscriptions, and those are tied to CDI bean lifecycle, not game lifecycle.

The fix is to not call `clear()` at all. Only `EventAccumulator` (which actually buffers events) needs resetting between games.

## What comes next

The summarisers are deterministic stubs — rule-based phase classification, template-based narratives. They prove the pipeline shape. When #180 (LLM advisory team) lands, the `Summariser` interface will need to go async. That's an API shape change I'd rather make before extracting to `casehub-blocks`, not after. So the generic layer stays in quarkmind for now, and the migration happens once the async contract is settled.

Three issues are now unblocked: #180 (LLM advisors can subscribe to Level 2/3 summaries instead of raw ticks), #181 (commentator can trigger on moments), and #183 (enemy strategy classifier can consume the phase abstraction). The infrastructure is the shared foundation — each consumer adds its own `MomentConsumer` implementation and subscribes.
