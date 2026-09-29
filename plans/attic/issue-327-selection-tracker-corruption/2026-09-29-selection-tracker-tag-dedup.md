# SelectionUnitLinkTracker Tag-Based Dedup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #327 — Fix SelectionUnitLinkTracker state accumulation causing P2:Marine over-counting
**Issue group:** #327 (part of epic #318)

**Goal:** Rewrite `SelectionUnitLinkTracker` to track individual unit tags alongside unitLinks, eliminating the accumulation bug that causes P2:Marine at 178% of oracle.

**Architecture:** Replace the flat `ArrayList<Integer>` with an ordered `ArrayList<TaggedUnit>` where each entry pairs a Scelight unit tag with its unitLink. RemoveMask operations work on indices (same semantics). AddSubgroups + addUnitTags are zipped positionally; duplicate tags are skipped (dedup). `countMatching()` counts entries whose unitLink matches, bounded by actual in-game units.

**Tech Stack:** Java 21, Scelight s2protocol (SelectionDeltaEvent, Delta, Subgroup APIs)

## Global Constraints

- No Quarkus dependency — `SelectionUnitLinkTracker` is a plain Java inner class
- No changes to `countMatching(Set<Integer>)` signature — callers must not change
- `MAX_MULTIPLICATION = 8` cap in the calling code stays as defence-in-depth
- `BUILDINGCAP_ABIL_LINKS` bypass for Larva/Nexus stays unchanged

---

## Batch 1: Rewrite tracker + unit tests

### Task 1: Rewrite SelectionUnitLinkTracker with tag-based tracking

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java:850-908`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/SelectionUnitLinkTrackerTest.java` (create)

**Interfaces:**
- Consumes: `SelectionDeltaEvent` (Scelight API — `getDelta()`, `Delta.getRemoveMask()`, `Delta.getAddSubgroups()`, `Delta.getAddUnitTags()`)
- Produces: `countMatching(Set<Integer>)` → int (unchanged signature), `unitLinksSnapshot()` → `List<Integer>` (unchanged signature)

- [ ] **Step 1: Write failing test — tag dedup prevents accumulation**

Create `SelectionUnitLinkTrackerTest.java`. The first test simulates two consecutive selection events with `removeMask=None` and the same `addSubgroups` + `addUnitTags` — asserts count stays at the real unit count, not doubled.

