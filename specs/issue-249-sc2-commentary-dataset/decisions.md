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
