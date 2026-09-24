# SC2 Commentary Dataset Phase 2+3 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #310 — epic: SC2 commentary dataset Phase 2+3 — automation, scale, and integration
**Issue group:** #310

**Goal:** Scale the commentary dataset pipeline from 1 tournament (30 games) to full SC2EGSet coverage, add quality scoring, and integrate the training data into QuarkMind's live commentary pipeline via few-shot examples and structured metadata RAG.

**Architecture:** Python data pipeline extending the Phase 1 foundation (`quarkmind-dataset/`). New modules for YouTube API VOD matching, tournament catalog expansion, quality scoring, and batch processing. Java integration adds a `FewShotRetriever` that loads pre-indexed JSON training examples and injects them into `CommentaryWorkerFactory`'s ChatRequest message list.

**Tech Stack:** Python 3.11+, YouTube Data API v3 (`google-api-python-client`), `sentence-transformers` + `faiss-cpu` (build-time curation), Java/Quarkus (commentary integration), LangChain4j ChatRequest API.

## Global Constraints

- All new Python modules go in `quarkmind-dataset/src/`
- All new Python tests go in `quarkmind-dataset/tests/`
- YouTube API key read from `YOUTUBE_API_KEY` env var — never hardcoded
- Existing modules (`parse_replays.py`, `extract_state.py`, `extract_subtitles.py`, `align.py`, `segment.py`, `estimate_offsets.py`) are not modified except where explicitly noted
- `pipeline.py` is modified only to add quality scoring to output
- Java changes only in `quarkmind-sc2/` — `io.quarkmind.plugin.commentary` package
- Test commands: `cd quarkmind-dataset && python -m pytest tests/ -v` (Python), `mvn test -pl quarkmind-sc2 -Dtest=<TestClass> -q` (Java)
- `data/sc2egset/` and `output/embeddings/` and `output/index/` must be gitignored

---

## Batch 1: Automation Infrastructure

### Task 1: VOD Matcher — YouTube Data API Integration

**Files:**
- Create: `quarkmind-dataset/src/vod_matcher.py`
- Create: `quarkmind-dataset/tests/test_vod_matcher.py`
- Modify: `quarkmind-dataset/pyproject.toml` (add `google-api-python-client` to optional deps)

**Interfaces:**
- Consumes: `catalog.Tournament` (tournament metadata), `parse_replays.ReplayData` (player names, map, duration)
- Produces: `VodCandidate(vod_id, title, duration_sec, match_score, score_breakdown)`, `MatchResult(replay_hash, candidates, best_match, confidence)`

- [ ] **Step 1: Write failing test — composite score computation**

```python
# tests/test_vod_matcher.py
from src.vod_matcher import compute_match_score


def test_perfect_match_scores_100():
    score, breakdown = compute_match_score(
        vod_title="ByuN vs Lilbow Game 1 - IEM Taipei Quarter Final",
        vod_duration_sec=500,
        player1="ByuN",
        player2="Lilbow",
        map_name="Lerilak Crest",
        replay_duration_sec=382,
        stage="Quarter Final",
        game_number=1,
    )
    assert score >= 60
    assert breakdown["players"] == 40  # both names present


def test_one_player_name_scores_15():
    score, breakdown = compute_match_score(
        vod_title="ByuN vs Unknown Game 1",
        vod_duration_sec=500,
        player1="ByuN",
        player2="Lilbow",
        map_name="Lerilak Crest",
        replay_duration_sec=382,
    )
    assert breakdown["players"] == 15


def test_no_player_names_scores_0():
    score, breakdown = compute_match_score(
        vod_title="StarCraft II Grand Final",
        vod_duration_sec=500,
        player1="ByuN",
        player2="Lilbow",
        map_name="Lerilak Crest",
        replay_duration_sec=382,
    )
    assert breakdown["players"] == 0


def test_map_name_in_title():
    score, breakdown = compute_match_score(
        vod_title="ByuN vs Lilbow on Lerilak Crest",
        vod_duration_sec=500,
        player1="ByuN",
        player2="Lilbow",
        map_name="Lerilak Crest",
        replay_duration_sec=382,
    )
    assert breakdown["map"] == 20


def test_duration_similarity_close():
    score, breakdown = compute_match_score(
        vod_title="ByuN vs Lilbow",
        vod_duration_sec=500,  # replay=382, diff=118, close to expected ~120
        player1="ByuN",
        player2="Lilbow",
        map_name="Lerilak Crest",
        replay_duration_sec=382,
    )
    assert breakdown["duration"] >= 15


def test_duration_similarity_far():
    score, breakdown = compute_match_score(
        vod_title="ByuN vs Lilbow",
        vod_duration_sec=3600,  # way too long
        player1="ByuN",
        player2="Lilbow",
        map_name="Lerilak Crest",
        replay_duration_sec=382,
    )
    assert breakdown["duration"] == 0
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd quarkmind-dataset && python -m pytest tests/test_vod_matcher.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'src.vod_matcher'`

- [ ] **Step 3: Implement composite scoring**

```python
# src/vod_matcher.py
"""Match SC2EGSet replays to YouTube VODs via composite key scoring."""
from dataclasses import dataclass


@dataclass
class VodCandidate:
    vod_id: str
    title: str
    duration_sec: int
    match_score: float
    score_breakdown: dict[str, float]


@dataclass
class MatchResult:
    replay_hash: str
    candidates: list[VodCandidate]
    best_match: VodCandidate | None
    confidence: str  # high, medium, low, none


def compute_match_score(
    vod_title: str,
    vod_duration_sec: int,
    player1: str,
    player2: str,
    map_name: str,
    replay_duration_sec: float,
    stage: str | None = None,
    game_number: int | None = None,
) -> tuple[float, dict[str, float]]:
    """Compute composite match score between a replay and a VOD candidate."""
    title_lower = vod_title.lower()
    breakdown = {}

    # Player name match (0-40)
    p1_found = player1.lower() in title_lower
    p2_found = player2.lower() in title_lower
    if p1_found and p2_found:
        breakdown["players"] = 40
    elif p1_found or p2_found:
        breakdown["players"] = 15
    else:
        breakdown["players"] = 0

    # Map name match (0-20)
    if map_name and map_name.lower() in title_lower:
        breakdown["map"] = 20
    elif map_name and any(word.lower() in title_lower for word in map_name.split() if len(word) > 3):
        breakdown["map"] = 10
    else:
        breakdown["map"] = 0

    # Duration similarity (0-20)
    expected_overhead = 120  # VODs typically 1-3 min longer
    diff = abs(vod_duration_sec - replay_duration_sec - expected_overhead)
    breakdown["duration"] = max(0, 20 - diff / 6)

    # Stage match (0-10)
    if stage and stage.lower() in title_lower:
        breakdown["stage"] = 10
    else:
        breakdown["stage"] = 0

    # Game number match (0-10)
    if game_number and f"game {game_number}" in title_lower:
        breakdown["game_number"] = 10
    else:
        breakdown["game_number"] = 0

    total = sum(breakdown.values())
    return round(total, 1), breakdown


def classify_confidence(score: float) -> str:
    if score >= 60:
        return "high"
    elif score >= 40:
        return "medium"
    else:
        return "low"


def match_replay_to_playlist(
    player1: str,
    player2: str,
    map_name: str,
    replay_duration_sec: float,
    playlist_items: list[dict],
    stage: str | None = None,
    game_number: int | None = None,
) -> MatchResult:
    """Match a single replay against a list of playlist items.

    Each playlist item is a dict with keys: vod_id, title, duration_sec.
    """
    candidates = []
    for item in playlist_items:
        score, breakdown = compute_match_score(
            vod_title=item["title"],
            vod_duration_sec=item.get("duration_sec", 0),
            player1=player1,
            player2=player2,
            map_name=map_name,
            replay_duration_sec=replay_duration_sec,
            stage=stage,
            game_number=game_number,
        )
        candidates.append(VodCandidate(
            vod_id=item["vod_id"],
            title=item["title"],
            duration_sec=item.get("duration_sec", 0),
            match_score=score,
            score_breakdown=breakdown,
        ))

    candidates.sort(key=lambda c: c.match_score, reverse=True)
    best = candidates[0] if candidates else None
    confidence = classify_confidence(best.match_score) if best else "none"

    return MatchResult(
        replay_hash="",
        candidates=candidates[:5],
        best_match=best if confidence != "none" else None,
        confidence=confidence,
    )
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd quarkmind-dataset && python -m pytest tests/test_vod_matcher.py -v`
Expected: all PASS

