## D1: Pipeline architecture

**Choice:** Offline batch learning with JSON export — a `@Tag("report")` test runs the full Java pipeline (EmulatedGame → StrategyFeatures → BinaryDetectorRunner) over reconstituted replays, collects observations at phase checkpoints (3min/5min), calls `ParameterLearner.fit()`, and serializes learned CPTs to JSON resource files per matchup.
**Alternatives:**
- Online learning via `OnlineParameterUpdater` — better for continuous refinement but requires labelled live games, not available yet; future enhancement on top of batch
- Python-side learning + JSON import — redundant since neocortex `ParameterLearner` is ready; loses production-path fidelity
**Rationale:** Reuses existing report-test pattern (like `ReconstitutionExportTest`), deterministic and reproducible, JSON output is versionable and reviewable, no new runtime dependencies, production ONNX inference path ensures learned CPTs match deployment behaviour
**Trade-offs:** Requires rerunning the report test to update CPTs when models or labels change; no automatic adaptation during live play
**Sources:** `SC2NetworkBuilder.java`, `CascadingPatternClassifier.java:385-415`, `ParameterLearner.java` (neocortex), `ReconstitutionExportTest.java`
**Exploration:** quick
**Status:** captured

## D2: Training data granularity

**Choice:** Per-phase checkpoint — one observation at each game phase boundary (3min for EARLY, 5min for MID). Matches how `DetectorCatalog` splits detectors by phase (EARLY/MID/LATE). Produces ~3x more training data than single-checkpoint.
**Alternatives:**
- Single checkpoint per replay (e.g., 5min) — simpler but phase-specific detectors learn from wrong time windows
- Per-tick sliding window — maximum data but heavily correlated samples, much slower processing
**Rationale:** Phase boundaries align with when each detector is active in production; EARLY detectors should learn from 3min state, MID detectors from 5min state
**Trade-offs:** LATE phase (7min+) has less replay coverage since some games end before then
**Sources:** `DetectorCatalog.java` (phase assignments), `CascadingPatternClassifier.java:378-383` (phase resolution)
**Exploration:** quick
**Status:** captured

## D3: Data source — production inference path

**Choice:** Java batch harness runs reconstituted replays through EmulatedGame → StrategyFeatures → BinaryDetectorRunner (production ONNX inference), paired with heuristic strategy labels from `.label.json` sidecars.
**Alternatives:**
- Python-only extraction — limited; Python ONNX path may not match production Java inference exactly
- Python extraction + Java ONNX — partial fidelity, still misses EmulatedGame feature pipeline
**Rationale:** Production-path fidelity ensures learned CPTs reflect actual detector behaviour at deployment; avoids subtle feature extraction mismatches between Python and Java pipelines
**Trade-offs:** More engineering work than pure Python; requires ONNX models on classpath for the report test
**Sources:** `BinaryDetectorRunner.java`, `ReplayValidationHarness`, `sc2egset_extractor.py`
**Exploration:** quick
**Status:** captured

## D4: Persistence format

**Choice:** JSON resource file — serialize learned CPTs as JSON bundled as a classpath resource, one file per matchup. Versionable, reviewable in PRs, consistent with how ONNX models are stored as resources.
**Alternatives:**
- NetworkSpec records + neocortex serializer — requires adding serialization to neocortex; more coupling
- Embedded Java constants — compile-time checked but requires regeneration on retrain
**Rationale:** Simple, human-readable, no new neocortex dependencies, follows existing resource-file conventions
**Trade-offs:** No compile-time schema validation; must parse and validate at load time
**Sources:** `DetectorCatalog.java` (modelPath pattern), neocortex `CptSpec`/`NetworkSpec` records
**Exploration:** quick
**Status:** captured

## D5: Fallback behaviour

**Choice:** Profile-gated — fail fast in profiles where classification matters (`%sc2`, `%emulated`, `%coach`), fall back to 0.8/0.2 defaults in `%mock`/`%test` where classification is incidental.
**Alternatives:**
- Always fall back to 0.8/0.2 — dangerous: silently degrades quality with no signal; if those values were good enough we wouldn't need this work
- Always fail fast — forces CI and tests to bundle learned CPTs, adds friction for no classification benefit
- Warn loudly, don't fail — visible signal but still runs with bad CPTs in production
**Rationale:** Classification accuracy is the point of this work; silently using garbage CPTs is worse than failing. But mock/test don't depend on classification quality.
**Trade-offs:** Adds a startup check gated on profile; must ensure learned-cpts.json ships with production deployments
**Sources:** `application.properties` profile definitions, `CascadingPatternClassifier.buildDefaultOrchestrator()`
**Exploration:** quick
**Status:** captured

## D6: Network loading vs reconstruction

**Choice:** Replace `SC2NetworkBuilder.build()` with direct BayesNetwork loading — the JSON resource stores the full learned network specification (structure + CPTs), and `SC2NetworkBuilder` gains a `load()` method alongside the existing `build()`. The `build()` path serves as fallback for profiles without learned CPTs.
**Alternatives:**
- Extract learned CPTs back into `Map<String, Map<String, Double>>` and feed through existing `build()` — extra conversion step, fragile archetype ordering dependency between `double[][]` layout and sorted archetype array
**Rationale:** The learned network already has correct structure and CPTs; reconstructing from maps is unnecessary indirection and risks CPT row ordering mismatches
**Trade-offs:** Two code paths for network construction (load vs build); JSON schema must capture the full network topology
**Depends on:** D4 (JSON resource persistence)
**Sources:** `SC2NetworkBuilder.java`, `ParameterLearner.java` (returns `BayesNetwork` directly), neocortex `NetworkSpec` records
**Exploration:** quick
**Status:** captured
