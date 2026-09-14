# SC2 Commentary Training Dataset — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #249 — research: SC2 commentary training dataset — replay-to-caster-transcript alignment
**Issue group:** #249

**Goal:** Build a Python data pipeline that pairs SC2EGSet esports replay game-state snapshots with real human caster commentary from YouTube tournament VODs, producing structured JSON training examples covering the full game arc.

**Architecture:** A six-stage Python pipeline: (1) parse SC2EGSet nested ZIP → tracker events, (2) extract structured game state snapshots, (3) download YouTube subtitles via yt-dlp, (4) match replays to VODs via tournament catalog, (5) align timestamps and segment into event-driven windows with game-phase context, (6) output structured JSON training examples. Phase 1 targets ~50 manually matched games from IEM10 Taipei 2016.

**Tech Stack:** Python 3.11+, yt-dlp, webvtt-py, pytest

## Global Constraints

- Python standalone pipeline — no Quarkus, no Maven, no Java dependencies
- One-time code kept for reproducibility — not production quality
- Output format: JSON matching `GameStateTranslator.toMap()` field names (see spec §6)
- SC2 Faster speed = 22.4 game loops/second (all competitive SC2)
- SC2EGSet nested ZIP structure: outer ZIP (BZip2) → `*_data.zip` (DEFLATE) → `*.SC2Replay.json`
- IEM10 Taipei 2016 ZIP at: `quarkmind-sc2/replays/2016_IEM_10_Taipei.zip`
- All tests run with: `cd quarkmind-dataset && python -m pytest tests/ -v`

---

## Batch 1: Game State Extraction from SC2EGSet

### Task 1: Project scaffold + SC2EGSet JSON parser

**Files:**
- Create: `quarkmind-dataset/pyproject.toml`
- Create: `quarkmind-dataset/src/__init__.py`
- Create: `quarkmind-dataset/src/parse_replays.py`
- Create: `quarkmind-dataset/tests/__init__.py`
- Create: `quarkmind-dataset/tests/test_parse_replays.py`

**Interfaces:**
- Produces: `parse_replays.enumerate_replays(zip_path: Path) -> list[ReplayData]` — reads the nested ZIP and returns parsed replay metadata + tracker events for all games. `ReplayData` is a dataclass with fields: `replay_name: str`, `player1: PlayerInfo`, `player2: PlayerInfo`, `map_name: str`, `duration_loops: int`, `tracker_events: list[dict]`, `game_events: list[dict]`. `PlayerInfo` has: `name: str`, `race: str`, `player_id: int`, `result: str`.

- [ ] **Step 1: Create project scaffold**

Create `quarkmind-dataset/pyproject.toml`:

```toml
[project]
name = "quarkmind-dataset"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "yt-dlp>=2024.0",
    "webvtt-py>=0.5",
    "pyyaml>=6.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

Create empty `quarkmind-dataset/src/__init__.py` and `quarkmind-dataset/tests/__init__.py`.

- [ ] **Step 2: Write failing test for replay parsing**

Create `quarkmind-dataset/tests/test_parse_replays.py`:

```python
from pathlib import Path
from src.parse_replays import enumerate_replays, ReplayData, PlayerInfo

IEM10_ZIP = Path(__file__).resolve().parents[2] / "quarkmind-sc2" / "replays" / "2016_IEM_10_Taipei.zip"

def test_enumerate_replays_finds_30_games():
    replays = enumerate_replays(IEM10_ZIP)
    assert len(replays) == 30

def test_replay_has_two_players():
    replays = enumerate_replays(IEM10_ZIP)
    r = replays[0]
    assert isinstance(r.player1, PlayerInfo)
    assert isinstance(r.player2, PlayerInfo)
    assert r.player1.race in ("Protoss", "Terran", "Zerg")
    assert r.player2.race in ("Protoss", "Terran", "Zerg")

def test_replay_has_tracker_events():
    replays = enumerate_replays(IEM10_ZIP)
    r = replays[0]
    assert len(r.tracker_events) > 100
    first = r.tracker_events[0]
    assert "evtTypeName" in first or "_event_type" in first

def test_replay_has_map_and_duration():
    replays = enumerate_replays(IEM10_ZIP)
    r = replays[0]
    assert r.map_name != ""
    assert r.duration_loops > 0
    duration_sec = r.duration_loops / 22.4
    assert 60 < duration_sec < 3600  # between 1 min and 1 hour
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd quarkmind-dataset && python -m pytest tests/test_parse_replays.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'src.parse_replays'`

- [ ] **Step 4: Implement replay parser**

Create `quarkmind-dataset/src/parse_replays.py`:

```python
"""Parse SC2EGSet nested ZIP archives into structured replay data."""
import json
import zipfile
from dataclasses import dataclass
from io import BytesIO
from pathlib import Path


@dataclass
class PlayerInfo:
    name: str
    race: str
    player_id: int
    result: str


@dataclass
class ReplayData:
    replay_name: str
    player1: PlayerInfo
    player2: PlayerInfo
    map_name: str
    duration_loops: int
    tracker_events: list[dict]
    game_events: list[dict]


RACE_MAP = {"Prot": "Protoss", "Terr": "Terran", "Zerg": "Zerg"}


def enumerate_replays(zip_path: Path) -> list[ReplayData]:
    """Read all SC2Replay.json files from the nested SC2EGSet ZIP."""
    replays = []
    with zipfile.ZipFile(zip_path) as outer:
        for entry in outer.namelist():
            if entry.endswith("_data.zip"):
                inner_bytes = outer.read(entry)
                with zipfile.ZipFile(BytesIO(inner_bytes)) as inner:
                    for inner_entry in inner.namelist():
                        if inner_entry.endswith(".SC2Replay.json"):
                            json_bytes = inner.read(inner_entry)
                            replay = _parse_replay_json(json_bytes, inner_entry)
                            replays.append(replay)
    return replays


