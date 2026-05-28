# IEM10 gameEvents → TimedIntent Stream — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Parse IEM10 JSON `gameEvents` to produce `List<TimedIntent>` via `IEM10CommandExtractor`, and generalise `ReplayValidationHarness` to accept any `SimulatedGame` so IEM10 JSON games can be validated against `EmulatedGame`.

**Architecture:** `IEM10JsonSimulatedGame` gains `watchedUserId` and `gameEvents` fields (package-private accessors). New class `IEM10CommandExtractor` in the same package owns selection-state tracking and ability dispatch using IEM10 2016 patch constants. `ReplayValidationHarness.run(SimulatedGame, List<TimedIntent>, int)` is the new general overload; the existing `run(Path, int, int)` delegates to it.

**Tech Stack:** Java 21, Jackson `JsonNode`, JUnit 5, AssertJ. No new dependencies.

---

## Pre-work: IEM10 ability constants (already discovered)

The constants below were identified via narrow-window modal correlation across all 30 IEM10 games
(Python script, 2026-05-27). They are **different** from AI Arena 2023+ values — both the
abilLink numbers and the abilCmdIndex values shifted between SC2 patches.

The `IEM10AbilityDiscoveryTest` in Task 0 verifies these constants and prints derivation evidence.

| Building        | abilLink_2016 | abilCmdIndex | UnitType  | AI Arena 2023+ abilLink |
|-----------------|---------------|--------------|-----------|-------------------------|
| Nexus           | 167           | 0            | PROBE     | 175                     |
| Gateway         | 164           | 1            | STALKER   | 172 (idx 0)             |
| Gateway         | 164           | 6            | ADEPT     | 172 (idx 5)             |
| RoboticsFacility| 166           | 1            | OBSERVER  | 173 (idx 1)             |
| RoboticsFacility| 166           | 3            | IMMORTAL  | 173 (idx 0)             |
| RoboticsFacility| 166           | 18           | DISRUPTOR | 173 (idx ?)             |
| Stargate        | 165           | 0            | PHOENIX   | 174 (idx 0)             |
| Stargate        | 165           | 9            | TEMPEST   | 174 (new unit)          |
| Larva           | 185           | 0            | DRONE     | 193 (idx 0)             |
| Larva           | 185           | 1            | ZERGLING  | 193 (idx 1)             |
| Larva           | 185           | 9            | ROACH     | 193 (idx 2)             |
| Hatchery        | 235           | 0            | QUEEN     | 184 (idx 1)             |
| CommandCenter   | 147           | 0            | SCV       | N/A (not in AbilityMapping)|
| Barracks        | 151           | 0            | MARINE    | N/A                     |

Tag decode formula: `int index = (int)(packed >> 18); int recycle = (int)(packed & 0x3FFFF)`
(verified: packed=52166657 → index=199, recycle=1, matches tracker event unitTagIndex/unitTagRecycle ✓)

---

## File Map

| Action | File |
|--------|------|
| Modify | `src/main/java/io/quarkmind/sc2/mock/IEM10JsonSimulatedGame.java` |
| Create | `src/main/java/io/quarkmind/sc2/mock/IEM10CommandExtractor.java` |
| Modify | `src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java` |
| Create | `src/test/java/io/quarkmind/sc2/mock/IEM10AbilityDiscoveryTest.java` |
| Create | `src/test/java/io/quarkmind/sc2/mock/IEM10CommandExtractorTest.java` |
| Create | `src/test/java/io/quarkmind/sc2/mock/IEM10MultiGameValidationTest.java` |
| Verify | `src/test/java/io/quarkmind/sc2/replay/ReplayValidationHarnessTest.java` (existing) |

---

## Task 0: Write `IEM10AbilityDiscoveryTest`

Documents how the IEM10 2016 ability constants were derived. Tagged `@Tag("diagnostic")`,
excluded from CI. Run once to verify constants before implementing `IEM10CommandExtractor`.

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/mock/IEM10AbilityDiscoveryTest.java`

- [ ] **Step 0.1: Create the discovery test**

```java
package io.quarkmind.sc2.mock;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.commons.compress.archivers.zip.ZipArchiveEntry;
import org.apache.commons.compress.archivers.zip.ZipArchiveInputStream;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;
import java.util.zip.ZipEntry;
import java.util.zip.ZipInputStream;

/**
 * Diagnostic: prints the (abilLink, abilCmdIndex) → unit-born correlation table for all
 * three races across all 30 IEM10 games, using narrow-window modal matching.
 *
 * Run with: mvn test -Dtest=IEM10AbilityDiscoveryTest -Pdiagnostic -q
 * (or: mvn test -Dtest=IEM10AbilityDiscoveryTest -Dgroups=diagnostic)
 *
 * Constants hard-coded in IEM10CommandExtractor were derived from this output.
 * Expected top matches (IEM10 2016 build ~39948):
 *   Protoss: Probe=167/0, Stalker=164/1, Adept=164/6, Observer=166/1,
 *            Immortal=166/3, Disruptor=166/18, Phoenix=165/0, Tempest=165/9
 *   Zerg:    Drone=185/0, Zergling=185/1, Roach=185/9, Queen=235/0
 *   Terran:  SCV=147/0, Marine=151/0
 */
