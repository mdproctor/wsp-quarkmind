# SC2 Commentary Dataset — Phase 2 (Automation) + Phase 3 (Scale + Integration)

**Issue:** casehubio/quarkmind#310
**Parent:** casehubio/quarkmind#252 (SC2 strategy knowledge infrastructure)
**Date:** 2026-09-14
**Decisions:** D1–D7 in `decisions.md`
**Builds on:** Phase 1 spec (`docs/specs/issue-249-sc2-commentary-dataset/2026-09-14-sc2-commentary-dataset-design.md`)

## 1. Overview

Phase 1 delivered a working end-to-end pipeline: 30 IEM10 Taipei games, 235 validated training examples, 54 tests passing. The pipeline extracts game state from SC2EGSet replays, aligns YouTube caster commentary via timestamp mapping, segments into event-driven windows with game-phase context, and outputs structured JSON training examples.

Phase 2 automates the manual bottlenecks (VOD matching, tournament catalog curation) and scales from 1 to 10-15 tournaments (~500 aligned games). Phase 3 extends to full SC2EGSet coverage where VODs exist, adds per-example quality scoring, and integrates the dataset into QuarkMind's live commentary pipeline via few-shot examples and RAG retrieval.

**Target outputs:**
- ~500 aligned games (Phase 2) → full SC2EGSet coverage (Phase 3)
- Per-example quality scores (0.0–1.0) for downstream filtering
- Prompt template library keyed by (game_phase, event_type)
- RAG-backed few-shot example selection for the live commentary pipeline

## 2. Extended Pipeline Architecture

```
SC2EGSet replays (JSON)          YouTube Data API v3
        |                              |
        v                              v
  Parse tracker events         Search playlists by
  Extract game state           player names + map
        |                              |
        v                              v
  Event detection              VOD matching (composite key)
  Phase tagging                Confidence scoring
        |                              |
        +----------+-------------------+
                   v
          Subtitle extraction (yt-dlp)
          Priority: manual > auto-gen > Whisper
                   |
                   v
          Transcript-based offset estimation
          (forward cues + backward cues + interpolation)
                   |
                   v
          Timestamp alignment + segmentation
                   |
                   v
          Quality scoring (multi-factor composite)
                   |
                   v
          Training examples (JSON)
                   |
           +-------+-------+
           v               v
    RAG index          Prompt template
    (embedding +       library (phase ×
     vector store)     event matrix)
           |               |
           +-------+-------+
                   v
          Commentary pipeline integration
          (few-shot example selection at inference time)
```

### New pipeline stages (vs Phase 1)

| Stage | Input | Output | Tooling |
|-------|-------|--------|---------|
| Automated VOD matching | Tournament playlist + replay metadata | replay→VOD pairs with confidence | YouTube Data API v3 |
| Catalog expansion | SC2EGSet tournament list | tournament→channel→playlist mapping | Semi-automated script + human review |
| Quality scoring | Training examples + alignment metadata | Per-example confidence 0.0–1.0 | Multi-factor composite scorer |
| RAG indexing | High-quality training examples | Searchable vector index | Embedding model + vector store |
| Template library | Scored examples + phase/event tags | (phase, event_type) → prompt templates | Curation script |

## 3. Phase 2: Automation

### 3.1 Automated VOD matching (D3)

Replace manual `vod_id` curation with YouTube Data API search within known playlists.

**Matching algorithm:**

1. For each replay in a tournament, extract: `player1_name`, `player2_name`, `map_name`, `duration_seconds`
2. Query the tournament's YouTube playlist via `playlistItems.list` API
3. For each playlist item, compute a composite match score:
   - **Player name match** (0–40 points): both names present in title = 40, one name = 15
   - **Map name match** (0–20 points): map name in title = 20, partial match = 10
   - **Duration similarity** (0–20 points): `max(0, 20 - abs(vod_duration - replay_duration - 120) / 6)` — VODs are typically 1-3 min longer than replays (pre/post-game segments)
   - **Stage match** (0–10 points): if stage info (Quarter Final, Semifinal, etc.) is known and matches title
   - **Game number** (0–10 points): "Game N" in title matches replay sequence number
4. Accept matches with score >= 60; flag 40-60 for manual review; reject < 40

**Output:** Updated match YAML with `vod_id`, `confidence`, `match_score`, `match_method: "api"`.

**New module:** `src/vod_matcher.py`

```python
@dataclass
class VodCandidate:
    vod_id: str
    title: str
    duration_sec: int
    match_score: float
    score_breakdown: dict[str, float]

def match_replays_to_vods(
    tournament: TournamentEntry,
    replays: list[ReplayData],
    api_key: str,
) -> list[MatchResult]:
    """Match replays to VODs within a tournament's YouTube playlists."""
```

