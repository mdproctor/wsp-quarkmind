---
title: "Wiring the detection stack"
date: 2026-10-24
author: mdp
tags: [bjt, detection, onnx, neocortex, workbench]
refs: ["#449", "#445", "#446", "#443", "#448"]
epic: "#453"
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
---

# Wiring the detection stack

Phase 1 of the SC2 Intelligence Stack epic is about making the existing detection pipeline cheaper without changing what it detects. Five issues, all in the detector layer, all wiring neocortex BJT capabilities that already existed but weren't connected.

The ordering mattered. Builder API first (#449), because it turns the Bayesian network construction from procedural node-and-edge wiring into a declaration — `variable("Strategy", archetypes).prior(priors)` then `.variable(detName, "HIGH", "LOW").given("Strategy").cpt(flatCpt)` for each detector. The topology reads as a statement, not an imperative sequence. That clean foundation made the next four changes simpler to reason about.

Batch evidence (#445) replaced the reset-loop-set-update cycle with a single `setAllEvidence(map)` call. The BJT's `batchSetEvidence` handles retraction internally — if a variable had evidence last tick but not this tick, it's unset automatically. One fewer thing for the caller to manage.

Hard evidence (#446) was the one that surprised me. I expected `setHardEvidence` to be a fundamentally different code path — clamping a variable, skipping it during propagation, special retraction logic. It's not. Under the hood, hard evidence is a degenerate likelihood distribution: `[1.0, 0.0]` for HIGH, `[0.0, 1.0]` for LOW. It flows through the same `setLikelyhood` path as soft evidence. That meant no API change on the fusion engine — Claude and I just built degenerate distributions directly in the evidence map when a detector output exceeds the threshold (default 0.95). Same batch call, same propagation pass, same retraction. The only difference is the distribution shape.

Sensitivity scheduling (#443) was the M/Med piece, and the one where the design needed the most thought. The BJT's `sensitivity()` method computes KL divergence between posteriors with and without a given piece of evidence — how much does this detector actually shift the classification? If the answer is "barely at all" (below 0.01 KL divergence), skip the ONNX call entirely and reuse last tick's evidence.

The catch: `initializeNetwork` was rebuilding the entire BayesNetwork every tick, destroying all sensitivity data. The fix was making initialization idempotent — compare the incoming detector set with the stored one, only rebuild if they differ. Within a game phase the active detector set is stable, so sensitivity rankings persist across ticks. The orchestrator stores previous tick's outputs and merges fresh with stale, so skipped detectors still contribute evidence.

The wiring through `CascadingPatternClassifier` required one subtlety: the network must be initialized with ALL active detector keys (not just the ones being evaluated), so that stale evidence has nodes to attach to. The scheduling query happens after initialization but before the ONNX pass.

Workbench stats (#448) tied it together — propagation nanos, dirty variable count, retraction count, per-detector sensitivity rankings. Standard CDI event chain: `DetectorStatsPublished` fired alongside the pattern assessment, observed by `WorkbenchEnricher`, broadcast as a `detector_stats` WebSocket event. Zero overhead when no workbench client is connected — the broadcaster checks `sessions.isEmpty()` before serialising.

The neocortex side needed one small addition: `EvidenceFusionEngine.getStats()` delegating to `BayesInstance.getStats()`. Everything else was already there — the Builder API, batch evidence, hard evidence, sensitivity analysis. Phase 1 was entirely about connecting existing capabilities to the right call sites.

What this opens: with sensitivity rankings visible in the workbench, Phase 2's CPT learning (#450) has an observable feedback loop. Learn CPTs from replay data, see immediately which detectors become more or less influential. And the scheduling infrastructure means Phase 3's temporal smoothing (#444) can layer on top without adding per-tick ONNX cost — smoothing happens in the BJT, not before it.