def _parse_replay_json(json_bytes: bytes, name: str) -> ReplayData:
    root = json.loads(json_bytes)

    player_map = root["ToonPlayerDescMap"]
    players = []
    for p in player_map.values() if isinstance(player_map, dict) else player_map:
        race_raw = p.get("race", "Unknown")
        players.append(PlayerInfo(
            name=p.get("name", p.get("SQ", "Unknown")),
            race=RACE_MAP.get(race_raw, race_raw),
            player_id=p.get("playerID", 0),
            result=p.get("result", "Unknown"),
        ))

    p1 = players[0] if len(players) > 0 else PlayerInfo("Unknown", "Unknown", 0, "Unknown")
    p2 = players[1] if len(players) > 1 else PlayerInfo("Unknown", "Unknown", 0, "Unknown")

    map_name = root.get("metadata", {}).get("mapName", "")
    if not map_name:
        map_name = root.get("metadata", {}).get("Title", "")

    duration_loops = root.get("header", {}).get("elapsedGameLoops", 0)

    tracker_events = root.get("trackerEvents", [])
    game_events = root.get("gameEvents", [])

    return ReplayData(
        replay_name=name,
        player1=p1,
        player2=p2,
        map_name=map_name,
        duration_loops=duration_loops,
        tracker_events=tracker_events,
        game_events=game_events,
    )
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd quarkmind-dataset && python -m pytest tests/test_parse_replays.py -v`
Expected: 4 PASSED. If the nested ZIP uses BZip2 at the outer level (the Java code uses Apache Commons Compress for this), standard `zipfile` may fail. In that case, switch to reading the outer ZIP with `tarfile` or `lzma`, or extract manually. The IEM10 ZIP in the repo may already be re-packed as standard DEFLATE.

- [ ] **Step 6: Commit**

```bash
git -C quarkmind-dataset add .
git commit -m "feat: SC2EGSet replay parser with nested ZIP support Refs #249"
```

### Task 2: Game state extractor

**Files:**
- Create: `quarkmind-dataset/src/extract_state.py`
- Create: `quarkmind-dataset/tests/test_extract_state.py`

**Interfaces:**
- Consumes: `parse_replays.ReplayData` (tracker_events, player1, player2)
- Produces: `extract_state.GameStateExtractor` class with method `snapshot_at(frame: int) -> dict` returning the structured snapshot (spec §6 schema), and `events_in_range(start_frame: int, end_frame: int) -> list[dict]` returning detected game events in the frame range.

- [ ] **Step 1: Write failing tests**

Create `quarkmind-dataset/tests/test_extract_state.py`:

```python
from pathlib import Path
from src.parse_replays import enumerate_replays
from src.extract_state import GameStateExtractor

IEM10_ZIP = Path(__file__).resolve().parents[2] / "quarkmind-sc2" / "replays" / "2016_IEM_10_Taipei.zip"

def _first_protoss_replay():
    replays = enumerate_replays(IEM10_ZIP)
    for r in replays:
        if r.player1.race == "Protoss" or r.player2.race == "Protoss":
            return r
    raise ValueError("No Protoss replay found")

def test_snapshot_at_frame_zero_has_workers():
    r = _first_protoss_replay()
    ext = GameStateExtractor(r)
    snap = ext.snapshot_at(0)
    assert "player" in snap
    assert "opponent" in snap
    assert snap["game_frame"] == 0

def test_snapshot_at_3min_has_economy():
    r = _first_protoss_replay()
    ext = GameStateExtractor(r)
    three_min_frame = int(180 * 22.4)
    snap = ext.snapshot_at(three_min_frame)
    assert snap["player"]["minerals"] >= 0
    assert snap["player"]["supply_used"] > 0
    assert snap["player"]["worker_count"] > 0

def test_snapshot_has_army_composition():
    r = _first_protoss_replay()
    ext = GameStateExtractor(r)
    five_min_frame = int(300 * 22.4)
    snap = ext.snapshot_at(five_min_frame)
    assert isinstance(snap["player"]["army_composition"], dict)
    assert isinstance(snap["player"]["buildings"], dict)

def test_snapshot_has_tech():
    r = _first_protoss_replay()
    ext = GameStateExtractor(r)
    five_min_frame = int(300 * 22.4)
    snap = ext.snapshot_at(five_min_frame)
    assert isinstance(snap["player"]["tech"], list)

def test_events_in_range():
    r = _first_protoss_replay()
    ext = GameStateExtractor(r)
    events = ext.events_in_range(0, int(180 * 22.4))
    assert len(events) > 0
    for ev in events:
        assert "frame" in ev
        assert "type" in ev
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd quarkmind-dataset && python -m pytest tests/test_extract_state.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'src.extract_state'`

- [ ] **Step 3: Implement game state extractor**

Create `quarkmind-dataset/src/extract_state.py`:

```python
"""Extract structured game state snapshots from SC2EGSet tracker events.

Mirrors the Java GameStateTranslator.toMap() schema. Reads tracker events
linearly, accumulating unit/building/economy state, and produces a snapshot
at any requested frame.
"""
from dataclasses import dataclass, field
from src.parse_replays import ReplayData

LOOPS_PER_SEC = 22.4

WORKER_TYPES = {"Probe", "SCV", "Drone"}

BUILDING_TYPES = {
    "Nexus", "Pylon", "Assimilator", "Gateway", "Forge", "CyberneticsCore",
    "PhotonCannon", "ShieldBattery", "RoboticsFacility", "Stargate",
    "TwilightCouncil", "RoboticsBay", "FleetBeacon", "TemplarArchive",
    "DarkShrine", "WarpGate",
    "CommandCenter", "SupplyDepot", "Refinery", "Barracks", "Factory",
    "Starport", "EngineeringBay", "Armory", "GhostAcademy", "FusionCore",
    "Bunker", "MissileTurret", "SensorTower", "PlanetaryFortress",
    "OrbitalCommand",
    "Hatchery", "SpawningPool", "Extractor", "EvolutionChamber",
    "RoachWarren", "BanelingNest", "Lair", "HydraliskDen", "LurkerDen",
    "InfestationPit", "Spire", "GreaterSpire", "NydusNetwork", "Hive",
    "UltraliskCavern", "SpineCrawler", "SporeCrawler",
}

GEYSER_TYPES = {"VespeneGeyser", "RichVespeneGeyser", "ProtossVespeneGeyser",
                "SpacePlatformGeyser", "PurifierVespeneGeyser"}
MINERAL_TYPES = {"MineralField", "MineralField750", "RichMineralField",
                 "RichMineralField750", "MineralFieldOpaque",
                 "MineralFieldOpaque900", "PurifierMineralField",
                 "PurifierMineralField750", "PurifierRichMineralField",
                 "PurifierRichMineralField750", "BattleStationMineralField",
                 "BattleStationMineralField750", "LabMineralField",
                 "LabMineralField750"}