**API key management:** Read from `YOUTUBE_API_KEY` environment variable. No key hardcoded in code or config. The `pyproject.toml` dependency is `google-api-python-client` for the YouTube Data API.

### 3.2 Offset detection — transcript-based (D2)

Use the existing `estimate_offsets.py` as-is. No audio fingerprinting.

The estimator already handles series VODs (multiple games in one video) with:
- Forward cues: "loaded into game N", "game N is underway"
- Backward cues: "GG", "takes game N" minus replay duration
- Sequential constraint enforcement
- Interpolation for missing games
- VOD bounds validation

**At scale:** The estimator will encounter more subtitle quality variation (auto-generated vs. Whisper vs. manual). The quality scoring system (§4.2) flags games with low-confidence offsets so downstream consumers can filter.

**No changes needed** to `estimate_offsets.py` for Phase 2.

### 3.3 Tournament catalog expansion (D4)

Semi-automated expansion from 1 to 10-15 tournaments.

**New module:** `src/catalog_builder.py`

```python
def build_catalog_entry(
    sc2egset_name: str,
    api_key: str,
) -> TournamentEntry | None:
    """Search YouTube for a tournament's channel and playlists.
    
    1. Derive search terms from SC2EGSet tournament name
       (e.g., "2017_GSL_Season_1" → "GSL 2017 Season 1 StarCraft")
    2. Search for the channel (ESL, AfreecaTV, DreamHack, etc.)
    3. List playlists on the channel matching the tournament
    4. Return a proposed TournamentEntry for human review
    """
```

**Known YouTube channels for SC2 esports:**

| Channel | SC2EGSet tournaments |
|---------|---------------------|
| ESL Archives | IEM (all seasons), ESL Pro League |
| AfreecaTV (GSL) | GSL (all seasons) |
| DreamHackSC2 | DreamHack (all events) |
| StarCraft | WCS (all seasons), BlizzCon |
| BaseTradeTV | HomeStory Cup |

The script proposes entries; human reviews and confirms before adding to `tournament-catalog.yaml`. Each tournament entry includes the channel ID, playlist URL, and a list of `vod_series` entries (one per series in the bracket).

**SC2EGSet download:** The 72 tournament ZIPs are hosted on Zenodo (`https://zenodo.org/records/14963484`). Add a download script (`src/download_sc2egset.py`) that fetches specific tournaments by name, caches them locally under `data/sc2egset/`, and extracts the JSON replay files.

### 3.4 Quality validation at scale

**Automated spot-check protocol:**

1. For each tournament processed, randomly sample 5 examples
2. Compute alignment drift: `abs(estimated_game_end - vod_caster_gg_timestamp)`
3. Flag tournaments with mean drift > 10 seconds for manual review
4. Report per-tournament alignment accuracy in `stats.json`

**Low-confidence flagging:**

- Games where `estimate_offsets` produces `confidence: "low"` are flagged
- Games where VOD match score < 60 are excluded from the dataset
- Games with commentary density < 0.5 tokens/second are flagged (likely ads/analysis desk content)

## 4. Phase 3: Scale + Integration

### 4.1 Full SC2EGSet coverage

Process all 55 SC2EGSet tournaments where VODs exist. Expected yield:
- SC2EGSet total: 17,930 games across 55 tournaments
- Estimated VOD coverage: ~40-60% (many older tournaments have VODs removed)
- Estimated aligned examples: 3,000-8,000 (depending on VOD availability and subtitle quality)

**Batch processing:** The pipeline already processes one tournament at a time. Add a batch orchestrator that iterates over the catalog, processes each tournament, and aggregates statistics.

**New module:** `src/batch_pipeline.py`

```python
def run_batch(
    catalog_path: Path,
    sc2egset_dir: Path,
    subtitles_dir: Path,
    output_dir: Path,
) -> BatchResult:
    """Process all tournaments in the catalog sequentially."""
```

**Coverage report:** After a full run, produce a coverage report showing:
- Per-tournament: games with VODs, games processed, examples generated, quality distribution
- Overall: total examples, matchup distribution, phase distribution, quality histogram

### 4.2 Per-example quality scoring (D5)

Multi-factor composite score (0.0–1.0) computed for each training example.

**Factors and weights:**

| Factor | Weight | Computation |
|--------|--------|-------------|
| Subtitle source | 0.25 | manual=1.0, auto-generated=0.7, whisper=0.5 |
| Offset confidence | 0.30 | high=1.0, medium=0.6, low=0.2 |
| Commentary density | 0.20 | `min(1.0, tokens / (duration_sec * 2.0))` — caps at 2 tokens/sec |
| Segment coherence | 0.25 | Keyword overlap ratio between commentary text and game events |

**Segment coherence heuristic:**

Extract domain keywords from the game state snapshot (unit names, building names, upgrade names, player names) and compute the fraction of commentary tokens that match domain terms. Higher overlap = commentary is about the game, not ads or analysis desk.

