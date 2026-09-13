# SC2 Commentary Training Dataset — Replay-to-Caster-Transcript Alignment

**Issue:** casehubio/quarkmind#249
**Date:** 2026-09-14
**Decisions:** D1–D5 in `decisions.md`

## 1. Overview

No SC2-specific commentary-replay paired dataset exists publicly. Training or grounding commentary/coaching LLMs on SC2 data requires pairing game state with human expert commentary. The methodology exists from LoL19-21 (Wang & Yoshinaga 2021/2024) but hasn't been applied to SC2.

This spec defines a pipeline that:
1. Matches SC2EGSet esports replays to YouTube tournament VODs
2. Extracts caster commentary via subtitle tracks (manual > auto-generated > Whisper ASR)
3. Aligns transcripts to replay timestamps using game-start detection + deterministic frame mapping
4. Segments into event-driven windows with game-phase context
5. Outputs structured JSON training examples pairing game state snapshots with commentary

**Downstream uses:**
- Fine-tuning a commentary generation model (game state → caster-style narration)
- Grounding the existing QuarkMind commentary pipeline with few-shot examples and improved prompts
- Correcting the early-game bias in current commentary/coaching — casters narrate the full game arc

**What this is NOT:**
- Not a real-time system — this is a batch data pipeline, run offline to produce a static dataset
- Not production code — one-time Python scripts kept for reproducibility
- Not SC2-specific tooling for QuarkMind's game loop — the output is training data consumed downstream

## 2. Pipeline Architecture

```
SC2EGSet replays (JSON)          YouTube VODs
        |                              |
        v                              v
  Parse tracker events         Download subtitles
  Extract game state           (manual > auto-gen > Whisper)
        |                              |
        v                              v
  Event detection <---- Tournament Catalog ----> VOD matching
  Phase tagging                (player, map, duration)
        |                              |
        +----------+-------------------+
                   v
          Timestamp alignment
          (game-start offset + linear mapping)
                   |
                   v
          Event-driven segmentation
          with game-phase context
                   |
                   v
          Training examples (JSON)
          {game_state, phase, commentary, metadata}
```

### Pipeline stages

| Stage | Input | Output | Tooling |
|-------|-------|--------|---------|
| Tournament catalog | SC2EGSet tournament metadata | tournament→playlist mapping (YAML) | Manual curation + Liquipedia |
| VOD matching | Catalog + replay metadata | replay→VOD pairs with confidence | `yt-dlp` metadata, composite key matching |
| Subtitle extraction | VOD URLs | Timestamped transcript (`.vtt`) | `yt-dlp --write-subs`, Whisper fallback |
| Game state extraction | SC2EGSet JSON | Structured snapshots per frame range | Python extractor (reimplements GameStateTranslator logic) |
| Timestamp alignment | VOD subtitle timestamps + replay frame numbers | Frame-aligned commentary segments | Game-start offset + `frame / 22.4` |
| Segmentation | Aligned commentary + game events | Event-driven windows with phase tags | Python segmenter using tracker events |
| Output | Segments | Training example JSON | Schema serialisation |

## 3. Tournament Catalog (D1)

The catalog is the foundation of the matching strategy. It maps each SC2EGSet tournament to its YouTube broadcast source.

### Format

```yaml
# tournament-catalog.yaml
tournaments:
  - sc2egset_name: "2016_IEM_10_Taipei"
    display_name: "IEM Season 10 Taipei"
    year: 2016
    youtube_channels:
      - channel_id: "UCy1Ms_5qBTawC-k7PVjHXKQ"  # ESL SC2
        playlist_id: "PLhCH_nPE4JTqKvJvmCrfR5Jk..."
    liquipedia_url: "https://liquipedia.net/starcraft2/IEM_Season_X/..."
    game_speed: "Faster"  # 22.4 loops/sec (all competitive SC2)
    notes: "Already have 30 replays in quarkmind-sc2/replays/"

  - sc2egset_name: "2017_GSL_Season_1"
    display_name: "GSL 2017 Season 1"
    year: 2017
    youtube_channels:
      - channel_id: "..."  # AfreecaTV
    liquipedia_url: "..."
```

