# HANDOFF — quarkmind

## Last Session (2026-10-22)

Branch: main — no active branch

### What Was Done

**#419 closed — Remove duplicated bayes engine.** Created `casehub-neocortex-inference-bayes` module in neocortex (zero-dependency jar). Deleted 4,273 lines of duplicated BJT from quarkmind-core, replaced with dependency. Landed as `8cc7f7e2`.

**#418 closed — Upgrade BJT to incremental.** Ported full incremental BJT from drools-beliefs into neocortex inference-bayes: dirty subtree tracking, separator snapshots, fast evidence retraction, BitMask hierarchy. 39 tests. `EvidenceFusionEngine.reset()` uses retraction instead of instance rebuild.

**Neocortex BJT roadmap completed (both phases).** All 15 Phase 1 issues (#620-#639) and all Phase 2 issues (#643-#652) landed in neocortex. The BJT is now a full-featured engine: log-space arithmetic, parallel propagation, MAP inference, parameter learning, causal inference (do-calculus + counterfactuals), explanation traces, influence diagrams, hard evidence, LLM-elicited structure, continuous refinement, observability.

**10 SC2 consumer issues created (#443-#452)** — quarkmind-side work to adopt the new BJT capabilities. None started yet.

### Next Session Plan

Adopt neocortex BJT capabilities in quarkmind's detector/strategy pipeline. Ordered by impact:

| Priority | Issue | Title | Scale | Depends on |
|----------|-------|-------|-------|------------|
| 1 | #443 | Sensitivity-driven detector scheduling — skip low-value ONNX calls | M/Med | — |
| 2 | #445 | Batch evidence API adoption | S/Low | — |
| 3 | #446 | Hard evidence for high-confidence detectors | S/Low | — |
| 4 | #449 | Builder API for SC2NetworkBuilder | S/Low | — |
| 5 | #448 | Observability stats in workbench | S/Low | — |
| 6 | #450 | Learn CPTs from replay data | M/Med | — |
| 7 | #451 | Explanation traces for coaching | S/Med | — |
| 8 | #447 | MAP inference for coherent classification | S/Med | — |
| 9 | #444 | Temporal BJT for cross-tick smoothing | L/High | — |
| 10 | #452 | Causal inference for strategy evaluation | M/High | — |

**Recommended start:** #443 (sensitivity-driven scheduling) — highest impact, skipping ONNX calls saves more than any BJT optimization. Then #445 + #446 as quick wins. Three missing SC2 issues not yet created: influence diagram strategy selection, continuous refinement during play, LLM-elicited network structure.

### Key Context

**Neocortex inference-bayes installed locally** — `mvn install` done. Quarkmind can compile against the full BJT API immediately.

**Pre-existing test failure:** `DroolsDominanceWeightStrategyTest` — `quarkmind.detector.models-dir` config missing in `%test` profile. From ONNX detector work (#422), unrelated to BJT.

**Dependency issue (ongoing):** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM is invalid (missing langchain4j version).

### Cross-Repo State

**Neocortex** — main at `275d8c78` (feat(#651): NetworkElicitor). All BJT work merged. origin pushed.

---

## Previous Session (2026-10-21)

Branch: `issue-422-onnx-detector-integration` — **closed**, landed as `25502441` on main

**#422 closed — ONNX binary detectors loaded in the Java game loop.**

`neocortex-inference-runtime` promoted from `runtime` to `compile` scope — `BinaryDetectorRunner` directly references `OnnxInferenceModel`.

---

## Paused Work

| Branch | Issue | State | Resume with |
|--------|-------|-------|-------------|
| `issue-388-train-onnx-reconstituted-ladder` | #388 | Perspectives format fixed, trained on 148K | `work resume` |