@Tag("diagnostic")
class IEM10AbilityDiscoveryTest {

    private static final Path IEM10_ZIP = Path.of("replays/2016_IEM_10_Taipei.zip");

    // Narrow training-time windows in loops (±150 slack around expected SC2Data values)
    private static final Map<String, int[]> WINDOWS = new LinkedHashMap<>();
    static {
        // Protoss
        WINDOWS.put("Probe",         new int[]{120,  400});
        WINDOWS.put("Stalker",       new int[]{544,  844});
        WINDOWS.put("Zealot",        new int[]{477,  777});
        WINDOWS.put("Adept",         new int[]{477,  777});
        WINDOWS.put("Immortal",      new int[]{746, 1046});
        WINDOWS.put("Observer",      new int[]{342,  642});
        WINDOWS.put("Disruptor",     new int[]{656,  956});
        WINDOWS.put("WarpPrism",     new int[]{450,  750});
        WINDOWS.put("Phoenix",       new int[]{377,  677});
        WINDOWS.put("Tempest",       new int[]{756, 1056});
        // Zerg
        WINDOWS.put("Drone",         new int[]{200,  500});
        WINDOWS.put("Zergling",      new int[]{500,  850});
        WINDOWS.put("Roach",         new int[]{550,  850});
        WINDOWS.put("Mutalisk",      new int[]{600,  950});
        WINDOWS.put("Queen",         new int[]{650,  950});
        // Terran
        WINDOWS.put("SCV",           new int[]{200,  450});
        WINDOWS.put("Marine",        new int[]{350,  650});
        WINDOWS.put("Marauder",      new int[]{450,  750});
        WINDOWS.put("Medivac",       new int[]{550,  850});
    }

    @Test
    void printAbilityCorrelations() throws IOException {
        // race -> unit -> (abilLink,idx) -> pair count
        Map<String, Map<String, Map<String, Integer>>> results = new TreeMap<>();
        for (String r : List.of("Prot", "Zerg", "Terr"))
            results.put(r, new TreeMap<>());

        try (ZipArchiveInputStream outer = new ZipArchiveInputStream(
                Files.newInputStream(IEM10_ZIP), "UTF-8", true, true)) {
            ZipArchiveEntry outerEntry;
            while ((outerEntry = outer.getNextEntry()) != null) {
                if (!outerEntry.getName().endsWith("_data.zip")) continue;
                byte[] innerBytes = outer.readAllBytes();
                try (ZipInputStream inner = new ZipInputStream(new ByteArrayInputStream(innerBytes))) {
                    ZipEntry innerEntry;
                    while ((innerEntry = inner.getNextEntry()) != null) {
                        if (!innerEntry.getName().endsWith(".SC2Replay.json")) continue;
                        JsonNode root = new ObjectMapper().readTree(inner.readAllBytes());
                        processGame(root, results);
                    }
                }
            }
        }

        System.out.println("\n=== IEM10 Ability Discovery — narrow-window modal correlation ===");
        for (var raceEntry : results.entrySet()) {
            System.out.println("\n[" + raceEntry.getKey() + "]");
            for (var unitEntry : raceEntry.getValue().entrySet()) {
                var counts = new TreeMap<>(Comparator.comparingInt(
                    (String k) -> -unitEntry.getValue().get(k)));
                counts.putAll(unitEntry.getValue());
                var top = counts.entrySet().stream().limit(3).toList();
                if (top.isEmpty()) continue;
                System.out.printf("  %-16s best=%s(%d)  alts=%s%n",
                    unitEntry.getKey(),
                    top.get(0).getKey(), top.get(0).getValue(),
                    top.subList(1, top.size()));
            }
        }
    }