### Matching algorithm

For each replay in a tournament:

1. Extract metadata: `player1_name`, `player2_name`, `map_name`, `duration_seconds` from SC2EGSet JSON (`ToonPlayerDescMap`, `metadata.mapName`, `header.elapsedGameLoops / 22.4`)
2. Search the tournament's YouTube playlist for VODs containing both player names in the title
3. Score candidates by: title match (player names, map name), duration similarity (±2 min tolerance for pre/post-game segments in VODs)
4. Accept matches above confidence threshold; flag ambiguous matches for manual review

### Match output

```yaml
# matches/2016_IEM_10_Taipei.yaml
matches:
  - replay_hash: "095724b..."
    players: ["ByuN", "Lilbow"]
    map: "Lerilak Crest"
    replay_duration_sec: 382
    vod:
      url: "https://youtube.com/watch?v=..."
      title: "ByuN vs Lilbow Game 1 - IEM Taipei 2016"
      vod_duration_sec: 540
      game_start_offset_sec: 85  # manual in Phase 1, automated in Phase 2
      subtitle_source: "manual"  # manual | auto-generated | whisper
    confidence: "high"  # high | medium | manual-review
```

## 4. Subtitle Extraction

Priority order for transcript quality (D1 supplementary):

1. **Manually uploaded subtitles** — highest quality, some tournament channels provide edited transcripts
2. **YouTube auto-generated subtitles** — proven viable (LoL19-21: 6.8% WER), free, already timestamped
3. **Whisper ASR** — fallback for VODs without any subtitle track; use `whisper-large-v3` for best accuracy on SC2-specific terminology

### Extraction command

```bash
# Prefer manual subs, fall back to auto-generated
yt-dlp --write-subs --sub-langs "en" --skip-download "$VOD_URL"

# If no manual subs available, get auto-generated
yt-dlp --write-auto-subs --sub-langs "en" --skip-download "$VOD_URL"

# Output: .vtt file with timestamped caption segments
```

### VTT format (what we get)

```
00:01:25.000 --> 00:01:28.500
and ByuN is going for a very aggressive
opening here with the reaper expand

00:01:28.500 --> 00:01:32.000
meanwhile Lilbow hasn't even scouted yet
he's completely in the dark
```

Each segment has a start/end timestamp and 1-3 lines of text. Segments are typically 2-5 seconds long.

## 5. Timestamp Alignment (D2)

### The alignment equation

SC2 competitive games run at Faster speed = exactly 22.4 game loops per second. Replays are frame-exact — no drift, no variation.

```
vod_timestamp = game_start_offset + (game_frame / 22.4)
game_frame = (vod_timestamp - game_start_offset) * 22.4
```

### Game-start offset detection

**Phase 1 (manual):** Human watches the VOD, records the timestamp where the game clock starts (after loading screen, player intros, countdown). Stored in the match YAML as `game_start_offset_sec`.

**Phase 2 (automated):** Detect the game-start moment programmatically:
- **Audio detection:** SC2 has a distinctive "game started" sound effect. Use audio fingerprinting (e.g., `librosa` cross-correlation against a reference clip) to find the offset.
- **Visual detection (backup):** The transition from loading screen (black with player info) to in-game (minimap visible, resource counters appear) is a detectable visual landmark.

### Alignment validation

After computing the offset, validate with spot checks:
- First supply depot/pylon typically happens at ~25-35 game seconds. If the caster says "first pylon going down" near `offset + 30s`, the alignment is correct.
- Game end: `offset + (total_game_loops / 22.4)` should approximately match when the caster announces the winner.

## 6. Game State Extraction (D5)

