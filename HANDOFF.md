# HANDOFF — quarkmind

## Last Session

**Phase 3 ONNX completion — #364, #365, #357 closed.** Three issues landed:

1. **#364** — Regenerated alignment fixture for 82-upgrade feature vector. Added `regenerateFixture()` @Tag("report") method for repeatable Java-driven generation.

2. **#365** — Moved upgrade_mapping.py generation from @Tag("report") test to Maven compile phase via exec-maven-plugin. UpgradeMappingGeneratorTest is now verification-only.

3. **#357** — Full ONNX retraining pipeline with 82-upgrade 409-feature schema:
   - Aligned Java StrategyFeatureExtractor to produce 409-feature ONNX windows (strips spatial/ratio/delta not in Python training data)
   - Fixed reconstitution pipeline dual-perspective label extraction (sidecar `perspectives` structure)
   - Expanded OnnxLabelMapping for full archetype class lists (8/7/9 per matchup)
   - Added ZERG_TECH_RUSH to StrategyArchetype and strategy-taxonomy.yaml
   - Retrained 3 per-matchup models on 227K samples from 50K reconstituted replays
   - Calibration test: rush accuracy 56%, early game 100%. Mid-game accuracy limited by ground truth alignment gap (deriveGroundTruth produces finer archetypes than model vocabulary)

## Immediate Next Step

**#358 (P3-3: Cross-version generalisation validation)** — still open. Requires adding era metadata to NPZ output to split accuracy by SC2 patch version. The retrained model was trained on mixed-version data but cross-version accuracy hasn't been formally measured. Run `work start #358`.

After #358: Phase 4 candidates — #359 (autonomous play) or #361 (continuous training loop).

## Known Issues

- **Ground truth alignment gap**: OnnxClassificationCalibrationTest mid-game accuracy (minutes 5-8) is low because `deriveGroundTruth()` produces archetypes (MARINE_TANK, BATTLE_MECH) not in the ONNX model's vocabulary. The model predicts BIO_TIMING/MECH_PUSH correctly per its labels, but the Java ground truth maps to different StrategyArchetype values.
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid. `DroolsDominanceWeightStrategyTest` config missing in `%test` profile.