- [ ] **Step 5: Write failing test — playlist matching**

```python
# append to tests/test_vod_matcher.py
from src.vod_matcher import match_replay_to_playlist, classify_confidence


def test_match_replay_to_playlist_finds_best():
    items = [
        {"vod_id": "abc", "title": "ByuN vs Lilbow Game 1 - IEM Taipei QF", "duration_sec": 500},
        {"vod_id": "def", "title": "StarCraft II Preview Show", "duration_sec": 1800},
        {"vod_id": "ghi", "title": "ByuN vs Lilbow Game 2 - IEM Taipei QF", "duration_sec": 500},
    ]
    result = match_replay_to_playlist(
        player1="ByuN", player2="Lilbow",
        map_name="Lerilak Crest", replay_duration_sec=382,
        playlist_items=items, stage="QF", game_number=1,
    )
    assert result.best_match is not None
    assert result.best_match.vod_id == "abc"
    assert result.confidence == "high"


def test_match_replay_empty_playlist():
    result = match_replay_to_playlist(
        player1="ByuN", player2="Lilbow",
        map_name="Lerilak Crest", replay_duration_sec=382,
        playlist_items=[],
    )
    assert result.best_match is None
    assert result.confidence == "none"


def test_classify_confidence_thresholds():
    assert classify_confidence(80) == "high"
    assert classify_confidence(60) == "high"
    assert classify_confidence(50) == "medium"
    assert classify_confidence(30) == "low"
```

- [ ] **Step 6: Run tests — should pass (implementation already covers this)**

Run: `cd quarkmind-dataset && python -m pytest tests/test_vod_matcher.py -v`
Expected: all PASS

- [ ] **Step 7: Write failing test — YouTube API integration**

```python
# append to tests/test_vod_matcher.py
from unittest.mock import patch, MagicMock
from src.vod_matcher import fetch_playlist_items


def test_fetch_playlist_items_parses_api_response():
    mock_service = MagicMock()
    mock_service.playlistItems.return_value.list.return_value.execute.return_value = {
        "items": [
            {
                "snippet": {"title": "ByuN vs Lilbow Game 1", "resourceId": {"videoId": "abc123"}},
                "contentDetails": {"videoId": "abc123"},
            }
        ],
        "nextPageToken": None,
    }
    mock_service.videos.return_value.list.return_value.execute.return_value = {
        "items": [
            {"id": "abc123", "contentDetails": {"duration": "PT8M20S"}},
        ]
    }
    items = fetch_playlist_items(mock_service, "PLtest123")
    assert len(items) == 1
    assert items[0]["vod_id"] == "abc123"
    assert items[0]["title"] == "ByuN vs Lilbow Game 1"
    assert items[0]["duration_sec"] == 500
```

- [ ] **Step 8: Implement YouTube API integration**

```python
# add to src/vod_matcher.py
import re
import os


def parse_iso8601_duration(duration: str) -> int:
    """Parse ISO 8601 duration (PT1H2M3S) to seconds."""
    match = re.match(r"PT(?:(\d+)H)?(?:(\d+)M)?(?:(\d+)S)?", duration)
    if not match:
        return 0
    h, m, s = (int(g) if g else 0 for g in match.groups())
    return h * 3600 + m * 60 + s


def build_youtube_service(api_key: str | None = None):
    """Build a YouTube Data API v3 service client."""
    from googleapiclient.discovery import build
    key = api_key or os.environ.get("YOUTUBE_API_KEY", "")
    return build("youtube", "v3", developerKey=key)


def fetch_playlist_items(service, playlist_id: str) -> list[dict]:
    """Fetch all items from a YouTube playlist with durations."""
    items = []
    page_token = None
    while True:
        request = service.playlistItems().list(
            part="snippet,contentDetails",
            playlistId=playlist_id,
            maxResults=50,
            pageToken=page_token,
        )
        response = request.execute()
        for item in response.get("items", []):
            video_id = item["contentDetails"]["videoId"]
            items.append({
                "vod_id": video_id,
                "title": item["snippet"]["title"],
                "duration_sec": 0,
            })
        page_token = response.get("nextPageToken")
        if not page_token:
            break

    # Batch-fetch durations
    video_ids = [it["vod_id"] for it in items]
    for i in range(0, len(video_ids), 50):
        batch = video_ids[i:i + 50]
        vresp = service.videos().list(
            part="contentDetails",
            id=",".join(batch),
        ).execute()
        dur_map = {}
        for v in vresp.get("items", []):
            dur_map[v["id"]] = parse_iso8601_duration(
                v["contentDetails"]["duration"]
            )
        for it in items:
            if it["vod_id"] in dur_map:
                it["duration_sec"] = dur_map[it["vod_id"]]

    return items
```

- [ ] **Step 9: Run all tests**

Run: `cd quarkmind-dataset && python -m pytest tests/test_vod_matcher.py -v`
Expected: all PASS

- [ ] **Step 10: Add optional dependency to pyproject.toml**

```toml
# In pyproject.toml [project.optional-dependencies]
api = ["google-api-python-client>=2.100"]
```

- [ ] **Step 11: Commit**

```bash
git add quarkmind-dataset/src/vod_matcher.py quarkmind-dataset/tests/test_vod_matcher.py quarkmind-dataset/pyproject.toml
git commit -m "feat: YouTube Data API VOD matcher with composite scoring Refs #310"
```

---

### Task 2: Catalog Builder + SC2EGSet Downloader

**Files:**
- Create: `quarkmind-dataset/src/catalog_builder.py`
- Create: `quarkmind-dataset/src/download_sc2egset.py`
- Create: `quarkmind-dataset/tests/test_catalog_builder.py`
- Create: `quarkmind-dataset/tests/test_download_sc2egset.py`
- Modify: `quarkmind-dataset/.gitignore` (add `data/sc2egset/`)

**Interfaces:**
- Consumes: `vod_matcher.fetch_playlist_items`, `vod_matcher.match_replay_to_playlist`
- Produces: `build_catalog_entry(sc2egset_name, api_key) -> TournamentEntry | None`, `download_tournament(tournament_name, output_dir) -> Path | None`

- [ ] **Step 1: Write failing test — tournament name parsing**

