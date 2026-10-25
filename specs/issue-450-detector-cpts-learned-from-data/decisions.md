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

**Choice:** Per-phase checkpoint — one observation at each game phase boundary (3min for EARLY, 5min for MID, 7min for LATE). Matches how `DetectorCatalog` splits detectors by phase (EARLY/MID/LATE). Produces ~3x more training data than single-checkpoint. Replays shorter than 7min contribute EARLY/MID data only — partial coverage is better than no LATE-phase learning.
**Alternatives:**
- Single checkpoint per replay (e.g., 5min) — simpler but phase-specific detectors learn from wrong time windows
- Per-tick sliding window — maximum data but heavily correlated samples, much slower processing
**Rationale:** Phase boundaries align with when each detector is active in production; EARLY detectors should learn from 3min state, MID detectors from 5min state, LATE detectors from 7min state
**Trade-offs:** LATE phase (7min+) has fewer replays reaching that length; data volume for LATE detectors will be lower
**Sources:** `DetectorCatalog.java` (phase assignments), `CascadingPatternClassifier.java:378-383` (phase resolution)
**Exploration:** quick
**Status:** revised (decision review: LATE-phase detectors had no learning pathway without 7min checkpoint)

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

**Choice:** NetworkSpec JSON — serialize the learned `BayesNetwork` as a `NetworkSpec` record via Jackson, wrapped in a metadata envelope with matchup, replay count, and extractor version. `NetworkSpec`, `VariableSpec`, `EdgeSpec`, and `CptSpec` are all records — Jackson handles them natively. Deserialize with `ObjectMapper.readValue()`, convert to live network with `NetworkSpec.toBayesNetwork()`. One file per matchup as a classpath resource.
**Alternatives:**
- Custom JSON format — unnecessary duplication of what NetworkSpec already models; creates schema drift risk if neocortex evolves CptSpec
- Embedded Java constants — compile-time checked but requires regeneration on retrain
**Rationale:** NetworkSpec is purpose-built for this. The dependency already exists (quarkmind-sc2 → casehub-neocortex-inference-bayes). `toBayesNetwork()` eliminates the need for a custom loader. Records are human-readable as JSON.
**Trade-offs:** Requires Jackson annotations on CptSpec sealed interface hierarchy (may need `@JsonTypeInfo`/`@JsonSubTypes` — verify); JSON is less pretty than a purpose-built format but has the correct semantics
**Sources:** `NetworkSpec.java`, `CptSpec.java` (neocortex), `ResponseParser.java` (existing Jackson deserialization in inference-bayes-llm)
**Exploration:** quick
**Status:** revised (decision review: NetworkSpec already exists and eliminates custom format duplication)

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

**Choice:** Use `NetworkSpec.toBayesNetwork()` for loading learned networks — the JSON resource deserializes to a `NetworkSpec`, which converts directly to a `BayesNetwork`. No custom loader needed. `SC2NetworkBuilder.build()` remains unchanged as the fallback path for profiles without learned CPTs. Learned networks are stored per matchup × phase (9 files: 3 matchups × 3 phases) so the loaded network exactly matches the active detector set for that phase — no sub-network extraction needed.
**Alternatives:**
- Per-matchup only (3 files), filter at load time — marginalization over absent variables produces different CPTs than learning from the subset; learned priors change
- Extract learned CPTs back into `Map<String, Map<String, Double>>` and feed through existing `build()` — extra conversion step, fragile archetype ordering dependency
**Rationale:** Per-phase networks avoid the structural mismatch between a full-matchup learned network and the phase-specific active detector set. `NetworkSpec.toBayesNetwork()` eliminates custom loading code.
**Trade-offs:** 9 resource files instead of 3; each file has fewer training observations than the matchup-wide pool
**Depends on:** D2 (per-phase checkpoints), D4 (NetworkSpec JSON persistence)
**Sources:** `SC2NetworkBuilder.java`, `NetworkSpec.toBayesNetwork()`, `DetectorOrchestrator.initializeNetwork()`
**Exploration:** quick
**Status:** revised (decision review: phase-specific subsets must match learned network structure)

## D7: Detector output discretization

**Choice:** Three-bucket discretization — LOW (<0.3), MEDIUM (0.3–0.7), HIGH (>0.7). Each detector variable has three outcomes instead of two. `ParameterLearner.fit()` handles arbitrary outcome counts.
**Alternatives:**
- Binary HIGH/LOW at 0.5 — loses magnitude information; a detector scoring 0.95 and one scoring 0.51 both become "HIGH" in training but produce very different soft evidence at inference time
- Beta distribution parameterization — principally correct for continuous soft evidence but goes beyond `ParameterLearner.fit()` which requires discrete observations
- Match hardEvidenceThreshold (0.95/0.05) — only two buckets plus a wide "uncertain" middle zone; barely different from binary
**Rationale:** Three buckets capture the distinction between confident detection (HIGH), borderline detection (MEDIUM), and confident absence (LOW). The learned CPTs then model P(HIGH|RUSH), P(MEDIUM|RUSH), P(LOW|RUSH) separately, giving the inference engine better discrimination than binary. The `BayesNetworkBuilder` already supports multi-outcome variables.
**Trade-offs:** 3x more CPT cells per detector (3 outcomes × N archetypes instead of 2 × N); requires updating `SC2NetworkBuilder.build()` fallback to use 3 outcomes; at inference time, detector outputs must be discretized to the same 3 buckets before setting evidence
**Depends on:** D3 (data source — discretization happens during extraction)
**Sources:** `DetectorOrchestrator.classify()` (soft evidence path), `BayesNetworkBuilder.variable()` (multi-outcome API)
**Exploration:** quick
**Status:** captured (surfaced by decision review — implicit choice made explicit)
