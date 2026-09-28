# Blizzard Ladder Replay Restoration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #317 — Restore Blizzard ladder replay tracker events — 151K replays for ONNX classifier coverage
**Issue group:** #317

**Goal:** Build a Java-native pipeline that extracts classifier features from 151K stripped Blizzard ladder replays, validated against Docker/SC2 headless oracle ground truth.

**Architecture:** Game events in stripped replays contain player commands (train, build, upgrade, move). A new `StrippedReplayFeatureExtractor` applies calibrated SC2Data timings to these commands to reconstruct synthetic tracker events (UnitBorn, UnitInit, UnitDone, PlayerStats, Upgrade). Output is game_json consumed by the existing Python training pipeline. A ~200-replay Docker-restored oracle set provides ground truth for abilLink discovery and validation. Every divergence fix hardens EmulatedGame for all consumers.

**Tech Stack:** Java 21 (quarkmind-sc2), Python 3 (quarkmind-classifier), Scelight replay parser, Podman (Docker-compatible), SC2 headless Linux (via ARM64 emulation)

## Global Constraints

- SC2Data train/build times are calibrated constants — always validate against oracle, never estimate
- Python pipeline string names (BUILDINGS, UNITS, UPGRADES lists in `sc2egset_extractor.py`) are the canonical vocabulary — Java output must match exactly
- Building double-counting: Python pipeline increments building_counts for both UnitInit AND UnitDone (+2 per completed building)
- All new Java code in `quarkmind-sc2` module, package `io.quarkmind.sc2.replay`
- All new tests are plain JUnit (no `@QuarkusTest`)
- Commit after every task with `Refs #317`

---

## Batch 1: Docker Oracle Infrastructure

Safe wrap point: Docker tooling fixed, SC2 headless extracted, oracle sampler ready. Oracle restoration can run in background.

### Task 1: Fix Docker tooling paths and extract SC2 headless

**Files:**
- Modify: `quarkmind-classifier/docker/sc2-restore/run.sh`
- Modify: `quarkmind-classifier/docker/sc2-restore/setup.sh`

**Interfaces:**
- Consumes: `/tmp/SC2.4.9.3.zip` (existing download), Podman container runtime
- Produces: Working `run.sh` that mounts correct directories, SC2 v4.9.3 headless with executable binary at `data/sc2_headless/4.9.3/SC2.4.9.3/StarCraftII/`

- [ ] **Step 1: Update run.sh paths**

Replace all occurrences of `/Users/mdproctor/claude/casehub/neocortex/evaluation/strategy_classifier/data` with the quarkmind-classifier-relative path. Change `DATA_BASE` to:
```bash
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
DATA_BASE="$(cd "$SCRIPT_DIR/../.." && pwd)/data"
```
This makes run.sh location-independent — it derives the data path from its own location in `docker/sc2-restore/`.

- [ ] **Step 2: Update setup.sh paths**

Same path change for `SC2_BASE`:
```bash
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
SC2_BASE="$(cd "$SCRIPT_DIR/../.." && pwd)/data/sc2_headless"
```

- [ ] **Step 3: Complete SC2 v4.9.3 headless extraction**

Extract all remaining files from `/tmp/SC2.4.9.3.zip`:
```bash
unzip -P iagreetotheeula -o /tmp/SC2.4.9.3.zip 'SC2.4.9.3/StarCraftII/Versions/*' 'SC2.4.9.3/StarCraftII/SC2Data/*' 'SC2.4.9.3/StarCraftII/.build.info' 'SC2.4.9.3/StarCraftII/Battle.net/*' 'SC2.4.9.3/StarCraftII/Interfaces/*' -d quarkmind-classifier/data/sc2_headless/4.9.3/
chmod +x quarkmind-classifier/data/sc2_headless/4.9.3/SC2.4.9.3/StarCraftII/Versions/Base75025/SC2_x64
```

- [ ] **Step 4: Verify with single replay restore**

```bash
podman run --rm --platform linux/amd64 \
  -v $PWD/quarkmind-classifier/data/sc2_headless/4.9.3/SC2.4.9.3/StarCraftII:/opt/StarCraftII:ro \
  -v $PWD/quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3/replays:/data/input:ro \
  -v /tmp/sc2-verify:/data/output \
  sc2-restore:latest --input /data/input --output /data/output --workers 1 --limit 1
```
Expected: `1 restored, 0 failed`

- [ ] **Step 5: Commit**

```bash
git add quarkmind-classifier/docker/sc2-restore/run.sh quarkmind-classifier/docker/sc2-restore/setup.sh
git commit -m "fix: update Docker sc2-restore paths from neocortex to quarkmind-classifier Refs #317"
```

### Task 2: Oracle sampler and restoration launcher

**Files:**
- Create: `quarkmind-classifier/scripts/sample_oracle.py`

**Interfaces:**
- Consumes: `data/replay_packs/blizzard_ladder/4.9.3/replays/` (148,640 replays)
- Produces: `data/replay_packs/blizzard_ladder/4.9.3_oracle/input/` (~200 replay symlinks stratified by matchup), `scripts/restore_oracle.sh` launcher

- [ ] **Step 1: Write oracle sampler script**

