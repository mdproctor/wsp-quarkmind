---
layout: post
title: "Milestone attestations for advisors — same signal, different eligibility"
date: 2026-08-14
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [trust, milestones, advisors, dominance]
---

The milestone trust scoring infrastructure from #191 evaluates strategies at game checkpoints — frame thresholds and phase transitions. A strategy dominating at the 3-minute mark gets partial credit even if the game is lost late. The same logic applies to advisors, with one twist: strategies are selected at game start, but advisors arrive mid-game at unpredictable frames.

An advisor invoked at frame 5000 shouldn't receive an attestation for a milestone at frame 4032. It didn't exist yet. The obvious fix — an explicit frame comparison — turns out to be unnecessary. The `AdvisoryInvocationCounter` only contains advisors that have already completed. If `evaluateMilestones()` runs at frame 4032, the counter's `snapshot()` naturally excludes any advisor invoked later. Temporal ordering handles eligibility for free.

The more interesting discovery was in the trigger infrastructure. `MilestoneTrigger.check()` takes a `MilestoneSession` — a concrete class, not an interface. It looks like triggers are stateful, but they're not. All fired-state tracking is delegated to the passed session object. The trigger itself only holds configuration (frame thresholds, phase weights). This meant we could share the exact same trigger instances between strategy and advisory milestones by extracting a `MilestoneTracker` interface with `hasFired()` and `markFired()`. Both `MilestoneSession` (strategy) and the new `AdvisoryMilestoneSession` (advisory) implement it. The triggers don't care which they receive.

The recording lifecycle mirrors the strategy side: single ledger entry per advisor per game, first milestone creates it via `recordAndReturnId()`, subsequent milestones and game-end append via `addAttestation()`. This preserves `decisionCount = games played` — critical for the `minimumObservations` threshold that gates trust routing out of BOOTSTRAP phase.

`AdvisoryMilestoneOutcomeRecorder` replaces `AdvisoryGameOutcomeRecorder` entirely. One class owns both milestone evaluation and game-end recording, avoiding the coordination complexity of split entry ownership. The confidence formula is identical to strategies: `temporalWeight × |dominanceScore|`. Advisory LLM confidence isn't mixed in — that signal is already captured separately by `DeferredAdvisoryEvaluator` in the `recommendation-quality` dimension. Dominance trajectory and recommendation quality are independent measurements of different things.

What this opens up: the trust pipeline now has per-checkpoint resolution for both strategies and advisors. An advisor that consistently correlates with dominant positions earns trust faster than one that happens to be active during lucky wins. The next step would be proportional game-end attribution for advisors invoked at different times — analogous to #225 for strategy pivots — but the current equal-weight model is a reasonable starting point.
