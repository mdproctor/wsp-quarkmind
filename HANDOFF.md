# HANDOFF — quarkmind

## Last Session (2026-10-25)

Branch: main — no active branch

### What Was Done

**#447 closed — MAP inference for coherent joint classification.** `DetectorOrchestrator.getMapStrategy()`/`getMapAssignment()` expose MAP results after BJT fusion. `CascadeResult.mapStrategy` carries the MAP winner on fusion-tier results. Workbench receives MAP data via `DetectorStatsPublished`. Landed as `841e2083`.

**#451 closed — BJT explanation traces for coaching.** `EvidenceFusionEngine.explain()` added to neocortex (delegates to `ExplanationEngine`). `DetectorOrchestrator.getExplanationTrace()`/`getExplanationSummary()` produce human-readable influence rankings. `DroolsScoutingTask` writes explanation to `SCOUTING_EXPLANATION` CaseFile key. `CoachingWorkerFactory` includes `EVIDENCE REASONING` in LLM prompt. Landed as `9996c47e`.

### Cross-Repo State

**Neocortex** — two commits on main: `mapInference()` (`b5d35c2b`) and `explain()` (`7a403dca`) on `EvidenceFusionEngine`. Installed locally. Not pushed to origin.

### Key Context

**Pre-existing test failure:** `DroolsDominanceWeightStrategyTest` — `quarkmind.detector.models-dir` config missing in `%test` profile.

**Dependency issue (ongoing):** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid.

**`patternAssessment` in coaching trigger is a dead code path.** `CoachingWorkerFactory.buildUserMessage()` reads `triggerMap.get("patternAssessment")` but nothing in production writes it. The assessment data exists at `SCOUTING_FINAL_ASSESSMENT` but isn't copied to the trigger map. Only test code constructs it.

### Next Session

Remaining #453 epic issues (BJT adoption):

| Priority | Issue | Title | Scale |
|----------|-------|-------|-------|
| 1 | #450 | Learn CPTs from replay data | M/Med |
| 2 | #444 | Temporal BJT for cross-tick smoothing | L/High |
| 3 | #452 | Causal inference for strategy evaluation | M/High |

**Recommended start:** #450 (learn CPTs from replay data) — replaces hand-tuned conditional probability tables with values learned from the replay corpus.

---

## Paused Work

| Branch | Issue | State | Resume with |
|--------|-------|-------|-------------|
| `issue-388-train-onnx-reconstituted-ladder` | #388 | Perspectives format fixed, trained on 148K | `work resume` |