    private void processGame(JsonNode root,
                             Map<String, Map<String, Map<String, Integer>>> results) {
        // userId -> race, playerId -> race
        Map<Integer, String> userToRace   = new HashMap<>();
        Map<Integer, String> playerToRace = new HashMap<>();
        for (JsonNode v : root.get("ToonPlayerDescMap")) {
            userToRace.put(v.get("userID").asInt(),    v.get("race").asText());
            playerToRace.put(v.get("playerID").asInt(), v.get("race").asText());
        }

        // Collect born loops per (race, unitType)
        Map<String, List<Long>> bornByUnit = new HashMap<>(); // "race:unit" -> loops
        for (JsonNode e : root.get("trackerEvents")) {
            if (!"UnitBorn".equals(e.path("evtTypeName").asText())) continue;
            long loop = e.path("loop").asLong();
            if (loop == 0) continue;
            int pid  = e.path("controlPlayerId").asInt();
            String race = playerToRace.get(pid);
            String unit = e.path("unitTypeName").asText();
            if (race != null && WINDOWS.containsKey(unit))
                bornByUnit.computeIfAbsent(race + ":" + unit, k -> new ArrayList<>()).add(loop);
        }

        // Collect no-target Cmd events per race
        Map<String, List<long[]>> cmdByRace = new HashMap<>(); // race -> [loop, abilLink, idx]
        for (JsonNode e : root.get("gameEvents")) {
            if (!"Cmd".equals(e.path("evtTypeName").asText())) continue;
            if (!isNoTarget(e)) continue;
            int uid = e.path("userid").path("userId").asInt();
            String race = userToRace.get(uid);
            if (race == null) continue;
            JsonNode abil = e.path("abil");
            if (abil.isMissingNode()) continue;
            int link = abil.path("abilLink").asInt(-1);
            int idx  = abil.path("abilCmdIndex").asInt(0);
            if (link < 0) continue;
            cmdByRace.computeIfAbsent(race, k -> new ArrayList<>())
                     .add(new long[]{e.path("loop").asLong(), link, idx});
        }

        // Cross-reference: for each born event find cmds within its window
        for (var entry : bornByUnit.entrySet()) {
            String[] parts = entry.getKey().split(":", 2);
            String race = parts[0], unit = parts[1];
            int[] window = WINDOWS.get(unit);
            List<long[]> cmds = cmdByRace.getOrDefault(race, List.of());
            Map<String, Integer> unitMap = results.get(race)
                .computeIfAbsent(unit, k -> new HashMap<>());
            for (long bornLoop : entry.getValue()) {
                for (long[] cmd : cmds) {
                    long diff = bornLoop - cmd[0];
                    if (diff >= window[0] && diff <= window[1]) {
                        String key = cmd[1] + "/" + cmd[2];
                        unitMap.merge(key, 1, Integer::sum);
                    }
                }
            }
        }
    }

    private boolean isNoTarget(JsonNode event) {
        JsonNode data = event.path("data");
        if (data.isMissingNode()) return true;
        return data.has("None") && data.size() == 1;
    }
}
```

- [ ] **Step 0.2: Run the discovery test and verify constants**

```bash
mvn test -Dtest=IEM10AbilityDiscoveryTest -q
```

Expected output shows best matches aligning with the Pre-work table:
- `Probe: best=167/0`, `Stalker: best=164/1`, `Disruptor: best=166/18` (CLEAR)
- `Drone: best=185/0`, `Zergling: best=185/1`, `Roach: best=185/9` (CLEAR)
- `SCV: best=147/0`, `Marine: best=151/0` (CLEAR)

If any constant conflicts with the Pre-work table, update the constant in Task 2 before proceeding.

---

## Task 1: Extend `IEM10JsonSimulatedGame`

Add `watchedUserId` and `gameEvents` fields with package-private accessors.

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/mock/IEM10JsonSimulatedGame.java`

- [ ] **Step 1.1: Add `watchedUserId` and `gameEvents` fields**

In the class, add two fields after the existing `watchedPlayerId` field:

```java
private final int watchedPlayerId;
private final int             watchedUserId;   // ADD — for gameEvents filtering
private final List<JsonNode>  gameEvents;       // ADD — raw game events from JSON
```

- [ ] **Step 1.2: Extract `watchedUserId` in the constructor**

In the constructor, after the two `for (JsonNode player : playerMap)` loops that set
`protossId` and `enemyRace`, add a third loop to find the Protoss player's `userID`:

```java
// After the existing loops that find protossId and enemyRace:
int protossUserId = 0;
for (JsonNode player : playerMap) {
    if (player.get("playerID").asInt() == protossId) {
        protossUserId = player.get("userID").asInt();
        break;
    }
}
this.watchedUserId = protossUserId;
```

- [ ] **Step 1.3: Parse and store `gameEvents` in the constructor**

After the existing `trackerEvents` list is built (the `for (JsonNode e : root.get("trackerEvents"))` block), add:

```java
List<JsonNode> gameEventList = new ArrayList<>();
JsonNode geNode = root.get("gameEvents");
if (geNode != null) {
    for (JsonNode e : geNode) gameEventList.add(e);
}
this.gameEvents = Collections.unmodifiableList(gameEventList);
```

- [ ] **Step 1.4: Add package-private accessors**

At the end of the class, before the closing brace, add:

```java
// Package-private — used by IEM10CommandExtractor
int watchedUserId() { return watchedUserId; }
List<JsonNode> gameEvents() { return gameEvents; }
```

- [ ] **Step 1.5: Verify existing tests still pass**

```bash
mvn test -Dtest=IEM10JsonSimulatedGameTest -q
```

Expected: all 18 tests pass. The additions are additive-only.

- [ ] **Step 1.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/sc2/mock/IEM10JsonSimulatedGame.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#150): add watchedUserId + gameEvents to IEM10JsonSimulatedGame"
```

---

## Task 2: Implement `IEM10CommandExtractor`

New class in `io.quarkmind.sc2.mock`. Stateful selection tracking over `gameEvents`;
all-species ability dispatch table using IEM10 2016 patch constants.

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/mock/IEM10CommandExtractor.java`

- [ ] **Step 2.1: Write the failing test first** (see Task 3, Step 3.1 — write this test before implementing)

Skip to Task 3 Step 3.1, write the failing test, run it to confirm it fails, then return here.

- [ ] **Step 2.2: Create `IEM10CommandExtractor.java`**