```python
"""Sample ~200 replays stratified by race matchup for oracle restoration.

Reads gamemetadata.json from each replay (via mpyq) to determine races.
Stratifies: ~33 per matchup (PvT, PvZ, PvP, TvZ, TvT, ZvZ).
Creates symlinks in 4.9.3_oracle/input/ for the Podman restore job.

Usage:
  python3 scripts/sample_oracle.py --count 200
"""
import argparse, json, io, random, sys
from pathlib import Path
from collections import defaultdict
import mpyq

MATCHUPS = ["PvT", "PvZ", "PvP", "TvZ", "TvT", "ZvZ"]
RACE_SHORT = {"Prot": "P", "Terr": "T", "Zerg": "Z"}

def get_matchup(replay_path: Path) -> str | None:
    try:
        archive = mpyq.MPQArchive(str(replay_path))
        meta = json.loads(archive.read_file("replay.gamemetadata.json"))
        players = meta.get("Players", [])
        if len(players) != 2:
            return None
        r1 = RACE_SHORT.get(players[0].get("SelectedRace", ""), "")
        r2 = RACE_SHORT.get(players[1].get("SelectedRace", ""), "")
        if not r1 or not r2:
            return None
        key = r1 + "v" + r2
        mirror = r2 + "v" + r1
        return key if key in MATCHUPS else mirror if mirror in MATCHUPS else None
    except Exception:
        return None

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--count", type=int, default=200)
    parser.add_argument("--seed", type=int, default=42)
    args = parser.parse_args()

    base = Path(__file__).resolve().parent.parent
    replay_dir = base / "data/replay_packs/blizzard_ladder/4.9.3/replays"
    output_dir = base / "data/replay_packs/blizzard_ladder/4.9.3_oracle/input"
    output_dir.mkdir(parents=True, exist_ok=True)

    replays = sorted(replay_dir.glob("*.SC2Replay"))
    rng = random.Random(args.seed)
    rng.shuffle(replays)

    by_matchup = defaultdict(list)
    per_matchup = args.count // len(MATCHUPS)
    scanned = 0

    for rp in replays:
        if all(len(v) >= per_matchup for v in by_matchup.values()) and len(by_matchup) == len(MATCHUPS):
            break
        matchup = get_matchup(rp)
        scanned += 1
        if scanned % 500 == 0:
            counts = {m: len(by_matchup[m]) for m in MATCHUPS}
            print(f"  Scanned {scanned}... {counts}", flush=True)
        if matchup and len(by_matchup[matchup]) < per_matchup:
            by_matchup[matchup].append(rp)
            (output_dir / rp.name).symlink_to(rp)

    total = sum(len(v) for v in by_matchup.values())
    print(f"Sampled {total} replays from {scanned} scanned:")
    for m in MATCHUPS:
        print(f"  {m}: {len(by_matchup[m])}")

if __name__ == "__main__":
    main()
```

- [ ] **Step 2: Run sampler**

```bash
cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 scripts/sample_oracle.py --count 200
```
Expected: ~200 symlinks in `data/replay_packs/blizzard_ladder/4.9.3_oracle/input/`, ~33 per matchup.

- [ ] **Step 3: Launch oracle restoration (background, ~3 hours)**

```bash
mkdir -p data/replay_packs/blizzard_ladder/4.9.3_oracle/restored
podman run --rm --platform linux/amd64 \
  -v $PWD/data/sc2_headless/4.9.3/SC2.4.9.3/StarCraftII:/opt/StarCraftII:ro \
  -v $PWD/data/replay_packs/blizzard_ladder/4.9.3_oracle/input:/data/input:ro \
  -v $PWD/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored:/data/output \
  sc2-restore:latest --input /data/input --output /data/output --workers 2
```
Run in background. Checkpoint-safe — already-restored replays are skipped on restart.

- [ ] **Step 4: Commit sampler**

```bash
git add quarkmind-classifier/scripts/sample_oracle.py
git commit -m "feat: oracle sampler — stratified replay selection for Docker restoration Refs #317"
```

---

## Batch 2: abilLink Discovery

Safe wrap point: All human replay abilLinks discovered, AbilityMapping extended, UpgradeType enum created, calibration validated. Requires oracle restoration to be complete.

