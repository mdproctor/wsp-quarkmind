# HANDOFF — quarkmind

## Last Session

**#450 closed — learned CPTs from replay data.** Python exporter runs ONNX detectors on 521K training observations, discretizes to HIGH/MEDIUM/LOW. Java `CptLearningExportTest` feeds observations to `ParameterLearner.fit()`, writes 9 learned CPT JSON files (3 phases × 3 matchups). Validation: learned CPTs outperform hand-tuned across all 9 matchup-phases (+10.7% to +75.7% accuracy). Landed as `dcae900d`.

**#360 scaffolded** — branch created, no implementation work started.

## Immediate Next Step

Brainstorm and plan #360 (feature extraction from emulated games, S/Med). Read the issue, understand what StrategyFeatures extraction from EmulatedGame requires vs the existing replay-based extraction path.

## Cross-Module

**Neocortex** — `ParameterLearner` used by #450. No pending changes.

**Pre-existing issues:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid. `DroolsDominanceWeightStrategyTest` config missing in `%test` profile.

## References

- `.plan` — issue queue with #360 active
- `docs/protocols/extractor-separate-from-simulated-game.md` — relevant protocol for #360
- `blog/2026-10-25-mdp03-teaching-the-network-to-count.md` — diary entry for #450