```python
def compute_quality_score(example: dict) -> float:
    """Compute composite quality score for a training example."""
```

**Quality tiers:**
- **High** (≥ 0.7): suitable for few-shot examples and prompt templates
- **Medium** (0.4–0.7): suitable for RAG retrieval, not template curation
- **Low** (< 0.4): excluded from downstream use, kept for statistics

The score is written into each training example JSON as `"quality_score": 0.85`.

### 4.3 Commentary pipeline integration — few-shot + RAG (D6)

Connect the training dataset to QuarkMind's live commentary pipeline.

**Layer 1: Static few-shot examples**

Curate the top-scoring examples (quality ≥ 0.7) as static few-shot examples. Select 2-3 per (phase, event_type) cell in the template matrix. These are committed to the project repo as JSON files.

**Output:** `quarkmind-sc2/src/main/resources/commentary/few-shot/` — one JSON file per template cell, containing 2-3 curated examples.

**Layer 2: RAG retrieval**

Two stages: build-time curation (Python, FAISS) and runtime selection (Java, structured matching).

**Build-time (Python):** Embed all medium+ quality examples (quality ≥ 0.4) using `all-MiniLM-L6-v2` via `sentence-transformers`. Build a FAISS index for curation assistance — finding similar examples, detecting duplicates, selecting diverse few-shot sets per template cell. The FAISS index is an offline tool, not a runtime dependency.

**Runtime (Java):** Use structured metadata matching rather than vector similarity. Each example is pre-indexed by `(phase, event_type, matchup)`. At inference time, `FewShotRetriever` filters the example pool by the current game phase and event type, then ranks by matchup similarity and quality score. For a dataset of ~5,000-8,000 examples, brute-force filtering over pre-loaded JSON is fast (<1ms).

This avoids embedding model or vector store dependencies at runtime. The structured fields (phase, event_type, matchup) are the natural retrieval dimensions — the same ones casters use to calibrate their commentary style.

**New modules:**
- `src/embed.py` — embedding generation (build-time curation only)
- `src/index.py` — FAISS index build/query (build-time curation only)
- `src/export_examples.py` — exports scored examples as JSON for Java consumption
- Java: `io.quarkmind.plugin.commentary.FewShotRetriever` — loads pre-indexed JSON examples, filters by structured metadata at runtime

**Integration point:** `CommentaryWorkerFactory` (static factory, not CDI). Two methods build prompts:
- `executeReactiveCommentary()` — play-by-play (1-2 sentences, triggered by L2 moments)
- `executeNarrativeCommentary()` — color commentary (2-3 sentences, accumulated ~45s window)

Both construct `ChatRequest.builder().messages(systemMessage, userMessage)`. Few-shot examples inject as `UserMessage`/`AiMessage` pairs between the system message and the final user message:

```java
ChatRequest request = ChatRequest.builder()
    .messages(
        systemMessage,
        // few-shot examples injected here
        new UserMessage(fewShotGameState1),
        new AiMessage(fewShotCommentary1),
        new UserMessage(fewShotGameState2),
        new AiMessage(fewShotCommentary2),
        // actual user message
        userMessage
    )
    .build();
```

**Precedent:** `CoachingWorkerFactory` already embeds inline examples in the system prompt via `CoachingStyle`. The commentary integration uses true multi-turn few-shot (separate messages) for better in-context learning.

The `FewShotRetriever` is called by `CommentaryWorkerFactory` before building the ChatRequest. It takes the current game phase, event type, and game state, and returns 2-3 relevant (game_state, commentary) pairs from the FAISS index.

### 4.4 Prompt template library (D7)

Templates keyed by `(game_phase, event_type)`.

**Matrix:**

| | battle | expansion | tech_transition | macro_economy |
|---|---|---|---|---|
| opening | ✓ | ✓ | ✓ | ✓ |
| early_aggression | ✓ | — | ✓ | ✓ |
| mid_game | ✓ | ✓ | ✓ | ✓ |
| late_game | ✓ | ✓ | ✓ | ✓ |
| endgame | ✓ | — | — | ✓ |

Cells marked `—` are rare/empty in typical games and may not have enough high-quality examples. Leave them unpopulated rather than using low-quality filler.

**Template structure:**

```json
{
  "phase": "mid_game",
  "event_type": "battle",
  "system_prompt_fragment": "You are an esports commentator narrating a mid-game battle...",
  "few_shot_examples": [
    {
      "game_state_summary": "...",
      "commentary": "..."
    }
  ],
  "commentary_style_notes": "Casters emphasize army supply counts, unit composition advantages, micro highlights, and predict outcomes."
}
```

**Output:** `quarkmind-sc2/src/main/resources/commentary/templates/` — one JSON file per populated matrix cell.