```java
package io.quarkmind.sc2.replay;

import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class SelectionUnitLinkTrackerTest {

    private static final int USER_ID = 0;

    @Test
    void duplicateTagsAreNotAccumulated() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        // First selection: 3 Marines (unitLink=70), tags 101,102,103
        tracker.onSelection(selectionDelta(USER_ID, null, null,
            List.of(subgroup(70, 3)),
            new Integer[]{101, 102, 103}));

        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(3);

        // Second selection: same 3 Marines again (removeMask=None, same tags)
        tracker.onSelection(selectionDelta(USER_ID, null, null,
            List.of(subgroup(70, 3)),
            new Integer[]{101, 102, 103}));

        // Without dedup this would be 6; with dedup stays at 3
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(3);
    }

    @Test
    void incrementalAddWithNewTags() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        // Select 2 Marines
        tracker.onSelection(selectionDelta(USER_ID, null, null,
            List.of(subgroup(70, 2)),
            new Integer[]{101, 102}));
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(2);

        // Shift+click adds 1 more Marine (removeMask=None, new tag)
        tracker.onSelection(selectionDelta(USER_ID, null, null,
            List.of(subgroup(70, 1)),
            new Integer[]{103}));
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(3);
    }

    @Test
    void zeroIndicesKeepsOnlyListedPositions() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        // Select 3 units: Marine(70), Tank(75), Marine(70)
        tracker.onSelection(selectionDelta(USER_ID, null, null,
            List.of(subgroup(70, 2), subgroup(75, 1)),
            new Integer[]{101, 102, 201}));
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(2);
        assertThat(tracker.countMatching(Set.of(75))).isEqualTo(1);

        // ZeroIndices: keep indices 0 and 2 (both Marines), drop index 1 (Tank)
        tracker.onSelection(selectionDelta(USER_ID,
            "ZeroIndices", new Integer[]{0, 2},
            List.of(), new Integer[]{}));
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(2);
        assertThat(tracker.countMatching(Set.of(75))).isEqualTo(0);
    }

    @Test
    void oneIndicesRemovesListedPositions() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        tracker.onSelection(selectionDelta(USER_ID, null, null,
            List.of(subgroup(70, 3)),
            new Integer[]{101, 102, 103}));

        // OneIndices: remove index 1
        tracker.onSelection(selectionDelta(USER_ID,
            "OneIndices", new Integer[]{1},
            List.of(), new Integer[]{}));
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(2);
    }

    @Test
    void maskRemovesMarkedPositions() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        tracker.onSelection(selectionDelta(USER_ID, null, null,
            List.of(subgroup(70, 3)),
            new Integer[]{101, 102, 103}));

        // Mask: remove index 1 (bit pattern: 010)
        var bitArray = new hu.belicza.andras.util.type.BitArray(3);
        bitArray.setBit(1);
        tracker.onSelection(selectionDelta(USER_ID,
            "Mask", bitArray,
            List.of(), new Integer[]{}));
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(2);
    }

    @Test
    void nullAddUnitTagsFallsBackToSyntheticTags() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        // addUnitTags is null — tracker should still work
        tracker.onSelection(selectionDelta(USER_ID, null, null,
            List.of(subgroup(70, 2)),
            null));
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(2);
    }

    @Test
    void mixedSubgroupsZipCorrectly() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        // 2 Marines (70) + 1 Tank (75) — tags zip positionally
        tracker.onSelection(selectionDelta(USER_ID, null, null,
            List.of(subgroup(70, 2), subgroup(75, 1)),
            new Integer[]{101, 102, 201}));

        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(2);
        assertThat(tracker.countMatching(Set.of(75))).isEqualTo(1);
        assertThat(tracker.unitLinksSnapshot()).containsExactly(70, 70, 75);
    }

    @Test
    void wrongUserIdIsIgnored() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        tracker.onSelection(selectionDelta(1, null, null,
            List.of(subgroup(70, 3)),
            new Integer[]{101, 102, 103}));
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(0);
    }

    @Test
    void unknownVariantClearsAll() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        tracker.onSelection(selectionDelta(USER_ID, null, null,
            List.of(subgroup(70, 3)),
            new Integer[]{101, 102, 103}));

        // Unknown variant clears
        tracker.onSelection(selectionDelta(USER_ID,
            "SomeNewVariant", new Object(),
            List.of(), new Integer[]{}));
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(0);
    }

    // --- Test helpers ---

    record MockSubgroup(Integer unitLink, Integer count) {
        Integer getUnitLink() { return unitLink; }
        Integer getCount() { return count; }
        Integer getSubgroupPriority() { return 0; }
        Integer getIntraSubgroupPriority() { return 0; }
    }

    static MockSubgroup subgroup(int unitLink, int count) {
        return new MockSubgroup(unitLink, count);
    }

    /**
     * Build a mock SelectionDeltaEvent. This is the hardest part — Scelight's
     * SelectionDeltaEvent constructor requires a struct Map parsed from replay
     * binary data. Instead, build the tracker's onSelection to accept a
     * package-private test entry point, or use a test adapter.
     *
     * The actual implementation will need a test-friendly overload:
     *   void onSelection(int userId, String removeMaskVariant, Object removeMaskValue,
     *                    List<? extends HasUnitLinkAndCount> subgroups, Integer[] unitTags)
     */
    static Object selectionDelta(int userId, String variant, Object value,
                                  List<MockSubgroup> subgroups, Integer[] unitTags) {
        // Placeholder — actual wiring in Step 3
        return null;
    }
}
```

**Note on testability:** Scelight's `SelectionDeltaEvent` requires parsed binary replay data — it can't be constructed from scratch. The tracker needs a test-friendly internal method that accepts decomposed fields rather than the full event. The `onSelection(SelectionDeltaEvent)` method delegates to this internal method. Tests call the internal method directly.

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SelectionUnitLinkTrackerTest -q`
Expected: Compilation error — test helpers and internal API don't exist yet.

- [ ] **Step 3: Implement the rewrite**

In `StrippedReplayFeatureExtractor.java`, replace the `SelectionUnitLinkTracker` inner class (lines 850-908):

```java
static final class SelectionUnitLinkTracker {
    record TaggedUnit(int tag, int unitLink) {}

