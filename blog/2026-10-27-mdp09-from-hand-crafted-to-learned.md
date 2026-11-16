---
layout: post
title: "From Hand-Crafted to Learned: Teaching the Scouting Pipeline What 28,000 Replays Already Know"
date: 2026-10-27
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [hypothesis-tracker, bayesian, scouting, priors, strategy-detection, neocortex]
---

# From Hand-Crafted to Learned: Teaching the Scouting Pipeline What 28,000 Replays Already Know

The scouting pipeline detects what the opponent is doing. A three-tier cascade — Drools rules for high-confidence pattern matching, ONNX binary detectors fused through a Bayesian network, LLM fallback when both tiers are uncertain — produces a ranked list of strategy assessments each game tick. It works. But the Bayesian fusion layer has a problem I've been ignoring: the priors are made up.

When `SC2NetworkBuilder` constructs the fusion network, every strategy archetype gets an equal prior — `1.0 / N`. Twenty archetypes, each at 5%. That's saying a cannon rush is exactly as likely as a macro expand before any evidence arrives. In real SC2, roughly 80% of games open with a macro build. The uniform prior forces the detector to work against base rates instead of with them.

## Why Priors Matter More Than They Look

A uniform prior is a statement of total ignorance. Sometimes that's honest. Here it's lazy — we have 149,000 labelled replays sitting in the classifier pipeline. The base rate of every strategy archetype, per matchup, per game phase, is knowable. We're just not using it.

The damage is subtle. A detector that returns 0.6 confidence for "rush" against a uniform prior of 0.05 produces a posterior that's dominated by the likelihood. The same 0.6 against a learned prior of 0.15 (rush IS more common than the uniform suggests) produces a meaningfully different posterior — and against a learned prior of 0.01 (rush is rare in this matchup), it's different again. The prior doesn't just shift the needle; it determines how much evidence the detector needs to overcome the base rate. Get it wrong and you're either too trigger-happy or too slow to react.

This is where neocortex's `HypothesisTracker` enters the picture.

## The HypothesisTracker Isn't Just for Scouting

I'd been thinking of the `HypothesisTracker` as a scouting detection replacement — a better fusion layer for the existing cascade. That's too narrow. The tracker maintains competing hypotheses over a composed Bayesian network, updating weights as observations arrive and eliminating dead paths when evidence rules them out. It's a general-purpose decision tool.

The same mechanism that tracks "what is the opponent doing?" can track "what should our response be?" — and the two compose. A composed network where opponent detection feeds response selection, with learned priors on both sides, driven by the same observations. The opponent tracker narrows from 20 hypotheses to 3 as scouting data arrives; the response tracker adjusts strategy recommendations as the opponent assessment shifts.

The design review surfaced a critical subtlety here. The `HypothesisTracker.observe()` API cumulatively multiplies hypothesis weights by the observation likelihood on each call. That's correct for one-time observations — a scout spots a gateway, and that information permanently updates beliefs. But detector scores refresh every game tick. Calling `observe()` 22 times per second with unchanged evidence would exponentially converge on the best-fitting hypothesis regardless of whether new information arrived.

The fix is a dual-path architecture: one-time scouting observations go through `observeHard()` and update weights permanently. Per-tick detector and Drools evidence goes through a separate path that sets evidence on the junction tree without multiplying weights — the per-tick likelihood is computed transiently for assessment derivation, not committed to the hypothesis weights. Between scouting events, weight ratios stay constant; detector evidence shifts the adjusted posterior transiently. This preserves the current `DetectorOrchestrator` behaviour where strong detector evidence alone can push a hypothesis above the cascade's confidence threshold.

## Discrete Observations, Not Feature Vectors

The existing classifier pipeline produces continuous feature vectors — 53 building counts, 56 unit counts, 13 economy stats, per second, per player. That's the right input for a CNN. It's the wrong input for a Bayesian network, which operates on discrete named variables with categorical states.

The `ScoutingObservationExtractor` bridges this gap. It reads the same `game_json` that the classifier pipeline consumes, but instead of producing continuous per-second arrays, it produces discrete events: `first_building = barracks`, `gas_timing = early`, `tech_path = factory`, `army_composition = mixed`. Seven variables, each with a handful of named states. These map directly to the `HypothesisTracker`'s `observeHard("intel", "first_building", "barracks")` API.

The discretization thresholds — is gas at loop 2016 "early" or "normal"? — are design choices that affect sensitivity. I calibrated from replay data rather than intuition, which is also what the `sc2data-train-times-require-calibration` protocol requires. The boundaries aren't sacred; they're a starting point. The calibration tests will tell us where they need adjusting.

## What the Data Says

Running the pipeline against the reconstituted replay datasets produces the priors the fusion layer has been missing. The headline numbers match what any SC2 player would expect:

Against Terran in the early game, 79% of the prior mass is on FAST_EXPAND. Marine rush and proxy barracks each get about 9%. Banshee harass and reaper harass are in the noise — less than 1%. This is the metagame, quantified. A fusion layer starting from this distribution needs strong evidence to classify a game as a rush, which is correct — rushes are uncommon, and false positives are expensive.

The observation CPTs — P(first_building = barracks | RUSH) vs P(first_building = barracks | FAST_EXPAND) — encode the structural fingerprints of each strategy. A rush archetype has high probability on early barracks and early gas; a macro expand has high probability on command center expansion and late factory. The CPTs are learned with Laplace smoothing and clamped to [0.01, 0.99] to prevent zero-probability elimination — a single zero entry in a CPT would cause the `HypothesisTracker` to hard-eliminate that hypothesis on the first observation, which is too aggressive when the training data is sparse.

## What This Opens Up

The learned priors and CPTs are Phase 1 of a four-phase integration. Phase 2 builds the `SC2HypothesisNetworkBuilder` and `HypothesisTrackerFusion` — the replacement for `DetectorOrchestrator` that uses these learned parameters. Phase 3 wires it into the live cascade and adds a `LiveScoutingObservationExtractor` that produces discrete observations from `GameState` during play. Phase 4 adds response selection — the composed network where opponent detection feeds our strategy choice.

The interesting question isn't whether learned priors beat uniform priors. They obviously will. The interesting question is whether discrete scouting observations add signal beyond what the ONNX detectors already provide. The detectors already see the full continuous feature space. Discrete observations are a lossy compression of the same data. But they operate at a different granularity — "first building was a gateway" is a structural fact about the game, not a statistical feature. The Bayesian network reasons over structural facts; the CNN reasons over statistical features. If they're capturing different aspects of the same evidence, the combination should be stronger than either alone.

The calibration tests will answer that concretely. That's the point of Phase 3.