A Python extractor reads SC2EGSet JSON tracker events and produces structured snapshots at specified frame ranges.

### SC2EGSet tracker events used

| Event type | Fields extracted | Maps to |
|------------|-----------------|---------|
| `PlayerStatsEvent` | minerals, vespene, food_used, food_made, workers | Economy snapshot |
| `UnitBornEvent` | unit_type, x, y, owner | Army composition, unit positions |
| `UnitDiedEvent` | unit_tag, killer | Army losses, engagement detection |
| `UnitTypeChangeEvent` | unit_tag, new_type | Tech transitions (e.g., Siege Tank → Siege Mode) |
| `UpgradeEvent` | upgrade_name, player | Tech level |

### Structured snapshot schema

```json
{
  "game_frame": 5040,
  "game_time_sec": 225.0,
  "player": {
    "race": "Protoss",
    "minerals": 350,
    "gas": 200,
    "supply_used": 44,
    "supply_cap": 54,
    "worker_count": 22,
    "army_composition": {
      "Stalker": 6,
      "Zealot": 4,
      "Sentry": 2
    },
    "tech": ["WarpGate", "Blink"],
    "buildings": {
      "Nexus": 1,
      "Gateway": 3,
      "CyberneticsCore": 1,
      "TwilightCouncil": 1
    },
    "recent_events": [
      {"frame": 4900, "type": "UNIT_BORN", "unit": "Stalker"},
      {"frame": 5000, "type": "UPGRADE_COMPLETE", "upgrade": "Blink"}
    ]
  },
  "opponent": {
    "race": "Terran",
    "known_units": {
      "Marine": 8,
      "Marauder": 3
    },
    "known_buildings": {
      "CommandCenter": 1,
      "Barracks": 2
    }
  }
}
```

The schema intentionally mirrors `GameStateTranslator.toMap()` field names so the training input format aligns with what the commentary pipeline provides at inference time.

### Opponent visibility

SC2EGSet replays contain full information for both players (observer perspective). For training data, we include the full opponent state — this matches what a caster sees (they have observer view). At inference time in QuarkMind, the opponent state is partial (fog of war), but the model can learn to commentate on what is known and speculate on what isn't.

## 7. Event-Driven Segmentation with Game-Phase Context

### Event detection

The Python extractor identifies game events from tracker event patterns:

| Event type | Detection pattern |
|------------|-------------------|
| Battle | Cluster of `UnitDiedEvent` within a short frame window (>3 deaths within 100 frames) |
| Expansion | `UnitBornEvent` for Nexus/CommandCenter/Hatchery at a new location |
| Tech transition | `UpgradeEvent` or `UnitBornEvent` for tech buildings (e.g., RoboticsFacility, Starport) |
| Army movement | Centroid shift of army units > threshold between snapshots |
| Supply block | `food_used >= food_cap` persisting for >50 frames |
| Worker saturation | Worker count crossing saturation thresholds (16, 24 per base) |

### Segmentation algorithm

1. Detect all game events across the replay
2. Create segments anchored to each event: the segment spans from the previous event (or game start) to a configurable window after the event (default: 15 game-seconds post-event)
3. Merge overlapping segments (rapid consecutive events become one segment)
4. Fill gaps: quiet periods between events become "macro/economy" segments with a minimum length of 30 game-seconds

### Game-phase tagging

Each segment is tagged with its game phase. Phase boundaries are determined by game-time heuristics calibrated from the IEM10 dataset:

| Phase | Typical timing | Key indicators |
|-------|---------------|----------------|
| `opening` | 0–3 min | First buildings, worker production, initial scouting |
| `early_aggression` | 2–5 min | First combat units, proxy buildings, early harass |
| `mid_game` | 5–12 min | Tech transitions, expansion timing, army composition divergence |
| `late_game` | 12–20 min | Max supply approaches, high-tech units, positional play |
| `endgame` | 20+ min | Resource depletion, decisive engagements, base trades |