```python
# tests/test_catalog_builder.py
from src.catalog_builder import parse_tournament_name, KNOWN_CHANNELS


def test_parse_iem_tournament():
    result = parse_tournament_name("2016_IEM_10_Taipei")
    assert result["search_terms"] == "IEM Season 10 Taipei StarCraft"
    assert result["year"] == 2016
    assert result["probable_channel"] == "ESL Archives"


def test_parse_gsl_tournament():
    result = parse_tournament_name("2017_GSL_Season_1")
    assert result["search_terms"] == "GSL 2017 Season 1 StarCraft"
    assert result["year"] == 2017
    assert result["probable_channel"] == "AfreecaTV"


def test_parse_dreamhack():
    result = parse_tournament_name("2015_DreamHack_Open_Valencia")
    assert result["year"] == 2015
    assert result["probable_channel"] == "DreamHack"


def test_known_channels_has_entries():
    assert len(KNOWN_CHANNELS) >= 5
    assert "ESL Archives" in KNOWN_CHANNELS
    assert "AfreecaTV" in KNOWN_CHANNELS
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd quarkmind-dataset && python -m pytest tests/test_catalog_builder.py -v`
Expected: FAIL

- [ ] **Step 3: Implement catalog builder**

```python
# src/catalog_builder.py
"""Semi-automated tournament catalog expansion from SC2EGSet names."""
import re
from dataclasses import dataclass
from pathlib import Path

import yaml


KNOWN_CHANNELS = {
    "ESL Archives": {"channel_id": "UCy1Ms_5qBTawC-k7PVjHXKQ", "patterns": ["IEM", "ESL"]},
    "AfreecaTV": {"channel_id": "UCK5eBtuoj_HkdXKHNmBLAXg", "patterns": ["GSL", "ASL"]},
    "DreamHack": {"channel_id": "UCHNMSfOcEoMOcn7kPYmKDIQ", "patterns": ["DreamHack"]},
    "StarCraft": {"channel_id": "UCBNhiiTfCNPwyMmA0ygpkJQ", "patterns": ["WCS", "BlizzCon"]},
    "BaseTradeTV": {"channel_id": "UCzRuPIvIyiVGhGgqWfvJHfg", "patterns": ["HomeStory"]},
}


@dataclass
class CatalogProposal:
    sc2egset_name: str
    display_name: str
    year: int
    search_terms: str
    probable_channel: str
    channel_id: str


def parse_tournament_name(sc2egset_name: str) -> dict:
    """Parse an SC2EGSet tournament name into search components."""
    parts = sc2egset_name.split("_")
    year = int(parts[0]) if parts[0].isdigit() else 0

    name_parts = [p for p in parts[1:] if not p.isdigit() or len(p) > 2]
    raw_name = " ".join(name_parts)

    probable_channel = _guess_channel(raw_name)

    # Build search terms
    season_match = re.search(r"Season[_ ]?(\d+)", sc2egset_name, re.IGNORECASE)
    season_suffix = f" Season {season_match.group(1)}" if season_match else ""

    number_match = re.search(r"_(\d{1,2})_", sc2egset_name)
    number_suffix = f" Season {number_match.group(1)}" if number_match and not season_match else ""

    search_name = raw_name.replace("_", " ")
    search_terms = f"{search_name}{season_suffix}{number_suffix} {year} StarCraft".strip()
    search_terms = re.sub(r"\s+", " ", search_terms)

    return {
        "sc2egset_name": sc2egset_name,
        "display_name": f"{raw_name.replace('_', ' ')} {year}",
        "year": year,
        "search_terms": search_terms,
        "probable_channel": probable_channel,
    }


def _guess_channel(name: str) -> str:
    name_upper = name.upper()
    for channel, info in KNOWN_CHANNELS.items():
        if any(p.upper() in name_upper for p in info["patterns"]):
            return channel
    return "Unknown"


def propose_catalog_entry(sc2egset_name: str) -> CatalogProposal:
    """Create a catalog entry proposal for human review."""
    parsed = parse_tournament_name(sc2egset_name)
    channel = parsed["probable_channel"]
    channel_id = KNOWN_CHANNELS.get(channel, {}).get("channel_id", "")
    return CatalogProposal(
        sc2egset_name=sc2egset_name,
        display_name=parsed["display_name"],
        year=parsed["year"],
        search_terms=parsed["search_terms"],
        probable_channel=channel,
        channel_id=channel_id,
    )


def write_catalog_entry(proposal: CatalogProposal, catalog_path: Path) -> None:
    """Append a confirmed proposal to the tournament catalog YAML."""
    data = yaml.safe_load(catalog_path.read_text()) if catalog_path.exists() else {"tournaments": []}
    entry = {
        "sc2egset_name": proposal.sc2egset_name,
        "display_name": proposal.display_name,
        "year": proposal.year,
        "youtube_channels": [{"channel_name": proposal.probable_channel, "channel_id": proposal.channel_id}],
        "game_speed": "Faster",
        "notes": "",
    }
    data["tournaments"].append(entry)
    catalog_path.write_text(yaml.dump(data, default_flow_style=False, sort_keys=False))
```

- [ ] **Step 4: Run tests**

Run: `cd quarkmind-dataset && python -m pytest tests/test_catalog_builder.py -v`
Expected: all PASS

- [ ] **Step 5: Write failing test — SC2EGSet downloader**

```python
# tests/test_download_sc2egset.py
from src.download_sc2egset import build_zenodo_url, list_available_tournaments


def test_build_zenodo_url():
    url = build_zenodo_url("2016_IEM_10_Taipei")
    assert "zenodo.org" in url
    assert "2016_IEM_10_Taipei" in url


def test_list_available_tournaments_from_index():
    # The SC2EGSet index file lists all tournament names
    tournaments = list_available_tournaments()
    assert isinstance(tournaments, list)
    assert len(tournaments) > 0
```

- [ ] **Step 6: Implement SC2EGSet downloader**

```python
# src/download_sc2egset.py
"""Download SC2EGSet tournament replay archives from Zenodo."""
import json
import urllib.request
from pathlib import Path

ZENODO_RECORD_ID = "14963484"
ZENODO_API_URL = f"https://zenodo.org/api/records/{ZENODO_RECORD_ID}"

# Known tournament names from SC2EGSet (subset — full list discovered via API)
SC2EGSET_TOURNAMENTS = [
    "2016_IEM_10_Taipei", "2017_GSL_Season_1", "2016_DreamHack_Open_Leipzig",
    "2017_WCS_Austin", "2016_IEM_11_Shanghai", "2015_DreamHack_Open_Valencia",
]


def build_zenodo_url(tournament_name: str) -> str:
    """Build the Zenodo download URL for a tournament ZIP."""
    return f"https://zenodo.org/records/{ZENODO_RECORD_ID}/files/{tournament_name}.zip"


def list_available_tournaments() -> list[str]:
    """List all tournament names available in the SC2EGSet Zenodo record."""
    try:
        with urllib.request.urlopen(ZENODO_API_URL, timeout=30) as resp:
            data = json.loads(resp.read())
            files = data.get("files", [])
            return [f["key"].replace(".zip", "") for f in files if f["key"].endswith(".zip")]
    except Exception:
        return list(SC2EGSET_TOURNAMENTS)


def download_tournament(tournament_name: str, output_dir: Path) -> Path | None:
    """Download a tournament ZIP from Zenodo if not already cached."""
    output_dir.mkdir(parents=True, exist_ok=True)
    zip_path = output_dir / f"{tournament_name}.zip"
    if zip_path.exists():
        return zip_path
    url = build_zenodo_url(tournament_name)
    try:
        urllib.request.urlretrieve(url, zip_path)
        return zip_path
    except Exception:
        return None
```

