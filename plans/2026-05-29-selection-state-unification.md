# SelectionState Unification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract a shared `SelectionState` class from two parallel implementations (`AbilityMapping.onSelection` and `IEM10CommandExtractor.applySelectionDelta`) so the core selection-list logic lives and is tested in one place.

**Architecture:** New `SelectionState` in `io.quarkmind.sc2` holds an `ArrayList<String>` of current unit/building tags and exposes six mutation methods plus four accessors. `AbilityMapping` (binary SC2Replay via Scelight) and `IEM10CommandExtractor` (IEM10 JSON via Jackson) each decode their format-specific input and call `SelectionState` primitives. The descending-loop invariant for bit-based removal lives once in `SelectionState.removeIf(IntPredicate)`.

**Tech Stack:** Java 21, JUnit 5, AssertJ, Maven (`mvn test -Dtest=ClassName`). No Quarkus, CDI, or frameworks involved.

---

## File Map

| Action | Path |
|--------|------|
| **Create** | `src/main/java/io/quarkmind/sc2/SelectionState.java` |
| **Create** | `src/test/java/io/quarkmind/sc2/SelectionStateTest.java` |
| **Modify** | `src/main/java/io/quarkmind/sc2/mock/IEM10CommandExtractor.java` |
| **Modify** | `src/test/java/io/quarkmind/sc2/mock/IEM10CommandExtractorSelectionDeltaTest.java` |
| **Modify** | `src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java` |

`AbilityMappingTest` is not touched — `setSelectionForTest(int, List<String>)` keeps its signature.

---