    private final ArrayList<TaggedUnit> units = new ArrayList<>();
    private final int userId;
    private int syntheticTagCounter = Integer.MIN_VALUE;

    SelectionUnitLinkTracker(int userId) { this.userId = userId; }

    void onSelection(SelectionDeltaEvent sel) {
        if (sel.getUserId() != userId) return;
        var delta = sel.getDelta();
        if (delta == null) {
            units.clear();
            return;
        }

        applyRemoveMask(delta.getRemoveMask());
        addUnits(delta.getAddSubgroups(), delta.getAddUnitTags());
    }

    // Package-private — test entry point that bypasses SelectionDeltaEvent construction
    void applyDelta(String removeMaskVariant, Object removeMaskValue,
                    int[][] subgroups, Integer[] unitTags) {
        if (removeMaskVariant != null || removeMaskValue != null) {
            applyRemoveMaskFields(removeMaskVariant, removeMaskValue);
        }
        addUnitsFromArrays(subgroups, unitTags);
    }

    private void applyRemoveMask(hu.sllauncher.util.Pair<String, Object> removeMask) {
        if (removeMask == null) return;
        applyRemoveMaskFields(removeMask.value1, removeMask.value2);
    }

    private void applyRemoveMaskFields(String variant, Object value) {
        if ("ZeroIndices".equals(variant) && value instanceof Integer[] indices) {
            var kept = new ArrayList<TaggedUnit>();
            for (int idx : indices) {
                if (idx >= 0 && idx < units.size()) kept.add(units.get(idx));
            }
            units.clear();
            units.addAll(kept);
        } else if ("OneIndices".equals(variant) && value instanceof Integer[] indices) {
            for (int i = indices.length - 1; i >= 0; i--) {
                int idx = indices[i];
                if (idx >= 0 && idx < units.size()) units.remove(idx);
            }
        } else if ("Mask".equals(variant)
                   && value instanceof hu.belicza.andras.util.type.BitArray bitArray) {
            for (int i = units.size() - 1; i >= 0; i--) {
                if (i < bitArray.getCount() && bitArray.getBit(i)) units.remove(i);
            }
        } else if (variant != null && !"None".equals(variant)) {
            units.clear();
        }
    }

    private void addUnits(hu.scelight.sc2.rep.model.gameevents.selectiondelta.Subgroup[] subgroups,
                          Integer[] unitTags) {
        if (subgroups == null) return;
        int tagIdx = 0;
        for (var sg : subgroups) {
            Integer link = sg.getUnitLink();
            Integer count = sg.getCount();
            if (link == null || count == null) continue;
            for (int i = 0; i < count; i++) {
                int tag = (unitTags != null && tagIdx < unitTags.length)
                          ? unitTags[tagIdx] : syntheticTagCounter++;
                tagIdx++;
                boolean duplicate = false;
                for (TaggedUnit existing : units) {
                    if (existing.tag == tag) { duplicate = true; break; }
                }
                if (!duplicate) {
                    units.add(new TaggedUnit(tag, link));
                }
            }
        }
    }

    // Test entry point — accepts decomposed arrays instead of Scelight objects
    private void addUnitsFromArrays(int[][] subgroups, Integer[] unitTags) {
        if (subgroups == null) return;
        int tagIdx = 0;
        for (int[] sg : subgroups) {
            int link = sg[0];
            int count = sg[1];
            for (int i = 0; i < count; i++) {
                int tag = (unitTags != null && tagIdx < unitTags.length)
                          ? unitTags[tagIdx] : syntheticTagCounter++;
                tagIdx++;
                boolean duplicate = false;
                for (TaggedUnit existing : units) {
                    if (existing.tag == tag) { duplicate = true; break; }
                }
                if (!duplicate) {
                    units.add(new TaggedUnit(tag, link));
                }
            }
        }
    }

