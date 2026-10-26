# HANDOFF — quarkmind

## Last Session

**#358 closed — cross-version generalisation validation.** Evaluated retrained ONNX models across 6 tournament datasets (patches 4.1–5.0, 401 replays). Oracle→tournament gap: ~12pp at min 3, ~6pp at min 5. vs_zerg early-minute accuracy (43.5% at min 3) is the main weakness — ROACH_RUSH↔RUSH confusion dominates. Filed #455 as follow-up for coarse-level accuracy and label audit.

## Immediate Next Step

**#455 (vs_zerg early-minute accuracy)** — add coarse-level accuracy to validation, audit ROACH_RUSH labels at min 3. Or Phase 4: #359 (autonomous play) / #361 (continuous training loop).

## Known Issues

- **Ground truth alignment gap**: OnnxClassificationCalibrationTest mid-game accuracy limited because `deriveGroundTruth()` produces archetypes not in model vocabulary.
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid.