### Task 1: Create SelectionState (TDD)

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/SelectionStateTest.java`
- Create: `src/main/java/io/quarkmind/sc2/SelectionState.java`

- [ ] **Step 1.1 — Write the failing test class**

Create `src/test/java/io/quarkmind/sc2/SelectionStateTest.java`:

```java
package io.quarkmind.sc2;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class SelectionStateTest {

    // --- removeAt ---

    @Test
    void removeAtInBoundsRemovesElement() {
        SelectionState s = SelectionState.of("a", "b", "c");
        s.removeAt(1);
        assertThat(s.snapshot()).containsExactly("a", "c");
    }

    @Test
    void removeAtOutOfBoundsIsNoOp() {
        SelectionState s = SelectionState.of("a", "b");
        s.removeAt(5);
        assertThat(s.snapshot()).containsExactly("a", "b");
    }

    @Test
    void removeAtNegativeIsNoOp() {
        SelectionState s = SelectionState.of("a", "b");
        s.removeAt(-1);
        assertThat(s.snapshot()).containsExactly("a", "b");
    }

    // --- removeIf ---

    @Test
    void removeIfRemovesMatchingIndices() {
        // bits 0 and 2 set (mask=5 in binary) — removes "a" and "c", keeps "b" and "d"
        SelectionState s = SelectionState.of("a", "b", "c", "d");
        s.removeIf(i -> i == 0 || i == 2);
        assertThat(s.snapshot()).containsExactly("b", "d");
    }

    @Test
    void removeIfNoMatchIsNoOp() {
        SelectionState s = SelectionState.of("a", "b");
        s.removeIf(i -> false);
        assertThat(s.snapshot()).containsExactly("a", "b");
    }

    @Test
    void removeIfOnEmptyIsNoOp() {
        SelectionState s = new SelectionState();
        s.removeIf(i -> true);
        assertThat(s.isEmpty()).isTrue();
    }

    // --- truncateTo ---

    @Test
    void truncateToLargerThanSizeIsNoOp() {
        SelectionState s = SelectionState.of("a", "b", "c");
        s.truncateTo(5);
        assertThat(s.snapshot()).containsExactly("a", "b", "c");
    }

    @Test
    void truncateToTruncatesCorrectly() {
        SelectionState s = SelectionState.of("a", "b", "c");
        s.truncateTo(1);
        assertThat(s.snapshot()).containsExactly("a");
    }

    @Test
    void truncateToZeroClearsAll() {
        SelectionState s = SelectionState.of("a", "b");
        s.truncateTo(0);
        assertThat(s.isEmpty()).isTrue();
    }

    // --- retainOnly ---

    @Test
    void retainOnlyKeepsElementsAtGivenIndicesInIndicesArrayOrder() {
        SelectionState s = SelectionState.of("a", "b", "c", "d");
        s.retainOnly(new int[]{3, 1});
        assertThat(s.snapshot()).containsExactly("d", "b");
    }

    @Test
    void retainOnlyWithAscendingIndices() {
        SelectionState s = SelectionState.of("a", "b", "c");
        s.retainOnly(new int[]{0, 2});
        assertThat(s.snapshot()).containsExactly("a", "c");
    }

    @Test
    void retainOnlyEmptyArrayClearsAll() {
        SelectionState s = SelectionState.of("a", "b");
        s.retainOnly(new int[]{});
        assertThat(s.isEmpty()).isTrue();
    }

    @Test
    void retainOnlyOutOfBoundsIndexSilentlyIgnored() {
        SelectionState s = SelectionState.of("a", "b");
        s.retainOnly(new int[]{0, 99});
        assertThat(s.snapshot()).containsExactly("a");
    }

    // --- clear ---

    @Test
    void clearRemovesAllElements() {
        SelectionState s = SelectionState.of("a", "b", "c");
        s.clear();
        assertThat(s.isEmpty()).isTrue();
        assertThat(s.size()).isZero();
    }

    // --- addTag ---

    @Test
    void addTagAppendsNewTag() {
        SelectionState s = new SelectionState();
        s.addTag("x");
        assertThat(s.snapshot()).containsExactly("x");
    }

    @Test
    void addTagIgnoresDuplicate() {
        SelectionState s = SelectionState.of("x");
        s.addTag("x");
        assertThat(s.snapshot()).hasSize(1).containsExactly("x");
    }

    // --- accessors ---

    @Test
    void firstReturnsFirstElement() {
        SelectionState s = SelectionState.of("a", "b", "c");
        assertThat(s.first()).isEqualTo("a");
    }

    @Test
    void sizeReflectsCurrentCount() {
        SelectionState s = SelectionState.of("a", "b", "c");
        assertThat(s.size()).isEqualTo(3);
        s.removeAt(0);
        assertThat(s.size()).isEqualTo(2);
    }

    @Test
    void ofFactoryProducesCorrectInitialState() {
        SelectionState s = SelectionState.of("x", "y", "z");
        assertThat(s.snapshot()).containsExactly("x", "y", "z");
        assertThat(s.size()).isEqualTo(3);
        assertThat(s.isEmpty()).isFalse();
    }

    // --- combinations ---

    @Test
    void removeAfterAddMaintainsOrder() {
        SelectionState s = SelectionState.of("a", "b");
        s.addTag("c");
        s.removeAt(0);
        assertThat(s.snapshot()).containsExactly("b", "c");
    }

    @Test
    void retainThenAddProducesCorrectResult() {
        SelectionState s = SelectionState.of("a", "b", "c");
        s.retainOnly(new int[]{2});
        s.addTag("d");
        assertThat(s.snapshot()).containsExactly("c", "d");
    }
}
```

- [ ] **Step 1.2 — Run the test to confirm it fails (class not found)**

```bash
mvn test -Dtest=SelectionStateTest -q
```

Expected: compile failure — `SelectionState` does not exist.

- [ ] **Step 1.3 — Implement SelectionState**

Create `src/main/java/io/quarkmind/sc2/SelectionState.java`:

```java
package io.quarkmind.sc2;

import java.util.ArrayList;
import java.util.List;
import java.util.function.IntPredicate;

public final class SelectionState {

    private final ArrayList<String> tags = new ArrayList<>();

    /** Test factory — produces an instance pre-populated with the given tags. */
    public static SelectionState of(String... initial) {
        SelectionState s = new SelectionState();
        for (String tag : initial) s.addTag(tag);
        return s;
    }