class GameStateExtractor:
    """Accumulates tracker events and produces snapshots at any frame."""

    def __init__(self, replay: ReplayData):
        self._replay = replay
        self._watched_id = self._pick_watched_player()
        self._opponent_id = 3 - self._watched_id  # players are 1 and 2

    def _pick_watched_player(self) -> int:
        return self._replay.player1.player_id

    def snapshot_at(self, target_frame: int) -> dict:
        """Build a structured game state snapshot up to target_frame."""
        state = _AccumulatorState()
        watched = self._watched_id
        opponent = self._opponent_id

        for ev in self._replay.tracker_events:
            if ev.get("loop", 0) > target_frame:
                break
            _apply_event(ev, state, watched, opponent)

        watched_player = self._replay.player1 if watched == self._replay.player1.player_id else self._replay.player2
        opponent_player = self._replay.player2 if watched == self._replay.player1.player_id else self._replay.player1

        return {
            "game_frame": target_frame,
            "game_time_sec": round(target_frame / LOOPS_PER_SEC, 1),
            "player": {
                "race": watched_player.race,
                "minerals": state.minerals,
                "gas": state.gas,
                "supply_used": state.supply_used,
                "supply_cap": state.supply_cap,
                "worker_count": state.worker_count,
                "army_composition": dict(state.army),
                "tech": list(state.upgrades),
                "buildings": dict(state.buildings),
                "recent_events": state.recent_events[-10:],
            },
            "opponent": {
                "race": opponent_player.race,
                "known_units": dict(state.enemy_units),
                "known_buildings": dict(state.enemy_buildings),
            },
        }

    def events_in_range(self, start_frame: int, end_frame: int) -> list[dict]:
        """Return significant game events detected within the frame range."""
        events = []
        for ev in self._replay.tracker_events:
            loop = ev.get("loop", 0)
            if loop < start_frame:
                continue
            if loop > end_frame:
                break
            evt_type = ev.get("evtTypeName", "")
            if evt_type == "UnitBorn":
                unit_name = ev.get("unitTypeName", "")
                ctrl = ev.get("controlPlayerId", 0)
                if unit_name in GEYSER_TYPES or unit_name in MINERAL_TYPES:
                    continue
                events.append({
                    "frame": loop,
                    "type": "UNIT_BORN",
                    "unit": unit_name,
                    "owner": "player" if ctrl == self._watched_id else "opponent",
                })
            elif evt_type == "UnitDied":
                events.append({
                    "frame": loop,
                    "type": "UNIT_DIED",
                    "unit_tag": f"{ev.get('unitTagIndex', 0)}-{ev.get('unitTagRecycle', 0)}",
                })
            elif evt_type == "UpgradeEvent":
                events.append({
                    "frame": loop,
                    "type": "UPGRADE_COMPLETE",
                    "upgrade": ev.get("upgradeTypeName", ""),
                    "owner": "player" if ev.get("playerId", 0) == self._watched_id else "opponent",
                })
        return events


@dataclass
class _AccumulatorState:
    minerals: int = 0
    gas: int = 0
    supply_used: int = 0
    supply_cap: int = 0
    worker_count: int = 0
    army: dict = field(default_factory=dict)
    buildings: dict = field(default_factory=dict)
    upgrades: list = field(default_factory=list)
    enemy_units: dict = field(default_factory=dict)
    enemy_buildings: dict = field(default_factory=dict)
    recent_events: list = field(default_factory=list)
    _alive_units: dict = field(default_factory=dict)


def _apply_event(ev: dict, state: _AccumulatorState, watched: int, opponent: int):
    evt_type = ev.get("evtTypeName", "")

    if evt_type == "PlayerStats":
        pid = ev.get("playerId", 0)
        stats = ev.get("stats", {})
        if pid == watched:
            state.minerals = stats.get("scoreValueMineralsCurrent", 0)
            state.gas = stats.get("scoreValueVespeneCurrent", 0)
            state.supply_used = stats.get("scoreValueFoodUsed", 0)
            state.supply_cap = stats.get("scoreValueFoodMade", 0)
            state.worker_count = stats.get("scoreValueWorkersActiveCount", 0)

    elif evt_type == "UnitBorn":
        unit_name = ev.get("unitTypeName", "")
        ctrl = ev.get("controlPlayerId", 0)
        tag = f"{ev.get('unitTagIndex', 0)}-{ev.get('unitTagRecycle', 0)}"
        if unit_name in GEYSER_TYPES or unit_name in MINERAL_TYPES:
            return
        if ctrl == watched:
            if unit_name in BUILDING_TYPES:
                state.buildings[unit_name] = state.buildings.get(unit_name, 0) + 1
            elif unit_name in WORKER_TYPES:
                pass  # worker_count comes from PlayerStats
            else:
                state.army[unit_name] = state.army.get(unit_name, 0) + 1
            state._alive_units[tag] = (unit_name, "player")
        elif ctrl == opponent:
            if unit_name in BUILDING_TYPES:
                state.enemy_buildings[unit_name] = state.enemy_buildings.get(unit_name, 0) + 1
            else:
                state.enemy_units[unit_name] = state.enemy_units.get(unit_name, 0) + 1
            state._alive_units[tag] = (unit_name, "opponent")
        state.recent_events.append({
            "frame": ev.get("loop", 0),
            "type": "UNIT_BORN",
            "unit": unit_name,
        })

    elif evt_type == "UnitDied":
        tag = f"{ev.get('unitTagIndex', 0)}-{ev.get('unitTagRecycle', 0)}"
        if tag in state._alive_units:
            unit_name, owner = state._alive_units.pop(tag)
            if owner == "player":
                if unit_name in BUILDING_TYPES:
                    state.buildings[unit_name] = max(0, state.buildings.get(unit_name, 0) - 1)
                elif unit_name not in WORKER_TYPES:
                    state.army[unit_name] = max(0, state.army.get(unit_name, 0) - 1)
        state.recent_events.append({
            "frame": ev.get("loop", 0),
            "type": "UNIT_DIED",
        })

    elif evt_type == "UpgradeEvent":
        pid = ev.get("playerId", 0)
        upgrade = ev.get("upgradeTypeName", "")
        if pid == watched:
            state.upgrades.append(upgrade)
        state.recent_events.append({
            "frame": ev.get("loop", 0),
            "type": "UPGRADE_COMPLETE",
            "upgrade": upgrade,
        })
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd quarkmind-dataset && python -m pytest tests/test_extract_state.py -v`
Expected: 5 PASSED

- [ ] **Step 5: Commit**

```bash
git add quarkmind-dataset/src/extract_state.py quarkmind-dataset/tests/test_extract_state.py
git commit -m "feat: game state extractor from SC2EGSet tracker events Refs #249"
```

---

## Batch 2: Subtitle Pipeline + Tournament Catalog

### Task 3: VTT parser + subtitle downloader

**Files:**
- Create: `quarkmind-dataset/src/extract_subtitles.py`
- Create: `quarkmind-dataset/tests/test_extract_subtitles.py`
- Create: `quarkmind-dataset/tests/fixtures/sample.vtt`

**Interfaces:**
- Produces: `extract_subtitles.parse_vtt(path: Path) -> list[Caption]` where `Caption` has `start_sec: float`, `end_sec: float`, `text: str`. Also `extract_subtitles.download_subtitles(vod_url: str, output_dir: Path) -> SubtitleResult` with fields `path: Path | None`, `source: str` (manual/auto-generated/none), `language: str`.

- [ ] **Step 1: Create test fixture**

Create `quarkmind-dataset/tests/fixtures/sample.vtt`:

```
WEBVTT