```java
package io.quarkmind.sc2.mock;

import com.fasterxml.jackson.databind.JsonNode;
import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.intent.TimedIntent;
import io.quarkmind.sc2.intent.TrainIntent;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Extracts training intents from IEM10 JSON gameEvents for any player.
 *
 * Uses IEM10 2016 build (~39948) abilLink constants — different from AI Arena
 * 2023+ build (~67188). Both the abilLink numbers and abilCmdIndex values changed
 * between patches. See IEM10AbilityDiscoveryTest for derivation evidence.
 *
 * Selection tracking mirrors AbilityMapping: the first element of currentSelection
 * when a Cmd fires is the building tag for the resulting TrainIntent. Building tags
 * are "j-index-recycle" format, matching IEM10JsonSimulatedGame tracker event tags.
 */
public class IEM10CommandExtractor {

    // IEM10 2016 build ~39948 — DO NOT use these for AI Arena replays
    private static final int NEXUS_2016          = 167;   // Nexus: idx=0 → Probe
    private static final int GATEWAY_2016        = 164;   // Gateway: idx=1→Stalker, idx=6→Adept
    private static final int ROBOTICS_2016       = 166;   // Robotics: idx=1→Observer, idx=3→Immortal, idx=18→Disruptor
    private static final int STARGATE_2016       = 165;   // Stargate: idx=0→Phoenix, idx=9→Tempest
    private static final int LARVA_2016          = 185;   // Larva: idx=0→Drone, idx=1→Zergling, idx=9→Roach
    private static final int HATCHERY_2016       = 235;   // Hatchery: idx=0→Queen
    private static final int COMMAND_CENTER_2016 = 147;   // CommandCenter: idx=0→SCV
    private static final int BARRACKS_2016       = 151;   // Barracks: idx=0→Marine

    private IEM10CommandExtractor() {}

    /**
     * Extracts training intents for the watched (Protoss) player.
     */
    public static List<TimedIntent> extract(IEM10JsonSimulatedGame game) {
        return extract(game, game.watchedUserId());
    }

    /**
     * Extracts training intents for any player identified by their gameEvents userId.
     * The userId comes from ToonPlayerDescMap.userID — NOT playerID - 1.
     */
    public static List<TimedIntent> extract(IEM10JsonSimulatedGame game, int userId) {
        List<String> currentSelection = new ArrayList<>();
        List<TimedIntent> intents     = new ArrayList<>();

        for (JsonNode event : game.gameEvents()) {
            String evtType = event.path("evtTypeName").asText();
            int    uid     = event.path("userid").path("userId").asInt(-1);
            if (uid != userId) continue;

            if ("SelectionDelta".equals(evtType)) {
                applySelectionDelta(event, currentSelection);
            } else if ("Cmd".equals(evtType)) {
                if (!isTrainingCommand(event)) continue;
                if (currentSelection.isEmpty()) continue;
                JsonNode abil = event.path("abil");
                if (abil.isMissingNode()) continue;
                int    abilLink     = abil.path("abilLink").asInt(-1);
                int    abilCmdIndex = abil.path("abilCmdIndex").asInt(0);
                long   loop         = event.path("loop").asLong();
                if (abilLink < 0) continue;

                UnitType unitType = dispatch(abilLink, abilCmdIndex);
                if (unitType == null) continue;

                String buildingTag = currentSelection.get(0);
                intents.add(new TimedIntent(loop, new TrainIntent(buildingTag, unitType)));
            }
        }

        return Collections.unmodifiableList(intents);
    }

    // ---- Selection tracking ----

    private static void applySelectionDelta(JsonNode event, List<String> selection) {
        JsonNode delta      = event.path("delta");
        JsonNode removeMask = delta.path("removeMask");

        if (removeMask.has("Mask")) {
            int mask = removeMask.get("Mask").asInt();
            List<String> kept = new ArrayList<>();
            for (int i = 0; i < selection.size(); i++) {
                if ((mask & (1 << i)) == 0) kept.add(selection.get(i));
            }
            selection.clear();
            selection.addAll(kept);
        } else if (removeMask.has("SweepToEnd")) {
            int from = removeMask.get("SweepToEnd").asInt();
            while (selection.size() > from) selection.remove(selection.size() - 1);
        } else if (removeMask.has("OneIndice")) {
            int idx = removeMask.get("OneIndice").asInt();
            if (idx < selection.size()) selection.remove(idx);
        }
        // "None" → nothing removed; add new tags to existing selection

        JsonNode addTags = delta.path("addUnitTags");
        for (JsonNode tagNode : addTags) {
            long packed  = tagNode.asLong();
            int  index   = (int) (packed >> 18);
            int  recycle = (int) (packed & 0x3FFFF);
            String tag   = Sc2ReplayShared.makeTag(index, recycle);
            if (!selection.contains(tag)) selection.add(tag);
        }
    }

    /** Training commands have data: {None: null} — no target point or target unit. */
    private static boolean isTrainingCommand(JsonNode event) {
        JsonNode data = event.path("data");
        if (data.isMissingNode()) return true;
        return data.has("None") && data.size() == 1;
    }

    // ---- Ability dispatch ----

    private static UnitType dispatch(int abilLink, int abilCmdIndex) {
        return switch (abilLink) {
            case NEXUS_2016    -> UnitType.PROBE;
            case GATEWAY_2016  -> switch (abilCmdIndex) {
                case 1  -> UnitType.STALKER;
                case 6  -> UnitType.ADEPT;
                default -> null;
            };
            case ROBOTICS_2016 -> switch (abilCmdIndex) {
                case 1  -> UnitType.OBSERVER;
                case 3  -> UnitType.IMMORTAL;
                case 18 -> UnitType.DISRUPTOR;
                default -> null;
            };
            case STARGATE_2016 -> switch (abilCmdIndex) {
                case 0  -> UnitType.PHOENIX;
                case 9  -> UnitType.TEMPEST;
                default -> null;
            };
            case LARVA_2016    -> switch (abilCmdIndex) {
                case 0  -> UnitType.DRONE;
                case 1  -> UnitType.ZERGLING;
                case 9  -> UnitType.ROACH;
                default -> null;
            };
            case HATCHERY_2016       -> abilCmdIndex == 0 ? UnitType.QUEEN   : null;
            case COMMAND_CENTER_2016 -> abilCmdIndex == 0 ? UnitType.SCV     : null;
            case BARRACKS_2016       -> abilCmdIndex == 0 ? UnitType.MARINE  : null;
            default -> null;
        };
    }
}
```