    /** Remove element at {@code index}; no-op if out of bounds. */
    public void removeAt(int index) {
        if (index >= 0 && index < tags.size()) tags.remove(index);
    }

    /**
     * Iterate descending; remove element at index {@code i} when {@code predicate.test(i)} is true.
     * The descending order prevents index-shift bugs: removing at {@code i} does not affect
     * indices 0..i-1 examined in subsequent iterations.
     */
    public void removeIf(IntPredicate predicate) {
        for (int i = tags.size() - 1; i >= 0; i--) {
            if (predicate.test(i)) tags.remove(i);
        }
    }

    /** Keep first {@code size} elements; no-op if already shorter. */
    public void truncateTo(int size) {
        while (tags.size() > size) tags.remove(tags.size() - 1);
    }

    /**
     * Keep only elements at the given retained indices, in {@code indices} array order.
     * Out-of-bounds indices are silently ignored. Empty array clears all.
     */
    public void retainOnly(int[] indices) {
        List<String> kept = new ArrayList<>(indices.length);
        for (int idx : indices) {
            if (idx >= 0 && idx < tags.size()) kept.add(tags.get(idx));
        }
        tags.clear();
        tags.addAll(kept);
    }

    /** Remove all elements. */
    public void clear() {
        tags.clear();
    }

    /** Add {@code tag} to the end if not already present. */
    public void addTag(String tag) {
        if (!tags.contains(tag)) tags.add(tag);
    }

    public boolean isEmpty() { return tags.isEmpty(); }
    public int size()        { return tags.size(); }

    /** Returns the first element. Behaviour undefined if state is empty — caller must guard. */
    public String first()    { return tags.get(0); }

    /** Returns an immutable snapshot of the current tag list. */
    public List<String> snapshot() { return List.copyOf(tags); }
}
```

- [ ] **Step 1.4 — Run tests and confirm all pass**

```bash
mvn test -Dtest=SelectionStateTest -q
```

Expected: BUILD SUCCESS, 22 tests passing.

- [ ] **Step 1.5 — Commit**

```bash
git add src/main/java/io/quarkmind/sc2/SelectionState.java \
        src/test/java/io/quarkmind/sc2/SelectionStateTest.java
git commit -m "feat(#162): add SelectionState — shared ordered tag list with primitive mutation ops"
```

---

### Task 2: Migrate IEM10CommandExtractor

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/mock/IEM10CommandExtractorSelectionDeltaTest.java`
- Modify: `src/main/java/io/quarkmind/sc2/mock/IEM10CommandExtractor.java`

- [ ] **Step 2.1 — Update the test to use SelectionState (will not compile until Step 2.2)**

Replace the entire content of `src/test/java/io/quarkmind/sc2/mock/IEM10CommandExtractorSelectionDeltaTest.java`:

```java
package io.quarkmind.sc2.mock;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.quarkmind.sc2.SelectionState;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class IEM10CommandExtractorSelectionDeltaTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    // (3L << 18) | 7 = 786439 → decodes to "j-3-7"
    private static final long TAG_3_7 = (3L << 18) | 7;

    private static ObjectNode event(String maskVariant, Integer maskValue, long... addTags) {
        ObjectNode removeMask = MAPPER.createObjectNode();
        if (maskValue == null) {
            removeMask.putNull(maskVariant);
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
        SelectionState sel = SelectionState.of("a", "b", "c", "d");
        IEM10CommandExtractor.applySelectionDelta(event("Mask", 5), sel);
        assertThat(sel.snapshot()).containsExactly("b", "d");
    }

    @Test
    void maskVariantWithZeroMaskRemovesNothing() {
        SelectionState sel = SelectionState.of("a", "b");
        IEM10CommandExtractor.applySelectionDelta(event("Mask", 0), sel);
        assertThat(sel.snapshot()).containsExactly("a", "b");
    }

    @Test
    void sweepToEndTruncatesFromIndex() {
        SelectionState sel = SelectionState.of("a", "b", "c");
        IEM10CommandExtractor.applySelectionDelta(event("SweepToEnd", 1), sel);
        assertThat(sel.snapshot()).containsExactly("a");
    }

    @Test
    void sweepToEndZeroRemovesAll() {
        SelectionState sel = SelectionState.of("a", "b");
        IEM10CommandExtractor.applySelectionDelta(event("SweepToEnd", 0), sel);
        assertThat(sel.snapshot()).isEmpty();
    }

    @Test
    void oneIndiceRemovesSingleItem() {
        SelectionState sel = SelectionState.of("a", "b", "c");
        IEM10CommandExtractor.applySelectionDelta(event("OneIndice", 1), sel);
        assertThat(sel.snapshot()).containsExactly("a", "c");
    }

    @Test
    void oneIndiceOutOfBoundsIsNoOp() {
        SelectionState sel = SelectionState.of("a", "b");
        IEM10CommandExtractor.applySelectionDelta(event("OneIndice", 5), sel);
        assertThat(sel.snapshot()).containsExactly("a", "b");
    }

    @Test
    void noneVariantPreservesExistingSelection() {
        SelectionState sel = SelectionState.of("a", "b");
        IEM10CommandExtractor.applySelectionDelta(event("None", null), sel);
        assertThat(sel.snapshot()).containsExactly("a", "b");
    }

    @Test
    void addUnitTagsDecodesPackedTag() {
        SelectionState sel = new SelectionState();
        IEM10CommandExtractor.applySelectionDelta(event("None", null, TAG_3_7), sel);
        assertThat(sel.snapshot()).containsExactly("j-3-7");
    }

    @Test
    void noneVariantAppendsTagsToExisting() {
        SelectionState sel = SelectionState.of("a");
        IEM10CommandExtractor.applySelectionDelta(event("None", null, TAG_3_7), sel);
        assertThat(sel.snapshot()).containsExactly("a", "j-3-7");
    }

    @Test
    void duplicateTagNotAdded() {
        SelectionState sel = new SelectionState();
        IEM10CommandExtractor.applySelectionDelta(event("None", null, TAG_3_7, TAG_3_7), sel);
        assertThat(sel.snapshot()).hasSize(1).containsExactly("j-3-7");
    }
}
```

- [ ] **Step 2.2 — Update IEM10CommandExtractor production code**

Replace the entire content of `src/main/java/io/quarkmind/sc2/mock/IEM10CommandExtractor.java`:

```java
package io.quarkmind.sc2.mock;

import com.fasterxml.jackson.databind.JsonNode;
import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.SelectionState;
import io.quarkmind.sc2.intent.TimedIntent;
import io.quarkmind.sc2.intent.TrainIntent;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

// Note: tag format "j-index-recycle" mirrors IEM10JsonSimulatedGame.makeTag (package-private)
// Do NOT use Sc2ReplayShared.makeTag here — that produces "r-" prefix for .SC2Replay binary parser

/**
 * Extracts training intents from IEM10 JSON gameEvents for any player.
 *
 * Uses IEM10 2016 build (~39948) abilLink constants — different from AI Arena
 * 2023+ build (~67188). Both abilLink numbers and abilCmdIndex values changed
 * between patches. See IEM10AbilityDiscoveryTest for derivation evidence.
 *
 * Selection tracking delegates to SelectionState. The first element of the current
 * selection when a Cmd fires is the building tag for the resulting TrainIntent.
 * Building tags are "j-index-recycle" format, matching IEM10JsonSimulatedGame tracker event tags.
 */
public class IEM10CommandExtractor {

    // IEM10 2016 build ~39948 — DO NOT use for AI Arena replays
    private static final int NEXUS_2016          = 167;
    private static final int GATEWAY_2016        = 164;
    private static final int ROBOTICS_2016       = 166;
    private static final int STARGATE_2016       = 165;
    private static final int LARVA_2016          = 185;
    private static final int HATCHERY_2016       = 235;
    private static final int COMMAND_CENTER_2016 = 147;
    private static final int BARRACKS_2016       = 151;

    private IEM10CommandExtractor() {}

    /** Extracts training intents for the watched (Protoss) player. */
    public static List<TimedIntent> extract(IEM10JsonSimulatedGame game) {
        return extract(game, game.watchedUserId());
    }

    /**
     * Extracts training intents for any player identified by their gameEvents userId.
     * The userId comes from ToonPlayerDescMap.userID — NOT playerID - 1.
     */
    public static List<TimedIntent> extract(IEM10JsonSimulatedGame game, int userId) {
        SelectionState currentSelection = new SelectionState();
        List<TimedIntent> intents       = new ArrayList<>();

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
                int  abilLink     = abil.path("abilLink").asInt(-1);
                int  abilCmdIndex = abil.path("abilCmdIndex").asInt(0);
                long loop         = event.path("loop").asLong();
                if (abilLink < 0) continue;

                UnitType unitType = dispatch(abilLink, abilCmdIndex);
                if (unitType == null) continue;

                String buildingTag = currentSelection.first();
                intents.add(new TimedIntent(loop, new TrainIntent(buildingTag, unitType)));
            }
        }

        return Collections.unmodifiableList(intents);
    }

    static void applySelectionDelta(JsonNode event, SelectionState selection) {
        JsonNode delta      = event.path("delta");
        JsonNode removeMask = delta.path("removeMask");

        if (removeMask.has("Mask")) {
            int mask = removeMask.get("Mask").asInt();
            selection.removeIf(i -> (mask & (1 << i)) != 0);
        } else if (removeMask.has("SweepToEnd")) {
            int from = removeMask.get("SweepToEnd").asInt();
            selection.truncateTo(from);
        } else if (removeMask.has("OneIndice")) {
            int idx = removeMask.get("OneIndice").asInt();
            selection.removeAt(idx);
        }
        // "None" or absent removeMask → carry forward (no removal)

        JsonNode addTags = delta.path("addUnitTags");
        for (JsonNode tagNode : addTags) {
            long packed  = tagNode.asLong();
            int  index   = (int) (packed >> 18);
            int  recycle = (int) (packed & 0x3FFFF);
            selection.addTag("j-" + index + "-" + recycle);
        }
    }

    /** Training commands have data: {None: null} — no target point or target unit. */
    private static boolean isTrainingCommand(JsonNode event) {
        JsonNode data = event.path("data");
        if (data.isMissingNode()) return true;
        return data.has("None") && data.size() == 1;
    }

    // Units not mapped: Zealot and WarpPrism were not resolved in the narrow-window
    // correlation (IEM10AbilityDiscoveryTest) — signal too weak vs Probe noise.
    // Terran Factory/Starport units beyond SCV/Marine similarly unresolved.
    // See issue #160 for follow-up.
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
            case HATCHERY_2016       -> abilCmdIndex == 0 ? UnitType.QUEEN  : null;
            case COMMAND_CENTER_2016 -> abilCmdIndex == 0 ? UnitType.SCV    : null;
            case BARRACKS_2016       -> abilCmdIndex == 0 ? UnitType.MARINE : null;
            default -> null;
        };
    }
}
```

- [ ] **Step 2.3 — Run IEM10 tests and confirm all 10 pass**

```bash
mvn test -Dtest="IEM10CommandExtractorSelectionDeltaTest,IEM10CommandExtractorTest" -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 2.4 — Run full suite to catch any regressions**

```bash
mvn test -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 2.5 — Commit**

```bash
git add src/main/java/io/quarkmind/sc2/mock/IEM10CommandExtractor.java \
        src/test/java/io/quarkmind/sc2/mock/IEM10CommandExtractorSelectionDeltaTest.java
git commit -m "refactor(#162): migrate IEM10CommandExtractor to SelectionState"
```

---

