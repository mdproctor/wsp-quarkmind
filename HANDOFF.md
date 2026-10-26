# HANDOFF — quarkmind

## Last Session

**#360 closed — EmulatedGameFeatureExtractor.** Count-based diff emitter that converts EmulatedGame tick-by-tick GameState snapshots into standard game_json format for the Python training pipeline. Includes UPPER_SNAKE_CASE→PascalCase name conversion with overrides for Python pipeline compatibility (SCV, MULE, HellionTank, VikingFighter, TemplarArchive). Full round-trip validated: shape (3 tests), runtime feature pipeline (2 tests), oracle replay equivalence characterisation (40 player runs — units 95%→64%, buildings 2%→37%). Landed as `dca03de3`.

## Cross-Module

**Neocortex** — no pending changes.

**Pre-existing issues:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid. `DroolsDominanceWeightStrategyTest` config missing in `%test` profile.

## References

- Design spec: `specs/issue-360-feature-extraction-emulated-games/2026-10-26-emulated-game-feature-extraction-design.md`
- Plan: `plans/2026-10-26-emulated-game-feature-extraction.md`
- Decisions: `specs/issue-360-feature-extraction-emulated-games/decisions.md`