- [ ] **Step 2.3: Compile**

```bash
mvn compile -q
```

Expected: compiles cleanly. Fix any import errors before continuing.

---

## Task 3: Test `IEM10CommandExtractor`

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/mock/IEM10CommandExtractorTest.java`

- [ ] **Step 3.1: Write the failing tests**

```java
package io.quarkmind.sc2.mock;

import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.intent.TimedIntent;
import io.quarkmind.sc2.intent.TrainIntent;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.Path;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class IEM10CommandExtractorTest {

    private static final Path IEM10_ZIP = Path.of("replays/2016_IEM_10_Taipei.zip");

    // ---- Protoss (watched player) extraction ----

    @Test
    void extractReturnsNonEmptyForFirstGame() throws IOException {
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        List<TimedIntent> intents = IEM10CommandExtractor.extract(game);
        assertThat(intents).isNotEmpty();
    }

    @Test
    void extractedIntentsAreAllTrainIntent() throws IOException {
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        List<TimedIntent> intents = IEM10CommandExtractor.extract(game);
        assertThat(intents).allMatch(ti -> ti.intent() instanceof TrainIntent);
    }

    @Test
    void extractedIntentsHavePositiveLoops() throws IOException {
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        List<TimedIntent> intents = IEM10CommandExtractor.extract(game);
        assertThat(intents).allMatch(ti -> ti.loop() > 0);
    }

    @Test
    void extractedIntentsAreOrderedByLoop() throws IOException {
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        List<TimedIntent> intents = IEM10CommandExtractor.extract(game);
        for (int i = 1; i < intents.size(); i++) {
            assertThat(intents.get(i).loop()).isGreaterThanOrEqualTo(intents.get(i - 1).loop());
        }
    }

    @Test
    void probIntentsHaveNexusBuildingTag() throws IOException {
        // Nexus tags start with "j-" and Probe training always comes from a Nexus
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        List<TimedIntent> intents = IEM10CommandExtractor.extract(game);
        List<TimedIntent> probeTrains = intents.stream()
            .filter(ti -> ((TrainIntent) ti.intent()).unitType() == UnitType.PROBE)
            .toList();
        assertThat(probeTrains).isNotEmpty();
        assertThat(probeTrains).allMatch(
            ti -> ((TrainIntent) ti.intent()).buildingTag().startsWith("j-"),
            "All Probe trains should carry a 'j-' building tag");
    }

    @Test
    void allGamesProduceNonTrivialProbeCount() throws IOException {
        // Guard against abilLink/userId bugs that silently return 0 commands.
        // IEM10 PvX at IEM level: every game has at least 10 Probe trains in first 3 min.
        List<IEM10JsonSimulatedGame> games = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP);
        for (IEM10JsonSimulatedGame game : games) {
            List<TimedIntent> intents = IEM10CommandExtractor.extract(game);
            long probes = intents.stream()
                .filter(ti -> ((TrainIntent) ti.intent()).unitType() == UnitType.PROBE)
                .count();
            assertThat(probes)
                .as("Game %s should have at least 10 Probe trains",
                    game.replayName())
                .isGreaterThanOrEqualTo(10);
        }
    }

    // ---- All-species extraction via explicit userId ----

    @Test
    void extractWithExplicitUserIdReturnsZergIntentsInPvZGame() throws IOException {
        // Find a PvZ game; extract Zerg player's intents using their userID
        List<IEM10JsonSimulatedGame> games = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP);
        IEM10JsonSimulatedGame pvzGame = games.stream()
            .filter(g -> g.matchup().equals("PvZ"))
            .findFirst()
            .orElseThrow(() -> new AssertionError("No PvZ game found"));

        // Zerg userId: iterate all games.json ToonPlayerDescMap to find non-Protoss player.
        // Simplification: use watchedUserId() ≠ approach — instead load the JSON directly.
        // For test isolation, enumerate gives us the game object; the Zerg userId is NOT
        // watchedUserId (which is the Protoss player). We need to extract it a different way.
        //
        // Strategy: try userId values 0..7 and find one that returns Zerg units (Drone/Zergling/Roach).
        List<TimedIntent> zergIntents = List.of();
        for (int uid = 0; uid <= 7; uid++) {
            List<TimedIntent> candidate = IEM10CommandExtractor.extract(pvzGame, uid);
            if (uid == pvzGame.watchedUserId()) continue;  // skip Protoss player
            boolean hasZergUnits = candidate.stream().anyMatch(ti -> {
                UnitType ut = ((TrainIntent) ti.intent()).unitType();
                return ut == UnitType.DRONE || ut == UnitType.ZERGLING || ut == UnitType.ROACH;
            });
            if (hasZergUnits) {
                zergIntents = candidate;
                break;
            }
        }
        assertThat(zergIntents)
            .as("Should find Zerg training intents in PvZ game")
            .isNotEmpty();
    }
}
```

- [ ] **Step 3.2: Run to confirm all tests fail**

```bash
mvn test -Dtest=IEM10CommandExtractorTest -q
```

Expected: compilation error (class not found) or all tests fail. This confirms the tests are in place before the implementation.

- [ ] **Step 3.3: Run tests after Task 2 implementation**

```bash
mvn test -Dtest=IEM10CommandExtractorTest -q
```

Expected: all 7 tests pass. If `allGamesProduceNonTrivialProbeCount` fails for any game, check if that game's Protoss player has a non-standard `watchedUserId` — inspect the game's JSON `ToonPlayerDescMap`.

- [ ] **Step 3.4: Run full test suite (regression check)**

```bash
mvn test -q
```

Expected: all tests pass.

- [ ] **Step 3.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
    src/test/java/io/quarkmind/sc2/mock/IEM10AbilityDiscoveryTest.java \
    src/test/java/io/quarkmind/sc2/mock/IEM10CommandExtractorTest.java \
    src/main/java/io/quarkmind/sc2/mock/IEM10CommandExtractor.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#150): add IEM10CommandExtractor with selection tracking and 2016 ability table"
```

