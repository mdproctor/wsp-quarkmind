---
layout: post
title: "When Marginals Lie"
date: 2026-10-25
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [bayesian, bjt, map-inference, coaching, strategy-classification]
---

# When Marginals Lie

Ask a Bayesian Junction Tree "what strategy is the opponent playing?" and it gives you marginal probabilities. RUSH 35%, MACRO 33%, TECH 32%. The classifier picks RUSH — it's the highest. But that 2% margin means nothing. No single coherent explanation of the evidence actually favours RUSH over the other two. The marginal winner is an artefact of solving each variable independently.

This is the problem with marginal inference on small networks. Each variable gets its own posterior as if the others don't exist. When one strategy clearly dominates — RUSH at 85% — the marginals are fine. But in the ambiguous middle, where the game is young and evidence is thin, marginal winners can be incoherent. The "winning" strategy might be inconsistent with the detector states that produced it.

## MAP: One Coherent Answer

MAP inference — Maximum A Posteriori — solves this by asking a different question. Instead of "what's the most likely state for each variable independently?", it asks "what's the single most likely joint assignment across all variables simultaneously?" One configuration of Strategy + detectors that best explains the evidence as a whole.

When marginals are decisive, MAP agrees with them. When marginals are ambiguous, MAP picks the configuration that actually holds together. No threshold tuning, no tiebreaker logic, no special-case handling for the ambiguous zone.

The implementation is straightforward. The BJT engine already had `mapInference()` on `BayesInstance` — I just needed to expose it through `EvidenceFusionEngine` and wire it into the detector pipeline. `DetectorOrchestrator` now returns both the marginal assessments (for confidence reporting) and the MAP strategy (for classification decisions). The cascade classifier uses the MAP winner when the fusion tier resolves. Both results flow through to the workbench.

## Telling the Coach Why

A classification alone — even a coherent one — isn't enough for coaching. The coaching AI needs to explain *why* it thinks the opponent is rushing. "Because the Bayesian network said so" isn't coaching advice.

Explanation traces solve this. After the BJT computes its posteriors, `ExplanationEngine` walks through each evidence variable and measures its individual contribution. It creates an instance *without* that variable, re-runs inference, and computes the KL divergence between the two posteriors. High divergence means that variable matters. The result is a ranked list: which detectors shifted the posterior most, and in which direction.

```java
public ExplanationTrace explain() {
    return ExplanationEngine.explain(graph, instance, targetVariable);
}
```

The trace names detectors by their actual names — `early_rush` not `Variable_3` — and reports direction (increased/decreased) alongside influence magnitude. When the coaching AI generates advice, the prompt now includes:

```
EVIDENCE REASONING: early_rush (increased, influence=0.420);
                    eco_expansion (decreased, influence=0.080)
```

Instead of "the opponent may be rushing," the coach can say "early barracks detected with high confidence, and no economic expansion — this looks like a rush." The evidence becomes the explanation.

## What This Opens Up

Two things sit behind this that I find more interesting than the features themselves.

The explanation trace runs counterfactual analysis through `CausalEngine` — computing the causal effect of each detector, not just the statistical association. A detector might correlate with RUSH because it fires at the same game phase, but causally it's the economy detector that actually drives the classification. That distinction between correlation and causation in the network is available but not yet surfaced to coaching. It should be.

And the MAP assignment doesn't just tell you the strategy — it tells you the coherent state of every detector. When the MAP says `{Strategy=RUSH, early_rush=detected, eco_expansion=not_detected, air_threat=not_detected}`, that's a complete picture. The detectors that *didn't* fire are as informative as the ones that did. A coaching AI that can say "no air threat detected, so the rush is ground-based" is reasoning about absence, not just presence.
