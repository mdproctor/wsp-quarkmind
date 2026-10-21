# HANDOFF — quarkmind

## Last Session (2026-10-21)

Branch: `issue-422-onnx-detector-integration` — **closed**, landed as `25502441` on main

### What Was Done

**#422 closed — ONNX binary detectors loaded in the Java game loop.**

Created `DetectorCatalog` (9 defs, 23 models mirroring Python `binary_catalog.py`), `BinaryDetectorRunner` (ONNX loading + feature adaptation), and wired into `CascadingPatternClassifier` Tier 2 for Bayesian fusion. The key non-obvious finding: Java feature vectors are 438/window (218/player + extras) but ONNX models expect 409/window (204/player + has_vision). Adaptation strips spatial/ratio/delta features using `SPATIAL_OFFSET=204` prefix match.

Code review caught missing `initializeNetwork()` in the CDI path — fixed before merge.

Epic #414 updated — all child issues (#410, #388, #412, #413, #411, #422) now marked complete. Remaining work (DetectorSelector/Aggregator SPIs, YAML orchestration, CBR meta-learning) tracked as future items in the epic body.

### Key Context

**Dependency issue (ongoing):** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM is invalid (missing langchain4j version). Causes cascading test-compile warnings but doesn't block `mvn test -Dtest=<specific>`. Use `-Dsurefire.failIfNoSpecifiedTests=false` when running specific tests.

`neocortex-inference-runtime` promoted from `runtime` to `compile` scope — `BinaryDetectorRunner` directly references `OnnxInferenceModel`.

---

## Paused Work

| Branch | Issue | State | Resume with |
|--------|-------|-------|-------------|
| `issue-388-train-onnx-reconstituted-ladder` | #388 | Perspectives format fixed, trained on 148K | `work resume` |