- [ ] **Step 7: Run tests**

Run: `cd quarkmind-dataset && python -m pytest tests/test_download_sc2egset.py tests/test_catalog_builder.py -v`
Expected: all PASS

- [ ] **Step 8: Update .gitignore**

Add `data/sc2egset/` to `quarkmind-dataset/.gitignore`.

- [ ] **Step 9: Commit**

```bash
git add quarkmind-dataset/src/catalog_builder.py quarkmind-dataset/src/download_sc2egset.py quarkmind-dataset/tests/test_catalog_builder.py quarkmind-dataset/tests/test_download_sc2egset.py quarkmind-dataset/.gitignore
git commit -m "feat: semi-automated catalog builder and SC2EGSet downloader Refs #310"
```

---

## Batch 2: Scale + Quality

### Task 3: Quality Scorer

**Files:**
- Create: `quarkmind-dataset/src/quality_scorer.py`
- Create: `quarkmind-dataset/tests/test_quality_scorer.py`
- Modify: `quarkmind-dataset/src/pipeline.py` (add quality scoring to output)

**Interfaces:**
- Consumes: Training example dict (from `pipeline.build_training_example`)
- Produces: `compute_quality_score(example, offset_confidence, subtitle_source) -> float`, `classify_quality_tier(score) -> str`

- [ ] **Step 1: Write failing test — quality factor computation**

```python
# tests/test_quality_scorer.py
from src.quality_scorer import (
    compute_quality_score,
    subtitle_source_score,
    offset_confidence_score,
    commentary_density_score,
    segment_coherence_score,
    classify_quality_tier,
)


def test_subtitle_source_manual():
    assert subtitle_source_score("manual") == 1.0


def test_subtitle_source_auto():
    assert subtitle_source_score("auto-generated") == 0.7


def test_subtitle_source_whisper():
    assert subtitle_source_score("whisper") == 0.5


def test_offset_confidence_high():
    assert offset_confidence_score("high") == 1.0


def test_offset_confidence_medium():
    assert offset_confidence_score("medium") == 0.6


def test_offset_confidence_low():
    assert offset_confidence_score("low") == 0.2


def test_commentary_density_normal():
    # 50 words over 30 seconds = 1.67 words/sec — good density
    score = commentary_density_score("this is a test " * 12 + "extra", 30.0)
    assert 0.5 < score <= 1.0


def test_commentary_density_sparse():
    # 3 words over 60 seconds — very sparse
    score = commentary_density_score("just three words", 60.0)
    assert score < 0.3


def test_segment_coherence_high():
    game_state = {
        "player": {"army_composition": {"Stalker": 6, "Zealot": 4},
                    "buildings": {"Nexus": 1}, "tech": ["Blink"]},
        "opponent": {"known_units": {"Marine": 8}},
    }
    commentary = "The Stalker force with Blink is pushing towards the Marines near the Nexus"
    score = segment_coherence_score(commentary, game_state)
    assert score > 0.3


def test_segment_coherence_low():
    game_state = {
        "player": {"army_composition": {"Stalker": 6}, "buildings": {}, "tech": []},
        "opponent": {"known_units": {}},
    }
    commentary = "Welcome back to the studio we have a great show for you today"
    score = segment_coherence_score(commentary, game_state)
    assert score < 0.2


def test_composite_score_range():
    score = compute_quality_score(
        commentary="Stalker push with Blink coming in",
        game_state={"player": {"army_composition": {"Stalker": 6}, "buildings": {}, "tech": ["Blink"]},
                     "opponent": {"known_units": {}}},
        duration_sec=30.0,
        subtitle_source="manual",
        offset_confidence="high",
    )
    assert 0.0 <= score <= 1.0


def test_classify_quality_tier():
    assert classify_quality_tier(0.8) == "high"
    assert classify_quality_tier(0.7) == "high"
    assert classify_quality_tier(0.5) == "medium"
    assert classify_quality_tier(0.3) == "low"
```

- [ ] **Step 2: Run tests to verify failure**

Run: `cd quarkmind-dataset && python -m pytest tests/test_quality_scorer.py -v`
Expected: FAIL

- [ ] **Step 3: Implement quality scorer**

```python
# src/quality_scorer.py
"""Multi-factor composite quality scoring for training examples."""
import re

WEIGHTS = {
    "subtitle_source": 0.25,
    "offset_confidence": 0.30,
    "commentary_density": 0.20,
    "segment_coherence": 0.25,
}


def subtitle_source_score(source: str) -> float:
    return {"manual": 1.0, "auto-generated": 0.7, "whisper": 0.5}.get(source, 0.3)


def offset_confidence_score(confidence: str) -> float:
    return {"high": 1.0, "medium": 0.6, "low": 0.2}.get(confidence, 0.1)


def commentary_density_score(commentary: str, duration_sec: float) -> float:
    if duration_sec <= 0:
        return 0.0
    tokens = len(commentary.split())
    tokens_per_sec = tokens / duration_sec
    return min(1.0, tokens_per_sec / 2.0)


def segment_coherence_score(commentary: str, game_state: dict) -> float:
    keywords = set()
    player = game_state.get("player", {})
    opponent = game_state.get("opponent", {})

    for name in player.get("army_composition", {}):
        keywords.add(name.lower())
    for name in player.get("buildings", {}):
        keywords.add(name.lower())
    for name in player.get("tech", []):
        keywords.add(name.lower())
    for name in opponent.get("known_units", {}):
        keywords.add(name.lower())
    for name in opponent.get("known_buildings", {}):
        keywords.add(name.lower())

    if not keywords:
        return 0.5  # no game state to compare against

    words = re.findall(r"\b\w+\b", commentary.lower())
    if not words:
        return 0.0
    matches = sum(1 for w in words if w in keywords)
    return min(1.0, matches / max(len(words) * 0.1, 1))


def compute_quality_score(
    commentary: str,
    game_state: dict,
    duration_sec: float,
    subtitle_source: str,
    offset_confidence: str,
) -> float:
    factors = {
        "subtitle_source": subtitle_source_score(subtitle_source),
        "offset_confidence": offset_confidence_score(offset_confidence),
        "commentary_density": commentary_density_score(commentary, duration_sec),
        "segment_coherence": segment_coherence_score(commentary, game_state),
    }
    return round(sum(factors[k] * WEIGHTS[k] for k in WEIGHTS), 3)


def classify_quality_tier(score: float) -> str:
    if score >= 0.7:
        return "high"
    elif score >= 0.4:
        return "medium"
    return "low"
```

- [ ] **Step 4: Run tests**

Run: `cd quarkmind-dataset && python -m pytest tests/test_quality_scorer.py -v`
Expected: all PASS

- [ ] **Step 5: Integrate quality scoring into pipeline.py**

Modify `pipeline.py` to add `quality_score` and `quality_tier` to each training example.

Add to the per-segment loop in `run_pipeline()`, after building the example:

