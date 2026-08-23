# QuarkMind Handover — 2026-08-24

## Last Session

Closed issue #213 (IEM10 replay validation & accuracy benchmarking) and epic #208 (ONNX strategy classifier). Full work-end ceremony: code review, branch audit, forage sweep (2 garden entries), CLAUDE.md sync, squash (12→7 commits), land, and close.

Set up branch `issue-222-temporal-replay-matching` for #222 — scaffold committed, ready for brainstorming.

## Immediate Next Step

Run `/work continue` on branch `issue-222-temporal-replay-matching`. Start with brainstorming — #222 is a design-heavy issue (L/High) requiring sequence similarity algorithms, temporal feature alignment, and CBR integration.

This stage benefits from Opus. Type `/model` to check your current model.

## What #222 Is About

Temporal game replay matching for CBR (Case-Based Reasoning). The existing CBR system (#214 epic, 7/8 children closed) uses flat feature vectors to find similar past games. #222 adds **sequence similarity** — matching games by their temporal evolution (strategy transitions over time), not just end-state snapshots.

Key design questions to explore in brainstorming:
- Sequence representation: how to encode the temporal cascade assessments (per-tick StrategyArchetype + confidence) as a matchable timeline
- Similarity metric: DTW (Dynamic Time Warping), edit distance, or learned embeddings for game timelines
- Integration with existing `SC2GameCbrCase` — temporal features alongside the flat feature vector
- Query semantics: "find games where the opponent transitioned from rush to macro around minute 4"

## Infrastructure Available

From #213 (just landed):
- `TemporalWindowAccumulator` — 10×30s windows of 269-feature vectors
- `CascadingPatternClassifier` — produces `PatternAssessment` (archetype + confidence + source) each tick
- `StrategyFeatureExtractor` — 2690 temporal + 6 map features for ONNX
- Per-race ONNX models (vs_terran, vs_zerg, vs_protoss)

From #214 children (already landed):
- `SC2GameCbrCase` — flat feature case representation
- `SC2CbrRetentionObserver` — case retention pipeline
- `SC2StrategyRouterTask` — CBR-based strategy routing
- CBR schemas registered: `sc2-strategy`, `sc2-advisory`

## Pre-existing CDI Issues Fixed

During #213 work-end, three pre-existing CDI issues were fixed:
- Added `quarkus-micrometer` dependency (MeterRegistry injection)
- Added `quarkmind-core` to `quarkus.index-dependency` (MilestoneSession discovery)
- Excluded `ChannelResource` from CDI (QhorusDashboardService dependency)

These are on main now — no action needed.

## References

- Issue: casehubio/quarkmind#222
- Parent epic: casehubio/quarkmind#214 (CBR game experience — 7/8 children closed, #222 is the last)
- Garden: GE-20260823-22f56f (Quarkus index-dependency gotcha), GE-20260823-43c64b (SmallRye empty string gotcha)
