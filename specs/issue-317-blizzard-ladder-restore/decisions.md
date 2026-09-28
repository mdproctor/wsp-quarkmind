# Decisions — issue-317-blizzard-ladder-restore

## D1: Restoration strategy — Java-native with Docker oracle validation

**Choice:** Build a Java-native command-to-features pipeline using calibrated SC2Data constants, validated against a Docker/SC2 headless oracle set.
**Alternatives:**
- Docker/SC2 headless for all 151K replays — proven but ~50s/replay under ARM64 emulation = months of compute
- EmulatedGame full physics simulation — fast but combat divergence risks mixing two ground truths in training data
**Rationale:** Stripped replays contain game events (player commands) which are deterministic inputs. Combined with SC2Data calibrated train/build times (validated by SC2TrainTimeCalibrationTest), unit births and economy are exactly reconstructable without running SC2. A small Docker-restored oracle set validates the Java pipeline before bulk processing. Every divergence fix also hardens EmulatedGame.
**Trade-offs:** Requires building a new Java pipeline component (StrippedReplayFeatureExtractor). Combat deaths won't be reconstructed — accepted because classifier targets early-game strategy archetypes where build order, not combat outcomes, is the primary signal.
**Sources:** SC2TrainTimeCalibrationTest.java, ReplayCommandExtractor.java, ReplaySimulatedGame.java, RepParserEngine.java (lines 104-114: .backup fallback), prepare_replay_pack.py
**Exploration:** deep-analysis
**Status:** captured

## D2: Oracle set composition

**Choice:** ~200 replays from the 4.9.3 batch, restored via Docker/SC2 headless locally (~3 hours under emulation).
**Alternatives:**
- Use 4.10.1 replays — requires Base75800 build which we don't have installed (Base75689 = v4.10.0 only)
- Use existing tournament replays as oracle — different patch versions, different abilLink constants
**Rationale:** 4.9.3 Docker restoration is validated end-to-end (5/5 replays restored successfully, tracker events confirmed). Base75025 matches PySC2's version registry. 200 replays gives sufficient coverage across races/matchups while completing in ~3 hours.
**Trade-offs:** Oracle only covers v4.9.3 patch. 4.10.1 replays validated separately once the Java pipeline works (abilLink differences between patches are the main risk).
**Sources:** Docker test results (5 replays, 257s, all OK), PySC2 version registry (4.9.3 = Base75025)
**Exploration:** quick
**Status:** captured

## D3: Scelight parser compatibility with stripped replays

**Choice:** Use Scelight's existing .backup fallback in RepParserEngine — no parser changes needed.
**Alternatives:**
- Pre-process stripped replays to rename .backup files — unnecessary given built-in fallback
- Use Python sc2reader instead — loses access to Java pipeline and calibrated SC2Data constants
**Rationale:** RepParserEngine lines 104-114 already try DETAILS, fall back to DETAILS_BACKUP; same for INIT_DATA. DETAILS and INIT_DATA are "always parsed" (line 63 comment). Game events parsing gets the protocol version it needs from initData. Stripped replay format is explicitly supported.
**Trade-offs:** None — this is existing functionality.
**Sources:** RepParserEngine.java:104-114
**Exploration:** quick
**Status:** captured

## D4: abilLink patch version risk

**Choice:** Validate abilLink constants from ReplayCommandExtractor against 4.9.3 ladder replays before bulk processing. Build a calibration test that cross-references game event commands with Docker-restored tracker events (same pattern as SC2TrainTimeCalibrationTest).
**Alternatives:**
- Assume abilLinks are stable across patches — risky, IEM10 (2016) already has different values
- Hard-code separate abilLink tables per patch — brittle, doesn't scale
**Rationale:** ReplayCommandExtractor's abilLink constants are calibrated against 2023+ AI Arena replays. The 4.9.3 ladder replays are from 2019. The calibration test note explicitly warns about cross-patch abilLink differences. The oracle set gives us ground truth to calibrate against.
**Trade-offs:** May require building a version-aware AbilityMapping if 2019 abilLinks differ from 2023+.
**Sources:** SC2TrainTimeCalibrationTest.java (cross-validation note lines 37-38), AbilityMapping.java
**Exploration:** quick
**Status:** captured

## D5: Feature output format

**Choice:** Java pipeline emits the same game_json dict structure that sc2reader_to_game_json() produces, serialised as JSON. The existing Python pipeline (prepare_replay_pack.py → extract_replay() → build_temporal_features()) consumes it unchanged.
**Alternatives:**
- Emit NPZ directly from Java — duplicates Python feature engineering in Java, two implementations to maintain
- Build entirely new Python pipeline — throws away validated prepare_replay_pack.py
**Rationale:** Minimises change surface. The Python pipeline is validated against tournament replays. By emitting the same intermediate format, we reuse all downstream feature engineering, labelling, and training code. Only the data source changes (Java commands → game_json vs sc2reader tracker events → game_json).
**Trade-offs:** Adds a Java → JSON → Python boundary. Acceptable for a batch pipeline; not suitable for real-time.
**Sources:** prepare_replay_pack.py:sc2reader_to_game_json(), extract_replay()
**Exploration:** quick
**Status:** captured