```python
from src.quality_scorer import compute_quality_score, classify_quality_tier

# After: example = build_training_example(...)
duration_sec = (seg.game_frame_end - seg.game_frame_start) / LOOPS_PER_SEC
quality = compute_quality_score(
    commentary=seg.commentary,
    game_state=game_state,
    duration_sec=duration_sec,
    subtitle_source=match.subtitle_source,
    offset_confidence=getattr(match, 'offset_confidence', 'medium'),
)
example["quality_score"] = quality
example["quality_tier"] = classify_quality_tier(quality)
```

Also add `offset_confidence` to the `MatchEntry` dataclass in `catalog.py`:

```python
# In catalog.py MatchEntry
offset_confidence: str  # add after subtitle_source
```

- [ ] **Step 6: Run full test suite**

Run: `cd quarkmind-dataset && python -m pytest tests/ -v`
Expected: all PASS (existing tests still pass, new tests pass)

- [ ] **Step 7: Commit**

```bash
git add quarkmind-dataset/src/quality_scorer.py quarkmind-dataset/tests/test_quality_scorer.py quarkmind-dataset/src/pipeline.py quarkmind-dataset/src/catalog.py
git commit -m "feat: multi-factor quality scoring for training examples Refs #310"
```

---

### Task 4: Batch Pipeline + Coverage Report

**Files:**
- Create: `quarkmind-dataset/src/batch_pipeline.py`
- Create: `quarkmind-dataset/tests/test_batch_pipeline.py`
- Modify: `quarkmind-dataset/.gitignore` (add `output/embeddings/`, `output/index/`)

**Interfaces:**
- Consumes: `catalog.load_catalog`, `pipeline.run_pipeline`, `download_sc2egset.download_tournament`
- Produces: `run_batch(catalog_path, sc2egset_dir, subtitles_dir, output_dir) -> BatchResult`, `generate_coverage_report(batch_result) -> dict`

- [ ] **Step 1: Write failing test — batch result aggregation**

```python
# tests/test_batch_pipeline.py
from src.batch_pipeline import BatchResult, aggregate_stats, generate_coverage_report


def test_aggregate_stats_empty():
    result = aggregate_stats([])
    assert result["total_examples"] == 0
    assert result["games_processed"] == 0
    assert result["tournaments"] == 0


def test_aggregate_stats_multiple():
    stats = [
        {"total_examples": 100, "games_processed": 10, "tournaments": 1,
         "phase_distribution": {"opening": 20, "mid_game": 50, "late_game": 30},
         "segment_type_distribution": {"battle": 40, "macro_economy": 60},
         "matchup_distribution": {"PvT": 50, "TvZ": 50}},
        {"total_examples": 50, "games_processed": 5, "tournaments": 1,
         "phase_distribution": {"opening": 10, "mid_game": 30, "endgame": 10},
         "segment_type_distribution": {"battle": 20, "expansion": 30},
         "matchup_distribution": {"PvZ": 50}},
    ]
    result = aggregate_stats(stats)
    assert result["total_examples"] == 150
    assert result["games_processed"] == 15
    assert result["tournaments"] == 2
    assert result["phase_distribution"]["opening"] == 30
    assert result["matchup_distribution"]["PvT"] == 50


def test_coverage_report_structure():
    batch = BatchResult(
        total_examples=150,
        games_processed=15,
        tournaments_processed=2,
        tournaments_failed=[],
        per_tournament_stats={
            "IEM10": {"total_examples": 100, "games_processed": 10},
            "GSL": {"total_examples": 50, "games_processed": 5},
        },
        aggregate_stats={"total_examples": 150},
    )
    report = generate_coverage_report(batch)
    assert "IEM10" in report["per_tournament"]
    assert report["summary"]["total_examples"] == 150
```

- [ ] **Step 2: Run test to verify failure**

Run: `cd quarkmind-dataset && python -m pytest tests/test_batch_pipeline.py -v`
Expected: FAIL

- [ ] **Step 3: Implement batch pipeline**

```python
# src/batch_pipeline.py
"""Multi-tournament batch pipeline orchestrator."""
import json
from collections import Counter
from dataclasses import dataclass, field
from pathlib import Path

from src.catalog import load_catalog, Tournament
from src.pipeline import run_pipeline


@dataclass
class BatchResult:
    total_examples: int
    games_processed: int
    tournaments_processed: int
    tournaments_failed: list[str]
    per_tournament_stats: dict[str, dict]
    aggregate_stats: dict


def aggregate_stats(stats_list: list[dict]) -> dict:
    """Aggregate statistics from multiple tournament runs."""
    if not stats_list:
        return {"total_examples": 0, "games_processed": 0, "tournaments": 0,
                "phase_distribution": {}, "segment_type_distribution": {},
                "matchup_distribution": {}}

    total_examples = sum(s.get("total_examples", 0) for s in stats_list)
    games_processed = sum(s.get("games_processed", 0) for s in stats_list)

    phase_dist = Counter()
    type_dist = Counter()
    matchup_dist = Counter()
    for s in stats_list:
        phase_dist.update(s.get("phase_distribution", {}))
        type_dist.update(s.get("segment_type_distribution", {}))
        matchup_dist.update(s.get("matchup_distribution", {}))

    return {
        "total_examples": total_examples,
        "games_processed": games_processed,
        "tournaments": len(stats_list),
        "phase_distribution": dict(phase_dist),
        "segment_type_distribution": dict(type_dist),
        "matchup_distribution": dict(matchup_dist),
    }


def generate_coverage_report(batch: BatchResult) -> dict:
    """Generate a coverage report from batch results."""
    return {
        "summary": {
            "total_examples": batch.total_examples,
            "games_processed": batch.games_processed,
            "tournaments_processed": batch.tournaments_processed,
            "tournaments_failed": batch.tournaments_failed,
        },
        "per_tournament": batch.per_tournament_stats,
        "aggregate": batch.aggregate_stats,
    }


def run_batch(
    catalog_path: Path,
    sc2egset_dir: Path,
    subtitles_dir: Path,
    output_dir: Path,
) -> BatchResult:
    """Process all tournaments in the catalog."""
    catalog = load_catalog(catalog_path)
    all_stats = []
    per_tournament = {}
    failed = []

    for tournament in catalog:
        matches_path = catalog_path.parent / "matches" / f"{tournament.sc2egset_name}.yaml"
        replays_path = sc2egset_dir / f"{tournament.sc2egset_name}.zip"

        if not matches_path.exists():
            failed.append(f"{tournament.sc2egset_name}: no match file")
            continue
        if not replays_path.exists():
            failed.append(f"{tournament.sc2egset_name}: no replay ZIP")
            continue

        tournament_subtitles = subtitles_dir / tournament.sc2egset_name
        tournament_output = output_dir / tournament.sc2egset_name

        try:
            result = run_pipeline(
                match_file=matches_path,
                replays_zip=replays_path,
                subtitles_dir=tournament_subtitles,
                output_dir=tournament_output,
                tournament_name=tournament.display_name,
            )
            all_stats.append(result.stats)
            per_tournament[tournament.sc2egset_name] = result.stats
        except Exception as e:
            failed.append(f"{tournament.sc2egset_name}: {e}")

    agg = aggregate_stats(all_stats)
    total_examples = sum(s.get("total_examples", 0) for s in all_stats)
    total_games = sum(s.get("games_processed", 0) for s in all_stats)

    report = BatchResult(
        total_examples=total_examples,
        games_processed=total_games,
        tournaments_processed=len(all_stats),
        tournaments_failed=failed,
        per_tournament_stats=per_tournament,
        aggregate_stats=agg,
    )

    report_path = output_dir / "coverage_report.json"
    report_path.parent.mkdir(parents=True, exist_ok=True)
    report_path.write_text(json.dumps(generate_coverage_report(report), indent=2))

    return report
```