---

## Task 4: Generalise `ReplayValidationHarness`

Add `run(SimulatedGame, List<TimedIntent>, int)` overload. Move all harness logic there.
Existing `run(Path, int, int)` becomes a one-line delegate.

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java`

- [ ] **Step 4.1: Write the failing test first**

The existing `ReplayValidationHarnessTest` already covers the binary-relay path. Write one new test that verifies the general form compiles and is callable:

In `src/test/java/io/quarkmind/sc2/replay/ReplayValidationHarnessTest.java`, add:

```java
@Test
void generalFormAcceptsSimulatedGameArgument() throws IOException {
    // Smoke-test: the general overload exists and delegates correctly.
    // Full IEM10 multi-game validation is in IEM10MultiGameValidationTest (@Tag("report")).
    Path replayPath = Path.of("replays/aiarena_protoss/Nothing_4720936.SC2Replay");
    var game    = new io.quarkmind.sc2.mock.ReplaySimulatedGame(replayPath, 1);
    var intents = io.quarkmind.sc2.replay.ReplayCommandExtractor.extract(replayPath, 1).intents();
    DivergenceReport report = ReplayValidationHarness.run(game, intents, 183);
    assertThat(report).isNotNull();
    assertThat(report.snapshots()).hasSize(183);
}
```

Run to confirm it fails (method not found):
```bash
mvn test -Dtest=ReplayValidationHarnessTest -q
```

- [ ] **Step 4.2: Refactor `ReplayValidationHarness`**

The full current `run(Path, int, int)` implementation becomes `run(SimulatedGame, List<TimedIntent>, int)`.
The `Path`-based overload delegates.

Replace the entire class body with:

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.domain.Building;
import io.quarkmind.domain.GameState;
import io.quarkmind.domain.SC2Data;
import io.quarkmind.sc2.emulated.EmulatedGame;
import io.quarkmind.sc2.intent.TimedIntent;
import io.quarkmind.sc2.mock.ReplaySimulatedGame;
import io.quarkmind.sc2.mock.SimulatedGame;

import java.nio.file.Path;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

/**
 * Validates EmulatedGame economic accuracy against real SC2 replay data.
 * Accepts any SimulatedGame as ground truth — binary SC2Replay via ReplaySimulatedGame
 * or IEM10 JSON via IEM10JsonSimulatedGame.
 *
 * <p>The Path-based overload is a convenience delegate for binary replays.
 */
public final class ReplayValidationHarness {

    private ReplayValidationHarness() {}

    /**
     * General form: accepts any SimulatedGame as ground truth.
     *
     * @param groundTruth tick-based ground truth source (reset() before call is NOT needed —
     *                    this method calls reset() itself)
     * @param intents     training intents extracted for this game's watched player
     * @param tickLimit   maximum ticks to run (use Integer.MAX_VALUE for full replay)
     */
    public static DivergenceReport run(SimulatedGame groundTruth,
                                       List<TimedIntent> intents,
                                       int tickLimit) {
        EmulatedGame emulated = new EmulatedGame();

        groundTruth.reset();
        emulated.reset();

        String label = groundTruth.getClass().getSimpleName();
        assertInitialStateMatch(groundTruth.snapshot(), emulated.snapshot(), label);

        int cursor = 0;
        Set<String> injectedTags = new HashSet<>();
        List<DivergenceReport.TickSnapshot> snapshots =
            new ArrayList<>(Math.min(tickLimit, 10_000));

        for (int tick = 0; tick < tickLimit && !groundTruth.isComplete(); tick++) {
            long windowEnd = (long) (tick + 1) * SC2Data.LOOPS_PER_TICK;

            GameState gtBefore = groundTruth.snapshot();
            emulated.setMiningProbesPerBase(countProbesPerBase(gtBefore));

            emulated.tick();
            groundTruth.tick();

            GameState gt = groundTruth.snapshot();

            syncBuildings(emulated, gt, injectedTags);

            if (gt.supply() > 0)
                emulated.setSupplyCapForHarness(gt.supply());
            emulated.setVespeneForHarness(gtBefore.vespene());

            while (cursor < intents.size() && intents.get(cursor).loop() < windowEnd) {
                emulated.applyIntent(intents.get(cursor));
                cursor++;
            }

            GameState em = emulated.snapshot();
            snapshots.add(new DivergenceReport.TickSnapshot(
                tick,
                gt.myUnits().size(),     em.myUnits().size(),
                gt.myBuildings().size(), em.myBuildings().size(),
                gt.minerals(),           em.minerals(),
                gt.vespene(),            em.vespene()));
        }

        return DivergenceReport.from(snapshots);
    }

    /**
     * Convenience delegate for binary SC2Replay files.
     */
    public static DivergenceReport run(Path replayPath, int playerId, int tickLimit) {
        ReplaySimulatedGame game    = new ReplaySimulatedGame(replayPath, playerId);
        List<TimedIntent>   intents = ReplayCommandExtractor.extract(replayPath, playerId).intents();
        return run(game, intents, tickLimit);
    }

    // ---- Internals (package-private for ReplayValidationHarnessTest) ----

    private static void syncBuildings(EmulatedGame emulated, GameState gt,
                                      Set<String> injectedTags) {
        for (Building gtBuilding : gt.myBuildings()) {
            if (!injectedTags.contains(gtBuilding.tag())) {
                emulated.injectReplayBuilding(gtBuilding);
                injectedTags.add(gtBuilding.tag());
            } else if (gtBuilding.isComplete()) {
                emulated.markReplayBuildingComplete(gtBuilding.tag());
            }
        }
    }

    static int[] countProbesPerBase(GameState state) {
        return EmulatedGame.countProbesPerBase(state.myBuildings(), state.myUnits());
    }

    private static void assertInitialStateMatch(GameState gt, GameState em, String label) {
        if (gt.myUnits().size() != em.myUnits().size()) {
            throw new IllegalStateException(String.format(
                "Initial unit count mismatch for %s — gt: %d, emulated: %d",
                label, gt.myUnits().size(), em.myUnits().size()));
        }
    }
}
```

