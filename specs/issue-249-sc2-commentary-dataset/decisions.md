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