- [ ] **Step 4: Run tests**

Run: `cd quarkmind-dataset && python -m pytest tests/test_batch_pipeline.py -v`
Expected: all PASS

- [ ] **Step 5: Update .gitignore**

Add `output/embeddings/` and `output/index/` to `.gitignore`.

- [ ] **Step 6: Commit**

```bash
git add quarkmind-dataset/src/batch_pipeline.py quarkmind-dataset/tests/test_batch_pipeline.py quarkmind-dataset/.gitignore
git commit -m "feat: multi-tournament batch pipeline with coverage report Refs #310"
```

---

## Batch 3: Integration

### Task 5: Python RAG Tooling — Embed, Index, Export

**Files:**
- Create: `quarkmind-dataset/src/embed.py`
- Create: `quarkmind-dataset/src/export_examples.py`
- Create: `quarkmind-dataset/tests/test_embed.py`
- Create: `quarkmind-dataset/tests/test_export_examples.py`
- Modify: `quarkmind-dataset/pyproject.toml` (add `sentence-transformers` to optional deps)

**Interfaces:**
- Consumes: Training example JSON files (from `pipeline.py` output)
- Produces: `embed_examples(examples) -> list[list[float]]`, `export_for_java(examples, output_dir) -> Path`

- [ ] **Step 1: Write failing test — export for Java**

The export module is the critical output — it produces the JSON files that Java loads at runtime.

```python
# tests/test_export_examples.py
import json
from pathlib import Path
from src.export_examples import export_for_java, group_by_phase_event


def test_group_by_phase_event():
    examples = [
        {"phase": "mid_game", "segment": {"type": "battle"}, "quality_score": 0.8,
         "commentary": "big push", "game_state": {}, "metadata": {"matchup": "TvP"}},
        {"phase": "mid_game", "segment": {"type": "battle"}, "quality_score": 0.9,
         "commentary": "another push", "game_state": {}, "metadata": {"matchup": "PvZ"}},
        {"phase": "opening", "segment": {"type": "macro_economy"}, "quality_score": 0.6,
         "commentary": "standard opening", "game_state": {}, "metadata": {"matchup": "TvP"}},
    ]
    groups = group_by_phase_event(examples)
    assert ("mid_game", "battle") in groups
    assert len(groups[("mid_game", "battle")]) == 2
    assert ("opening", "macro_economy") in groups


def test_export_for_java_creates_files(tmp_path):
    examples = [
        {"id": "ex1", "phase": "mid_game", "segment": {"type": "battle"},
         "quality_score": 0.8, "quality_tier": "high",
         "commentary": "big push", "game_state": {"player": {"race": "Protoss"}},
         "metadata": {"matchup": "TvP", "player1": "ByuN"}},
        {"id": "ex2", "phase": "mid_game", "segment": {"type": "battle"},
         "quality_score": 0.5, "quality_tier": "medium",
         "commentary": "another push", "game_state": {"player": {"race": "Terran"}},
         "metadata": {"matchup": "PvZ", "player1": "Maru"}},
    ]
    output = export_for_java(examples, tmp_path, min_quality=0.4)
    few_shot_dir = tmp_path / "few-shot"
    assert few_shot_dir.exists()
    files = list(few_shot_dir.glob("*.json"))
    assert len(files) >= 1

    # Check file content
    data = json.loads(files[0].read_text())
    assert "phase" in data
    assert "event_type" in data
    assert "examples" in data


def test_export_filters_by_quality(tmp_path):
    examples = [
        {"id": "ex1", "phase": "mid_game", "segment": {"type": "battle"},
         "quality_score": 0.8, "quality_tier": "high",
         "commentary": "good", "game_state": {}, "metadata": {"matchup": "TvP"}},
        {"id": "ex2", "phase": "mid_game", "segment": {"type": "battle"},
         "quality_score": 0.2, "quality_tier": "low",
         "commentary": "bad", "game_state": {}, "metadata": {"matchup": "TvP"}},
    ]
    export_for_java(examples, tmp_path, min_quality=0.4)
    files = list((tmp_path / "few-shot").glob("*.json"))
    for f in files:
        data = json.loads(f.read_text())
        for ex in data["examples"]:
            assert ex["quality_score"] >= 0.4
```

- [ ] **Step 2: Run test to verify failure**

Run: `cd quarkmind-dataset && python -m pytest tests/test_export_examples.py -v`
Expected: FAIL

- [ ] **Step 3: Implement export module**

```python
# src/export_examples.py
"""Export scored training examples as JSON for Java consumption."""
import json
from collections import defaultdict
from pathlib import Path


def group_by_phase_event(examples: list[dict]) -> dict[tuple[str, str], list[dict]]:
    """Group examples by (phase, event_type)."""
    groups = defaultdict(list)
    for ex in examples:
        phase = ex.get("phase", "unknown")
        event_type = ex.get("segment", {}).get("type", "unknown")
        groups[(phase, event_type)].append(ex)
    return dict(groups)


def export_for_java(
    examples: list[dict],
    output_dir: Path,
    min_quality: float = 0.4,
    max_per_cell: int = 10,
) -> Path:
    """Export examples as JSON files for Java FewShotRetriever.

    Produces one JSON file per (phase, event_type) cell containing
    up to max_per_cell examples, sorted by quality_score descending.
    """
    few_shot_dir = output_dir / "few-shot"
    few_shot_dir.mkdir(parents=True, exist_ok=True)

    filtered = [ex for ex in examples if ex.get("quality_score", 0) >= min_quality]
    groups = group_by_phase_event(filtered)

    for (phase, event_type), group_examples in groups.items():
        sorted_examples = sorted(group_examples, key=lambda e: e.get("quality_score", 0), reverse=True)
        top_examples = sorted_examples[:max_per_cell]

        cell_data = {
            "phase": phase,
            "event_type": event_type,
            "example_count": len(top_examples),
            "examples": [
                {
                    "id": ex.get("id", ""),
                    "quality_score": ex.get("quality_score", 0),
                    "matchup": ex.get("metadata", {}).get("matchup", ""),
                    "game_state_summary": _summarize_game_state(ex.get("game_state", {})),
                    "commentary": ex.get("commentary", ""),
                }
                for ex in top_examples
            ],
        }

        filename = f"{phase}_{event_type}.json"
        (few_shot_dir / filename).write_text(json.dumps(cell_data, indent=2))

    # Write an index file listing all cells
    index = {
        "cells": [
            {"phase": p, "event_type": e, "count": len(exs)}
            for (p, e), exs in groups.items()
        ],
        "total_examples": sum(len(exs) for exs in groups.values()),
    }
    (few_shot_dir / "index.json").write_text(json.dumps(index, indent=2))

    return few_shot_dir


def _summarize_game_state(game_state: dict) -> str:
    """Create a compact text summary of game state for few-shot context."""
    parts = []
    player = game_state.get("player", {})
    opponent = game_state.get("opponent", {})

    if player.get("race"):
        parts.append(f"Player: {player['race']}")
    if player.get("supply_used") and player.get("supply_cap"):
        parts.append(f"Supply: {player['supply_used']}/{player['supply_cap']}")
    if player.get("army_composition"):
        army = ", ".join(f"{v} {k}" for k, v in player["army_composition"].items())
        parts.append(f"Army: {army}")
    if player.get("tech"):
        parts.append(f"Tech: {', '.join(player['tech'])}")
    if opponent.get("known_units"):
        enemy = ", ".join(f"{v} {k}" for k, v in opponent["known_units"].items())
        parts.append(f"Enemy: {enemy}")

    return "; ".join(parts) if parts else "No game state"
```