    int countMatching(Set<Integer> validLinks) {
        int count = 0;
        for (TaggedUnit u : units) {
            if (validLinks.contains(u.unitLink)) count++;
        }
        return count;
    }

    List<Integer> unitLinksSnapshot() {
        return units.stream().map(TaggedUnit::unitLink).toList();
    }
}
```

- [ ] **Step 4: Update the test to use the `applyDelta` test entry point**

Replace the placeholder `selectionDelta` helper and rewrite tests to use `tracker.applyDelta(...)`:

```java
package io.quarkmind.sc2.replay;

import org.junit.jupiter.api.Test;

import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class SelectionUnitLinkTrackerTest {

    private static final int USER_ID = 0;

    @Test
    void duplicateTagsAreNotAccumulated() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        tracker.applyDelta(null, null,
            new int[][]{{70, 3}},
            new Integer[]{101, 102, 103});
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(3);

        // Same tags again — should not accumulate
        tracker.applyDelta(null, null,
            new int[][]{{70, 3}},
            new Integer[]{101, 102, 103});
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(3);
    }

    @Test
    void incrementalAddWithNewTags() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        tracker.applyDelta(null, null,
            new int[][]{{70, 2}},
            new Integer[]{101, 102});
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(2);

        // New tag 103 — genuine add
        tracker.applyDelta(null, null,
            new int[][]{{70, 1}},
            new Integer[]{103});
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(3);
    }

    @Test
    void zeroIndicesKeepsOnlyListedPositions() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        tracker.applyDelta(null, null,
            new int[][]{{70, 2}, {75, 1}},
            new Integer[]{101, 102, 201});

        // Keep indices 0 and 2 (Marines), drop index 1 (Tank)
        tracker.applyDelta("ZeroIndices", new Integer[]{0, 2},
            null, null);
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(2);
        assertThat(tracker.countMatching(Set.of(75))).isEqualTo(0);
    }

    @Test
    void oneIndicesRemovesListedPositions() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        tracker.applyDelta(null, null,
            new int[][]{{70, 3}},
            new Integer[]{101, 102, 103});

        tracker.applyDelta("OneIndices", new Integer[]{1},
            null, null);
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(2);
    }

    @Test
    void maskRemovesMarkedPositions() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        tracker.applyDelta(null, null,
            new int[][]{{70, 3}},
            new Integer[]{101, 102, 103});

        var bitArray = new hu.belicza.andras.util.type.BitArray(3);
        bitArray.setBit(1);
        tracker.applyDelta("Mask", bitArray,
            null, null);
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(2);
    }

    @Test
    void nullUnitTagsFallsBackToSyntheticTags() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        tracker.applyDelta(null, null,
            new int[][]{{70, 2}}, null);
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(2);

        // Repeat without tags — synthetic tags differ, so these add
        tracker.applyDelta(null, null,
            new int[][]{{70, 2}}, null);
        // Synthetic tags are unique each call, so this accumulates to 4
        // This is the fallback behavior — no dedup without real tags
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(4);
    }

    @Test
    void mixedSubgroupsZipCorrectly() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        tracker.applyDelta(null, null,
            new int[][]{{70, 2}, {75, 1}},
            new Integer[]{101, 102, 201});
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(2);
        assertThat(tracker.countMatching(Set.of(75))).isEqualTo(1);
        assertThat(tracker.unitLinksSnapshot()).containsExactly(70, 70, 75);
    }

    @Test
    void unknownVariantClearsAll() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        tracker.applyDelta(null, null,
            new int[][]{{70, 3}},
            new Integer[]{101, 102, 103});

        tracker.applyDelta("SomeNewVariant", new Object(),
            null, null);
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(0);
    }

    @Test
    void removeAndAddInSameEvent() {
        var tracker = new StrippedReplayFeatureExtractor.SelectionUnitLinkTracker(USER_ID);

        // Start with 3 Marines
        tracker.applyDelta(null, null,
            new int[][]{{70, 3}},
            new Integer[]{101, 102, 103});

        // Remove index 0, add a Tank
        tracker.applyDelta("OneIndices", new Integer[]{0},
            new int[][]{{75, 1}},
            new Integer[]{201});
        assertThat(tracker.countMatching(Set.of(70))).isEqualTo(2);
        assertThat(tracker.countMatching(Set.of(75))).isEqualTo(1);
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SelectionUnitLinkTrackerTest -q`
Expected: All 9 tests PASS.

- [ ] **Step 6: Run existing unit test suite to check for regressions**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All tests pass. The `onSelection(SelectionDeltaEvent)` method signature is unchanged, so callers in `StrippedReplayFeatureExtractor.extract()` and `MarineMultiplicationDiagnosticTest` should work without modification.

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/SelectionUnitLinkTrackerTest.java
git commit -m "feat: rewrite SelectionUnitLinkTracker with tag-based dedup

Replace flat ArrayList<Integer> with tagged entries that pair each unit's
unique tag (from addUnitTags) with its unitLink (from addSubgroups).
Duplicate tags are skipped on add, preventing the unbounded accumulation
that caused P2:Marine at 178% of oracle.

Refs #327"
```

---

## Batch 2: Diagnostic validation

### Task 2: Run diagnostic tests and verify acceptance criteria

**Files:**
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/MarineMultiplicationDiagnosticTest.java` (existing — run only)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerCorruptionDiagnosticTest.java` (existing — run only)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayValidationTest.java` (existing — run only)

**Interfaces:**
- Consumes: Rewritten `SelectionUnitLinkTracker` from Task 1

- [ ] **Step 1: Run StrippedReplayValidationTest for baseline coverage**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=StrippedReplayValidationTest -q`
Expected: Report prints. Check P1:Marine and P2:Marine aggregate coverage percentages.

Acceptance criteria:
- P2:Marine aggregate between 80-120% (was 178%)
- P1:Marine preserved at >= 90% (was 97.6%)

- [ ] **Step 2: Run MarineMultiplicationDiagnosticTest**

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=MarineMultiplicationDiagnosticTest -q`
Expected: Per-replay table. Check that no replay shows maxMult > 20 for P2 (was 76 in worst case).

- [ ] **Step 3: Run TrackerCorruptionDiagnosticTest**

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=TrackerCorruptionDiagnosticTest -q`
Expected: Selection delta trace for worst replay. unitLink=70 count should stay bounded (max ~12-16, never 76).

- [ ] **Step 4: If acceptance criteria not met, diagnose and iterate**

Read the diagnostic output. If P2:Marine is still above 120%:
- Check whether `addUnitTags` is null for the problematic replays (synthetic tag fallback doesn't dedup)
- If so, add a secondary dedup strategy: when `removeMask=None` and `addUnitTags` is null, clear before adding (treat as full replacement)

If P1:Marine drops below 90%:
- Check whether tag dedup is incorrectly filtering genuine incremental additions
- Verify that distinct CmdEvents produce different addUnitTags (they should — different commands produce different selection states)

- [ ] **Step 5: Run full quarkmind-sc2 test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All tests pass — no regressions.

- [ ] **Step 6: Commit any diagnostic-driven fixes**

If Step 4 required changes:
```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java
git commit -m "fix: handle null addUnitTags in tag-based tracker

When addUnitTags is absent, synthetic tags prevent dedup. Add secondary
strategy: clear-before-add when removeMask=None and no real tags available.

Refs #327"
```

If no changes needed:
```bash
git commit --allow-empty -m "chore: diagnostic validation passed — acceptance criteria met Refs #327"
```

---

## References

- [2026-09-29-selection-tracker-tag-dedup-design.md] — design spec this plan implements
- [StrippedReplayFeatureExtractor.java:850-908] — current SelectionUnitLinkTracker (rewrite target)
- [AbilityMapping.java:214-259] — tag-based selection handling (pattern reference)
- [TrackerCorruptionDiagnosticTest.java] — traces selection growth for worst replay
- [MarineMultiplicationDiagnosticTest.java] — per-replay multiplication analysis
- [StrippedReplayValidationTest.java] — oracle comparison across 118 replays
- [GitHub #327] — focal issue
- [GitHub #318] — parent epic
