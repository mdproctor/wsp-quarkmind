# removeMask Tests, isComplete(), and Terran AbilityMapping — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add focused unit tests for `IEM10CommandExtractor.applySelectionDelta`, add `SimulatedGame.isComplete()` base-contract test, fix `AbilityMapping.onSelection` removeMask handling, and add Terran ability IDs to `AbilityMapping` with three-layer tests.

**Architecture:** Tasks flow in dependency order: #160 (visibility + unit tests) → #161 (isComplete test) → #140A (integration test written to fail) → #140B (onSelection fix) → #140C (Terran constants + unit tests → integration test passes). The integration test is written before the fix so it acts as a failing spec.

**Tech Stack:** Java 21, JUnit 5, AssertJ, Jackson `ObjectMapper` for JsonNode test fixtures, Scelight `SelectionDeltaEvent.getDelta().getRemoveMask()` returning `Pair<String,Object>`.

---

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `src/main/java/io/quarkmind/sc2/mock/IEM10CommandExtractor.java` | Modify | Change `applySelectionDelta` visibility |
| `src/test/java/io/quarkmind/sc2/mock/IEM10CommandExtractorSelectionDeltaTest.java` | **Create** | 10 unit tests for removeMask variants |
| `src/test/java/io/quarkmind/sc2/mock/SimulatedGameTest.java` | Modify | Add `isCompleteReturnsFalseByDefault` |
| `src/test/java/io/quarkmind/sc2/replay/TerranReplayCommandExtractorTest.java` | **Create** | Integration test (written first, initially fails) |
| `src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java` | Modify | Fix `onSelection` removeMask + add Terran constants |
| `src/test/java/io/quarkmind/sc2/replay/AbilityMappingTest.java` | Modify | Add Terran unit tests |
| `src/test/java/io/quarkmind/sc2/replay/AbilityDiscoveryTest.java` | Modify | Add PvT replays, fix stale javadoc |
| `src/test/java/io/quarkmind/sc2/replay/TerranDiscoveryTest.java` | **Delete** | Temporary diagnostic no longer needed |
| `CLAUDE.md` | Modify | Add two new test classes to inventory |

---

## Task 1: #160 — Write failing tests for applySelectionDelta