- [ ] **Step 4: Run tests**

Run: `cd quarkmind-dataset && python -m pytest tests/test_export_examples.py -v`
Expected: all PASS

- [ ] **Step 5: Write failing test — embedding generation**

```python
# tests/test_embed.py
from src.embed import embed_text, embed_examples


def test_embed_text_returns_vector():
    # Uses a mock/small model for unit testing
    vector = embed_text("Stalker push with Blink", model_name="mock")
    assert len(vector) > 0
    assert isinstance(vector[0], float)


def test_embed_examples_batch():
    examples = [
        {"commentary": "big push", "game_state": {"player": {"race": "Protoss"}}},
        {"commentary": "expanding", "game_state": {"player": {"race": "Terran"}}},
    ]
    vectors = embed_examples(examples, model_name="mock")
    assert len(vectors) == 2
```

- [ ] **Step 6: Implement embedding module**

```python
# src/embed.py
"""Sentence embedding generation for build-time RAG curation."""
import hashlib
import json
from pathlib import Path

MOCK_DIM = 64


def embed_text(text: str, model_name: str = "all-MiniLM-L6-v2") -> list[float]:
    """Embed a single text string."""
    if model_name == "mock":
        return _mock_embed(text)
    from sentence_transformers import SentenceTransformer
    model = _get_model(model_name)
    return model.encode(text).tolist()


def embed_examples(
    examples: list[dict],
    model_name: str = "all-MiniLM-L6-v2",
) -> list[list[float]]:
    """Embed a batch of training examples."""
    texts = [_example_to_text(ex) for ex in examples]
    if model_name == "mock":
        return [_mock_embed(t) for t in texts]
    from sentence_transformers import SentenceTransformer
    model = _get_model(model_name)
    return model.encode(texts).tolist()


def _example_to_text(example: dict) -> str:
    """Convert a training example to embedding text."""
    commentary = example.get("commentary", "")
    state = example.get("game_state", {})
    player = state.get("player", {})
    parts = [commentary]
    if player.get("race"):
        parts.append(player["race"])
    if player.get("army_composition"):
        parts.extend(player["army_composition"].keys())
    return " ".join(parts)


_model_cache = {}

def _get_model(name: str):
    if name not in _model_cache:
        from sentence_transformers import SentenceTransformer
        _model_cache[name] = SentenceTransformer(name)
    return _model_cache[name]


def _mock_embed(text: str) -> list[float]:
    """Deterministic mock embedding for testing."""
    h = hashlib.md5(text.encode()).hexdigest()
    return [int(h[i:i+2], 16) / 255.0 for i in range(0, MOCK_DIM * 2, 2)][:MOCK_DIM]
```

- [ ] **Step 7: Run tests**

Run: `cd quarkmind-dataset && python -m pytest tests/test_embed.py tests/test_export_examples.py -v`
Expected: all PASS

- [ ] **Step 8: Update pyproject.toml optional deps**

```toml
# In [project.optional-dependencies]
rag = ["sentence-transformers>=2.2"]
```

- [ ] **Step 9: Commit**

```bash
git add quarkmind-dataset/src/embed.py quarkmind-dataset/src/export_examples.py quarkmind-dataset/tests/test_embed.py quarkmind-dataset/tests/test_export_examples.py quarkmind-dataset/pyproject.toml
git commit -m "feat: Python RAG tooling — embed, export for Java consumption Refs #310"
```

---