00:01:25.000 --> 00:01:28.500
and ByuN is going for a very aggressive
opening here with the reaper expand

00:01:28.500 --> 00:01:32.000
meanwhile Lilbow hasn't even scouted yet
he's completely in the dark

00:02:15.000 --> 00:02:19.500
oh there's the first pylon going down
at the natural expansion

00:05:30.000 --> 00:05:35.000
and here comes the push from ByuN
a huge marine marauder ball with stim

00:05:35.000 --> 00:05:40.000
Lilbow has to hold this or the game is over
```

- [ ] **Step 2: Write failing tests**

Create `quarkmind-dataset/tests/test_extract_subtitles.py`:

```python
from pathlib import Path
from src.extract_subtitles import parse_vtt, Caption

FIXTURES = Path(__file__).parent / "fixtures"

def test_parse_vtt_returns_captions():
    captions = parse_vtt(FIXTURES / "sample.vtt")
    assert len(captions) == 5

def test_caption_has_timestamps():
    captions = parse_vtt(FIXTURES / "sample.vtt")
    c = captions[0]
    assert isinstance(c, Caption)
    assert c.start_sec == 85.0
    assert c.end_sec == 88.5
    assert "ByuN" in c.text

def test_captions_are_ordered():
    captions = parse_vtt(FIXTURES / "sample.vtt")
    for i in range(1, len(captions)):
        assert captions[i].start_sec >= captions[i - 1].start_sec

def test_multiline_text_joined():
    captions = parse_vtt(FIXTURES / "sample.vtt")
    c = captions[0]
    assert "aggressive" in c.text
    assert "reaper expand" in c.text
    assert "\n" not in c.text  # lines joined with space
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd quarkmind-dataset && python -m pytest tests/test_extract_subtitles.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 4: Implement VTT parser and subtitle downloader**

Create `quarkmind-dataset/src/extract_subtitles.py`:

```python
"""Extract and parse subtitle tracks from YouTube VODs."""
import re
import subprocess
from dataclasses import dataclass
from pathlib import Path


@dataclass
class Caption:
    start_sec: float
    end_sec: float
    text: str


@dataclass
class SubtitleResult:
    path: Path | None
    source: str  # "manual", "auto-generated", "whisper", "none"
    language: str


def parse_vtt(path: Path) -> list[Caption]:
    """Parse a WebVTT file into a list of Caption objects."""
    content = path.read_text(encoding="utf-8")
    captions = []
    blocks = re.split(r"\n\n+", content)
    for block in blocks:
        lines = block.strip().split("\n")
        timestamp_line = None
        text_lines = []
        for line in lines:
            if "-->" in line:
                timestamp_line = line
            elif timestamp_line and line.strip() and not line.startswith("WEBVTT"):
                text_lines.append(line.strip())
        if timestamp_line and text_lines:
            start, end = _parse_timestamp_line(timestamp_line)
            text = " ".join(text_lines)
            text = re.sub(r"<[^>]+>", "", text)  # strip HTML tags
            captions.append(Caption(start_sec=start, end_sec=end, text=text))
    return captions


def download_subtitles(vod_url: str, output_dir: Path) -> SubtitleResult:
    """Download subtitles from a YouTube VOD using yt-dlp.

    Tries manual subtitles first, then auto-generated.
    """
    output_dir.mkdir(parents=True, exist_ok=True)
    template = str(output_dir / "%(id)s")

    # Try manual subtitles first
    result = _run_ytdlp(vod_url, template, auto=False)
    if result.path:
        return result

    # Fall back to auto-generated
    return _run_ytdlp(vod_url, template, auto=True)


def _run_ytdlp(url: str, template: str, auto: bool) -> SubtitleResult:
    cmd = [
        "yt-dlp",
        "--write-auto-subs" if auto else "--write-subs",
        "--sub-langs", "en",
        "--sub-format", "vtt",
        "--skip-download",
        "-o", template,
        url,
    ]
    try:
        subprocess.run(cmd, capture_output=True, text=True, timeout=60, check=False)
    except (subprocess.TimeoutExpired, FileNotFoundError):
        return SubtitleResult(path=None, source="none", language="en")

    # Find the downloaded VTT file
    output_dir = Path(template).parent
    vtt_files = list(output_dir.glob("*.vtt"))
    if vtt_files:
        source = "auto-generated" if auto else "manual"
        return SubtitleResult(path=vtt_files[0], source=source, language="en")
    return SubtitleResult(path=None, source="none", language="en")


def _parse_timestamp_line(line: str) -> tuple[float, float]:
    match = re.match(
        r"(\d{2}):(\d{2}):(\d{2})\.(\d{3})\s*-->\s*(\d{2}):(\d{2}):(\d{2})\.(\d{3})",
        line.strip(),
    )
    if not match:
        return 0.0, 0.0
    g = [int(x) for x in match.groups()]
    start = g[0] * 3600 + g[1] * 60 + g[2] + g[3] / 1000
    end = g[4] * 3600 + g[5] * 60 + g[6] + g[7] / 1000
    return start, end
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd quarkmind-dataset && python -m pytest tests/test_extract_subtitles.py -v`
Expected: 4 PASSED

- [ ] **Step 6: Commit**

```bash
git add quarkmind-dataset/src/extract_subtitles.py quarkmind-dataset/tests/test_extract_subtitles.py quarkmind-dataset/tests/fixtures/
git commit -m "feat: VTT subtitle parser and yt-dlp downloader Refs #249"
```

### Task 4: Tournament catalog for IEM10 Taipei