### Task 3: AbilityDiscoveryCalibrationTest — discover human replay abilLinks

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AbilityDiscoveryCalibrationTest.java`

**Interfaces:**
- Consumes: Oracle restored replays (tracker + game events), `GameEventStream.events(Path)`, Scelight `RepParserEngine.parseReplay(Path, EnumSet.of(RepContent.TRACKER_EVENTS))`
- Produces: Printed abilLink → event type mapping table, assertion that all 53 building types, 53 unit types, and 15 upgrades have at least one abilLink discovered

This test follows the `SC2TrainTimeCalibrationTest` pattern: parse game events and tracker events from the same replay, cross-reference by player and temporal proximity.

- [ ] **Step 1: Write the discovery test skeleton**

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.factory.RepContent;
import hu.scelight.sc2.rep.factory.RepParserEngine;
import hu.scelight.sc2.rep.model.Replay;
import hu.scelight.sc2.rep.model.gameevents.cmd.CmdEvent;
import hu.scelight.sc2.rep.s2prot.Event;
import hu.scelightapi.sc2.rep.model.trackerevents.IBaseUnitEvent;
import hu.scelightapi.sc2.rep.model.trackerevents.ITrackerEvents;
import hu.scelightapi.sc2.rep.model.trackerevents.IUpgradeEvent;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Discovers abilLink → unit/building/upgrade mappings by cross-referencing
 * game event commands with tracker events in oracle-restored replays.
 *
 * Pattern: for each tracker event (UnitBorn, UnitInit, Upgrade), find the
 * game event CmdEvent from the same player in the preceding time window.
 * The modal (abilLink, abilCmdIndex) pair for each tracker event type
 * is the mapping.
 */
class AbilityDiscoveryCalibrationTest {

    private static final Path ORACLE_DIR = Path.of(
        "../quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored");

    static boolean oracleExists() {
        return Files.isDirectory(ORACLE_DIR)
            && ORACLE_DIR.toFile().listFiles(f -> f.getName().endsWith(".SC2Replay")).length > 0;
    }

    @Test
    @EnabledIf("oracleExists")
    void discoverAbilLinksFromOracleSet() throws Exception {
        // Map: trackerEventName → Map<(abilLink, abilCmdIndex), count>
        Map<String, Map<String, Integer>> discoveredMappings = new TreeMap<>();

        try (var stream = Files.list(ORACLE_DIR)) {
            for (Path replayPath : stream.filter(p -> p.toString().endsWith(".SC2Replay")).sorted().toList()) {
                discoverFromReplay(replayPath, discoveredMappings);
            }
        }

        System.out.println("=== Discovered abilLink Mappings ===");
        for (var entry : discoveredMappings.entrySet()) {
            System.out.println(entry.getKey() + ":");
            entry.getValue().entrySet().stream()
                .sorted((a, b) -> b.getValue() - a.getValue())
                .forEach(e -> System.out.println("  " + e.getKey() + " (count=" + e.getValue() + ")"));
        }

        assertThat(discoveredMappings).as("Must discover at least some mappings").isNotEmpty();
    }

    private void discoverFromReplay(Path replayPath,
                                      Map<String, Map<String, Integer>> mappings) {
        // Parse tracker events
        Replay rep = RepParserEngine.parseReplay(replayPath, EnumSet.of(RepContent.TRACKER_EVENTS));
        if (rep == null || rep.trackerEvents == null) return;

        // Parse game events
        List<Event> gameEvents;
        try { gameEvents = GameEventStream.events(replayPath); }
        catch (Exception e) { return; }

        // Index game event commands by (playerId, loop)
        List<CmdRecord> commands = new ArrayList<>();
        for (Event raw : gameEvents) {
            if (raw instanceof CmdEvent cmd && cmd.getAbilLink() != null) {
                commands.add(new CmdRecord(
                    cmd.getUserId() + 1, // convert 0-indexed userId to 1-indexed playerId
                    cmd.getLoop(),
                    cmd.getAbilLink(),
                    Objects.requireNonNullElse(cmd.getAbilCmdIndex(), 0)));
            }
        }

        // For each tracker event, find the closest preceding command from the same player
        for (Event raw : rep.trackerEvents.getEvents()) {
            switch (raw.getId()) {
                case ITrackerEvents.ID_UNIT_BORN -> {
                    IBaseUnitEvent born = (IBaseUnitEvent) raw;
                    if (born.getControlPlayerId() == null || born.getControlPlayerId() == 0) continue;
                    if (born.getLoop() == 0) continue; // skip initial units
                    String unitName = born.getUnitTypeName().toString();
                    findMatchingCommand(commands, born.getControlPlayerId(), born.getLoop(),
                        "UnitBorn:" + unitName, mappings);
                }
                case ITrackerEvents.ID_UNIT_INIT -> {
                    IBaseUnitEvent init = (IBaseUnitEvent) raw;
                    if (init.getControlPlayerId() == null || init.getControlPlayerId() == 0) continue;
                    String buildingName = init.getUnitTypeName().toString();
                    findMatchingCommand(commands, init.getControlPlayerId(), init.getLoop(),
                        "UnitInit:" + buildingName, mappings);
                }
                case ITrackerEvents.ID_UPGRADE -> {
                    IUpgradeEvent upgrade = (IUpgradeEvent) raw;
                    if (upgrade.getPlayerId() == null) continue;
                    String upgradeName = upgrade.getUpgradeTypeName().toString();
                    // Upgrades have a research time — command precedes tracker event by upgradeTime loops
                    findMatchingCommand(commands, upgrade.getPlayerId(), upgrade.getLoop(),
                        "Upgrade:" + upgradeName, mappings, 0, 5000);
                }
            }
        }
    }

    private void findMatchingCommand(List<CmdRecord> commands, int playerId, long trackerLoop,
                                       String trackerKey, Map<String, Map<String, Integer>> mappings) {
        findMatchingCommand(commands, playerId, trackerLoop, trackerKey, mappings, 0, 1500);
    }

    private void findMatchingCommand(List<CmdRecord> commands, int playerId, long trackerLoop,
                                       String trackerKey, Map<String, Map<String, Integer>> mappings,
                                       int minLookback, int maxLookback) {
        // Find the closest preceding command from the same player within the lookback window
        CmdRecord best = null;
        long bestDist = Long.MAX_VALUE;
        for (CmdRecord cmd : commands) {
            if (cmd.playerId != playerId) continue;
            long dist = trackerLoop - cmd.loop;
            if (dist >= minLookback && dist <= maxLookback && dist < bestDist) {
                bestDist = dist;
                best = cmd;
            }
        }
        if (best != null) {
            String key = "abilLink=" + best.abilLink + ",idx=" + best.abilCmdIndex;
            mappings.computeIfAbsent(trackerKey, k -> new TreeMap<>())
                .merge(key, 1, Integer::sum);
        }
    }

    record CmdRecord(int playerId, long loop, int abilLink, int abilCmdIndex) {}
}
```

