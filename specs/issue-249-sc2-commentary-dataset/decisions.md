# Decisions — #249 SC2 Commentary Training Dataset

## D1: Replay-to-VOD matching strategy — tournament-catalog matching

**Choice:** Build a tournament catalog mapping SC2EGSet tournament names to YouTube channels/playlists. Match replays to VODs within each tournament using a composite key (player1, player2, map, duration ±2min). Supplement with Liquipedia/Aligulac VOD links for validation and gap-filling.
**Alternatives:**
- Metadata-search matching — YouTube search per replay, noisy results, high false-positive rate without playlist boundaries
- Community-sourced matching — inconsistent coverage, good as supplementary but not primary
**Rationale:** Narrows search from all-of-YouTube to ~20-50 known playlists. SC2EGSet covers 55 Premiere/Major tournaments with official broadcast channels. Composite key of two player names + map is nearly always unique within a tournament.
**Trade-offs:** Requires upfront manual effort to build the tournament→playlist catalog (~55 tournaments). One-time cost.
**Sources:** SC2EGSet paper (Nature 2023) — tournament coverage, LoL19-21 methodology — matching via game IDs (not available for SC2)
**Exploration:** quick
**Status:** captured

## D2: Timestamp alignment — game-start detection + linear mapping

**Choice:** Detect the game-start moment in the VOD automatically (audio/visual landmark detection — SC2 has distinctive game-start sounds and loading-screen-to-game transition). After finding the game-start offset, map linearly: `vod_time = game_start_offset + (game_frame / 22.4)` (Faster speed is exactly 22.4 loops/second, deterministic). Phase 1 uses manual annotation as fallback; Phase 2 invests in automated detection.
**Alternatives:**
- Manual annotation only — most accurate but doesn't scale past Phase 1
- Subtitle content matching — match caster phrases to game events as anchor points; clever but brittle, casters don't always narrate at the exact moment
**Rationale:** SC2's game clock is deterministic — replays are frame-exact. Once the game-start offset in the VOD is found, the linear mapping has zero drift. The game-start moment is the most detectable landmark (distinct audio cue, visual transition from loading screen).
**Trade-offs:** Automated game-start detection requires audio/visual processing tooling. Deferred to Phase 2 — Phase 1 uses manual timestamps for 50-100 games.
**Sources:** SC2 game engine — Faster speed = 22.4 game loops/second (replay-index.md), existing IEM10JsonSimulatedGame frame arithmetic
**Exploration:** quick
**Status:** captured

## D3: Training example output format — game-state-centric JSON

**Choice:** Each training example is a structured JSON record with: `game_state` (army comp, economy, tech, supply, map control, recent events), `phase` (game phase tag), `phase_transition` (boolean + description), `commentary` (aligned caster text), `metadata` (tournament, players, map, matchup, frame range, VOD URL, segment type). Schema mirrors `GameStateTranslator.toMap()` output so training input matches inference input.
**Alternatives:**
- Conversation-format (chat-style) — system/user/assistant messages, ready for SFT but opinionated about target model
- Raw paired text — linearized game state → commentary string, simplest but loses structure
**Rationale:** Preserves full structure — can always flatten to chat or raw text for specific training runs, but can't recover structure from flattened text. JSON schema can evolve across phases without breaking earlier data.
**Trade-offs:** Requires a schema definition and Python extraction code that mirrors GameStateTranslator logic. More work upfront than raw text.
**Sources:** GameStateTranslator.toMap() — existing game state serialisation, LoL19-21 — linearized key-value pairs (~540 tokens average input)
**Exploration:** quick
**Depends on:** D2 (timestamp alignment provides frame range for each example)
**Status:** captured

## D4: Epic phasing — three-phase batched epic

**Choice:** Structure #249 as a three-phase epic with sub-issues, tracked as batches in the `.plan`:
- Phase 1 (Foundation): tournament catalog for 3-5 tournaments, manual VOD matching + manual game-start timestamps for ~50 games, subtitle extraction, replay JSON parsing, event-driven segmentation with phase tags, first batch of training examples. Deliverable: ~50 validated aligned pairs spanning full game arcs.
- Phase 2 (Automation): automated VOD matching within playlists (YouTube API + composite key), automated game-start detection (audio landmark), expand to ~500 games. Deliverable: semi-automated pipeline, hours not days per tournament.
- Phase 3 (Scale + Integration): full SC2EGSet coverage where VODs exist, quality scoring, integration with QuarkMind commentary pipeline (few-shot examples, prompt templates). Deliverable: production-grade dataset + pipeline integration.
**Alternatives:**
- Two-phase (compressed) — merge Phase 1+2, automate upfront. Higher initial risk if methodology assumptions are wrong.
- Four-phase (stretched) — split Phase 1 into catalog-only + first examples. More checkpoints but slower to first useful output.
**Rationale:** Phase 1 validates the methodology with minimal tooling investment. Phase 2 invests in automation only after the approach is proven. Phase 3 is the payoff. Each phase is an independent sub-issue that can be picked up in a separate session.
**Trade-offs:** Phase 1 includes manual work that Phase 2 automates — some effort is "throwaway." Acceptable because the manual work validates the pipeline design.
**Sources:** LoL19-21 — started with 220 games (LoL19) before scaling to 650 (LoL19-21)
**Exploration:** quick
**Status:** captured

## D5: Game state extraction — Python reimplementation

**Choice:** Write a lightweight Python extractor that reads SC2EGSet tracker events and produces structured snapshots (army comp, economy, tech, supply, map control, recent events). SC2EGSet JSON already has `PlayerStatsEvent` (minerals, gas, workers), `UnitBornEvent`/`UnitDiedEvent` (army composition), `UpgradeEvent` (tech) — no protobuf/MPQ parsing needed.
**Alternatives:**
- Call Java from Python — use `IEM10JsonSimulatedGame` + `GameStateTranslator` via subprocess/GraalPy. Guarantees exact parity with inference-time state but heavyweight dependency for a one-time pipeline.
**Rationale:** The SC2EGSet JSON is already structured — the Python extractor is straightforward. Exact parity with Java `GameStateTranslator` isn't critical for training data; close-enough representations are sufficient for learning commentary patterns.
**Trade-offs:** Minor divergence between training-time and inference-time game state representation. Acceptable because the model learns commentary patterns, not exact state formats.
**Sources:** IEM10JsonSimulatedGame.java — existing SC2EGSet JSON parsing, GameStateTranslator.toMap() — target schema reference
**Exploration:** quick
**Depends on:** D3 (output format defines the target schema the extractor must produce)
**Status:** captured
