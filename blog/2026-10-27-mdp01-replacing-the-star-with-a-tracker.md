---
layout: post
title: "Replacing the Star with a Tracker"
date: 2026-10-27
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [bayesian, hypothesis-tracker, scouting, detection, neocortex]
series: issue-457-hypothesis-tracker-integration
---

# Replacing the Star with a Tracker

The `DetectorOrchestrator` was always a blunt instrument. A naive Bayes star — one `Strategy` parent, uniform priors, N binary detector children — and every tick, you set all evidence, propagate, marginalize, read the posterior. It works. It works surprisingly well for how little it knows. But it has a fundamental problem: it treats every tick's evidence as if it arrived fresh, with no memory of what was observed before.

The HypothesisTracker fixes this by separating two kinds of evidence. Scouting observations — a scout sees a gateway, spots early gas, finds no expansion — are one-time events. They arrive, they permanently shift hypothesis weights via `observeHard()`, and they may eliminate dead paths entirely. A hypothesis whose prior assigns zero probability to an observed building is removed from consideration. The field narrows.

Detector scores and Drools markers are different. They refresh every game tick. Calling `observe()` on them repeatedly would compound the weights exponentially — the same unchanged evidence would push the leading hypothesis toward certainty in a dozen ticks, regardless of whether anything new happened. So these go through `setPerTickEvidence()`, which sets evidence on each hypothesis's junction tree without touching the weights. The assessment formula combines both: `P(archetype) ∝ weight × perTickLikelihood`. Weights evolve only from scouting. Per-tick likelihoods are transient — recomputed each tick from current detector state, influencing the assessment without committing to the weights.

The interesting part was the prior labels. Phase 1's learning pipeline extracted archetype frequencies from 28K replays and wrote them as JSON classpath resources. The labels came out race-unqualified — `FAST_EXPAND`, `MARINE_RUSH` — because the pipeline groups by matchup. But `StrategyArchetype` enum values are race-qualified: `TERRAN_FAST_EXPAND`. The mapping needed a fallback that tries the raw label first, then prepends the race. A small thing, but the kind of mismatch that produces "all assessments are empty" with no error if you don't catch it.

The observation CPTs had a similar data boundary issue. Some archetypes in the prior set weren't present in the training data for every observation variable — sparse combinations where that strategy simply didn't appear in the replay dataset. The CPT row comes back shorter than the state count. Without bounds checking, you get an `ArrayIndexOutOfBoundsException` deep inside the network construction — no hint about which archetype or variable caused it. The fix is uniform fallback for missing entries, which is the correct Bayesian treatment anyway: no data means no information, so uniform.

Three neocortex API additions were needed to make this work — `setPerTickEvidence()`, `clearPerTickEvidence()`, and `marginalize()` on `HypothesisTracker`, plus `sensitivity()` on `CompositionInstance`. All thin delegations to existing internals; the `compositions` map is private, so the tracker needed public pass-throughs. These are on a branch, not yet pushed — Phase 3 will need them installed before the cascade wiring can proceed.

Phase 2 produced the standalone classes. Phase 3 is where the wiring happens — replacing `DetectorOrchestrator` inside `CascadingPatternClassifier`, adding a `LiveScoutingObservationExtractor` that feeds discrete observations from `GameState` during each game tick, and modifying `DroolsScoutingTask` to call it. That's a different shape of work — CDI integration, cascade consumer contracts, and calibration tests comparing old vs new accuracy on replay datasets.