Segments that span a phase boundary are flagged with `phase_transition: true` and include a `transition_description` (e.g., "early_aggression → mid_game: third base started, first colossus produced").

## 8. Training Example Format (D3)

### Schema

```json
{
  "id": "iem10-taipei-2016-095724b-seg-003",
  "version": "1.0",
  "game_state": { "...structured snapshot from section 6..." },
  "phase": "mid_game",
  "phase_transition": false,
  "transition_description": null,
  "commentary": "ByuN is pushing out with a huge marine marauder force, he's got stim researched and he's looking to end this game before Lilbow can get that third base up and running",
  "segment": {
    "type": "battle",
    "game_frame_start": 4500,
    "game_frame_end": 5200,
    "game_time_start_sec": 200.9,
    "game_time_end_sec": 232.1
  },
  "metadata": {
    "tournament": "IEM Season 10 Taipei 2016",
    "player1": "ByuN",
    "player1_race": "Terran",
    "player2": "Lilbow",
    "player2_race": "Protoss",
    "matchup": "TvP",
    "map": "Lerilak Crest",
    "game_duration_sec": 382,
    "vod_url": "https://youtube.com/watch?v=...",
    "subtitle_source": "auto-generated",
    "replay_hash": "095724b..."
  }
}
```

### Dataset statistics file

Each pipeline run produces a `stats.json` alongside the training examples:

```json
{
  "total_examples": 847,
  "games_processed": 48,
  "tournaments": 3,
  "phase_distribution": {
    "opening": 95,
    "early_aggression": 142,
    "mid_game": 310,
    "late_game": 215,
    "endgame": 85
  },
  "segment_type_distribution": {
    "battle": 280,
    "expansion": 120,
    "tech_transition": 98,
    "macro_economy": 245,
    "phase_transition": 104
  },
  "matchup_distribution": {
    "PvT": 180, "TvP": 180,
    "PvZ": 150, "ZvP": 150,
    "PvP": 95, "TvZ": 92
  },
  "avg_commentary_tokens": 85,
  "avg_game_state_tokens": 320
}
```

## 9. Project Structure

```
quarkmind-dataset/                  # Python package, top-level in quarkmind repo
  pyproject.toml                    # minimal deps: yt-dlp, whisper, librosa (Phase 2)
  README.md                        # usage instructions
  catalog/
    tournament-catalog.yaml         # SC2EGSet tournament → YouTube mapping
    matches/                        # per-tournament match files (replay → VOD)
  src/
    extract_state.py               # SC2EGSet JSON → structured game state snapshots
    extract_subtitles.py           # VOD → timestamped transcript (.vtt)
    align.py                       # timestamp alignment (offset + linear mapping)
    segment.py                     # event-driven segmentation + phase tagging
    pipeline.py                    # orchestrates all stages end-to-end
  output/
    examples/                      # generated training examples (JSON-per-game)
    stats.json                     # dataset statistics
  tests/
    test_extract_state.py          # validates extraction against known IEM10 values
    test_segment.py                # validates segmentation boundaries
    test_align.py                  # validates timestamp math
```

## 10. Epic Structure (D4)

Three phases, each a sub-issue of #249, tracked as batches in `.plan`:

### Phase 1 — Foundation (target: ~50 aligned games)

Sub-issues:
1. **Build tournament catalog** — curate 3-5 well-known tournaments (IEM Taipei 2016, GSL, WCS) with YouTube playlist IDs
2. **Manual VOD matching** — match ~50 replays to VODs using composite key, manually record game-start offsets
3. **Subtitle extraction pipeline** — `yt-dlp` wrapper, VTT parsing, subtitle quality classification
4. **Game state extractor** — Python reimplementation reading SC2EGSet JSON, producing structured snapshots
5. **Alignment + segmentation** — timestamp alignment, event-driven segmentation with phase tags
6. **First dataset output** — end-to-end pipeline producing ~50 games of training examples, stats report

