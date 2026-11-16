---
layout: post
title: "Evidence, Not Cascade"
date: 2026-10-18
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [bayesian, junction-tree, onnx, evidence-fusion, architecture, sc2]
series: issue-411-onnx-detector-library
---

# Evidence, Not Cascade

The detector library issue started as "wire 23 ONNX models into the game loop." I expected the hard part to be model loading and feature alignment. It wasn't.

The hard part was realising the architecture was wrong.

## The cascade that couldn't compose

QuarkMind's `CascadingPatternClassifier` is a sequential fallback chain: try Drools rules first, if not confident enough try a multi-class ONNX model, if still not confident enough ask an LLM. Each tier produces a single archetype classification — "it's a rush" — and you take the first one confident enough to commit.

This worked fine when the ONNX tier was a single multi-class model. But we now have 23 binary detectors, each answering a different question about the game state. A rush detector and a hidden-tech detector can both fire on the same tick, and that *combination* — aggressive opening masking a tech switch — is the insight. The cascade can't express this. It picks the first confident answer and discards the rest.

The framing was wrong. Binary detectors don't fit a fallback chain. They produce concurrent, composable probability signals, not competing classifications.

## First principles: what are we actually doing?

Multiple evidence sources observe the same game state and produce signals about the opponent's strategy. Drools rules fire discrete markers ("saw a Dark Shrine"). Binary detectors output probabilities ("rush likelihood 0.82"). These aren't alternative answers to the same question — they're complementary evidence about different aspects of the situation.

What we need is evidence fusion, not a cascade.

I have a Bayesian Junction Tree implementation — wrote it years ago for Drools, standalone inference with no framework dependencies. It does exact probabilistic inference on arbitrary graph structures. The incremental update property means only affected nodes are recomputed when new evidence arrives. For a small network (3-6 detectors per game phase plus some Drools markers), this is microsecond-fast.

Claude pushed back on whether the BJT was over-engineering. Fair challenge. For a Naive Bayes star topology — which is where we're starting — the posterior is a ten-line closed-form computation. The BJT algorithm solves a scaling problem we don't have.

But the BJT separates network structure from inference. When we add intermediate nodes to capture detector correlations (and we will — the early rush detector and mid timing-attack detector are clearly not independent), only the network definition changes. No code changes. That separation is the value, not the algorithm's sophistication.

## The practical question: will it actually work?

This is where I insisted on a validation gate before writing any Java. Claude had been building toward the elegant architecture for a while, and I wanted empirical evidence that the detectors actually contain complementary signal worth fusing. An LLM will keep going in a direction, and by the end acknowledge it was never going to work. Test early.

We wrote a Python script that runs all 23 detectors on held-out test data, computes Naive Bayes posteriors using CPTs derived from training metrics, and compares fusion accuracy against the best individual detector.

Results: fusion outperformed the best individual detector in 8 of 9 phase×matchup combinations. Average improvement: +6.8%. The mid-game phase showed the strongest gains — +9.1% for vs_terran, +10.1% for vs_protoss — which makes sense because that's where four detectors provide the most complementary signal.

The detectors are genuinely complementary. Pairwise correlations are mostly moderate (0.2-0.6), with one expected high correlation: early_cheese vs early_rush at -0.887 for vs_protoss. Those are near-opposite signals by design.

CPT calibration has room for improvement — max deltas of 0.3-0.5 between manifest metrics and actual fire rates. But the decision tree from the design review separates this cleanly: fusion works + CPTs miscalibrated means fix the parameters, not abandon the architecture.

## Extracting the BJT

With validation passing, we extracted the standalone BJT classes from `drools-beliefs` into `quarkmind-core`. 32 files total — the inference engine, graph infrastructure, and bitmask utilities. Zero external dependencies after stripping the Drools TMS integration layer (`BayesBeliefSystem`, `BayesVariableConstructor`, `VarName` — all POC-grade, not ready for production use).

The extraction hit a few snags. The `JunctionTree` class implemented `ProcessedResource` from KIE — assembly plumbing that had no business in an inference engine. `OpenBitSet` extended `BitMask` which pulled in five more BitMask subtypes. `BayesInstance` had typed POJO marshalling via reflection annotations. All stripped cleanly — the standalone API surface (`BayesNetwork`, `JunctionTreeBuilder`, `BayesInstance`) is small and self-contained.

The `EvidenceFusionEngine` wraps the BJT in a clean API: `initialize(network, target)`, `setEvidence(variable, likelihood)`, `update()`, `marginalize()`, `reset()`. Domain-agnostic — it sits in `quarkmind-core` alongside the agency SPIs, reusable across game domains.

## What's next

Three tasks remain: the `TemporalAssessmentTracker` (evidence windowing replaces exponential decay — the design review caught that decaying BJT posteriors destroys their calibration), the `DetectorOrchestrator` (per-tick pipeline wiring detectors + Drools markers into the fusion engine), and the migration that decomposes `CascadingPatternClassifier`.

The design review's most important catch: applying the existing exponential decay to BJT posteriors would break them. The posteriors are calibrated probabilities that sum to 1.0. Decay makes them sum to less than 1.0, and absolute values become meaningless for threshold-based consumers. The fix is to manage temporal stability on the evidence *inputs* (a sliding window of observations), not the inference *outputs*. The BJT always produces fresh, calibrated posteriors from bounded recent evidence.

The broader question is whether this architecture generalises beyond SC2. The fusion engine, detector selection, and temporal tracking are inherently domain-agnostic — they don't know about rushes or expansions, they know about binary situation classifiers producing evidence for Bayesian inference. AML fraud detection, clinical diagnostics, any domain where multiple independent detectors observe the same situation and you need a coherent posterior. The SC2 living lab proves the pattern; the foundation is ready for the next domain to pick it up.
