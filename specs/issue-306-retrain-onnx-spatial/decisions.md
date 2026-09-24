# Decisions — #306 Retrain ONNX Strategy Classifier

## D1: Scope pivot — enrich feature vector, not just retrain

**Choice:** Extend the feature vector with spatial, temporal delta, and structural ratio features, then retrain. The original premise (spatial features shifted) was incorrect — the feature vector has no spatial features at all. The real gap is that the classifier operates on composition only (WHAT units exist) while missing spatial intent (WHERE), tempo (HOW FAST), and resource commitment (HOW MUCH).
**Alternatives:**
- Retrain with same features — no accuracy improvement expected since feature distribution didn't change
- Close as not-needed — misses the opportunity to fill a genuine information gap
**Rationale:** The spatial fix (#298/#300) made real position data available for the first time. The model should use it.
**Trade-offs:** Larger scope than original issue. Requires extending both Java and Python feature extractors.
**Sources:** `DroolsScoutingTask.buildSnapshot()` (quarkmind-sc2), `sc2egset_extractor.py` (neocortex), `FeatureIndexMaps.java`
**Exploration:** deep-analysis
**Status:** captured

## D2: Single model with modality dropout, not parallel ONNX specialists

**Choice:** Keep a single ONNX model per matchup with all feature dimensions concatenated. Use the existing `ModalityDropoutDataset` (40% drop probability) to handle cases where spatial data is unreliable (fog of war, pre-scout).
**Alternatives:**
- Parallel specialist models per dimension (spatial model, composition model) with fusion — cross-dimensional interactions are the primary signal (5 roaches + army at your base = rush, same roaches at home = macro); separating models loses this
- Separate models retrained independently — adds inference latency (multiple ONNX calls per 22.4Hz tick), operational complexity (more models to version/deploy), and the fusion layer needs its own training data
**Rationale:** The classification signal lives in the interaction between dimensions. Modality dropout already provides graceful degradation when features are unavailable. The three-tier cascade (Drools → ONNX → LLM) provides architectural tiering at the system level.
**Trade-offs:** No per-dimension versioning. All features must be in norm_stats.json.
**Sources:** `ModalityDropoutDataset` in `neocortex/evaluation/strategy_classifier/dataset.py`, `CascadingPatternClassifier.java`
**Exploration:** deep-analysis
**Status:** captured

## D3: Feature additions — spatial (15), temporal deltas (8), structural ratios (6)

**Choice:** Add 29 new features per window across three dimensions:

**Spatial (15 per window):** army centroid X/Y, distance to own/enemy base, army spread, max forward projection, proxy building score (×2 players) + army gap (inter-player).

**Temporal deltas (8 per window):** delta army supply, delta workers, delta production buildings, delta tech buildings (×2 players). Window 0 = zeros.

**Structural ratios (6 per window):** army supply ratio, worker saturation, gas/mineral ratio (×2 players). Denominators clipped to min=1.

**Alternatives:**
- Spatial only (15 features) — ratios and deltas deferred — ratios carry genuine signal the CNN struggles to learn (division operations), and deltas free conv layer capacity for higher-order patterns
- Add engagement proximity — correlated with army_gap, O(n×m) compute, max_forward_dist captures outlier harass units
- Add production density — learnable from raw small-integer counts (production buildings + base counts both present)
- Add tech tier — learnable from existing building existence flags (ordinal encoding doesn't add information)
- Add per-window max/std — low within-window variance in pre-engagement phase; mean is sufficient
**Rationale:** Each dimension fills a specific gap: spatial encodes WHERE (completely orthogonal to composition), deltas encode rate-of-change explicitly (saves CNN capacity), ratios encode relative commitment (division is hard for CNNs). Feature vector grows 269→298 (~11%), trivially handled by wider first conv layer.
**Trade-offs:** 29 more features to keep aligned between Java and Python extractors. Training data must include position data (available in SC2EGSet/IEM10/AI Arena replays, currently not extracted by Python pipeline).
**Sources:** First-principles analysis of strategy classification axes (composition, spatial intent, tempo, resource commitment), `TemporalWindowAccumulator.java`, `feature_engineering.py`
**Exploration:** deep-analysis
**Status:** captured

## D4: Keep 10×30s window structure, defer finer resolution

**Choice:** Keep the current 10 windows of 30 seconds (covering 5 minutes). Do not change to 20×15s in this issue.
**Alternatives:**
- 20 windows of 15 seconds — doubles temporal resolution, better timing discrimination — but changes input tensor shape, CNN receptive field, norm_stats, and all tests simultaneously with 29 new features, making it impossible to attribute accuracy changes
**Rationale:** Isolate variables. Adding 29 new features is one change; changing temporal resolution is another. Measure spatial feature impact first, then refine temporal resolution as a follow-up if needed.
**Trade-offs:** Less temporal resolution for timing-sensitive patterns (e.g., 30s vs 45s proxy timing). Acceptable because the new temporal delta features partially compensate.
**Sources:** `TemporalWindowAccumulator` (MAX_WINDOWS=10, TICKS_PER_WINDOW=60), `config.py` (window_seconds=30, max_windows=10)
**Exploration:** deep-analysis
**Status:** captured

## D5: ONNX tier is the right place for spatial reasoning

**Choice:** Add spatial features to the ONNX tier (learned model), not to the Drools tier (handcrafted rules).
**Alternatives:**
- Drools rules with spatial thresholds (IF army_dist_enemy_base < 0.3 AND army_supply > 10 AND game_time < 3min THEN RUSH 0.8) — brittle, requires per-map/per-matchup threshold tuning, misses non-obvious combinations
**Rationale:** The ONNX model learns thresholds from data and captures multi-dimensional interactions. Drools handles high-confidence pattern-matched signals; ONNX handles learned multi-dimensional patterns; LLM handles ambiguous cases. Each tier uses the reasoning approach best suited to its signal type.
**Trade-offs:** Spatial patterns not available until model is retrained (Drools rules would be immediately usable). Acceptable because the retrain is part of this issue.
**Sources:** `CascadingPatternClassifier.java` (three-tier cascade architecture)
**Exploration:** quick
**Status:** captured