### Phase 2 — Automation (target: ~500 aligned games)

Sub-issues:
1. **Automated VOD matching** — YouTube Data API integration, composite key scoring, confidence thresholds
2. **Game-start detection** — audio fingerprinting for automated offset detection
3. **Scale to 10-15 tournaments** — expand catalog, batch-process matches
4. **Quality validation** — spot-check alignment accuracy, flag low-confidence examples

### Phase 3 — Scale + Integration (target: full SC2EGSet where VODs exist)

Sub-issues:
1. **Full SC2EGSet coverage** — process all 55 tournaments, report coverage statistics
2. **Quality scoring** — per-example confidence score based on subtitle quality, alignment accuracy, segment coherence
3. **Commentary pipeline integration** — generate few-shot prompt examples from highest-quality training pairs
4. **Prompt template library** — phase-specific and event-specific commentary templates grounded in real caster language

## 11. Quality and Validation

### Alignment accuracy

- **Spot-check protocol:** For each tournament in Phase 1, manually verify 5 examples by watching the VOD at the computed timestamp and confirming the commentary matches the game state.
- **Drift detection:** Compare game-end timestamp (`offset + total_frames / 22.4`) against caster's "GG" announcement. Drift > 5 seconds indicates a bad offset.

### Commentary quality

- **Subtitle source tracking:** Each example records whether its transcript came from manual subs, auto-generated, or Whisper. Downstream consumers can filter by quality.
- **Language filter:** Only English-language broadcasts. Multi-language tournaments may have separate English streams.
- **Non-commentary filtering:** Remove pre-game analysis, post-game interviews, and ad breaks. Detect via: subtitle gaps > 30 seconds (ads), phrases like "let's go to the desk" (studio segments), timestamps outside the game-start/game-end window.

### Game state accuracy

- **Cross-validate against existing tests:** The Python extractor's output for IEM10 Taipei games can be compared against `IEM10JsonSimulatedGameTest` assertions — same input data, should produce equivalent army counts, economy values, and event sequences.

## 12. Scope Boundaries

**In scope:**
- Tournament catalog curation (YAML)
- Replay-to-VOD matching (composite key within playlists)
- Subtitle extraction (`yt-dlp`, Whisper fallback)
- Game state extraction from SC2EGSet JSON
- Timestamp alignment (manual Phase 1, automated Phase 2)
- Event-driven segmentation with game-phase context
- Structured JSON training example output
- Phase 1 deliverable: ~50 aligned games

**Out of scope:**
- Model fine-tuning (consumes the dataset, not part of the pipeline)
- Real-time commentary generation (that's the existing commentary pipeline + #290)
- Non-English commentary (future extension)
- Replays without matching VODs (skipped, not synthesised)
- Video frame extraction (we use subtitle text, not visual frames)

## References

- SC2EGSet: https://www.nature.com/articles/s41597-023-02510-7 — replay dataset (17,930 games)
- LoL19-21 (Wang & Yoshinaga 2021/2024) — methodology for game-state-to-commentary alignment
- TacticCraft (2025): https://arxiv.org/html/2507.15618 — tactical classification from community text
- AI-Generated Game Commentary survey (2025): https://arxiv.org/html/2506.17294v1 — approaches and datasets
- `IEM10JsonSimulatedGame.java` — existing SC2EGSet JSON parsing in QuarkMind
- `GameStateTranslator.java` — target schema reference for game state snapshots
- Commentary design spec (2026-07-06, #181) — dual-pattern commentary architecture
- Sync replay-commentary spec (2026-09-12, #290) — synchronized playback (consumption side)
- Coach mode spec (2026-07-19, #230) — where #249 was identified
- `quarkmind-sc2/replays/replay-index.md` — existing replay datasets and frame rate constants
