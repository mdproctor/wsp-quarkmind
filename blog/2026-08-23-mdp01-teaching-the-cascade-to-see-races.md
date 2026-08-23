---
title: "Teaching the cascade to see races"
date: 2026-08-23
author: Mark Proctor
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [onnx, classifier, cascade, feature-extraction, scouting]
series: quarkmind-diary
publish: false
---

# Teaching the cascade to see races

Last session extended `GameState` with economy stats and upgrades — seventy-one constructor sites touched in one SSR pass. This session wired the rest: the feature pipeline that converts game state into ONNX-compatible tensors, and the per-race model routing that makes the cascade aware of which matchup it's classifying.

## The 269-feature window

The neocortex models expect a specific layout: 134 features per player (53 buildings, 53 units, 13 economy stats, 15 upgrade flags), concatenated for both players with a scouting visibility flag — 269 per time window, 10 windows of 30 seconds each. The training pipeline (`sc2egset_extractor.py`) defines the exact ordering. Getting this wrong means the model sees marines where it expects pylons.

The mapping between QuarkMind's Java enums and the neocortex ordering is non-trivial. The Python training pipeline has `BarracksReactor` and `WarpGate` as distinct buildings; Java's `BuildingType` doesn't. Java has `HELLBAT`; Python calls it `HellionTank`. Java has `VIKING` and `VIKING_ASSAULT` as separate types; Python has `VikingFighter` and `VikingAssault`. `FeatureIndexMaps` is the translation layer — static maps from each Java enum value to its neocortex feature index, with unmapped types silently ignored.

## Softmax bites in testing

The ONNX tier tests caught me out. `InMemoryInferenceModel.returning(0.9f, 0.05f, 0.05f)` looks like it returns a confident classifier — 90% on the top class. But `TensorClassifier` applies softmax internally before comparing to the confidence threshold. After softmax, those logits produce roughly 0.33 — well below the 0.5 threshold. The ONNX tier silently falls through, the cascade returns DROOLS, and the test assertion fails with no exception to explain why.

The fix is simple once you see it: use raw logit values with large gaps. `(5.0, 0.0, 0.0, 0.0, 0.0, 0.0)` gives ~0.97 after softmax. But the failure mode is invisible — no error, no warning, just the wrong `AssessmentSource` on the result.

## Per-race routing

The single `@Inference("strategy-classifier")` model becomes three: `strategy-vs-terran`, `strategy-vs-zerg`, `strategy-vs-protoss`. Each has its own class count (5, 6, 7 respectively) and its own label vocabulary. `OnnxLabelMapping` resolves ONNX labels to `StrategyArchetype` values per race — `"RUSH"` for Terran maps to `TERRAN_MARINE_RUSH`, for Zerg to `ZERG_ZERGLING_RUSH`, for Protoss to `PROTOSS_GATEWAY_RUSH`. The broadest existing archetype for each race, so the ONNX confidence doesn't fragment across fine-grained Drools archetypes.

Five new `StrategyArchetype` values: `PROTOSS_DT_RUSH`, `TERRAN_AIR_SUPERIORITY`, `ZERG_HYDRA_PUSH`, `PROTOSS_COLOSSUS_PUSH`, `PROTOSS_AIR_SUPERIORITY`. These are the mid-game compositions the ONNX models classify that Drools didn't previously name.

## The accumulator

`DroolsScoutingTask` now carries a `TemporalWindowAccumulator` — each tick builds a `WindowSnapshot` from the current `GameState` via `FeatureIndexMaps`, the accumulator groups snapshots into 30-second windows, `StrategyFeatureExtractor` flattens and z-score normalises the result. The normalisation stats come from the neocortex training pipeline (`norm_stats.npz` exported as JSON). The `has_vision` flag at index 268 is exempted — it's binary, not continuous.

All production code for the cascade is now in place. What remains is proving it works: a validation test running all 59 replay games through three modes (Drools-only, ONNX-only, cascade) at minutes 1–5, and a latency benchmark confirming p99 stays under 10ms.
