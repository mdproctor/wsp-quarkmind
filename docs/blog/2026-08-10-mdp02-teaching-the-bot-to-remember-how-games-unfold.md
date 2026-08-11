---
title: "Teaching the Bot to Remember How Games Unfold"
date: 2026-08-10
entry_type: note
subtype: diary
tags: [cbr, case-enrichment, game-experience, observability]
issues: ["#215", "#217", "#218", "#219", "#220", "#221"]
epic: "#214"
projects: [quarkmind]
status: draft
---

# Teaching the Bot to Remember How Games Unfold

QuarkMind's CBR system could already remember *who won* — strategy X beat archetype Y. What it couldn't remember was *how the game played out*. The phase sequence, the dominance shifts, whether scouting was early or late, how many expansions were running at close. All the texture that separates a lucky win from a decisive one.

We built the Tier 2 enrichment layer across six domains. The core idea is simple: the `SC2CbrRetentionObserver` subscribes to the event summarisation buses (moments, phases, arcs) and accumulates data throughout the game. At close, it reads both the CaseFile snapshot and its own accumulated state, computes aggregate metrics, and builds a fat case with ~20 features instead of 4.

The design decision that shaped everything: one enriched case per game, not separate case types per domain. A game *is* one thing. Splitting economics, tactics, and scouting into separate `sc2-economics` and `sc2-tactics` case types would have created a cross-type join problem at retrieval time. With a single enriched `sc2-strategy` case, "find games similar to this one" stays as one `CbrQuery` call. Neocortex handles sparse features natively — old Tier 1 cases with missing enrichment fields score neutrally.

The bus subscription pattern already existed in the codebase. `MultiFactorDominanceAssessor` subscribes to `phaseBus` for phase-aware weight adjustment. `AdvisoryInvocationCounter` accumulates advisor IDs and clears on `GameStarted`. We followed both patterns: `CopyOnWriteArrayList` accumulators, `AtomicReference` for the latest arc, clear on `@Observes GameStarted`. No new abstractions needed.

Three pieces of instrumentation went in alongside the enrichment:

**Supply block detection.** `SUPPLY_BLOCK` existed in `GameMomentType` but no Drools rule produced it. We added `supplyUsed` and `supplyCap` fields to `MomentDetectionRuleUnit`, a rule at salience 130, and 224-frame cooldown dedup to avoid firing every tick during a block.

**Scout dispatch tracking.** `DroolsScoutingTask` now records the frame of its first scout dispatch via a `scoutFirstDispatchFrame` field, written to a CaseFile key. The observer converts it to minutes for the case feature.

**CBR influence tracking.** `SC2StrategyRouterTask` now calls `select()` twice — once with CBR experiences (the real selection) and once with empty experiences (trust-only baseline). If the winners differ, `CBR_INFLUENCED_SELECTION` is written as true. This feeds the key validation metric: what fraction of games would the bot have played differently without CBR?

The observability endpoints (`/qa/cbr/learning-curve`, `/qa/cbr/strategy-evolution`, `/qa/cbr/case-stats`) compute everything on demand from the case store. No pre-computed aggregates, no mutable state. The case base is small enough — under 1000 games — that query-time computation is fast. `retrievalInfluenceRate` is the number I'll be watching: if CBR never changes the routing decision, the entire infrastructure is overhead.

Three items were deferred and filed as follow-up issues: posture correctness evaluation (#269 — needs a definition of "correct"), per-engagement outcome tracking (#270 — `BATTLE_ENDED` detection doesn't exist yet), and SC2 mode opponent identity (#271 — needs protocol layer changes). Each was small enough to pull in but depended on instrumentation outside the CBR layer.

The schema grew from 4 fields to 21. Weights intentionally sum to 1.2 — neocortex normalises — with archetype still dominant at 0.35. The balance will need calibration once enough games accumulate for retrieval to matter. Right now it's infrastructure; the proof comes when `retrievalInfluenceRate` climbs above zero.
