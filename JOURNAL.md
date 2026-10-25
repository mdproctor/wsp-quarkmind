# Design Journal — issue-450-detector-cpts-learned-from-data

## 2026-10-25 — Session 1: Design + Batches 0–2

### Decisions made
- D4 revised: NetworkSpec JSON instead of custom format (decision review caught duplication)
- D2 revised: added 7min checkpoint for LATE-phase detectors (review caught missing learning pathway)
- D7 added: 3-bucket discretization (HIGH/MEDIUM/LOW) instead of binary (review caught information loss)
- D6 revised: per-matchup×phase networks (9 files) to match active detector sets exactly
- Fallback: profile-gated fail-fast in sc2/emulated/coach, warn+fallback in mock/test (diverges from issue #450 acceptance criteria — intentional)

### Cross-repo work
- neocortex#694: CptSpec Jackson annotations + NetworkSpec.fromBayesNetwork() — merged to main, installed locally

### Open question
- #454 filed: are ONNX detector outputs bimodal? If MEDIUM bucket is empty, 3-bucket adds noise. Diagnostic needed before bulk pipeline run.

### What's done
- Batch 0: neocortex prerequisites (merged)
- Batch 1: CatalogLabelMapper + 3-outcome evidence mapping in DetectorOrchestrator
- Batch 2: LearnedCptLoader CDI bean + wired into CascadingPatternClassifier

### What's next
- Batch 3: CptLearningExportTest — the bulk pipeline (biggest task)
- Batch 4: CptLearningValidationTest + integration verification
- Consider running #454 diagnostic before Batch 3 to validate discretization thresholds