- [ ] **Step 2: Run the discovery test (requires oracle set)**

```bash
mvn test -pl quarkmind-sc2 -Dtest=AbilityDiscoveryCalibrationTest -q
```
Expected: prints abilLink → tracker event type mapping table. Each building/unit/upgrade type should have a dominant (abilLink, abilCmdIndex) pair.

- [ ] **Step 3: Analyse output and document discovered abilLinks**

Review the output table. For each tracker event type, the modal abilLink pair is the mapping. Document in a `discovered-abillinks.md` file in the spec directory for reference during AbilityMapping extension.

- [ ] **Step 4: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AbilityDiscoveryCalibrationTest.java
git commit -m "feat: AbilityDiscoveryCalibrationTest — cross-references game+tracker events for abilLink mapping Refs #317"
```

### Task 4: UpgradeType enum + SC2Data.upgradeTimeInLoops

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/domain/UpgradeType.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java` (add upgradeTimeInLoops)
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/domain/UpgradeTypeTest.java`

**Interfaces:**
- Consumes: Discovered upgrade abilLinks from Task 3 output
- Produces: `UpgradeType` enum with 15 values matching Python's UPGRADES list, `SC2Data.upgradeTimeInLoops(UpgradeType)` returning calibrated research times

- [ ] **Step 1: Write failing test for UpgradeType enum**

```java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class UpgradeTypeTest {

    @Test
    void allClassifierUpgradesCovered() {
        // Python UPGRADES list from sc2egset_extractor.py
        String[] expected = {
            "Stimpack", "ShieldWall", "PunisherGrenades", "BansheeCloak",
            "TerranVehicleWeaponsLevel1", "PersonalCloaking", "DrillClaws",
            "zerglingmovementspeed", "GlialReconstitution", "CentrificalHooks",
            "Burrow", "WarpGateResearch", "BlinkTech", "Charge",
            "AdeptPiercingAttack"
        };
        assertThat(UpgradeType.values()).hasSize(expected.length);
        for (String name : expected) {
            assertThat(UpgradeType.fromPythonName(name))
                .as("UpgradeType for Python name '%s'", name)
                .isNotNull();
        }
    }

    @Test
    void upgradeTimesArePositive() {
        for (UpgradeType ut : UpgradeType.values()) {
            assertThat(SC2Data.upgradeTimeInLoops(ut))
                .as("upgradeTimeInLoops(%s)", ut)
                .isGreaterThan(0);
        }
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
mvn test -pl quarkmind-sc2 -Dtest=UpgradeTypeTest -q
```
Expected: compilation error — `UpgradeType` does not exist.

- [ ] **Step 3: Create UpgradeType enum**

Use `ide_create_file` to create `quarkmind-sc2/src/main/java/io/quarkmind/domain/UpgradeType.java`:

```java
package io.quarkmind.domain;

import java.util.Map;

public enum UpgradeType {
    STIMPACK("Stimpack"),
    COMBAT_SHIELD("ShieldWall"),
    CONCUSSIVE_SHELLS("PunisherGrenades"),
    BANSHEE_CLOAK("BansheeCloak"),
    TERRAN_VEHICLE_WEAPONS_1("TerranVehicleWeaponsLevel1"),
    PERSONAL_CLOAKING("PersonalCloaking"),
    DRILL_CLAWS("DrillClaws"),
    ZERGLING_SPEED("zerglingmovementspeed"),
    GLIAL_RECONSTITUTION("GlialReconstitution"),
    CENTRIFUGAL_HOOKS("CentrificalHooks"),
    BURROW("Burrow"),
    WARP_GATE_RESEARCH("WarpGateResearch"),
    BLINK("BlinkTech"),
    CHARGE("Charge"),
    ADEPT_PIERCING("AdeptPiercingAttack");

    private final String pythonName;

    private static final Map<String, UpgradeType> BY_PYTHON_NAME;
    static {
        var map = new java.util.HashMap<String, UpgradeType>();
        for (UpgradeType ut : values()) map.put(ut.pythonName, ut);
        BY_PYTHON_NAME = Map.copyOf(map);
    }

    UpgradeType(String pythonName) { this.pythonName = pythonName; }
    public String pythonName() { return pythonName; }
    public static UpgradeType fromPythonName(String name) { return BY_PYTHON_NAME.get(name); }
}
```

- [ ] **Step 4: Add SC2Data.upgradeTimeInLoops**

Use `ide_insert_member` to add to `SC2Data.java` after `buildTimeInTicks`:

```java
private static final Map<UpgradeType, Integer> UPGRADE_TIMES = Map.ofEntries(
    Map.entry(UpgradeType.STIMPACK, (int)(100 * GAME_LOOPS_PER_SECOND)),
    Map.entry(UpgradeType.COMBAT_SHIELD, (int)(100 * GAME_LOOPS_PER_SECOND)),
    Map.entry(UpgradeType.CONCUSSIVE_SHELLS, (int)(43 * GAME_LOOPS_PER_SECOND)),
    Map.entry(UpgradeType.BANSHEE_CLOAK, (int)(79 * GAME_LOOPS_PER_SECOND)),
    Map.entry(UpgradeType.TERRAN_VEHICLE_WEAPONS_1, (int)(114 * GAME_LOOPS_PER_SECOND)),
    Map.entry(UpgradeType.PERSONAL_CLOAKING, (int)(86 * GAME_LOOPS_PER_SECOND)),
    Map.entry(UpgradeType.DRILL_CLAWS, (int)(79 * GAME_LOOPS_PER_SECOND)),
    Map.entry(UpgradeType.ZERGLING_SPEED, (int)(71 * GAME_LOOPS_PER_SECOND)),
    Map.entry(UpgradeType.GLIAL_RECONSTITUTION, (int)(57 * GAME_LOOPS_PER_SECOND)),
    Map.entry(UpgradeType.CENTRIFUGAL_HOOKS, (int)(43 * GAME_LOOPS_PER_SECOND)),
    Map.entry(UpgradeType.BURROW, (int)(71 * GAME_LOOPS_PER_SECOND)),
    Map.entry(UpgradeType.WARP_GATE_RESEARCH, (int)(114 * GAME_LOOPS_PER_SECOND)),
    Map.entry(UpgradeType.BLINK, (int)(121 * GAME_LOOPS_PER_SECOND)),
    Map.entry(UpgradeType.CHARGE, (int)(100 * GAME_LOOPS_PER_SECOND)),
    Map.entry(UpgradeType.ADEPT_PIERCING, (int)(100 * GAME_LOOPS_PER_SECOND))
);

public static int upgradeTimeInLoops(UpgradeType type) {
    return UPGRADE_TIMES.getOrDefault(type, (int)(100 * GAME_LOOPS_PER_SECOND));
}
```

Note: these are wiki-derived estimates. Phase 2d calibration will refine them from oracle data.

- [ ] **Step 5: Run tests — verify pass**

```bash
mvn test -pl quarkmind-sc2 -Dtest=UpgradeTypeTest -q
```
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/domain/UpgradeType.java quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java quarkmind-sc2/src/test/java/io/quarkmind/domain/UpgradeTypeTest.java
git commit -m "feat: UpgradeType enum + SC2Data.upgradeTimeInLoops — 15 classifier-tracked upgrades Refs #317"
```

### Task 5: Extend AbilityMapping with discovered abilLinks

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayCommand.java` (add BuildCommand, UpgradeCommand, MorphCommand variants)
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ExtendedAbilityMappingTest.java`

**Interfaces:**
- Consumes: Discovered abilLink table from Task 3, `UpgradeType` from Task 4
- Produces: `ReplayCommand.BuildCommand(loop, buildingPythonName, position)`, `ReplayCommand.UpgradeCommand(loop, UpgradeType)`, `ReplayCommand.MorphCommand(loop, sourcePythonName, targetPythonName)`, `ReplayCommand.CancelCommand(loop, buildingTag)`

This task depends on Task 3's discovery output — the exact abilLink numbers are populated from the discovery test results. The test structure and ReplayCommand variants are defined here; the specific constant values are filled in from discovery data.

- [ ] **Step 1: Write failing test for new ReplayCommand variants**

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.domain.UpgradeType;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ExtendedAbilityMappingTest {

    @Test
    void buildCommandVariantExists() {
        var cmd = new ReplayCommand.BuildCommand(100, "Pylon", null);
        assertThat(cmd.buildingPythonName()).isEqualTo("Pylon");
        assertThat(cmd.loop()).isEqualTo(100);
    }

    @Test
    void upgradeCommandVariantExists() {
        var cmd = new ReplayCommand.UpgradeCommand(200, UpgradeType.WARP_GATE_RESEARCH);
        assertThat(cmd.upgradeType()).isEqualTo(UpgradeType.WARP_GATE_RESEARCH);
    }

    @Test
    void morphCommandVariantExists() {
        var cmd = new ReplayCommand.MorphCommand(300, "Zergling", "Baneling");
        assertThat(cmd.sourcePythonName()).isEqualTo("Zergling");
        assertThat(cmd.targetPythonName()).isEqualTo("Baneling");
    }

    @Test
    void cancelCommandVariantExists() {
        var cmd = new ReplayCommand.CancelCommand(400, "r-42-1");
        assertThat(cmd.buildingTag()).isEqualTo("r-42-1");
    }
}
```

- [ ] **Step 2: Run — verify compilation fails**

- [ ] **Step 3: Add ReplayCommand variants**

Use `ide_insert_member` on `ReplayCommand.java` to add:
```java
record BuildCommand(long loop, String buildingPythonName, Point2d position) implements ReplayCommand {}
record UpgradeCommand(long loop, UpgradeType upgradeType) implements ReplayCommand {}
record MorphCommand(long loop, String sourcePythonName, String targetPythonName) implements ReplayCommand {}
record CancelCommand(long loop, String buildingTag) implements ReplayCommand {}
```

- [ ] **Step 4: Extend AbilityMapping dispatch table**

Use `ide_edit_member` on `AbilityMapping.dispatch()` to add cases for discovered building, upgrade, morph, and cancel abilLinks. The specific constant values come from Task 3's discovery output. Add private static final maps for each category (e.g., `BUILDING_ABILLINKS`, `UPGRADE_ABILLINKS`, `MORPH_ABILLINKS`).

Example pattern for buildings:
```java
// --- Building placement (discovered from oracle) ---
private static final Map<Integer, String> BUILDING_ABILLINKS = Map.ofEntries(
    // Values populated from AbilityDiscoveryCalibrationTest output
    // Map.entry(abilLink, "PythonBuildingName")
);
```

The dispatch switch adds:
```java
default -> {
    String building = BUILDING_ABILLINKS.get(abilLink);
    if (building != null) {
        yield buildCommand(loop, building, event);
    }
    String upgrade = UPGRADE_ABILLINKS.get(abilLink + ":" + idx);
    if (upgrade != null) {
        yield upgradeCommand(loop, UpgradeType.fromPythonName(upgrade));
    }
    // ... morphs, cancels
    yield unknown(abilLink, idx);
}
```

- [ ] **Step 5: Run tests — verify pass**

```bash
mvn test -pl quarkmind-sc2 -Dtest=ExtendedAbilityMappingTest -q
```

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayCommand.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ExtendedAbilityMappingTest.java
git commit -m "feat: extend AbilityMapping — buildings, upgrades, morphs, cancels for human replays Refs #317"
```

---

## Batch 3: StrippedReplayFeatureExtractor

Safe wrap point: Feature extractor produces game_json from stripped replays with production queues, morphs, economy, and WarpGate auto-morph. Unit-tested independently of oracle.

### Task 6: StrippedReplayFeatureExtractor core — production queue + unit births

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java`
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractorTest.java`

**Interfaces:**
- Consumes: `GameEventStream.events(Path)`, `AbilityMapping` (extended), `SC2Data.trainTimeInLoops()`, `SC2Data.buildTimeInLoops()`, `SC2Data.upgradeTimeInLoops()`
- Produces: `Map<String, Object> extractGameJson(Path replayPath)` — game_json dict with synthetic trackerEvents list

This task implements §3a (core extractor), §3a-1 (building double-counting), §3a-2 (production queues), and the game_json output structure (§3c). Morphs, WarpGate auto-morph, and economy are separate tasks.

- [ ] **Step 1: Write failing test — basic probe train extraction**

```java
package io.quarkmind.sc2.replay;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class StrippedReplayFeatureExtractorTest {

    private static final Path LADDER_493 = Path.of(
        "../quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3/replays");

    static boolean ladderReplaysExist() {
        return Files.isDirectory(LADDER_493);
    }

    @Test
    @EnabledIf("ladderReplaysExist")
    void extractsUnitBornEventsFromStrippedReplay() throws Exception {
        Path replay = Files.list(LADDER_493)
            .filter(p -> p.toString().endsWith(".SC2Replay"))
            .sorted().findFirst().orElseThrow();

        var extractor = new StrippedReplayFeatureExtractor();
        Map<String, Object> gameJson = extractor.extract(replay);

        assertThat(gameJson).containsKey("trackerEvents");
        assertThat(gameJson).containsKey("ToonPlayerDescMap");
        assertThat(gameJson).containsKey("header");

        @SuppressWarnings("unchecked")
        List<Map<String, Object>> events = (List<Map<String, Object>>) gameJson.get("trackerEvents");
        long unitBornCount = events.stream()
            .filter(e -> "UnitBorn".equals(e.get("evtTypeName")))
            .count();

        assertThat(unitBornCount).as("Must have synthetic UnitBorn events").isGreaterThan(0);

        // Check first Probe train — should appear at loop ~268 (calibrated probe train time)
        var firstProbe = events.stream()
            .filter(e -> "UnitBorn".equals(e.get("evtTypeName")) && "Probe".equals(e.get("unitTypeName")))
            .findFirst();
        assertThat(firstProbe).isPresent();
    }

    @Test
    @EnabledIf("ladderReplaysExist")
    void productionQueueDelaysSecondUnit() throws Exception {
        // This test verifies that queued units don't all appear at the same loop
        Path replay = Files.list(LADDER_493)
            .filter(p -> p.toString().endsWith(".SC2Replay"))
            .sorted().findFirst().orElseThrow();

        var extractor = new StrippedReplayFeatureExtractor();
        Map<String, Object> gameJson = extractor.extract(replay);

        @SuppressWarnings("unchecked")
        List<Map<String, Object>> events = (List<Map<String, Object>>) gameJson.get("trackerEvents");
        List<Long> probeLoops = events.stream()
            .filter(e -> "UnitBorn".equals(e.get("evtTypeName")) && "Probe".equals(e.get("unitTypeName")))
            .map(e -> ((Number) e.get("loop")).longValue())
            .toList();

        if (probeLoops.size() >= 2) {
            // Second probe should be at least trainTime later than first
            assertThat(probeLoops.get(1) - probeLoops.get(0))
                .as("Second probe must be delayed by at least one train cycle")
                .isGreaterThanOrEqualTo(200); // probe train time is ~268 loops
        }
    }
}
```

- [ ] **Step 2: Run — verify compilation fails**

- [ ] **Step 3: Implement StrippedReplayFeatureExtractor**

Create the class with:
- `extract(Path replayPath)` → `Map<String, Object>` (game_json)
- Internal state: `Map<String, Long> buildingBusyUntil` (production queues)
- Event scheduling: collect all commands, compute birth/completion loops, sort by loop, emit as trackerEvents list
- Player metadata from `gamemetadata.json` via Scelight's `RepParserEngine` (replay.details gives player info)

The implementation processes commands in loop order, maintains per-building production queues per §3a-2, and emits events using Python-compatible string names.

- [ ] **Step 4: Run tests — verify pass**

```bash
mvn test -pl quarkmind-sc2 -Dtest=StrippedReplayFeatureExtractorTest -q
```

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractorTest.java
git commit -m "feat: StrippedReplayFeatureExtractor — synthetic tracker events from game commands Refs #317"
```

### Task 7: Morph-death semantics + WarpGate auto-morph

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractorTest.java`

**Interfaces:**
- Consumes: `ReplayCommand.MorphCommand` from extended AbilityMapping
- Produces: UnitDied events for morph sources (Zergling→Baneling kills Zergling), UnitInit/UnitBorn for morph targets, Gateway→WarpGate auto-morph on WarpGateResearch completion

- [ ] **Step 1: Write failing test for morph-death source emission**

Test that a morph command produces both a UnitDied for the source and a UnitBorn/UnitInit for the target. Use a synthetic command stream (mock AbilityMapping output) rather than a real replay.

- [ ] **Step 2: Write failing test for WarpGate auto-morph**

Test that after a WarpGateResearch upgrade event, all tracked Gateways emit UnitDied(Gateway) + UnitInit(WarpGate) + UnitDone(WarpGate).

- [ ] **Step 3: Implement morph handling in extractor**

Add morph table mapping source→target per §3a-3. On MorphCommand, emit source UnitDied + target UnitBorn/UnitInit. Handle Archon merge (2 source deaths). Handle Drone→Building (source death = worker count impact).

- [ ] **Step 4: Implement WarpGate auto-morph per §3a-4**

Track WarpGateResearch completion loop. When reached, iterate all tracked Gateway buildings for the player, emit morph events.

- [ ] **Step 5: Run tests — verify pass**

- [ ] **Step 6: Commit**

```bash
git commit -m "feat: morph-death semantics + WarpGate auto-morph in feature extractor Refs #317"
```

### Task 8: Economy reconstruction (3 tiers)

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractorTest.java`

**Interfaces:**
- Consumes: `SC2Data.mineralCost()`, `SC2Data.gasCost()`, `SC2Data.supplyCost()`, `SC2Data.supplyBonus()`, `SC2Data.mineralIncomePerTick()`
- Produces: Synthetic PlayerStats events at regular intervals (every 160 loops, matching SC2's reporting frequency) with all 13 economy stats per §3b tiers

- [ ] **Step 1: Write failing test for Tier 1 stats (exact from commands)**

Test that `MineralsUsedCurrentArmy` accumulates correctly from unit train costs.

- [ ] **Step 2: Write failing test for Tier 2 stats (mining model)**

Test that `MineralsCurrent` is positive and decreases after spending.

- [ ] **Step 3: Implement economy tracking**

Add cumulative cost tracking (Tier 1), mining model using `SC2Data.mineralIncomePerTick()` (Tier 2), and supply tracking with morph-death awareness (Tier 3). Emit synthetic PlayerStats events every 160 loops.

- [ ] **Step 4: Run tests — verify pass**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat: economy reconstruction — 3-tier accuracy model for PlayerStats events Refs #317"
```

---

## Batch 4: Oracle Validation + Bulk Processing

Safe wrap point: Java pipeline validated against oracle, prepare_replay_pack.py accepts JSON input, bulk processing complete, ONNX retrained.

### Task 9: Oracle validation test — Java vs Docker ground truth

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayValidationTest.java`

**Interfaces:**
- Consumes: Oracle restored replays (tracker events), stripped originals (game events), `StrippedReplayFeatureExtractor`
- Produces: Per-replay divergence report. Asserts zero divergence on deterministic features (unit births, buildings, upgrades). Reports economy divergence per §3b tolerance tiers.

- [ ] **Step 1: Write validation test**

For each oracle replay:
1. Extract game_json via `StrippedReplayFeatureExtractor` from the STRIPPED original
2. Extract ground truth tracker events from the RESTORED oracle replay via Scelight
3. Compare: UnitBorn events (type + loop ±1 tick), UnitInit events, Upgrade events
4. Report: economy stats divergence per tier

```java
@Test
@EnabledIf("oracleAndOriginalsExist")
void javaOutputMatchesOracleForDeterministicFeatures() throws Exception {
    // For each oracle replay, compare Java extraction of the STRIPPED original
    // against the tracker events in the RESTORED oracle
    var extractor = new StrippedReplayFeatureExtractor();
    int totalReplays = 0, passedReplays = 0;

    try (var oracleStream = Files.list(ORACLE_RESTORED_DIR)) {
        for (Path oracleReplay : oracleStream.filter(p -> p.toString().endsWith(".SC2Replay")).sorted().toList()) {
            Path stripped = STRIPPED_DIR.resolve(oracleReplay.getFileName());
            if (!Files.exists(stripped)) continue;

            totalReplays++;
            Map<String, Object> javaJson = extractor.extract(stripped);
            // Parse oracle tracker events via Scelight
            // Compare unit births, buildings, upgrades
            // Report economy divergence
            passedReplays++;
        }
    }

    assertThat(passedReplays).as("Most oracle replays must pass validation")
        .isGreaterThan(totalReplays * 9 / 10); // >90% pass rate
}
```

- [ ] **Step 2: Run — fix divergences iteratively per §4c**

Each divergence drives a fix in SC2Data, AbilityMapping, or the extractor. Re-run until oracle set passes.

- [ ] **Step 3: Commit**

```bash
git commit -m "feat: StrippedReplayValidationTest — Java vs oracle validation loop Refs #317"
```

### Task 10: prepare_replay_pack.py JSON input + Java batch runner

**Files:**
- Modify: `quarkmind-classifier/src/prepare_replay_pack.py` (add `--format json`)
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/BulkFeatureExtractor.java`

**Interfaces:**
- Consumes: 151K stripped replays, validated `StrippedReplayFeatureExtractor`
- Produces: Per-replay game_json files, training data via Python pipeline, retrained ONNX models

- [ ] **Step 1: Add --format json to prepare_replay_pack.py**

Add a new code path in `__main__` that reads JSON files instead of `.SC2Replay` files:
```python
parser.add_argument("--format", choices=["replay", "json"], default="replay",
                    help="Input format: 'replay' for .SC2Replay, 'json' for game_json files")
```

When `--format json`, the worker reads JSON directly and calls `extract_replay()`:
```python
def _process_one_json(args):
    json_path, replay_seed = args
    with open(json_path) as f:
        game_json = json.load(f)
    # ... same pipeline as _process_one_replay after sc2reader_to_game_json
```

- [ ] **Step 2: Write BulkFeatureExtractor.java**

```java
public class BulkFeatureExtractor {
    public static void main(String[] args) {
        Path inputDir = Path.of(args[0]);  // stripped replay directory
        Path outputDir = Path.of(args[1]); // game_json output directory
        int threads = args.length > 2 ? Integer.parseInt(args[2]) : 4;
        // Parallel processing with crash-resume (skip existing output files)
    }
}
```

- [ ] **Step 3: Run bulk extraction**

```bash
mvn exec:java -pl quarkmind-sc2 -Dexec.mainClass=io.quarkmind.sc2.replay.BulkFeatureExtractor \
  -Dexec.args="quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3/replays /tmp/game_json_output 8"
```

- [ ] **Step 4: Feed through Python pipeline**

```bash
cd quarkmind-classifier
PYTHONPATH=. .venv/bin/python3 -m src.prepare_replay_pack --dir /tmp/game_json_output --format json --name blizzard_ladder --batch-size 5000 --workers 10
PYTHONPATH=. .venv/bin/python3 -m src.normalize --source blizzard_ladder
PYTHONPATH=. .venv/bin/python3 -m src.run_pipeline --data combined
```

- [ ] **Step 5: Commit**

```bash
git add quarkmind-classifier/src/prepare_replay_pack.py quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/BulkFeatureExtractor.java
git commit -m "feat: bulk feature extraction + ONNX retrain with 151K ladder replays Refs #317"
```

---

## References

- [2026-09-28-blizzard-ladder-restore-design.md] — design spec this plan implements
- [decisions.md] — 7 design decisions (D1-D7) with trade-offs
- [AbilityMapping.java] — existing abilLink dispatch (lines 183-228)
- [SC2Data.java] — calibrated train/build times, unit costs
- [ReplayCommand.java] — sealed interface with Movement and IntentCommand variants
- [ReplayCommandExtractor.java] — existing command extraction pipeline
- [sc2egset_extractor.py:25-88] — 134 feature definitions (BUILDINGS, UNITS, STAT_KEYS, UPGRADES)
- [prepare_replay_pack.py] — Python feature engineering pipeline
- [SC2TrainTimeCalibrationTest.java] — abilLink discovery pattern
- [StrippedReplayParseTest.java] — validated: Scelight parses stripped replays
- [docker/sc2-restore/restore_tracker.py] — Docker restoration pipeline
- [GitHub #317] — focal issue