Note: `SimulatedGame` is in `io.quarkmind.sc2.mock` — add this import.

- [ ] **Step 4.3: Verify both tests pass**

```bash
mvn test -Dtest=ReplayValidationHarnessTest -q
```

Expected: all tests pass (existing + new smoke test).

- [ ] **Step 4.4: Run full test suite**

```bash
mvn test -q
```

Expected: all tests pass.

- [ ] **Step 4.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
    src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java \
    src/test/java/io/quarkmind/sc2/replay/ReplayValidationHarnessTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "refactor(#150): generalise ReplayValidationHarness to accept SimulatedGame"
```

---

## Task 5: Multi-game validation test

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/mock/IEM10MultiGameValidationTest.java`

- [ ] **Step 5.1: Create the test**

```java
package io.quarkmind.sc2.mock;

import io.quarkmind.sc2.intent.TimedIntent;
import io.quarkmind.sc2.replay.DivergenceReport;
import io.quarkmind.sc2.replay.ReplayValidationHarness;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.Path;
import java.util.*;

/**
 * Runs ReplayValidationHarness across all 30 IEM10 games and prints aggregate
 * divergence stats per matchup (PvT, PvZ, PvP).
 *
 * Run with: mvn test -Preport
 * Excluded from default surefire run via excludedGroups=benchmark,browser,report.
 *
 * This is the first multi-game IEM10 validation run — establishes a baseline.
 * Re-run after any change to EmulatedGame physics, IEM10CommandExtractor constants,
 * or ReplayValidationHarness to detect regressions.
 */
@Tag("report")
class IEM10MultiGameValidationTest {

    private static final Path IEM10_ZIP = Path.of("replays/2016_IEM_10_Taipei.zip");

    @Test
    void multiGameDivergenceReport() throws IOException {
        List<IEM10JsonSimulatedGame> games = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP);

        // Per-matchup stats
        Map<String, List<Double>> meanDivByMatchup  = new TreeMap<>();
        Map<String, List<Integer>> maxDivByMatchup  = new TreeMap<>();
        int totalGames = 0;

        for (IEM10JsonSimulatedGame game : games) {
            List<TimedIntent> intents = IEM10CommandExtractor.extract(game);
            DivergenceReport report   = ReplayValidationHarness.run(game, intents, Integer.MAX_VALUE);

            String matchup = game.matchup();
            double meanDiv = report.snapshots().stream()
                .mapToInt(s -> Math.abs(s.gtUnits() - s.emUnits()))
                .average()
                .orElse(0.0);
            int maxDiv = report.snapshots().stream()
                .mapToInt(s -> Math.abs(s.gtUnits() - s.emUnits()))
                .max()
                .orElse(0);

            meanDivByMatchup.computeIfAbsent(matchup, k -> new ArrayList<>()).add(meanDiv);
            maxDivByMatchup.computeIfAbsent(matchup,  k -> new ArrayList<>()).add(maxDiv);
            totalGames++;

            System.out.printf("  [%s] %s  mean=%.1f  max=%d  intents=%d%n",
                matchup, game.replayName().replaceAll(".*/(.*).SC2Replay.json", "$1"),
                meanDiv, maxDiv, intents.size());
        }

        System.out.println("\n=== IEM10 Multi-Game Validation Report ===");
        System.out.printf("  Total games: %d%n%n", totalGames);
        for (String matchup : meanDivByMatchup.keySet()) {
            var means = meanDivByMatchup.get(matchup);
            var maxes = maxDivByMatchup.get(matchup);
            System.out.printf("  [%s]  games=%d  avg_mean=%.1f  avg_max=%.1f%n",
                matchup, means.size(),
                means.stream().mapToDouble(d -> d).average().orElse(0),
                maxes.stream().mapToDouble(d -> d).average().orElse(0));
        }

        // Sanity guard: at least 28 of 30 games must produce non-empty intent streams
        // (a few games may have unusual player selection not yet handled)
        long gamesWithIntents = games.stream()
            .filter(g -> !IEM10CommandExtractor.extract(g).isEmpty())
            .count();
        org.assertj.core.api.Assertions.assertThat(gamesWithIntents)
            .as("At least 28 of 30 IEM10 games should produce non-empty intent streams")
            .isGreaterThanOrEqualTo(28);
    }
}
```

