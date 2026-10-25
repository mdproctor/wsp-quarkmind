---
layout: post
title: "Teaching the Network to Count"
date: 2026-10-25
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [bayesian, parameter-learning, cpt, detector, mle]
series: issue-450-detector-cpts-learned-from-data
---

# Teaching the Network to Count

The BJT network now has MAP inference and explanation traces. It can give a coherent joint classification and tell the coach exactly which detectors shifted the posterior. There's one problem: the probabilities driving all of this are made up.

Every detector in the network uses the same CPT: 0.8 if it's a positive indicator for a strategy, 0.2 if it isn't. The early rush detector and the mid-game air tech detector have identical conditional probability tables despite detecting fundamentally different things with different accuracy profiles. A detector that fires on 90% of actual rushes gets the same weight as one that fires on 55%.

This matters more than it sounds. The MAP inference I wrote about in the last entry finds the single most coherent joint assignment across all variables. But if the CPTs feeding that inference are uniform guesses, "most coherent" is doing arithmetic on fiction. The network has the right topology — the right questions — but the wrong answers.

## From Guesses to Counts

We have 149K labelled replays with strategy labels from the heuristic classifier, and ONNX detector models trained on this data. The neocortex library already has `ParameterLearner.fit()` — closed-form MLE with Laplace smoothing. The inputs exist. The machinery exists. The gap was a pipeline to connect them.

The pipeline has two stages. A Python exporter runs each ONNX detector on the training data — the same pre-computed feature vectors the detectors were trained on — and discretizes the raw scores into observation records pairing strategy labels with detector outputs. A Java test then reads those observations, builds the naive Bayes structure, and calls `ParameterLearner.fit()` to produce the learned network. The result is serialized as `NetworkSpec` JSON and dropped onto the classpath where `LearnedCptLoader` picks it up at startup.

## Three Buckets, Not Two

The original plan was binary discretization: detector output ≥ 0.5 maps to HIGH, below to LOW. Simple. But the production inference path doesn't use binary evidence — `DetectorOrchestrator.classify()` feeds continuous soft evidence to the BJT. A detector scoring 0.95 produces very different posteriors from one scoring 0.51, even though both would be "HIGH" in training.

Three buckets — LOW, MEDIUM, HIGH — capture the distinction between confident detection, borderline detection, and confident absence. The learned CPTs model P(HIGH | RUSH), P(MEDIUM | RUSH), P(LOW | RUSH) separately. At inference time, a piecewise linear function maps the continuous score to a 3-element likelihood vector, preserving the magnitude information that binary discretization destroys.

## The Fallback Question

The spec's original acceptance criterion said "hand-tuned CPTs used if learned data unavailable." I pushed back on this during design. If the uniform 0.8/0.2 values were good enough, we wouldn't need parameter learning at all. Silently falling back to known-bad CPTs is worse than failing — the system looks like it's classifying, but the numbers driving the classification are fiction. No error, no warning, just quietly wrong.

The compromise: profile-gated fallback. In production profiles (`%sc2`, `%emulated`, `%coach`), missing learned CPTs throw an `IllegalStateException` at startup. Classification quality matters in those contexts and silent degradation is dangerous. In dev and test profiles, the fallback fires with a WARN log — classification accuracy is incidental there and hard failures would just add friction.

## What the Data Says

521K observations across nine matchup-phase combinations. The learned priors alone are worth the effort — they expose how badly uniform priors misrepresent reality.

Early game vs Zerg: RUSH accounts for 79% of the prior mass. MACRO_ECONOMY takes 20%. The hand-tuned network split the prior evenly across all archetypes, treating macro play as equally likely as a zergling rush in the first two minutes. It isn't. Early game vs Protoss: PROXY dominates at 62%. The early game in Protoss is structurally skewed toward proxy-style builds, and the network now reflects that instead of pretending all openings are equiprobable.

Mid game vs Terran is the most striking. TECH_RUSH takes 58% of the learned prior — the mid-game Terran landscape is dominated by tech-oriented play. The hand-tuned network had no way to know this.

The validation test runs both networks — learned and hand-tuned — against a held-out test split, comparing MAP accuracy. Learned CPTs outperform hand-tuned across all nine matchup-phases:

| Phase | Matchup | Learned | Hand-tuned | Delta |
|-------|---------|---------|------------|-------|
| EARLY | vs Protoss | 72.3% | 47.2% | +25.1 |
| EARLY | vs Terran | 57.5% | 46.8% | +10.7 |
| EARLY | vs Zerg | 88.8% | 13.1% | +75.7 |
| MID | vs Terran | 95.1% | 75.7% | +19.4 |
| LATE | vs Terran | 69.2% | 24.9% | +44.2 |

The early vs Zerg number is the most dramatic: 88.8% vs 13.1%. The hand-tuned network was essentially random — uniform priors and uniform CPTs made it incapable of exploiting the overwhelming RUSH signal in early Zerg games. The learned network picks it up immediately because the data tells it that 79% of early Zerg games look like rushes and the rush detector fires HIGH on most of them.

9/9 improved. Not a single matchup-phase where hand-tuned wins. The guesses were consistently worse than counting.
