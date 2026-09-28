# Decisions — issue-317-blizzard-ladder-restore

## D1: Restoration strategy — cloud VM primary, Java pipeline secondary

**Choice:** Use a cloud x86_64 VM with Docker/SC2 headless to restore all 151K replays with full tracker events. Develop the Java-native command-to-features pipeline as a secondary hardening effort, validated against cloud-restored ground truth.
**Alternatives:**
- Java-native only (original) — can reconstruct only ~18% of classifier features (unit training timing for the 24 unit types mapped in AbilityMapping); insufficient for training
- EmulatedGame for early-game features — requires solving the building/upgrade abilLink discovery first (see D7)
**Rationale:** The classifier uses 134 features per player (53 buildings, 53 units, 13 economic stats, 15 upgrades), all derived from tracker events (D6). Game events alone can produce partial unit birth timing for ~24 of 53 mapped unit types via AbilityMapping — zero building counts, zero economic stats, zero upgrades. A cloud x86_64 VM processes replays at 2-4/second (per issue #317), completing all 151K replays in ~10-21 hours for ~$5-20 of compute. This produces complete, accurate training data without new code.
**Trade-offs:** Requires cloud VM provisioning (one-time DevOps). Doesn't directly harden EmulatedGame — but the Java pipeline development as a secondary effort preserves that benefit. Combat deaths are reconstructed by the full SC2 engine, so unit counts are accurate (unlike game-event-only extraction where deaths are missing).
**Hardening loop (preserved):** The local Docker oracle set (D2) provides the ground truth that D4's abilLink discovery needs. The full cloud-restored dataset then serves as validation ground truth for the Java pipeline at scale. The Java pipeline development — extending AbilityMapping for human replay abilLinks, building StrippedReplayFeatureExtractor — proceeds in parallel, validated against cloud-restored data. Every divergence fix still hardens EmulatedGame for all consumers (agent loop, coaching mode, replay validation).
**Sources:** Issue #317 (cloud VM timing: 2-4 replays/s), sc2egset_extractor.py (134 features), AbilityMapping.java (24 unit types mapped, zero buildings/upgrades), ReplayValidationHarness.java (buildings injected from tracker GT)
**Exploration:** deep-analysis
**Status:** revised (R1-02, R1-03: corrected feature coverage analysis and VM timing)

## D2: Oracle set composition

**Choice:** ~200 replays from the 4.9.3 batch, restored via Docker/SC2 headless locally (~3 hours under emulation), stratified by race matchup (6 matchups × ~33 replays, proportional to matchup frequency in the dataset).
**Alternatives:**
- Use cloud-restored replays as oracle — viable but local emulation gives immediate feedback during development
- Use existing tournament replays — different patch versions, different abilLink constants
**Rationale:** The oracle serves two purposes: (1) validate abilLink constant discovery for human ladder replays (D4), and (2) provide regression test data for the Java pipeline (D1 secondary effort). 200 replays stratified across 6 matchups provides ~33 per matchup. If the oracle reveals fundamental feature extraction gaps (e.g., unmapped building abilLinks), the fallback is the cloud VM primary strategy — no pivot required because the cloud VM produces complete data regardless.
**Fallback:** If oracle analysis reveals the game-event-only approach is not viable for the Java pipeline, the training data production continues unblocked via the cloud VM path. The oracle data is still valuable for AbilityMapping extension work.
**Trade-offs:** Oracle only covers v4.9.3 patch. 4.10.1 replays validated separately once abilLinks are mapped.
**Sources:** Docker test results (5 replays, 257s, all OK), PySC2 version registry (4.9.3 = Base75025)
**Exploration:** quick
**Status:** revised (R1-04: added stratification rationale and fallback plan)

## D3: Scelight parser compatibility with stripped replays

**Choice:** Use Scelight's existing .backup fallback in RepParserEngine — no parser changes needed.
**Alternatives:**
- Pre-process stripped replays to rename .backup files — unnecessary given built-in fallback
- Use Python sc2reader instead — loses access to Java pipeline and calibrated SC2Data constants
**Rationale:** RepParserEngine lines 104-114 already try DETAILS, fall back to DETAILS_BACKUP; same for INIT_DATA. Game events parsing gets the protocol version it needs from initData. Stripped replay format is explicitly supported by the parser.
**Trade-offs:** Parser compatibility confirms the file can be opened and game events read. It does NOT imply data completeness — stripped replays lack tracker events (UnitBorn, UnitInit, UnitDied, PlayerStats, Upgrade, UnitPositions), which are the data source for 100% of classifier features. Parser support is a necessary prerequisite, not a sufficient condition for feature extraction.
**Sources:** RepParserEngine.java:104-114
**Exploration:** quick
**Status:** revised (R1-05: correctly scoped what parser compatibility confirms vs doesn't confirm)

## D4: abilLink vocabulary discovery for human replays

**Choice:** Discover human ladder replay abilLinks using the oracle set (D2) by cross-referencing game event commands with Docker-restored tracker events. Extend AbilityMapping with human-replay-specific mappings for building placement and upgrade research.
**Alternatives:**
- Assume bot and human abilLinks are identical — incorrect; AbilityMapping comment (line 21-24) documents that bots use abilLink=42 (Smart) for building placement, while human UI sends distinct per-building abilLinks
- Hard-code separate abilLink tables per patch — same approach as version-aware mapping, just named differently
**Rationale:** The bot/human abilLink difference is a vocabulary gap, not a patch-version calibration issue. Bot replays (AI Arena) use the SC2 API which sends Smart commands for building placement. Human replays use the UI-driven ability system where each building type, upgrade research, and morph command has its own abilLink. The current AbilityMapping has zero building placement mappings and zero upgrade research mappings because these commands are invisible in bot replays. Discovery requires the same approach as the original AbilityDiscoveryTest: cross-reference game event abilLinks against tracker events from oracle-restored replays.
**Coverage gaps to address:**
- Building placement: all 53 building types (zero currently mapped)
- Upgrade research: all 15 tracked upgrades (zero currently mapped)
- Terran production: Factory, Starport, all add-ons (only CC and Barracks mapped)
- Zerg morphs: Baneling, Ravager, Lurker, Brood Lord, Lair, Hive (zero mapped)
- WarpGate warp-in: currently treated as movement (abilLink=170), but produces units. Note: once WarpGate research completes mid-game, Protoss Gateway production shifts entirely to WarpGate warp-in — making the effective Protoss unit coverage lower than 10 types for mid-game replays
**Trade-offs:** AbilityMapping extension is a substantial discovery exercise. This work is non-blocking for training data production (cloud VM path), but required for the Java-native pipeline to become viable.
**Sources:** AbilityMapping.java (lines 21-24 comment, abilLink constants), AbilityDiscoveryTest pattern
**Exploration:** quick
**Status:** revised (R1-06: reframed from patch calibration to vocabulary discovery, documented coverage gaps)

## D5: Feature output format

**Choice:** For cloud-restored replays: feed directly through the existing Python pipeline (sc2reader → game_json → extract_replay → feature engineering). For the future Java pipeline: emit the same game_json dict structure with synthetic tracker events, validated against cloud-restored ground truth.
**Alternatives:**
- Emit NPZ directly from Java — duplicates Python feature engineering in Java, two implementations to maintain
- Build entirely new Python pipeline — throws away validated prepare_replay_pack.py
**Rationale:** Cloud-restored replays have full tracker events, so the existing Python pipeline (prepare_replay_pack.py) processes them unchanged — both features AND labels work correctly. The labelling pipeline (extract_build_order in prepare_real_data.py) also requires tracker events: UnitInit for buildings, UnitBorn for units. The Java pipeline, when developed, must produce synthetic tracker events for all event types the downstream pipeline consumes: UnitInit, UnitDone (buildings), UnitBorn, UnitDied (units), PlayerStats (economy), Upgrade (upgrades). This is substantially more than format compatibility — it requires the Java pipeline to reconstruct data that only exists in tracker events.
**Trade-offs:** Cloud-restored replays go through the existing pipeline with zero changes. The Java pipeline path requires solving D4 (abilLink discovery) and building economic simulation output before format compatibility becomes relevant.
**Sources:** prepare_replay_pack.py:sc2reader_to_game_json(), prepare_real_data.py:extract_build_order(), sc2egset_extractor.py:extract_replay()
**Exploration:** quick
**Status:** revised (R1-07: acknowledged labelling pipeline dependency on tracker events, separated cloud-VM and Java pipeline paths)

## D6: Feature requirements analysis

**Choice:** Perform a feature requirements analysis before designing any feature extraction pipeline. The analysis maps each of the classifier's 134 features to its tracker event source and determines which can be reconstructed from game events alone.
**Alternatives:**
- Proceed without analysis (original implicit approach) — leads to discovering feature gaps during implementation
- Limit classifier to extractable features only — reduces model accuracy by training on a subset
**Rationale:** The classifier uses 134 features per player: 53 building counts (from UnitInit/UnitDone tracker events), 53 unit counts (from UnitBorn/UnitDied), 13 economic stats (from PlayerStats), and 15 upgrade flags (from Upgrade). A systematic analysis before strategy selection would have revealed that only ~24 unit birth events are extractable from game events via AbilityMapping — zero building events, zero economic stats, zero upgrades. This analysis should be step 1 of any alternative extraction strategy.
**Sources:** sc2egset_extractor.py (BUILDINGS, UNITS, STAT_KEYS, UPGRADES lists), AbilityMapping.java (mapped abilLinks)
**Exploration:** N/A (surfaced by reviewer)
**Status:** captured (R1-09: implicit omission made explicit)

## D7: EmulatedGame for early-game feature extraction

**Choice:** Defer EmulatedGame-based feature extraction until AbilityMapping has been extended for human replays (D4). Do not reject outright — EmulatedGame is architecturally suited for early-game features once building/upgrade abilLinks are mapped.
**Alternatives:**
- Use EmulatedGame now (for minutes 2-4) — blocked by the same AbilityMapping gaps that block the Java pipeline (buildings, upgrades)
- Reject EmulatedGame entirely — overly dismissive; the combat divergence concern that motivated the original rejection does not apply to pre-combat game phases (minutes 2-4)
**Rationale:** EmulatedGame has full economic simulation (handleTrain, handleBuild, mineral/gas costs, PlayerState tracking) and existing replay integration (injectReplayBuilding, markReplayBuildingComplete, spawnUnit). For early-game time windows (classifier uses minutes 2, 3, 4, 5), combat divergence is minimal — most games have no significant combat before minute 4. The original rejection ("combat divergence risks mixing two ground truths") was applied as a blanket argument without evaluating the early-game-only use case where combat divergence is not a factor. However, EmulatedGame STILL requires building and upgrade events as input — currently injected from tracker events via ReplayValidationHarness. Without solving D4 (abilLink discovery), EmulatedGame cannot be driven from game events alone.
**Future value:** Once D4 is complete, EmulatedGame becomes a viable path for extracting features from any replay (including future stripped replays) without requiring SC2 or cloud compute. This is valuable for the agent loop, coaching mode, and processing new replay batches.
**Sources:** EmulatedGame.java (handleTrain, handleBuild, injectReplayBuilding), ReplayValidationHarness.java (building sync from tracker GT), prepare_real_data.py (MINUTES = [2, 3, 4, 5])
**Exploration:** N/A (surfaced by reviewer)
**Status:** captured (R1-08: implicit rejection made explicit, combat divergence scoped to late-game only)