Note: `DivergenceReport.TickSnapshot` accessors (`gtUnits()`, `emUnits()`) — verify the
actual field names in `DivergenceReport.java` before coding. Adjust if they differ.

- [ ] **Step 5.2: Check `DivergenceReport.TickSnapshot` accessor names**

```bash
grep -n "record TickSnapshot\|gtUnits\|emUnits\|myUnits\|gt\b\|em\b" \
    /Users/mdproctor/claude/casehub/quarkmind/src/main/java/io/quarkmind/sc2/replay/DivergenceReport.java | head -20
```

Adjust the accessor calls in Step 5.1 to match the actual field names.

- [ ] **Step 5.3: Run the validation report**

```bash
mvn test -Preport -Dtest=IEM10MultiGameValidationTest -q
```

Expected runtime: ~2–5 minutes (30 full-replay runs).

Expected output: divergence table per matchup. First-run baseline — record the mean/max values
in a comment at the top of the test or in `docs/benchmarks/iem10-validation-baseline.txt`.

- [ ] **Step 5.4: Run full test suite (final regression)**

```bash
mvn test -q
```

Expected: all tests pass.

- [ ] **Step 5.5: Final commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
    src/test/java/io/quarkmind/sc2/mock/IEM10MultiGameValidationTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#150): add IEM10MultiGameValidationTest — closes #150"
```

---

## Self-Review

**Spec coverage:**
- ✅ Parse Cmd events from IEM10 JSON gameEvents — `IEM10CommandExtractor.extract()`
- ✅ Return `List<TimedIntent>` — done, wrapped in `TrainIntent` with real "j-" building tags
- ✅ Multi-game validation test — `IEM10MultiGameValidationTest` @Tag("report")
- ✅ `watchedUserId` vs `playerID-1` gotcha — explicit `ToonPlayerDescMap.userID` extraction
- ✅ IEM10 2016 abilLink table — all-species, narrowly derived
- ✅ `ReplayValidationHarness` generalized — `SimulatedGame` overload, existing path delegates
- ✅ Tag decode formula verified — `index = packed >> 18; recycle = packed & 0x3FFFF`

**Placeholder scan:** None. All code is complete. `DivergenceReport.TickSnapshot` accessor names flagged for verification in Step 5.2.

**Type consistency:** `IEM10CommandExtractor.extract()` returns `List<TimedIntent>` → consumed by `ReplayValidationHarness.run(SimulatedGame, List<TimedIntent>, int)` ✓. `IEM10JsonSimulatedGame` extends `SimulatedGame` ✓.