**Template selection at inference time:** Match the current game phase and detected event type to select the template. The RAG retriever then fills in the most relevant few-shot examples from the full dataset.

## 5. New Dependencies

| Dependency | Purpose | Phase |
|------------|---------|-------|
| `google-api-python-client` | YouTube Data API v3 | Phase 2 |
| `sentence-transformers` | Embedding generation for RAG | Phase 3 |
| `faiss-cpu` | Vector index for RAG retrieval | Phase 3 |

All are optional dependencies (in `pyproject.toml` `[project.optional-dependencies]`), not core.

## 6. Project Structure Changes

```
quarkmind-dataset/
  src/
    vod_matcher.py          # NEW: YouTube Data API VOD matching
    catalog_builder.py      # NEW: semi-automated catalog expansion
    download_sc2egset.py    # NEW: Zenodo tournament ZIP downloader
    batch_pipeline.py       # NEW: multi-tournament batch orchestrator
    quality_scorer.py       # NEW: multi-factor quality scoring
    embed.py                # NEW: sentence embedding generation (build-time curation)
    index.py                # NEW: FAISS index build/query (build-time curation)
    export_examples.py      # NEW: export scored examples as JSON for Java
    pipeline.py             # MODIFIED: add quality scoring to output
    estimate_offsets.py     # UNCHANGED
    ...existing modules...
  data/
    sc2egset/               # NEW: downloaded SC2EGSet tournament ZIPs (gitignored)
  output/
    embeddings/             # NEW: generated embeddings (gitignored)
    index/                  # NEW: FAISS index files (gitignored)

quarkmind-sc2/
  src/main/resources/commentary/
    few-shot/               # NEW: curated few-shot examples per template cell
    templates/              # NEW: prompt template library JSON files
  src/main/java/io/quarkmind/plugin/commentary/
    FewShotRetriever.java   # NEW: loads pre-indexed JSON, structured metadata matching
```

## 7. Batch Structure for .plan

The 8 tasks are organized into 3 batches with natural exit points:

**Batch 1 — Automation infrastructure (Phase 2 core)**
1. Automated VOD matching (YouTube Data API integration)
2. Semi-automated tournament catalog expansion
3. Scale to 10-15 tournaments (first batch run)

**Batch 2 — Quality + coverage (Phase 2 validation + Phase 3 scale)**
4. Per-example quality scoring (multi-factor composite)
5. Quality validation at scale (spot-check protocol, flagging)
6. Full SC2EGSet coverage (batch processing all 55 tournaments)

**Batch 3 — Integration (Phase 3 integration)**
7. Commentary pipeline integration (few-shot + RAG)
8. Prompt template library (phase × event matrix)

Child issues will be created during the planning phase (writing-plans).

Each batch is a safe exit point — the branch can be closed after any batch with useful output delivered.

## 8. Testing Strategy

**Unit tests (pytest, no external dependencies):**
- `test_vod_matcher.py` — composite score computation, candidate ranking
- `test_catalog_builder.py` — tournament name parsing, search term derivation
- `test_quality_scorer.py` — factor computation, composite scoring, tier classification
- `test_embed.py` — embedding dimension validation, batch processing
- `test_index.py` — FAISS index build/query round-trip
- `test_batch_pipeline.py` — multi-tournament aggregation, coverage reporting

**Integration tests (require API key or network):**
- `test_vod_matcher_live.py` — live YouTube API search (marked with `@pytest.mark.live`)
- `test_download_sc2egset.py` — Zenodo download (marked with `@pytest.mark.live`)

**Validation tests (require full dataset run):**
- `test_quality_distribution.py` — asserts quality score distribution is reasonable (mean > 0.5, < 10% low-quality)
- `test_coverage_report.py` — asserts per-tournament statistics are populated

All live/validation tests are excluded from the default `pytest` run via markers.

## References

- Phase 1 spec: `docs/specs/issue-249-sc2-commentary-dataset/2026-09-14-sc2-commentary-dataset-design.md`
- Phase 1 decisions: `docs/specs/issue-249-sc2-commentary-dataset/decisions.md`
- SC2EGSet paper (Nature 2023): https://www.nature.com/articles/s41597-023-02510-7
- SC2EGSet Zenodo: https://zenodo.org/records/14963484
- Existing offset estimator: `quarkmind-dataset/src/estimate_offsets.py`
- Existing segmentation: `quarkmind-dataset/src/segment.py`
- Tournament catalog: `quarkmind-dataset/catalog/tournament-catalog.yaml`
- Commentary design spec (2026-07-06, #181)
- `CommentaryWorkerFactory.java` — LLM prompt construction and chat execution
- `InlineCommentaryDispatcher.java` — async/sync dispatch modes
- `CoachingWorkerFactory.java` — precedent for inline examples in system prompt
- Issue #310 body — Phase 2/3 task breakdown