The method `IEM10CommandExtractor.applySelectionDelta` is currently `private static`. Tests in the same package cannot see `private` members. Write the tests first so they fail for the right reason (access), then change visibility.

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/mock/IEM10CommandExtractorSelectionDeltaTest.java`

- [ ] **Step 1.1: Create the test class**

```java
package io.quarkmind.sc2.mock;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class IEM10CommandExtractorSelectionDeltaTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    // packed = (index << 18) | recycle → "j-index-recycle"
    // (3 << 18) | 7 = 786439 → "j-3-7"
    // (5 << 18) | 1 = 1310721 → "j-5-1"

    private static ObjectNode event(String maskVariant, Integer maskValue, long... addTags) {
        ObjectNode removeMask = MAPPER.createObjectNode();
        if (maskValue == null) {
            removeMask.putNull(maskVariant);  // {"None": null}
        } else {
            removeMask.put(maskVariant, maskValue);
        }
        ArrayNode addUnitTags = MAPPER.createArrayNode();
        for (long tag : addTags) addUnitTags.add(tag);
        ObjectNode delta = MAPPER.createObjectNode();
        delta.set("removeMask", removeMask);
        delta.set("addUnitTags", addUnitTags);
        ObjectNode evt = MAPPER.createObjectNode();
        evt.set("delta", delta);
        return evt;
    }

    @Test
    void maskVariantRemovesCorrectBitPositions() {
        // mask=5=0b0101 → remove indices 0 and 2
        List<String> sel = new ArrayList<>(List.of("a", "b", "c", "d"));
        IEM10CommandExtractor.applySelectionDelta(event("Mask", 5), sel);
        assertThat(sel).containsExactly("b", "d");
    }

    @Test
    void maskVariantWithZeroMaskRemovesNothing() {
        List<String> sel = new ArrayList<>(List.of("a", "b"));
        IEM10CommandExtractor.applySelectionDelta(event("Mask", 0), sel);
        assertThat(sel).containsExactly("a", "b");
    }

    @Test
    void sweepToEndTruncatesFromIndex() {
        List<String> sel = new ArrayList<>(List.of("a", "b", "c"));
        IEM10CommandExtractor.applySelectionDelta(event("SweepToEnd", 1), sel);
        assertThat(sel).containsExactly("a");
    }

    @Test
    void sweepToEndZeroRemovesAll() {
        List<String> sel = new ArrayList<>(List.of("a", "b"));
        IEM10CommandExtractor.applySelectionDelta(event("SweepToEnd", 0), sel);
        assertThat(sel).isEmpty();
    }

    @Test
    void oneIndiceRemovesSingleItem() {
        List<String> sel = new ArrayList<>(List.of("a", "b", "c"));
        IEM10CommandExtractor.applySelectionDelta(event("OneIndice", 1), sel);
        assertThat(sel).containsExactly("a", "c");
    }

    @Test
    void oneIndiceOutOfBoundsIsNoOp() {
        List<String> sel = new ArrayList<>(List.of("a", "b"));
        IEM10CommandExtractor.applySelectionDelta(event("OneIndice", 5), sel);
        assertThat(sel).containsExactly("a", "b");
    }

    @Test
    void noneVariantPreservesExistingSelection() {
        List<String> sel = new ArrayList<>(List.of("a", "b"));
        IEM10CommandExtractor.applySelectionDelta(event("None", null), sel);
        assertThat(sel).containsExactly("a", "b");
    }

    @Test
    void addUnitTagsDecodesPackedTag() {
        // (3 << 18) | 7 = 786439 → "j-3-7"
        List<String> sel = new ArrayList<>();
        IEM10CommandExtractor.applySelectionDelta(event("None", null, 786439L), sel);
        assertThat(sel).containsExactly("j-3-7");
    }

    @Test
    void noneVariantAppendsTagsToExisting() {
        List<String> sel = new ArrayList<>(List.of("a"));
        IEM10CommandExtractor.applySelectionDelta(event("None", null, 786439L), sel);
        assertThat(sel).containsExactly("a", "j-3-7");
    }

    @Test
    void duplicateTagNotAdded() {
        List<String> sel = new ArrayList<>();
        IEM10CommandExtractor.applySelectionDelta(event("None", null, 786439L, 786439L), sel);
        assertThat(sel).hasSize(1).containsExactly("j-3-7");
    }
}
```

- [ ] **Step 1.2: Run tests — verify they fail to compile (private access)**

```bash
mvn test -Dtest=IEM10CommandExtractorSelectionDeltaTest -q 2>&1 | head -20
```

Expected: compilation error — `applySelectionDelta` has private access.

- [ ] **Step 1.3: Change visibility in IEM10CommandExtractor**

In `src/main/java/io/quarkmind/sc2/mock/IEM10CommandExtractor.java`, line ~81, change:

```java
private static void applySelectionDelta(JsonNode event, List<String> selection) {
```

to:

```java
static void applySelectionDelta(JsonNode event, List<String> selection) {
```

(Remove `private`. Keep `static`.)

- [ ] **Step 1.4: Run tests — verify all 10 pass**

```bash
mvn test -Dtest=IEM10CommandExtractorSelectionDeltaTest -q
```

Expected: `BUILD SUCCESS` with 10 tests passing.

- [ ] **Step 1.5: Run the full IEM10CommandExtractor suite to confirm no regressions**

```bash
mvn test -Dtest="IEM10CommandExtractorSelectionDeltaTest,IEM10CommandExtractorTest" -q
```

Expected: `BUILD SUCCESS`.

---

## Task 2: #161 — Add isComplete() base-contract test

`SimulatedGame.isComplete()` already returns `false` — this test just documents the contract. It will pass immediately after being written.

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/mock/SimulatedGameTest.java`

- [ ] **Step 2.1: Add test to SimulatedGameTest**

Append inside the `SimulatedGameTest` class (after the last existing `@Test` method, before the closing `}`):

```java
    @Test
    void isCompleteReturnsFalseByDefault() {
        assertThat(game.isComplete()).isFalse();
    }
```

The `game` field is already declared and reset in `@BeforeEach`.

- [ ] **Step 2.2: Run to verify it passes**

```bash
mvn test -Dtest=SimulatedGameTest -q
```

Expected: `BUILD SUCCESS`.

---

## Task 3: #140A — Write TerranReplayCommandExtractorTest (initially failing)

Write the full integration test now, before fixing `AbilityMapping`. It will fail because there are no Terran constants and `onSelection` has bugs — this is the failing spec that Tasks 4–5 will satisfy.

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/replay/TerranReplayCommandExtractorTest.java`

Player mapping: `Nothing_4720935.SC2Replay` — Nothing=Protoss (player 1, userId 0), RustyNikolaj=Terran (player 2, userId 1). Pass `playerId=2` to `ReplayCommandExtractor.extract()`. Inside `AbilityMapping`, `userId = playerId - 1 = 1`.

- [ ] **Step 3.1: Create the test class**

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.domain.SC2Data;
import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.intent.TrainIntent;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestInstance;

import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class TerranReplayCommandExtractorTest {

    // Nothing=Protoss(player1, userId0), RustyNikolaj=Terran(player2, userId1)
    static final Path REPLAY =
        Path.of("replays/aiarena_protoss/Nothing_4720935.SC2Replay");

    ReplayCommandStream stream;

    @BeforeAll
    void extract() {
        stream = ReplayCommandExtractor.extract(REPLAY, 2); // player 2 → Terran
    }

    @Test
    void extractsNonEmptyIntentListForTerranPlayer() {
        assertThat(stream.intents()).isNotEmpty();
    }

    @Test
    void extractsNonEmptyMovementOrdersForTerranPlayer() {
        assertThat(stream.movementOrders()).isNotEmpty();
    }

    @Test
    void intentsIncludeScvTrain() {
        assertThat(stream.intents()).anyMatch(ti ->
            ti.intent() instanceof TrainIntent t && t.unitType() == UnitType.SCV);
    }

    @Test
    void intentsIncludeMarineTrain() {
        assertThat(stream.intents()).anyMatch(ti ->
            ti.intent() instanceof TrainIntent t && t.unitType() == UnitType.MARINE);
    }

    @Test
    void intentsAreOrderedByLoop() {
        var intents = stream.intents();
        for (int i = 1; i < intents.size(); i++) {
            assertThat(intents.get(i).loop())
                .isGreaterThanOrEqualTo(intents.get(i - 1).loop());
        }
    }

    @Test
    void allIntentLoopsAreNonNegative() {
        assertThat(stream.intents()).allMatch(ti -> ti.loop() >= 0);
    }

    @Test
    void movementOrdersAreOrderedByLoop() {
        var orders = stream.movementOrders();
        for (int i = 1; i < orders.size(); i++) {
            assertThat(orders.get(i).loop())
                .isGreaterThanOrEqualTo(orders.get(i - 1).loop());
        }
    }

    @Test
    void allMovementOrderTagsMatchTrackerFormat() {
        assertThat(stream.movementOrders())
            .allMatch(o -> o.unitTag().startsWith("r-"));
    }

    @Test
    void trainIntentsReferenceKnownUnitTypes() {
        stream.intents().stream()
            .filter(ti -> ti.intent() instanceof TrainIntent)
            .map(ti -> (TrainIntent) ti.intent())
            .forEach(t -> assertThat(SC2Data.trainTimeInTicks(t.unitType()))
                .as("trainTimeInTicks must be > 0 for %s", t.unitType())
                .isGreaterThan(0));
    }

    @Test
    void listsAreUnmodifiable() {
        assertThat(stream.movementOrders()).isUnmodifiable();
        assertThat(stream.intents()).isUnmodifiable();
    }
}
```

- [ ] **Step 3.2: Run — verify it fails (no Terran constants)**

```bash
mvn test -Dtest=TerranReplayCommandExtractorTest -q 2>&1 | head -30
```

Expected: `FAIL` — `extractsNonEmptyIntentListForTerranPlayer` fails (intents empty because AbilityMapping has no Terran dispatch cases). Movement assertions may pass. SCV/Marine assertions definitely fail.

---

## Task 4: #140B — Fix AbilityMapping.onSelection removeMask handling

`AbilityMapping.onSelection` currently replaces `currentSelection` wholesale with `addUnitTags` regardless of removeMask variant. The `Mask` variant appears ~1,800 times in the Nothing_4720935 human replay — building tags will be wrong without this fix.

**Variants from empirical discovery on PvT replays:**
- `ZeroIndices` (dominant: 56K–109K/replay) — wholesale replacement; current behavior is correct
- `Mask` (1,787–1,998 in human replays) — bitmask: bit i set → remove index i
- `None` (1,613–1,834 in human replays) — add without removing
- `SweepToEnd`, `OneIndice` — zero occurrences found, handle for correctness
- `null` — treat as ZeroIndices (full replacement)

**`getRemoveMask()` API:** `delta.getRemoveMask()` returns `Pair<String, Object>` from `hu.sllauncher.util.Pair`. Access via `pair.value1` (String variant name) and `pair.value2` (Object payload). For Mask: `((Number) pair.value2).intValue()` gives the bitmask.

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java`

- [ ] **Step 4.1: Replace the body of `onSelection` in AbilityMapping**

Find the method (currently around line 97):
```java
public void onSelection(SelectionDeltaEvent event) {
    if (event.getUserId() != userId) return;
    var delta = event.getDelta();
    if (delta == null || delta.getAddUnitTags() == null) {
        currentSelection = List.of();
        return;
    }
    currentSelection = java.util.Arrays.stream(delta.getAddUnitTags())
            .filter(Objects::nonNull)
            .map(GameEventStream::decodeTag)
            .toList();
}
```

Replace with:
```java
public void onSelection(SelectionDeltaEvent event) {
    if (event.getUserId() != userId) return;
    var delta = event.getDelta();
    if (delta == null) {
        currentSelection = List.of();
        return;
    }

    var removeMask = delta.getRemoveMask();
    String variant = removeMask != null ? removeMask.value1 : null;
    boolean fullReplacement = variant == null || "ZeroIndices".equals(variant);

    List<String> result = fullReplacement ? new ArrayList<>() : new ArrayList<>(currentSelection);

    if ("Mask".equals(variant)) {
        int mask = ((Number) removeMask.value2).intValue();
        for (int i = result.size() - 1; i >= 0; i--) {
            if ((mask & (1 << i)) != 0) result.remove(i);
        }
    } else if ("SweepToEnd".equals(variant)) {
        int from = ((Number) removeMask.value2).intValue();
        while (result.size() > from) result.remove(result.size() - 1);
    } else if ("OneIndice".equals(variant)) {
        int idx = ((Number) removeMask.value2).intValue();
        if (idx < result.size()) result.remove(idx);
    }
    // "None": no removal — result starts as copy of currentSelection, addUnitTags appended below

    if (delta.getAddUnitTags() != null) {
        for (Integer rawTag : delta.getAddUnitTags()) {
            if (rawTag != null) {
                String tag = GameEventStream.decodeTag(rawTag);
                if (!result.contains(tag)) result.add(tag);
            }
        }
    }

    currentSelection = List.copyOf(result);
}
```

- [ ] **Step 4.2: Verify the new method compiles**

```bash
mvn compile -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 4.3: Run existing AbilityMappingTest to confirm no regressions**

```bash
mvn test -Dtest=AbilityMappingTest -q
```

Expected: `BUILD SUCCESS` — all existing tests pass. (Existing tests use `setSelectionForTest()` to bypass `onSelection`, so this validates the `process()` path is unchanged.)

- [ ] **Step 4.4: Run TerranReplayCommandExtractorTest — still fails but differently**

```bash
mvn test -Dtest=TerranReplayCommandExtractorTest -q 2>&1 | head -20
```

Expected: still `FAIL` on SCV/Marine assertions (no constants yet), but movement assertions should pass and the error is purely "no matching intents" rather than exceptions.

---

## Task 5: #140C — Add Terran constants, unit tests, update discovery, delete diagnostic

This task adds the Terran ability constants to `AbilityMapping`, adds unit tests to `AbilityMappingTest`, updates `AbilityDiscoveryTest`, and deletes the temporary `TerranDiscoveryTest`.

**Terran abilLink constants derived from TerranDiscoveryTest (run 2026-05-29 on Nothing_4720935/Tyckles_4721034/Starlight_4721165):**

No-target train commands cross-referenced across three Terran-wins PvT replays:
- `155, idx=0` — appears in all 4 replays (61–92 counts); single index only → **SCV** (Command Center, single-unit building)
- `159, idx=0` — appears in all 4 replays; dominant in 6m early game → **Marine** (Barracks)
- `159, idx=3` — appears in 2 replays in mid-game only (24–41 counts) → **Marauder** (Barracks + Tech Lab)

All other abilLinks (157, 158, 161, 112, 131, etc.) have insufficient or conflicting cross-replay evidence. They go through `unknown()` and are logged at DEBUG.

**Subtask 5A: Write failing unit tests first**

- [ ] **Step 5.1: Add Terran constants and unit tests to AbilityMappingTest**

Add after the existing constants block (before `AbilityMapping mapping;`):
```java
    // Terran constants — AI Arena build 75689 (derived from TerranDiscoveryTest 2026-05-29)
    static final int TERRAN_CC_ABIL       = 155; // Command Center → SCV (idx=0)
    static final int TERRAN_BARRACKS_ABIL = 159; // Barracks → Marine (idx=0), Marauder (idx=3)
    static final int TERRAN_MARINE_IDX    = 0;
    static final int TERRAN_MARAUDER_IDX  = 3;
```

Add after the existing test methods (before closing `}`):
```java
    @Test
    void trainScvProducesSingleTrainIntent() {
        mapping.setSelectionForTest(0, List.of("r-cc-1"));
        List<ReplayCommand> result = mapping.process(
                fakeCmdEvent(TERRAN_CC_ABIL, 0, 500, null, null, 0));
        assertThat(result).hasSize(1);
        ReplayCommand.IntentCommand ic = (ReplayCommand.IntentCommand) result.get(0);
        assertThat(ic.intent().loop()).isEqualTo(500);
        TrainIntent t = (TrainIntent) ic.intent().intent();
        assertThat(t.unitType()).isEqualTo(UnitType.SCV);
        assertThat(t.buildingTag()).isEqualTo("r-cc-1");
    }

    @Test
    void trainMarineProducesSingleTrainIntent() {
        mapping.setSelectionForTest(0, List.of("r-bx-1"));
        List<ReplayCommand> result = mapping.process(
                fakeCmdEvent(TERRAN_BARRACKS_ABIL, 0, 600, null, null, TERRAN_MARINE_IDX));
        assertThat(result).hasSize(1);
        TrainIntent t = (TrainIntent) ((ReplayCommand.IntentCommand) result.get(0)).intent().intent();
        assertThat(t.unitType()).isEqualTo(UnitType.MARINE);
    }

    @Test
    void trainMarauderProducesSingleTrainIntent() {
        mapping.setSelectionForTest(0, List.of("r-bx-2"));
        List<ReplayCommand> result = mapping.process(
                fakeCmdEvent(TERRAN_BARRACKS_ABIL, 0, 700, null, null, TERRAN_MARAUDER_IDX));
        assertThat(result).hasSize(1);
        TrainIntent t = (TrainIntent) ((ReplayCommand.IntentCommand) result.get(0)).intent().intent();
        assertThat(t.unitType()).isEqualTo(UnitType.MARAUDER);
    }

    @Test
    void unknownBarracksIndexReturnsEmpty() {
        mapping.setSelectionForTest(0, List.of("r-bx-3"));
        List<ReplayCommand> result = mapping.process(
                fakeCmdEvent(TERRAN_BARRACKS_ABIL, 0, 800, null, null, 99));
        assertThat(result).isEmpty();
    }
```

- [ ] **Step 5.2: Run AbilityMappingTest — verify new Terran tests fail**

```bash
mvn test -Dtest=AbilityMappingTest -q 2>&1 | grep -E "FAIL|Tests run"
```

Expected: FAIL on the 4 new Terran tests.

**Subtask 5B: Add Terran constants to AbilityMapping**

- [ ] **Step 5.3: Add Terran constants to AbilityMapping**

In `src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java`, after the Zerg constants block (before `private final int userId`), add:

```java
    // --- Terran train (AI Arena build 75689) ---
    // Derived from TerranDiscoveryTest: no-target Cmd events, cross-referenced across
    // Nothing_4720935 (18m), Tyckles_4721034 (15m), Starlight_4721165 (6m) Terran-wins PvT.
    // Other abilLinks (157, 158, 161) have insufficient cross-replay evidence — logged as unknown.
    private static final int ABIL_COMMAND_CENTER = 155; // idx=0 only → SCV
    private static final int ABIL_BARRACKS       = 159; // idx=0 → Marine, idx=3 → Marauder

    private static final Map<Integer, UnitType> BARRACKS_UNITS = Map.of(
            0, UnitType.MARINE,
            3, UnitType.MARAUDER
    );
```

- [ ] **Step 5.4: Add Terran cases to `dispatch()` in AbilityMapping**

In the `dispatch()` method's switch expression, after the `ABIL_HATCHERY` case (before `default`), add:

```java
            case ABIL_COMMAND_CENTER ->
                    trainIntent(loop, UnitType.SCV);

            case ABIL_BARRACKS -> {
                UnitType unit = BARRACKS_UNITS.get(idx);
                yield unit != null ? trainIntent(loop, unit) : unknown(abilLink, idx);
            }
```

- [ ] **Step 5.5: Run AbilityMappingTest — verify all tests pass**

```bash
mvn test -Dtest=AbilityMappingTest -q
```

Expected: `BUILD SUCCESS`.

**Subtask 5C: Update AbilityDiscoveryTest, delete TerranDiscoveryTest**

- [ ] **Step 5.6: Update AbilityDiscoveryTest — add PvT replays and fix javadoc**

Replace the class javadoc:
```java
/**
 * Scans replays and prints all observed (abilLink, abilCmdIndex) tuples with counts.
 * No assertions — output populates AbilityMapping's static table.
 * Covers Protoss (userId=0) and Zerg (userId=1) from PvZ aiarena replays.
 * Terran coverage requires .SC2Replay files — see issue #140.
 */
```
with:
```java
/**
 * Scans replays and prints all observed (abilLink, abilCmdIndex) tuples with counts.
 * No assertions — output populates AbilityMapping's static table.
 *
 * <p>Coverage by replay:
 * <ul>
 *   <li>Nothing_4720936 (PvZ): Protoss userId=0, Zerg userId=1</li>
 *   <li>ArgoBot_4721229 (PvT): Protoss userId=1, Terran userId=0 (bot opponent)</li>
 *   <li>Nothing_4720935 (PvT): Protoss userId=0, Terran userId=1 (RustyNikolaj)</li>
 *   <li>Tyckles_4721034 (PvT): Protoss userId=0, Terran userId=1 (RustyNikolaj)</li>
 * </ul>
 */
```

Replace `replayFiles()`:
```java
    static Stream<Path> replayFiles() {
        return Stream.of(
            Path.of("replays/aiarena_protoss/Nothing_4720936.SC2Replay"),  // PvZ
            Path.of("replays/aiarena_protoss/ArgoBot_4721229.SC2Replay"),  // PvT Terran=userId=0
            Path.of("replays/aiarena_protoss/Nothing_4720935.SC2Replay"),  // PvT Terran=userId=1
            Path.of("replays/aiarena_protoss/Tyckles_4721034.SC2Replay")   // PvT Terran=userId=1
        );
    }
```

- [ ] **Step 5.7: Delete TerranDiscoveryTest**

```bash
rm src/test/java/io/quarkmind/sc2/replay/TerranDiscoveryTest.java
```

- [ ] **Step 5.8: Run AbilityDiscoveryTest to confirm it still works with the new files**

```bash
mvn test -Dtest=AbilityDiscoveryTest -q 2>&1 | grep -E "BUILD|Tests run|ERROR"
```

Expected: `BUILD SUCCESS` (no assertions — it always passes if replays parse).

**Subtask 5D: Verify the integration test now passes**

- [ ] **Step 5.9: Run TerranReplayCommandExtractorTest — verify all 10 tests pass**

```bash
mvn test -Dtest=TerranReplayCommandExtractorTest -q
```

Expected: `BUILD SUCCESS` with all 10 tests passing. If any assertion fails:
- `extractsNonEmptyIntentListForTerranPlayer` or SCV/Marine tests fail → Terran constants wrong; re-examine TerranDiscoveryTest output for userId=1 no-target commands.
- `allMovementOrderTagsMatchTrackerFormat` fails → tag decoding bug in `onSelection`.
- `intentsAreOrderedByLoop` fails → sorting bug in `ReplayCommandExtractor` (not introduced here, file a bug).

---

## Task 6: Update CLAUDE.md test inventory

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 6.1: Add new test classes to the unit test inventory**

In `CLAUDE.md`, find the line in the **Unit tests** section that lists test class names. It currently includes `IEM10CommandExtractorTest`. Add the two new classes:

Find:
```
`IEM10CommandExtractorTest`,
```
Add after it (keeping alphabetical order within the IEM10 group):
```
`IEM10CommandExtractorSelectionDeltaTest`,
```

Find the `ReplayCommandExtractorTest` entry and add after it:
```
`TerranReplayCommandExtractorTest`,
```

- [ ] **Step 6.2: Verify the CLAUDE.md diff looks right**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind diff CLAUDE.md | grep "^[+-]" | grep -v "^---\|^+++"
```

Expected: two added lines (IEM10CommandExtractorSelectionDeltaTest and TerranReplayCommandExtractorTest).

---

## Task 7: Full test suite, code review, commit

- [ ] **Step 7.1: Run the full default test suite**

```bash
mvn test -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`. If any test fails, diagnose before proceeding.

- [ ] **Step 7.2: Invoke superpowers:requesting-code-review**

Required before any commit. Any finding Minor or above that is not fixed in this session must be captured as a GitHub issue.

- [ ] **Step 7.3: Commit all changes**

Stage files:
```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/sc2/mock/IEM10CommandExtractor.java \
  src/test/java/io/quarkmind/sc2/mock/IEM10CommandExtractorSelectionDeltaTest.java \
  src/test/java/io/quarkmind/sc2/mock/SimulatedGameTest.java \
  src/test/java/io/quarkmind/sc2/replay/TerranReplayCommandExtractorTest.java \
  src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java \
  src/test/java/io/quarkmind/sc2/replay/AbilityMappingTest.java \
  src/test/java/io/quarkmind/sc2/replay/AbilityDiscoveryTest.java \
  CLAUDE.md

git -C /Users/mdproctor/claude/casehub/quarkmind rm src/test/java/io/quarkmind/sc2/replay/TerranDiscoveryTest.java
```

Commit:
```bash
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "$(cat <<'EOF'
feat(#160,#161,#140): removeMask unit tests, isComplete, Terran AbilityMapping

#160: IEM10CommandExtractor.applySelectionDelta made package-private;
IEM10CommandExtractorSelectionDeltaTest covers all 4 removeMask variants
(Mask bitmask removal, SweepToEnd truncation, OneIndice single removal, None
preservation) plus addUnitTags packed-tag decoding and duplicate suppression.

#161: isCompleteReturnsFalseByDefault added to SimulatedGameTest.

#140: AbilityMapping.onSelection fixed to handle Mask/None/SweepToEnd/OneIndice
variants from Scelight Delta.getRemoveMask() (Pair<String,Object> API); ZeroIndices
keeps wholesale-replacement behaviour. Terran abilLink constants added for AI Arena
build 75689: ABIL_COMMAND_CENTER=155 (SCV), ABIL_BARRACKS=159 (Marine idx=0,
Marauder idx=3). TerranReplayCommandExtractorTest validates against
Nothing_4720935.SC2Replay (player 2, Terran). AbilityDiscoveryTest extended with
PvT replays; stale javadoc fixed.

Refs #140, Closes #160, Closes #161
EOF
)"
```

---

## Task 8: implementation-doc-sync

- [ ] **Step 8.1: Invoke implementation-doc-sync**

Required after committing. Scoped to what changed this session.

---

## Self-Review

**Spec coverage check:**
- #160: visibility change ✓ (Task 1.3), all 10 tests ✓ (Task 1.1), CLAUDE.md ✓ (Task 6)
- #161: isComplete test ✓ (Task 2), CLAUDE.md note (bca77f2 already done) ✓
- #140 discovery: TerranDiscoveryTest deletion ✓ (Task 5.7), AbilityDiscoveryTest PvT extension ✓ (Task 5.6), javadoc fix ✓ (Task 5.6)
- #140 onSelection fix: Mask ✓, None ✓, SweepToEnd ✓, OneIndice ✓, ZeroIndices ✓ (Task 4)
- #140 constants: SCV ✓, Marine ✓, Marauder ✓ (Task 5.3–5.4)
- #140 unit tests: trainSCV ✓, trainMarine ✓, trainMarauder ✓, unknownBarracksIdx ✓ (Task 5.1)
- #140 integration: TerranReplayCommandExtractorTest, all 10 assertions ✓ (Task 3), passes after fix ✓ (Task 5.9)
- CLAUDE.md: IEM10CommandExtractorSelectionDeltaTest ✓, TerranReplayCommandExtractorTest ✓ (Task 6)
- Code review ✓ (Task 7.2), commit ✓ (Task 7.3), doc-sync ✓ (Task 8)

**Placeholder scan:** No TBDs, no "implement later", all code blocks are complete. Test commands show exact expected output.

**Type consistency:** `ReplayCommandStream`, `TrainIntent`, `UnitType.SCV/MARINE/MARAUDER` all used consistently. `fakeCmdEvent()` signature is consistent with existing AbilityMappingTest usage. `TERRAN_CC_ABIL=155`, `TERRAN_BARRACKS_ABIL=159` used identically in constants block and tests.
