# Exact Building Routing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #407 — Exact replay reconstruction via game event building routing
**Issue group:** #407

**Goal:** Route TrackerIntentExtractor's TrainIntents, ResearchIntents, and AbilityIntents to the exact building the player selected, instead of auto-resolving via shortestQueue().

**Architecture:** Pre-process game events (SelectionDeltaEvent + CmdEvent) into a `TreeMap<Integer, String>` mapping CmdEvent loop → selected building tag. When extracting intents from tracker events, look up the back-calculated command loop in the index within a ±5 loop window. Fall back to existing `"r-"` auto-resolution when no match is found (non-oracle replays).

**Tech Stack:** Java 21, JUnit 5, Scelight replay parser

## Global Constraints

- Tags must use `"r-"` prefix per protocol PP-20260528-d9f967
- Extraction logic stays in TrackerIntentExtractor per protocol PP-20260528-612dee
- No API signature changes — `extract(Path, int)` stays the same
- Non-oracle replays must have zero behavior change

---

## Batch 1: Building index and exact routing

### Task 1: Add game event building index and wire into all intent paths

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/TrackerIntentExtractor.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerIntentExtractorIntentTest.java`

**Interfaces:**
- Consumes: `SelectionState` (existing), `GameEventStream.decodeTag(int)` (existing, package-private), `SelectionDeltaEvent.getDelta()`, `Delta.getRemoveMask()`, `Delta.getAddUnitTags()`
- Produces: No new public API — all changes are internal to TrackerIntentExtractor

- [ ] **Step 1: Write the failing test**

Add to `TrackerIntentExtractorIntentTest.java`:

```java
@Test
@EnabledIf("oracleExists")
void trainIntentsUseBuildingTagsFromGameEvents() throws Exception {
    Path replay = Files.list(ORACLE_DIR)
        .filter(p -> p.toString().endsWith(".SC2Replay"))
        .sorted().findFirst().orElseThrow();

    Replay parsed = RepParserEngine.parseReplay(replay,
        EnumSet.of(RepContent.TRACKER_EVENTS, RepContent.DETAILS, RepContent.GAME_EVENTS));

    Set<String> buildingTags = new HashSet<>();
    for (Event raw : parsed.trackerEvents.getEvents()) {
        if (raw.getId() == ITrackerEvents.ID_UNIT_INIT && raw instanceof IBaseUnitEvent ue) {
            Integer pid = ue.getControlPlayerId();
            if (pid != null && pid == 1) {
                String typeName = String.valueOf(ue.getUnitTypeName()).trim();
                if (Sc2ReplayShared.BUILDING_NAMES.contains(typeName)) {
                    buildingTags.add("r-" + ue.getUnitTagIndex() + "-" + ue.getUnitTagRecycle());
                }
            }
        }
    }

    List<TimedIntent> intents = TrackerIntentExtractor.extractFromReplay(parsed, 1);

    long matchingBuildingTag = intents.stream()
        .filter(ti -> ti.intent() instanceof TrainIntent)
        .map(ti -> ((TrainIntent) ti.intent()).buildingTag())
        .filter(buildingTags::contains)
        .count();

    long totalTrain = intents.stream()
        .filter(ti -> ti.intent() instanceof TrainIntent).count();

    System.out.printf("TrainIntents with building tags: %d / %d (%.1f%%)%n",
        matchingBuildingTag, totalTrain,
        totalTrain > 0 ? matchingBuildingTag * 100.0 / totalTrain : 0);

    assertTrue(matchingBuildingTag > 0,
        "With game events, some TrainIntents should have building tags matching actual buildings");
}
```

Imports to add to the test file:

```java
import hu.scelight.sc2.rep.factory.RepContent;
import hu.scelight.sc2.rep.factory.RepParserEngine;
import hu.scelight.sc2.rep.model.Replay;
import hu.scelight.sc2.rep.s2prot.Event;
import hu.scelightapi.sc2.rep.model.trackerevents.IBaseUnitEvent;
import hu.scelightapi.sc2.rep.model.trackerevents.ITrackerEvents;
import io.quarkmind.sc2.mock.Sc2ReplayShared;
import java.util.EnumSet;
import java.util.HashSet;
import java.util.Set;
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TrackerIntentExtractorIntentTest#trainIntentsUseBuildingTagsFromGameEvents -q`
Expected: FAIL — current TrainIntent building tags are unit tracker tags (`"r-{unitTagIndex}-{unitTagRecycle}"`), not building tags, so `matchingBuildingTag` will be 0.

- [ ] **Step 3: Implement buildBuildingIndex**

Add to `TrackerIntentExtractor.java` — new private static method:

```java
private static TreeMap<Integer, String> buildBuildingIndex(Event[] gameEvents, int userId) {
    TreeMap<Integer, String> index = new TreeMap<>();
    SelectionState selection = new SelectionState();

    for (Event raw : gameEvents) {
        if (raw instanceof SelectionDeltaEvent sel) {
            if (sel.getUserId() != userId) continue;
            var delta = sel.getDelta();
            if (delta == null) {
                selection.clear();
                continue;
            }

            var    removeMask = delta.getRemoveMask();
            String variant    = removeMask != null ? removeMask.value1 : null;

            if (variant == null || "None".equals(variant)) {
                // carry forward
            } else if ("ZeroIndices".equals(variant)) {
                if (removeMask.value2 instanceof Integer[] indices) {
                    int[] prims = new int[indices.length];
                    for (int i = 0; i < indices.length; i++) prims[i] = indices[i];
                    selection.retainOnly(prims);
                } else {
                    selection.clear();
                }
            } else if ("Mask".equals(variant)) {
                if (removeMask.value2 instanceof BitArray bitArray) {
                    selection.removeIf(i -> i < bitArray.getCount() && bitArray.getBit(i));
                } else {
                    selection.clear();
                }
            } else if ("OneIndices".equals(variant)) {
                if (removeMask.value2 instanceof Integer[] indices) {
                    for (int i = indices.length - 1; i >= 0; i--) {
                        selection.removeAt(indices[i]);
                    }
                } else {
                    selection.clear();
                }
            } else {
                selection.clear();
            }

            var addTags = delta.getAddUnitTags();
            if (addTags != null) {
                for (Integer rawTag : addTags) {
                    if (rawTag != null) {
                        selection.addTag(GameEventStream.decodeTag(rawTag));
                    }
                }
            }
        } else if (raw instanceof CmdEvent cmd) {
            if (cmd.getUserId() != userId) continue;
            if (!selection.isEmpty()) {
                index.put(cmd.getLoop(), selection.first());
            }
        }
    }

    return index;
}
```

Add private static lookup helper:

```java
private static String lookupBuildingTag(TreeMap<Integer, String> index, int commandLoop) {
    Map.Entry<Integer, String> entry = index.floorEntry(commandLoop + 5);
    if (entry != null && commandLoop - entry.getKey() <= 5) {
        return entry.getValue();
    }
    return null;
}
```

Add imports to `TrackerIntentExtractor.java`:

```java
import hu.belicza.andras.util.type.BitArray;
import hu.scelight.sc2.rep.model.gameevents.selectiondelta.SelectionDeltaEvent;
import io.quarkmind.sc2.SelectionState;
import java.util.Map;
import java.util.TreeMap;
```

- [ ] **Step 4: Wire building index into extractFromReplay**

At the start of `extractFromReplay`, build the index:

```java
static List<TimedIntent> extractFromReplay(Replay replay, int playerId) {
    List<TimedIntent> intents = new ArrayList<>();

    int userId = playerId - 1;
    TreeMap<Integer, String> buildingIndex =
        replay.gameEvents != null
            ? buildBuildingIndex(replay.gameEvents.getEvents(), userId)
            : new TreeMap<>();
```

In the `ID_UNIT_BORN` case, after computing `commandLoop`, replace the tag line:

```java
// Before:
String tag = "r-" + ue.getUnitTagIndex() + "-" + ue.getUnitTagRecycle();

// After:
String resolved = lookupBuildingTag(buildingIndex, commandLoop);
String tag = resolved != null ? resolved
    : "r-" + ue.getUnitTagIndex() + "-" + ue.getUnitTagRecycle();
```

In the `ID_UPGRADE` case, after computing `commandLoop`, replace the tag:

```java
// Before:
intents.add(new TimedIntent(commandLoop,
    new ResearchIntent("r-tracker", ut, commandLoop)));

// After:
String resolvedBuilding = lookupBuildingTag(buildingIndex, commandLoop);
intents.add(new TimedIntent(commandLoop,
    new ResearchIntent(resolvedBuilding != null ? resolvedBuilding : "r-tracker",
        ut, commandLoop)));
```

In the chrono boost game event loop, replace the caster tag:

```java
// Before:
intents.add(new TimedIntent(cmd.getLoop(),
    new AbilityIntent("r-NEXUS", "CHRONO_BOOST", targetTag)));

// After:
String resolvedCaster = lookupBuildingTag(buildingIndex, cmd.getLoop());
intents.add(new TimedIntent(cmd.getLoop(),
    new AbilityIntent(resolvedCaster != null ? resolvedCaster : "r-NEXUS",
        "CHRONO_BOOST", targetTag)));
```

Also move the `int userId = playerId - 1;` line from the chrono boost section (line 100) to the top of the method since it's now used by both the building index and chrono boost loop. Remove the duplicate declaration at line 100.

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TrackerIntentExtractorIntentTest#trainIntentsUseBuildingTagsFromGameEvents -q`
Expected: PASS — TrainIntents now have building tags from game event selection state that match actual building tracker tags.

- [ ] **Step 6: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All existing tests pass. No behavior change for non-oracle paths.

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/TrackerIntentExtractor.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerIntentExtractorIntentTest.java
git commit -m "feat: exact building routing via game event selection tracking

TrackerIntentExtractor now pre-indexes game events (SelectionDeltaEvent +
CmdEvent) to build a TreeMap<loop, buildingTag>. TrainIntent, ResearchIntent,
and AbilityIntent building tags are resolved from the index within a ±5
loop window. Falls back to existing r- auto-resolution for non-oracle replays.

Refs #407"
```

## References

- [2026-10-24-exact-building-routing-design.md] — design spec this plan implements
- [TrackerIntentExtractor.java] — primary modification target
- [AbilityMapping.java:391-465] — SelectionDelta processing pattern replicated
- [GameEventStream.java:34-36] — decodeTag() for tag format
- [SelectionState.java] — tag tracking data structure
- [TrackerIntentExtractorIntentTest.java] — test file extended
- [PP-20260528-d9f967] — replay tag prefix protocol
- [PP-20260528-612dee] — extractor separation protocol
- [GitHub #407] — focal issue