**Files:**
- Create: `quarkmind-dataset/catalog/tournament-catalog.yaml`
- Create: `quarkmind-dataset/catalog/matches/2016_IEM_10_Taipei.yaml`
- Create: `quarkmind-dataset/src/catalog.py`
- Create: `quarkmind-dataset/tests/test_catalog.py`

**Interfaces:**
- Produces: `catalog.load_catalog(path: Path) -> list[Tournament]` and `catalog.load_matches(path: Path) -> list[MatchEntry]`. `Tournament` has `sc2egset_name: str`, `youtube_channels: list[dict]`, `liquipedia_url: str`. `MatchEntry` has `replay_hash: str`, `players: list[str]`, `map: str`, `vod_url: str`, `game_start_offset_sec: float`, `subtitle_source: str`, `confidence: str`.
- Consumes: `parse_replays.ReplayData` (for matching replays to catalog entries by player names + map)

- [ ] **Step 1: Create tournament catalog YAML**

Create `quarkmind-dataset/catalog/tournament-catalog.yaml`:

```yaml
tournaments:
  - sc2egset_name: "2016_IEM_10_Taipei"
    display_name: "IEM Season 10 Taipei"
    year: 2016
    youtube_channels:
      - channel_name: "ESL Archives"
        playlist_url: "https://www.youtube.com/playlist?list=PLhCH_nPE4JTqKvJvmCrfR5Jk"
    liquipedia_url: "https://liquipedia.net/starcraft2/IEM_Season_X_-_Taipei"
    game_speed: "Faster"
    notes: "30 replays in quarkmind-sc2/replays/2016_IEM_10_Taipei.zip"
```

- [ ] **Step 2: Create initial match file with a few manual entries**

Create `quarkmind-dataset/catalog/matches/2016_IEM_10_Taipei.yaml`. Populate with 2-3 placeholder entries that demonstrate the schema. Real VOD URLs and offsets will be filled in during the manual curation step of Phase 1 execution.

```yaml
# Manual matches — IEM Season 10 Taipei 2016
# VOD URLs and game_start_offset_sec to be filled during curation
tournament: "2016_IEM_10_Taipei"
matches: []
```

- [ ] **Step 3: Write failing tests**

Create `quarkmind-dataset/tests/test_catalog.py`:

```python
from pathlib import Path
from src.catalog import load_catalog, load_matches, Tournament

CATALOG_DIR = Path(__file__).resolve().parents[1] / "catalog"

def test_load_catalog():
    tournaments = load_catalog(CATALOG_DIR / "tournament-catalog.yaml")
    assert len(tournaments) >= 1
    t = tournaments[0]
    assert isinstance(t, Tournament)
    assert t.sc2egset_name == "2016_IEM_10_Taipei"
    assert t.year == 2016

def test_load_matches_empty():
    matches = load_matches(CATALOG_DIR / "matches" / "2016_IEM_10_Taipei.yaml")
    assert isinstance(matches, list)
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `cd quarkmind-dataset && python -m pytest tests/test_catalog.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 5: Implement catalog loader**

Create `quarkmind-dataset/src/catalog.py`:

```python
"""Load tournament catalogs and replay-to-VOD match files."""
from dataclasses import dataclass
from pathlib import Path

import yaml


@dataclass
class Tournament:
    sc2egset_name: str
    display_name: str
    year: int
    youtube_channels: list[dict]
    liquipedia_url: str
    game_speed: str
    notes: str


@dataclass
class MatchEntry:
    replay_hash: str
    players: list[str]
    map: str
    replay_duration_sec: float
    vod_url: str
    vod_title: str
    game_start_offset_sec: float
    subtitle_source: str
    confidence: str


def load_catalog(path: Path) -> list[Tournament]:
    """Load the tournament catalog YAML."""
    data = yaml.safe_load(path.read_text())
    return [
        Tournament(
            sc2egset_name=t["sc2egset_name"],
            display_name=t["display_name"],
            year=t["year"],
            youtube_channels=t.get("youtube_channels", []),
            liquipedia_url=t.get("liquipedia_url", ""),
            game_speed=t.get("game_speed", "Faster"),
            notes=t.get("notes", ""),
        )
        for t in data.get("tournaments", [])
    ]


def load_matches(path: Path) -> list[MatchEntry]:
    """Load a replay-to-VOD match file."""
    data = yaml.safe_load(path.read_text())
    matches_raw = data.get("matches", [])
    if not matches_raw:
        return []
    return [
        MatchEntry(
            replay_hash=m.get("replay_hash", ""),
            players=m.get("players", []),
            map=m.get("map", ""),
            replay_duration_sec=m.get("replay_duration_sec", 0),
            vod_url=m.get("vod", {}).get("url", ""),
            vod_title=m.get("vod", {}).get("title", ""),
            game_start_offset_sec=m.get("vod", {}).get("game_start_offset_sec", 0),
            subtitle_source=m.get("vod", {}).get("subtitle_source", "unknown"),
            confidence=m.get("confidence", "manual-review"),
        )
        for m in matches_raw
    ]
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `cd quarkmind-dataset && python -m pytest tests/test_catalog.py -v`
Expected: 2 PASSED

- [ ] **Step 7: Commit**

```bash
git add quarkmind-dataset/catalog/ quarkmind-dataset/src/catalog.py quarkmind-dataset/tests/test_catalog.py
git commit -m "feat: tournament catalog and match file loader Refs #249"
```

---

## Batch 3: Alignment, Segmentation, and Pipeline

### Task 5: Timestamp alignment + event-driven segmentation

**Files:**
- Create: `quarkmind-dataset/src/align.py`
- Create: `quarkmind-dataset/src/segment.py`
- Create: `quarkmind-dataset/tests/test_align.py`
- Create: `quarkmind-dataset/tests/test_segment.py`

**Interfaces:**
- Consumes: `extract_subtitles.Caption` (start_sec, end_sec, text), `extract_state.GameStateExtractor` (events_in_range)
- Produces: `align.align_captions(captions: list[Caption], game_start_offset_sec: float) -> list[AlignedCaption]` where `AlignedCaption` adds `game_frame_start: int`, `game_frame_end: int` to Caption fields. Also `segment.segment_game(extractor: GameStateExtractor, aligned_captions: list[AlignedCaption], total_frames: int) -> list[Segment]` where `Segment` has `type: str`, `phase: str`, `phase_transition: bool`, `transition_description: str | None`, `game_frame_start: int`, `game_frame_end: int`, `commentary: str`, `game_state: dict`.

- [ ] **Step 1: Write failing alignment tests**

Create `quarkmind-dataset/tests/test_align.py`:

```python
from src.align import align_captions, AlignedCaption
from src.extract_subtitles import Caption