### Task 6: Java FewShotRetriever + CommentaryWorkerFactory Integration

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/commentary/FewShotRetriever.java`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/commentary/FewShotExample.java`
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/commentary/FewShotRetrieverTest.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/commentary/CommentaryWorkerFactory.java` (inject few-shot messages)
- Create: `quarkmind-sc2/src/main/resources/commentary/few-shot/` (directory for JSON files)

**Interfaces:**
- Consumes: JSON files from `export_examples.py` (loaded as classpath resources)
- Produces: `FewShotRetriever.retrieve(phase, eventType, matchup, limit) -> List<FewShotExample>`

- [ ] **Step 1: Write failing test — FewShotRetriever**

```java
// quarkmind-sc2/src/test/java/io/quarkmind/plugin/commentary/FewShotRetrieverTest.java
package io.quarkmind.plugin.commentary;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class FewShotRetrieverTest {

    @Test
    void retrievesByPhaseAndEventType(@TempDir Path tempDir) throws IOException {
        Path fewShotDir = tempDir.resolve("few-shot");
        Files.createDirectories(fewShotDir);
        Files.writeString(fewShotDir.resolve("mid_game_battle.json"), """
            {
              "phase": "mid_game",
              "event_type": "battle",
              "example_count": 2,
              "examples": [
                {"id": "ex1", "quality_score": 0.9, "matchup": "TvP",
                 "game_state_summary": "Player: Terran; Army: 8 Marine, 3 Marauder",
                 "commentary": "Big push coming in from the Terran army"},
                {"id": "ex2", "quality_score": 0.8, "matchup": "PvZ",
                 "game_state_summary": "Player: Protoss; Army: 6 Stalker",
                 "commentary": "Stalker force moving across the map"}
              ]
            }
            """);
        Files.writeString(fewShotDir.resolve("index.json"), """
            {"cells": [{"phase": "mid_game", "event_type": "battle", "count": 2}],
             "total_examples": 2}
            """);

        FewShotRetriever retriever = FewShotRetriever.fromDirectory(fewShotDir);
        List<FewShotExample> results = retriever.retrieve("mid_game", "battle", null, 3);

        assertEquals(2, results.size());
        assertEquals("ex1", results.get(0).id());
        assertTrue(results.get(0).qualityScore() > results.get(1).qualityScore());
    }

    @Test
    void returnsEmptyForUnknownCell(@TempDir Path tempDir) throws IOException {
        Path fewShotDir = tempDir.resolve("few-shot");
        Files.createDirectories(fewShotDir);
        Files.writeString(fewShotDir.resolve("index.json"), """
            {"cells": [], "total_examples": 0}
            """);

        FewShotRetriever retriever = FewShotRetriever.fromDirectory(fewShotDir);
        List<FewShotExample> results = retriever.retrieve("endgame", "battle", null, 3);

        assertTrue(results.isEmpty());
    }

    @Test
    void filtersAndSortsByMatchup(@TempDir Path tempDir) throws IOException {
        Path fewShotDir = tempDir.resolve("few-shot");
        Files.createDirectories(fewShotDir);
        Files.writeString(fewShotDir.resolve("mid_game_battle.json"), """
            {
              "phase": "mid_game", "event_type": "battle", "example_count": 3,
              "examples": [
                {"id": "ex1", "quality_score": 0.9, "matchup": "TvP",
                 "game_state_summary": "...", "commentary": "..."},
                {"id": "ex2", "quality_score": 0.85, "matchup": "PvZ",
                 "game_state_summary": "...", "commentary": "..."},
                {"id": "ex3", "quality_score": 0.8, "matchup": "TvP",
                 "game_state_summary": "...", "commentary": "..."}
              ]
            }
            """);
        Files.writeString(fewShotDir.resolve("index.json"), """
            {"cells": [{"phase": "mid_game", "event_type": "battle", "count": 3}],
             "total_examples": 3}
            """);

        FewShotRetriever retriever = FewShotRetriever.fromDirectory(fewShotDir);
        List<FewShotExample> results = retriever.retrieve("mid_game", "battle", "TvP", 2);

        assertEquals(2, results.size());
        // TvP matches should be preferred
        assertEquals("TvP", results.get(0).matchup());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=FewShotRetrieverTest -q`
Expected: compilation failure — classes don't exist

- [ ] **Step 3: Implement FewShotExample record**

```java
// quarkmind-sc2/src/main/java/io/quarkmind/plugin/commentary/FewShotExample.java
package io.quarkmind.plugin.commentary;

public record FewShotExample(
    String id,
    double qualityScore,
    String matchup,
    String gameStateSummary,
    String commentary
) {}
```

- [ ] **Step 4: Implement FewShotRetriever**

```java
// quarkmind-sc2/src/main/java/io/quarkmind/plugin/commentary/FewShotRetriever.java
package io.quarkmind.plugin.commentary;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.io.IOException;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class FewShotRetriever {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private final Map<String, List<FewShotExample>> cellCache = new ConcurrentHashMap<>();
    private final Path baseDir;

    private FewShotRetriever(Path baseDir) {
        this.baseDir = baseDir;
    }

    public static FewShotRetriever fromDirectory(Path dir) {
        return new FewShotRetriever(dir);
    }

    public static FewShotRetriever fromClasspath(String resourcePath) {
        // Resolve classpath resource to a Path
        var url = FewShotRetriever.class.getClassLoader().getResource(resourcePath);
        if (url == null) {
            return new FewShotRetriever(Path.of("nonexistent"));
        }
        return new FewShotRetriever(Path.of(url.getPath()));
    }

    public List<FewShotExample> retrieve(String phase, String eventType,
                                          String matchup, int limit) {
        String cellKey = phase + "_" + eventType;
        List<FewShotExample> examples = cellCache.computeIfAbsent(cellKey, this::loadCell);

        if (examples.isEmpty()) {
            return List.of();
        }

        List<FewShotExample> ranked = new ArrayList<>(examples);
        if (matchup != null && !matchup.isEmpty()) {
            ranked.sort((a, b) -> {
                boolean aMatch = matchup.equals(a.matchup());
                boolean bMatch = matchup.equals(b.matchup());
                if (aMatch != bMatch) return aMatch ? -1 : 1;
                return Double.compare(b.qualityScore(), a.qualityScore());
            });
        }

        return ranked.subList(0, Math.min(limit, ranked.size()));
    }

    private List<FewShotExample> loadCell(String cellKey) {
        Path cellFile = baseDir.resolve(cellKey + ".json");
        if (!Files.exists(cellFile)) {
            return List.of();
        }
        try {
            JsonNode root = MAPPER.readTree(Files.readString(cellFile));
            JsonNode examplesNode = root.get("examples");
            if (examplesNode == null || !examplesNode.isArray()) {
                return List.of();
            }
            List<FewShotExample> result = new ArrayList<>();
            for (JsonNode ex : examplesNode) {
                result.add(new FewShotExample(
                    ex.path("id").asText(""),
                    ex.path("quality_score").asDouble(0),
                    ex.path("matchup").asText(""),
                    ex.path("game_state_summary").asText(""),
                    ex.path("commentary").asText("")
                ));
            }
            return result;
        } catch (IOException e) {
            return List.of();
        }
    }
}
```

- [ ] **Step 5: Run test**

Run: `mvn test -pl quarkmind-sc2 -Dtest=FewShotRetrieverTest -q`
Expected: all PASS

- [ ] **Step 6: Modify CommentaryWorkerFactory to inject few-shot examples**

In `CommentaryWorkerFactory.java`, modify `executeReactiveCommentary` (line ~117) and `executeNarrativeCommentary` (line ~152) to accept an optional `FewShotRetriever` parameter and inject few-shot messages into the ChatRequest.

Add a new static method:

```java
static List<ChatMessage> buildFewShotMessages(FewShotRetriever retriever,
                                                String phase, String eventType,
                                                String matchup) {
    if (retriever == null) {
        return List.of();
    }
    List<FewShotExample> examples = retriever.retrieve(phase, eventType, matchup, 2);
    List<ChatMessage> messages = new ArrayList<>();
    for (FewShotExample ex : examples) {
        messages.add(new UserMessage("Game state: " + ex.gameStateSummary()
                                     + "\nProvide commentary."));
        messages.add(new AiMessage(ex.commentary()));
    }
    return messages;
}
```

Modify the ChatRequest construction in both methods to include few-shot messages:

```java
List<ChatMessage> allMessages = new ArrayList<>();
allMessages.add(systemMessage);
allMessages.addAll(buildFewShotMessages(retriever, phase, eventType, matchup));
allMessages.add(userMessage);

ChatRequest request = ChatRequest.builder()
                                 .messages(allMessages)
                                 .build();
```

The `phase` and `eventType` are extracted from the input map — add helper methods to parse them from the game state keys already present in the trigger data.

- [ ] **Step 7: Run existing CommentaryWorkerFactory tests to verify no regression**

Run: `mvn test -pl quarkmind-sc2 -Dtest=CommentaryWorkerFactoryTest -q`
Expected: all PASS (existing tests should work since retriever is null by default)

- [ ] **Step 8: Create resources directory**

```bash
mkdir -p quarkmind-sc2/src/main/resources/commentary/few-shot
```

- [ ] **Step 9: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/plugin/commentary/FewShotRetriever.java quarkmind-sc2/src/main/java/io/quarkmind/plugin/commentary/FewShotExample.java quarkmind-sc2/src/test/java/io/quarkmind/plugin/commentary/FewShotRetrieverTest.java quarkmind-sc2/src/main/java/io/quarkmind/plugin/commentary/CommentaryWorkerFactory.java quarkmind-sc2/src/main/resources/commentary/few-shot/
git commit -m "feat: FewShotRetriever and CommentaryWorkerFactory integration Refs #310"
```

---

## References

- `specs/issue-310-sc2-commentary-phase2-3/2026-09-14-sc2-commentary-phase2-3-design.md` — design spec this plan implements
- `docs/specs/issue-249-sc2-commentary-dataset/2026-09-14-sc2-commentary-dataset-design.md` — Phase 1 spec
- `quarkmind-dataset/src/pipeline.py` — existing pipeline orchestrator
- `quarkmind-dataset/src/estimate_offsets.py` — existing offset estimator (unchanged)
- `quarkmind-dataset/src/catalog.py` — existing catalog/match loader
- `quarkmind-dataset/src/segment.py` — existing segmentation (event types + phases)
- `quarkmind-sc2/src/main/java/io/quarkmind/plugin/commentary/CommentaryWorkerFactory.java:117-129` — reactive commentary ChatRequest construction
- `quarkmind-sc2/src/main/java/io/quarkmind/plugin/commentary/CommentaryWorkerFactory.java:152-164` — narrative commentary ChatRequest construction
- GitHub #310 — epic: SC2 commentary dataset Phase 2+3
- GitHub #249 — parent: SC2 commentary dataset research