### Task 3: Migrate AbilityMapping

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java`

- [ ] **Step 3.1 — Replace AbilityMapping with SelectionState-based implementation**

Replace the entire content of `src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java`:

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.model.gameevents.cmd.CmdEvent;
import hu.scelight.sc2.rep.model.gameevents.selectiondelta.SelectionDeltaEvent;
import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.SelectionState;
import io.quarkmind.sc2.intent.TimedIntent;
import io.quarkmind.sc2.intent.TrainIntent;
import org.jboss.logging.Logger;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Objects;

/**
 * Stateful SC2 command interpreter scoped to one player.
 * Owns selection state and maps CmdEvents to ReplayCommands.
 *
 * <p>Ability IDs discovered via AbilityDiscoveryTest from aiarena_protoss PvZ replays.
 * Note: Build commands (probe placing structures) use abilLink=42 (Smart) in bot play,
 * indistinguishable from movement orders. BuildIntent extraction is not attempted here;
 * buildings are synchronised from ground truth in ReplayValidationHarness.
 *
 * <p>Discovered ability table (Nothing_4720936.SC2Replay PvZ):
 * <pre>
 *   userId=0 (Protoss): 42=Smart/Move, 175=TrainProbe, 172=GatewayTrain(idx1=Zealot,idx0=Stalker)
 *   userId=0 (Protoss): 173=RoboticsTrain(idx0=Immortal), 170=WarpGateTrain(hasTP=location)
 *   userId=1 (Zerg):    42=Smart/Move, 193=LarvaTrain(variousIdx), 184=Queen/Inject
 * </pre>
 * Terran coverage added in issue #140 (ABIL_COMMAND_CENTER=155 → SCV, ABIL_BARRACKS=159 → Marine/Marauder).
 */
public class AbilityMapping {

    private static final Logger log = Logger.getLogger(AbilityMapping.class);

    // --- Movement ---
    private static final int ABIL_SMART       = 42;  // Smart/RightClick — move, attack, harvest
    private static final int ABIL_ATTACK_MOVE = 45;  // Attack-move command

    // --- Protoss train ---
    private static final int ABIL_GATEWAY     = 172; // Gateway normal train (abilCmdIndex selects unit)
    private static final int ABIL_ROBOTICS    = 173; // Robotics Facility train
    private static final int ABIL_NEXUS       = 175; // Nexus train (Probe)
    private static final int ABIL_STARGATE    = 174; // Stargate train

    // WarpGate warp-in — abilLink=170 with hasTP=true; treated as movement (location-targeted)
    private static final int ABIL_WARPGATE    = 170;

    // --- Zerg train (from larva, abilLink=193, abilCmdIndex selects unit) ---
    private static final int ABIL_LARVA       = 193;
    // Queen is trained from Hatchery via abilLink=184 abilCmdIndex=1; other indices are macro (inject)
    private static final int ABIL_HATCHERY    = 184;

    // --- Terran train (AI Arena build 75689) ---
    // Derived from TerranDiscoveryTest: no-target Cmd events cross-referenced across
    // Nothing_4720935 (18m), Tyckles_4721034 (15m), Starlight_4721165 (6m) Terran-wins PvT.
    // Other abilLinks (157, 158, 161) have insufficient cross-replay evidence — logged as unknown.
    private static final int ABIL_COMMAND_CENTER = 155; // idx=0 only → SCV
    private static final int ABIL_BARRACKS       = 159; // idx=0 → Marine, idx=3 → Marauder

    private static final Map<Integer, UnitType> BARRACKS_UNITS = Map.of(
            0, UnitType.MARINE,
            3, UnitType.MARAUDER
    );

    // Gateway abilCmdIndex → UnitType (from discovery: idx1=Zealot most common, idx0=Stalker)
    private static final Map<Integer, UnitType> GATEWAY_UNITS = Map.of(
            0, UnitType.STALKER,
            1, UnitType.ZEALOT,
            5, UnitType.ADEPT
    );

    // Robotics abilCmdIndex → UnitType
    private static final Map<Integer, UnitType> ROBOTICS_UNITS = Map.of(
            0, UnitType.IMMORTAL,
            1, UnitType.OBSERVER,
            2, UnitType.COLOSSUS
    );

    // Stargate abilCmdIndex → UnitType
    private static final Map<Integer, UnitType> STARGATE_UNITS = Map.of(
            0, UnitType.PHOENIX,
            1, UnitType.VOID_RAY,
            2, UnitType.ORACLE
    );

    // Zerg larva abilCmdIndex → UnitType
    private static final Map<Integer, UnitType> LARVA_UNITS = Map.of(
            0, UnitType.DRONE,
            1, UnitType.ZERGLING,
            2, UnitType.ROACH,
            3, UnitType.HYDRALISK,
            4, UnitType.MUTALISK,
            5, UnitType.CORRUPTOR,
            6, UnitType.ULTRALISK,
            7, UnitType.INFESTOR,
            8, UnitType.SWARM_HOST,
            9, UnitType.VIPER
    );

    private final int userId;  // 0-indexed game event userId = (playerId - 1)
    private final SelectionState selection = new SelectionState();

    public AbilityMapping(int playerId) {
        this.userId = playerId - 1;
    }

    public void onSelection(SelectionDeltaEvent event) {
        if (event.getUserId() != userId) return;
        var delta = event.getDelta();
        if (delta == null) {
            selection.clear();
            return;
        }

        var removeMask = delta.getRemoveMask();
        String variant = removeMask != null ? removeMask.value1 : null;

        if (variant == null || "None".equals(variant)) {
            // carry forward — no removal
        } else if ("ZeroIndices".equals(variant)) {
            if (removeMask.value2 instanceof Integer[] indices) {
                selection.retainOnly(toPrimitiveInts(indices));
            } else {
                selection.clear();
            }
        } else if ("Mask".equals(variant)) {
            if (removeMask.value2 instanceof hu.belicza.andras.util.type.BitArray bitArray) {
                selection.removeIf(i -> i < bitArray.getCount() && bitArray.getBit(i));
            } else {
                log.debugf("[SELECTION] Mask variant has unexpected payload type: %s", removeMask.value2);
            }
        } else if ("OneIndices".equals(variant)) {
            if (removeMask.value2 instanceof Integer[] indices) {
                for (int i = indices.length - 1; i >= 0; i--) {
                    selection.removeAt(indices[i]);
                }
            } else {
                log.debugf("[SELECTION] OneIndices variant has unexpected payload type: %s", removeMask.value2);
            }
        } else {
            log.debugf("[SELECTION] Unknown removeMask variant '%s' — treating as full clear", variant);
            selection.clear();
        }

        if (delta.getAddUnitTags() != null) {
            for (Integer rawTag : delta.getAddUnitTags()) {
                if (rawTag != null) {
                    selection.addTag(GameEventStream.decodeTag(rawTag));
                }
            }
        }
    }

    public List<ReplayCommand> process(CmdEvent event) {
        if (event.getUserId() != userId) return List.of();
        if (selection.isEmpty()) return List.of();
        Integer abilLink = event.getAbilLink();
        if (abilLink == null) return List.of();
        int idx = Objects.requireNonNullElse(event.getAbilCmdIndex(), 0);
        return dispatch(abilLink, idx, event);
    }

    public void reset() {
        selection.clear();
    }

    /** Package-private — used by AbilityMappingTest to prime selection without replay parsing. */
    void setSelectionForTest(int forUserId, List<String> tags) {
        if (forUserId == this.userId) {
            selection.clear();
            tags.forEach(selection::addTag);
        }
    }

    private List<ReplayCommand> dispatch(int abilLink, int idx, CmdEvent event) {
        long loop = event.getLoop();

        return switch (abilLink) {
            case ABIL_SMART, ABIL_ATTACK_MOVE, ABIL_WARPGATE ->
                    moveOrders(event, loop);

            case ABIL_NEXUS ->
                    trainIntent(loop, UnitType.PROBE);

            case ABIL_GATEWAY -> {
                UnitType unit = GATEWAY_UNITS.get(idx);
                yield unit != null ? trainIntent(loop, unit) : unknown(abilLink, idx);
            }

            case ABIL_ROBOTICS -> {
                UnitType unit = ROBOTICS_UNITS.get(idx);
                yield unit != null ? trainIntent(loop, unit) : unknown(abilLink, idx);
            }

            case ABIL_STARGATE -> {
                UnitType unit = STARGATE_UNITS.get(idx);
                yield unit != null ? trainIntent(loop, unit) : unknown(abilLink, idx);
            }

            case ABIL_LARVA -> {
                UnitType unit = LARVA_UNITS.get(idx);
                yield unit != null ? trainIntent(loop, unit) : unknown(abilLink, idx);
            }

            case ABIL_HATCHERY ->
                    idx == 1 ? trainIntent(loop, UnitType.QUEEN) : List.of();

            case ABIL_COMMAND_CENTER ->
                    idx == 0 ? trainIntent(loop, UnitType.SCV) : unknown(abilLink, idx);

            case ABIL_BARRACKS -> {
                UnitType unit = BARRACKS_UNITS.get(idx);
                yield unit != null ? trainIntent(loop, unit) : unknown(abilLink, idx);
            }

            default -> unknown(abilLink, idx);
        };
    }

    private List<ReplayCommand> trainIntent(long loop, UnitType unitType) {
        String buildingTag = selection.first();
        return List.of(new ReplayCommand.IntentCommand(
                new TimedIntent(loop, new TrainIntent(buildingTag, unitType))));
    }

    private List<ReplayCommand> moveOrders(CmdEvent event, long loop) {
        var tp = event.getTargetPoint();
        var tu = event.getTargetUnit();
        List<ReplayCommand> orders = new ArrayList<>(selection.size());
        for (String tag : selection.snapshot()) {
            if (tp != null) {
                float x = tp.getXFloat();
                float y = tp.getYFloat();
                if (x >= 0 && x <= 256 && y >= 0 && y <= 256) {
                    orders.add(new ReplayCommand.Movement(
                            new UnitOrder(tag, loop, new Point2d(x, y), null)));
                }
            } else if (tu != null && tu.getTag() != null) {
                orders.add(new ReplayCommand.Movement(
                        new UnitOrder(tag, loop, null, GameEventStream.decodeTag(tu.getTag()))));
            } else {
                log.debugf("[ABILITY] Move cmd at loop %d has no target — skipped for %s", loop, tag);
            }
        }
        return orders;
    }

    private List<ReplayCommand> unknown(int abilLink, int idx) {
        log.debugf("[ABILITY] Unknown abilLink=%d abilCmdIndex=%d — skipped", abilLink, idx);
        return List.of();
    }

    private static int[] toPrimitiveInts(Integer[] boxed) {
        int[] result = new int[boxed.length];
        for (int i = 0; i < boxed.length; i++) result[i] = boxed[i];
        return result;
    }
}
```

- [ ] **Step 3.2 — Run AbilityMapping tests**

```bash
mvn test -Dtest="AbilityMappingTest,TerranReplayCommandExtractorTest" -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 3.3 — Run full test suite**

```bash
mvn test -q
```

Expected: BUILD SUCCESS. All tests that passed before this branch must still pass.

- [ ] **Step 3.4 — Commit**

```bash
git add src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java
git commit -m "refactor(#162): migrate AbilityMapping to SelectionState"
```

---

### Task 4: Final verification

- [ ] **Step 4.1 — Run full suite one final time**

```bash
mvn test -q
```

Expected: BUILD SUCCESS with no failures or errors.

- [ ] **Step 4.2 — Spot-check intent extraction integration**

Run the replay-based integration tests to verify that selection state is being applied correctly end-to-end:

```bash
mvn test -Dtest="ReplayCommandExtractorTest,TerranReplayCommandExtractorTest,IEM10CommandExtractorTest" -q
```

Expected: BUILD SUCCESS.