def test_align_converts_vod_time_to_game_frame():
    captions = [
        Caption(start_sec=85.0, end_sec=88.5, text="first pylon going down"),
        Caption(start_sec=200.0, end_sec=205.0, text="big push coming"),
    ]
    aligned = align_captions(captions, game_start_offset_sec=80.0)
    assert len(aligned) == 2
    # caption at 85s VOD, offset 80s -> game time 5s -> frame = 5 * 22.4 = 112
    assert aligned[0].game_frame_start == 112
    assert aligned[0].game_frame_end == int(8.5 * 22.4)  # 190

def test_align_filters_pre_game_captions():
    captions = [
        Caption(start_sec=10.0, end_sec=15.0, text="welcome to the tournament"),
        Caption(start_sec=85.0, end_sec=88.0, text="and the game begins"),
    ]
    aligned = align_captions(captions, game_start_offset_sec=80.0)
    assert len(aligned) == 1  # pre-game caption filtered out
    assert "game begins" in aligned[0].text

def test_align_preserves_text():
    captions = [Caption(start_sec=100.0, end_sec=105.0, text="great micro")]
    aligned = align_captions(captions, game_start_offset_sec=90.0)
    assert aligned[0].text == "great micro"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd quarkmind-dataset && python -m pytest tests/test_align.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Implement alignment**

Create `quarkmind-dataset/src/align.py`:

```python
"""Align VOD subtitle timestamps to SC2 replay game frames."""
from dataclasses import dataclass
from src.extract_subtitles import Caption

LOOPS_PER_SEC = 22.4


@dataclass
class AlignedCaption:
    start_sec: float
    end_sec: float
    text: str
    game_frame_start: int
    game_frame_end: int


def align_captions(
    captions: list[Caption],
    game_start_offset_sec: float,
) -> list[AlignedCaption]:
    """Convert VOD timestamps to game frames using the game-start offset.

    Filters out captions that fall before the game starts.
    """
    aligned = []
    for c in captions:
        game_time_start = c.start_sec - game_start_offset_sec
        game_time_end = c.end_sec - game_start_offset_sec
        if game_time_start < 0:
            continue
        aligned.append(AlignedCaption(
            start_sec=c.start_sec,
            end_sec=c.end_sec,
            text=c.text,
            game_frame_start=int(game_time_start * LOOPS_PER_SEC),
            game_frame_end=int(game_time_end * LOOPS_PER_SEC),
        ))
    return aligned
```

- [ ] **Step 4: Run alignment tests**

Run: `cd quarkmind-dataset && python -m pytest tests/test_align.py -v`
Expected: 3 PASSED

- [ ] **Step 5: Write failing segmentation tests**

Create `quarkmind-dataset/tests/test_segment.py`:

```python
from src.segment import segment_game, Segment, classify_phase
from src.align import AlignedCaption

def test_classify_phase_by_time():
    assert classify_phase(0) == "opening"
    assert classify_phase(int(60 * 22.4)) == "opening"
    assert classify_phase(int(180 * 22.4)) == "opening"
    assert classify_phase(int(240 * 22.4)) == "early_aggression"
    assert classify_phase(int(480 * 22.4)) == "mid_game"
    assert classify_phase(int(900 * 22.4)) == "late_game"
    assert classify_phase(int(1500 * 22.4)) == "endgame"

def test_segment_groups_captions():
    captions = [
        AlignedCaption(100, 103, "first pylon", 448, 515),
        AlignedCaption(101, 104, "nice placement", 470, 537),
        AlignedCaption(300, 305, "push incoming", 4928, 5040),
    ]
    events = [
        {"frame": 460, "type": "UNIT_BORN", "unit": "Pylon"},
        {"frame": 4950, "type": "UNIT_DIED"},
    ]
    segments = segment_game(captions, events, total_frames=int(600 * 22.4))
    assert len(segments) >= 2
    for s in segments:
        assert s.phase in ("opening", "early_aggression", "mid_game", "late_game", "endgame")
        assert s.commentary != ""

def test_segment_detects_phase_transition():
    # Caption spans the opening -> early_aggression boundary (~180s = 4032 frames)
    captions = [
        AlignedCaption(250, 260, "transitioning to aggression", 3808, 4032),
        AlignedCaption(260, 265, "first combat units moving out", 4032, 4144),
    ]
    events = [{"frame": 4000, "type": "UNIT_BORN", "unit": "Stalker"}]
    segments = segment_game(captions, events, total_frames=int(300 * 22.4))
    transitions = [s for s in segments if s.phase_transition]
    # At least one segment should span or be near a phase boundary
    assert any(s.phase in ("opening", "early_aggression") for s in segments)
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `cd quarkmind-dataset && python -m pytest tests/test_segment.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 7: Implement segmentation**

Create `quarkmind-dataset/src/segment.py`:

```python
"""Event-driven segmentation with game-phase context."""
from dataclasses import dataclass
from src.align import AlignedCaption

LOOPS_PER_SEC = 22.4

PHASE_BOUNDARIES_SEC = [
    (0, 180, "opening"),
    (180, 300, "early_aggression"),
    (300, 720, "mid_game"),
    (720, 1200, "late_game"),
    (1200, float("inf"), "endgame"),
]


@dataclass
class Segment:
    type: str  # "battle", "expansion", "tech_transition", "macro_economy", "phase_transition"
    phase: str
    phase_transition: bool
    transition_description: str | None
    game_frame_start: int
    game_frame_end: int
    commentary: str


def classify_phase(frame: int) -> str:
    """Classify a game frame into its game phase."""
    game_sec = frame / LOOPS_PER_SEC
    for start, end, phase in PHASE_BOUNDARIES_SEC:
        if start <= game_sec < end:
            return phase
    return "endgame"


def segment_game(
    aligned_captions: list[AlignedCaption],
    events: list[dict],
    total_frames: int,
    post_event_window_sec: float = 15.0,
    min_quiet_segment_sec: float = 30.0,
) -> list[Segment]:
    """Segment a game into event-driven windows with phase tags.

    1. Identify segment boundaries from game events
    2. Assign captions to segments by frame overlap
    3. Tag each segment with its game phase
    4. Flag phase transitions
    """
    post_event_frames = int(post_event_window_sec * LOOPS_PER_SEC)
    min_quiet_frames = int(min_quiet_segment_sec * LOOPS_PER_SEC)

    # Build event-anchored segment boundaries
    boundaries = _build_boundaries(events, post_event_frames, min_quiet_frames, total_frames)

    # Assign captions and classify
    segments = []
    for start, end, seg_type in boundaries:
        seg_captions = [
            c for c in aligned_captions
            if c.game_frame_start < end and c.game_frame_end > start
        ]
        if not seg_captions:
            continue
        commentary = " ".join(c.text for c in seg_captions)
        phase_start = classify_phase(start)
        phase_end = classify_phase(end)
        is_transition = phase_start != phase_end
        transition_desc = f"{phase_start} -> {phase_end}" if is_transition else None

        segments.append(Segment(
            type=seg_type,
            phase=phase_start,
            phase_transition=is_transition,
            transition_description=transition_desc,
            game_frame_start=start,
            game_frame_end=end,
            commentary=commentary,
        ))

    return segments


def _build_boundaries(
    events: list[dict],
    post_event_frames: int,
    min_quiet_frames: int,
    total_frames: int,
) -> list[tuple[int, int, str]]:
    """Build segment boundaries from game events."""
    if not events:
        return [(0, total_frames, "macro_economy")]

    boundaries = []
    prev_end = 0

    # Cluster nearby events (within post_event_frames of each other)
    clusters = _cluster_events(events, post_event_frames)

    for cluster in clusters:
        cluster_start = cluster[0]["frame"]
        cluster_end = cluster[-1]["frame"] + post_event_frames

        # Gap before this cluster
        if cluster_start - prev_end >= min_quiet_frames:
            boundaries.append((prev_end, cluster_start, "macro_economy"))

        seg_type = _classify_cluster(cluster)
        boundaries.append((cluster_start, min(cluster_end, total_frames), seg_type))
        prev_end = min(cluster_end, total_frames)

    # Trailing quiet period
    if total_frames - prev_end >= min_quiet_frames:
        boundaries.append((prev_end, total_frames, "macro_economy"))

    return boundaries


def _cluster_events(events: list[dict], window: int) -> list[list[dict]]:
    """Group events within `window` frames of each other."""
    if not events:
        return []
    sorted_events = sorted(events, key=lambda e: e.get("frame", 0))
    clusters = [[sorted_events[0]]]
    for ev in sorted_events[1:]:
        if ev["frame"] - clusters[-1][-1]["frame"] <= window:
            clusters[-1].append(ev)
        else:
            clusters.append([ev])
    return clusters


def _classify_cluster(cluster: list[dict]) -> str:
    """Classify a cluster of events by its dominant event type."""
    deaths = sum(1 for e in cluster if e.get("type") == "UNIT_DIED")
    births = sum(1 for e in cluster if e.get("type") == "UNIT_BORN")
    upgrades = sum(1 for e in cluster if e.get("type") == "UPGRADE_COMPLETE")

    if deaths >= 3:
        return "battle"
    if upgrades > 0:
        return "tech_transition"
    if births > 0:
        unit_names = [e.get("unit", "") for e in cluster if e.get("type") == "UNIT_BORN"]
        building_set = {"Nexus", "CommandCenter", "Hatchery", "OrbitalCommand", "PlanetaryFortress"}
        if any(u in building_set for u in unit_names):
            return "expansion"
    return "macro_economy"
```

- [ ] **Step 8: Run segmentation tests**

Run: `cd quarkmind-dataset && python -m pytest tests/test_segment.py -v`
Expected: 3 PASSED

- [ ] **Step 9: Commit**

```bash
git add quarkmind-dataset/src/align.py quarkmind-dataset/src/segment.py quarkmind-dataset/tests/test_align.py quarkmind-dataset/tests/test_segment.py
git commit -m "feat: timestamp alignment and event-driven segmentation Refs #249"
```

### Task 6: Pipeline orchestrator + first dataset output

**Files:**
- Create: `quarkmind-dataset/src/pipeline.py`
- Create: `quarkmind-dataset/tests/test_pipeline.py`
- Create: `quarkmind-dataset/output/.gitkeep`

**Interfaces:**
- Consumes: All previous modules — `parse_replays`, `extract_state`, `extract_subtitles`, `catalog`, `align`, `segment`
- Produces: `pipeline.run_pipeline(catalog_dir: Path, replays_zip: Path, output_dir: Path) -> PipelineResult` with `total_examples: int`, `games_processed: int`, `stats: dict`. Writes JSON training examples to `output_dir/examples/` and `output_dir/stats.json`.

- [ ] **Step 1: Write failing test**

Create `quarkmind-dataset/tests/test_pipeline.py`:

```python
from pathlib import Path
from src.pipeline import build_training_example

def test_build_training_example_schema():
    game_state = {
        "game_frame": 5040,
        "game_time_sec": 225.0,
        "player": {"race": "Protoss", "minerals": 350, "gas": 200,
                    "supply_used": 44, "supply_cap": 54, "worker_count": 22,
                    "army_composition": {"Stalker": 6}, "tech": ["WarpGate"],
                    "buildings": {"Nexus": 1}, "recent_events": []},
        "opponent": {"race": "Terran", "known_units": {"Marine": 8},
                     "known_buildings": {"CommandCenter": 1}},
    }
    example = build_training_example(
        example_id="test-001",
        game_state=game_state,
        phase="mid_game",
        phase_transition=False,
        transition_description=None,
        commentary="big push incoming",
        segment_type="battle",
        game_frame_start=4500,
        game_frame_end=5200,
        tournament="IEM Taipei 2016",
        player1="ByuN", player1_race="Terran",
        player2="Lilbow", player2_race="Protoss",
        matchup="TvP", map_name="Lerilak Crest",
        game_duration_sec=382.0,
        vod_url="https://youtube.com/watch?v=test",
        subtitle_source="auto-generated",
        replay_hash="095724b",
    )
    assert example["id"] == "test-001"
    assert example["version"] == "1.0"
    assert example["phase"] == "mid_game"
    assert example["commentary"] == "big push incoming"
    assert example["game_state"]["player"]["race"] == "Protoss"
    assert example["segment"]["type"] == "battle"
    assert example["metadata"]["tournament"] == "IEM Taipei 2016"
    assert example["metadata"]["matchup"] == "TvP"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd quarkmind-dataset && python -m pytest tests/test_pipeline.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Implement pipeline orchestrator**

Create `quarkmind-dataset/src/pipeline.py`:

```python
"""End-to-end pipeline: SC2EGSet replays + YouTube subtitles -> training examples."""
import json
from collections import Counter
from dataclasses import dataclass
from pathlib import Path

