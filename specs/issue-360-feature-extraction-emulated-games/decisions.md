# Decisions — #360 Feature Extraction from Emulated Games

## D1: Extraction approach

**Choice:** GameState diff-based synthetic event emitter
**Alternatives:**
- Direct snapshot serializer — simpler but requires Python pipeline changes, breaks single-format principle
- StrippedReplayFeatureExtractor adapter — large refactor of 1340-line class for minimal gain; EmulatedGame has full state and doesn't need reconstitution logic
**Rationale:** Produces standard game_json format with zero downstream changes. EmulatedGame's typed GameState collections (units, buildings, upgrades, economy) map 1:1 to synthetic tracker events. Matches #384 spec §6.4 preferred approach. Follows protocol PP-20260528-612dee (extraction in dedicated class).
**Trade-offs:** Diff logic needs care around combat-heavy ticks where many units die simultaneously. Slightly more complex than direct serialization.
**Sources:** #384 spec §6.4, PP-20260528-612dee, StrippedReplayFeatureExtractor.java, EmulatedGame.java
**Exploration:** quick
**Status:** captured

## D2: Diff strategy

**Choice:** Count-based diffs (compare unit/building type counts between ticks)
**Alternatives:**
- Tag-based tracking — track individual unit tags across ticks, more accurate event timeline but more complex; only matters if downstream needs per-unit events
**Rationale:** The Python pipeline (sc2egset_extractor.py) only consumes counts per type at each second, not individual unit identities. Count-based diffs produce exactly what's needed with less complexity. Tag tracking adds no training data value.
**Trade-offs:** Loses individual unit identity, so events can't distinguish "unit A died and unit B was born" from "net zero change." Training data doesn't need this distinction.
**Sources:** sc2egset_extractor.py, FeatureIndexMaps.java
**Exploration:** quick
**Depends on:** D1 (extraction approach)
**Status:** captured

## D3: PlayerStats emission source

**Choice:** Use EmulatedGame's existing PlayerEconomyStats (toFeatureVector() produces the 13-field array)
**Alternatives:**
- Recompute independently from raw unit/building counts and mineral/vespene state — redundant with what EmulatedGame already tracks
**Rationale:** EmulatedGame already maintains PlayerEconomyStats with the same 13 fields the tracker events carry. Using it directly avoids duplication and stays consistent with the runtime feature extraction path (DroolsScoutingTask.buildSnapshot uses the same data).
**Trade-offs:** Couples the extractor to EmulatedGame's economy accuracy — but that's the whole point; if economy is inaccurate, the training data should reflect that.
**Sources:** EmulatedGame.java (snapshot()), PlayerEconomyStats.java, TrackerEventFeatureExtractor.java
**Exploration:** quick
**Depends on:** D1 (extraction approach)
**Status:** captured