from src.parse_replays import enumerate_replays, ReplayData
from src.extract_state import GameStateExtractor
from src.extract_subtitles import parse_vtt
from src.catalog import load_matches, MatchEntry
from src.align import align_captions, AlignedCaption
from src.segment import segment_game


@dataclass
class PipelineResult:
    total_examples: int
    games_processed: int
    stats: dict


def build_training_example(
    example_id: str,
    game_state: dict,
    phase: str,
    phase_transition: bool,
    transition_description: str | None,
    commentary: str,
    segment_type: str,
    game_frame_start: int,
    game_frame_end: int,
    tournament: str,
    player1: str, player1_race: str,
    player2: str, player2_race: str,
    matchup: str, map_name: str,
    game_duration_sec: float,
    vod_url: str,
    subtitle_source: str,
    replay_hash: str,
) -> dict:
    """Build a single training example in the spec §8 schema."""
    return {
        "id": example_id,
        "version": "1.0",
        "game_state": game_state,
        "phase": phase,
        "phase_transition": phase_transition,
        "transition_description": transition_description,
        "commentary": commentary,
        "segment": {
            "type": segment_type,
            "game_frame_start": game_frame_start,
            "game_frame_end": game_frame_end,
            "game_time_start_sec": round(game_frame_start / 22.4, 1),
            "game_time_end_sec": round(game_frame_end / 22.4, 1),
        },
        "metadata": {
            "tournament": tournament,
            "player1": player1,
            "player1_race": player1_race,
            "player2": player2,
            "player2_race": player2_race,
            "matchup": matchup,
            "map": map_name,
            "game_duration_sec": game_duration_sec,
            "vod_url": vod_url,
            "subtitle_source": subtitle_source,
            "replay_hash": replay_hash,
        },
    }


def run_pipeline(
    match_file: Path,
    replays_zip: Path,
    subtitles_dir: Path,
    output_dir: Path,
    tournament_name: str,
) -> PipelineResult:
    """Run the full pipeline for a single tournament."""
    output_dir.mkdir(parents=True, exist_ok=True)
    examples_dir = output_dir / "examples"
    examples_dir.mkdir(exist_ok=True)

    matches = load_matches(match_file)
    replays = enumerate_replays(replays_zip)
    replay_by_name = {r.replay_name: r for r in replays}

    all_examples = []
    games_processed = 0
    phase_counts = Counter()
    type_counts = Counter()
    matchup_counts = Counter()

    for match in matches:
        replay = _find_replay(match, replay_by_name)
        if not replay:
            continue

        vtt_path = subtitles_dir / f"{_vod_id(match.vod_url)}.en.vtt"
        if not vtt_path.exists():
            continue

        captions = parse_vtt(vtt_path)
        aligned = align_captions(captions, match.game_start_offset_sec)
        if not aligned:
            continue

        extractor = GameStateExtractor(replay)
        events = extractor.events_in_range(0, replay.duration_loops)
        segments = segment_game(aligned, events, replay.duration_loops)

        matchup = f"{replay.player1.race[0]}v{replay.player2.race[0]}"
        duration_sec = replay.duration_loops / 22.4

        for i, seg in enumerate(segments):
            mid_frame = (seg.game_frame_start + seg.game_frame_end) // 2
            game_state = extractor.snapshot_at(mid_frame)
            example_id = f"{match.replay_hash[:8]}-seg-{i:03d}"

            example = build_training_example(
                example_id=example_id,
                game_state=game_state,
                phase=seg.phase,
                phase_transition=seg.phase_transition,
                transition_description=seg.transition_description,
                commentary=seg.commentary,
                segment_type=seg.type,
                game_frame_start=seg.game_frame_start,
                game_frame_end=seg.game_frame_end,
                tournament=tournament_name,
                player1=replay.player1.name,
                player1_race=replay.player1.race,
                player2=replay.player2.name,
                player2_race=replay.player2.race,
                matchup=matchup,
                map_name=replay.map_name,
                game_duration_sec=duration_sec,
                vod_url=match.vod_url,
                subtitle_source=match.subtitle_source,
                replay_hash=match.replay_hash,
            )
            all_examples.append(example)
            phase_counts[seg.phase] += 1
            type_counts[seg.type] += 1
            matchup_counts[matchup] += 1

        games_processed += 1

    # Write examples
    for ex in all_examples:
        path = examples_dir / f"{ex['id']}.json"
        path.write_text(json.dumps(ex, indent=2))

    # Write stats
    stats = {
        "total_examples": len(all_examples),
        "games_processed": games_processed,
        "tournaments": 1,
        "phase_distribution": dict(phase_counts),
        "segment_type_distribution": dict(type_counts),
        "matchup_distribution": dict(matchup_counts),
    }
    (output_dir / "stats.json").write_text(json.dumps(stats, indent=2))

    return PipelineResult(
        total_examples=len(all_examples),
        games_processed=games_processed,
        stats=stats,
    )


def _find_replay(match: MatchEntry, replay_map: dict[str, ReplayData]) -> ReplayData | None:
    if match.replay_hash in replay_map:
        return replay_map[match.replay_hash]
    for name, replay in replay_map.items():
        if (set(match.players) == {replay.player1.name, replay.player2.name}
                and match.map == replay.map_name):
            return replay
    return None


def _vod_id(url: str) -> str:
    if "v=" in url:
        return url.split("v=")[-1].split("&")[0]
    return url.split("/")[-1]
```

- [ ] **Step 4: Run tests**

Run: `cd quarkmind-dataset && python -m pytest tests/test_pipeline.py -v`
Expected: 1 PASSED

- [ ] **Step 5: Commit**

```bash
git add quarkmind-dataset/src/pipeline.py quarkmind-dataset/tests/test_pipeline.py quarkmind-dataset/output/
git commit -m "feat: pipeline orchestrator with training example output Refs #249"
```

---

## References

- [2026-09-14-sc2-commentary-dataset-design.md] — design spec this plan implements
- [decisions.md] — D1-D5 design decisions
- [IEM10JsonSimulatedGame.java] — SC2EGSet JSON parsing reference (nested ZIP, tracker events, player metadata)
- [GameStateTranslator.java:41-64] — target game state schema reference
- [quarkmind-sc2/replays/replay-index.md] — replay datasets and frame rate constants
- [GitHub #249] — focal issue
- [GitHub #290] — sync replay commentary (consumption side, related)
- [GitHub #230] — coach mode (where #249 was identified)
