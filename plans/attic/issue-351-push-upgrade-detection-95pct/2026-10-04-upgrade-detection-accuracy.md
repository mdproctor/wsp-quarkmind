# Upgrade Detection Accuracy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #351 — P1-2: Push upgrade detection accuracy to ≥95%
**Issue group:** #351 (child of #341)

**Goal:** Push upgrade detection from 87.7% toward 100% across 179
mixed-version replays by adding abilLink overrides for recoverable gaps
and documenting inherent limitations.

**Architecture:** Extend the existing `StrippedReplayValidationTest`
divergence report with per-upgrade-type accuracy metrics. Use the
existing `AbilityDiscoveryCalibrationTest` discovery diagnostics to
identify missing abilLink→UpgradeType mappings. Add overrides to
`AbilityProfile.buildHsc2025Overrides()` (tournament) or
`AbilityMapping` (4.9.3) following established patterns. Assert a
threshold in the validation test.

**Tech Stack:** Java 21, JUnit 5, Scelight replay parsing, AbilityMapping
dispatch framework

## Global Constraints

- Do not modify `StrippedReplayFeatureExtractor` parsing logic
- Do not add/remove `UpgradeType` enum values (all 83 already defined)
- Do not modify oracle replay infrastructure
- Follow existing override patterns in `AbilityProfile` (lambda-based
  `AbilityDispatch`)
- All new overrides require unit tests in `AbilityMappingTest`

---

## Batch 1: Per-upgrade-type accuracy report

### Task 1: Add per-upgrade-type accuracy breakdown to StrippedReplayValidationTest

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayValidationTest.java`

**Interfaces:**
- Consumes: Existing `upgradeDivergence` map (`Map<String, int[]>` keyed
  by `"P" + playerId + ":" + upgradeTypeName`, value `int[2]` =
  `[oracleCount, javaCount]`)
- Produces: Per-type accuracy table printed to stdout; aggregate accuracy
  percentage

The existing test accumulates per-player-per-upgrade divergence but only
prints per-key `✓`/`✗` markers with no aggregate. This task adds a
per-UpgradeType summary section and an overall accuracy figure.

- [ ] **Step 1: Write the per-type aggregation code**

After the existing per-replay loop completes (after the existing upgrade
divergence report section around line 222), add a new section that
aggregates `upgradeDivergence` entries by upgrade type name (stripping
the `"P" + playerId + ":"` prefix) and computes per-type totals:

```java
// --- Per-upgrade-type accuracy summary ---
Map<String, int[]> perTypeTotals = new TreeMap<>();
for (var entry : upgradeDivergence.entrySet()) {
    String upgradeType = entry.getKey().substring(entry.getKey().indexOf(':') + 1);
    int[] counts = perTypeTotals.computeIfAbsent(upgradeType, k -> new int[2]);
    counts[0] += entry.getValue()[0]; // oracle
    counts[1] += entry.getValue()[1]; // java
}

int totalOracle = 0, totalDetected = 0;
System.out.println("\n=== Per-Upgrade-Type Accuracy ===");
System.out.printf("%-45s %6s %6s %6s %8s%n", "UpgradeType", "Oracle", "Java", "Missed", "Accuracy");
System.out.println("-".repeat(75));
for (var entry : perTypeTotals.entrySet()) {
    int oracle = entry.getValue()[0];
    int java = entry.getValue()[1];
    int missed = oracle - java;
    double accuracy = oracle > 0 ? 100.0 * java / oracle : 100.0;
    totalOracle += oracle;
    totalDetected += java;
    String marker = missed == 0 ? "✓" : "✗";
    System.out.printf("%s %-43s %6d %6d %6d %7.1f%%%n",
        marker, entry.getKey(), oracle, java, missed, accuracy);
}
System.out.println("-".repeat(75));
double overallAccuracy = totalOracle > 0 ? 100.0 * totalDetected / totalOracle : 100.0;
System.out.printf("  %-43s %6d %6d %6d %7.1f%%%n",
    "TOTAL", totalOracle, totalDetected, totalOracle - totalDetected, overallAccuracy);
System.out.println("=================================");
```

- [ ] **Step 2: Run the report to verify output and get baseline accuracy**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=StrippedReplayValidationTest -q`

Expected: Test passes. The per-upgrade-type table prints showing current
accuracy (~87.7%). Save the output — it identifies which upgrade types
have misses.

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayValidationTest.java
git commit -m "feat: per-upgrade-type accuracy report in StrippedReplayValidationTest

Adds aggregate breakdown by UpgradeType showing oracle/java/missed/accuracy
per type plus overall accuracy. Foundation for #351 gap analysis.

Refs #351"
```

---

## Batch 2: Discovery, overrides, and gap documentation

### Task 2: Run discovery diagnostics and categorize gaps

This task is diagnostic — it runs existing tests to gather data, then
categorizes each missed upgrade type.

**Files:**
- No code changes — diagnostic runs only

- [ ] **Step 1: Run the upgrade abilLink discovery diagnostic**

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=AbilityDiscoveryCalibrationTest#discoverUpgradeResearchAbilLinks -q`

This correlates oracle `IUpgradeEvent` events with stripped `CmdEvent`
records across 118 oracle replays using a 500–5000 loop window. It prints
abilLink→upgrade mappings discovered from the data.

Save the output — it shows which abilLinks map to which upgrades.

- [ ] **Step 2: Run the frequency-based discovery diagnostic**

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=AbilityDiscoveryCalibrationTest#discoverUpgradeAbilLinksFrequencyBased -q`

This collects ALL CmdEvents within 500–5000 loops of each oracle upgrade
and ranks candidates by frequency. Entries marked `★` (>50% ratio) are
high-confidence candidates.

Save the output — cross-reference with Step 1 to confirm mappings.

- [ ] **Step 3: Cross-reference against existing mappings**

Compare discovered abilLink→upgrade mappings against:
- V4_9_3 constants in `AbilityMapping.java:98-131`
- HSC_2025 overrides in `AbilityProfile.buildHsc2025Overrides()`

Any discovered mapping NOT already in either location is a candidate
override to add in Task 3.

Any upgrade type with misses in the Batch 1 accuracy report but NO
correlating CmdEvent in either discovery diagnostic is an **inherent
gap** — document it in Task 4.

- [ ] **Step 4: Create a categorization table**

Write a temporary working file `docs/upgrade-gap-analysis.md`:

```markdown
# Upgrade Gap Analysis — #351

## Recoverable (abilLink found, override missing)

| UpgradeType | abilLink | abilCmdIndex | Profile | Confidence |
|---|---|---|---|---|
| ... | ... | ... | V4_9_3 or HSC_2025 | high/medium |

## Inherent gaps (no CmdEvent signal)

| UpgradeType | Oracle count | Reason |
|---|---|---|
| ... | ... | No CmdEvent emitted in stripped replay data |

## Already mapped (no action needed)

| UpgradeType | abilLink | Profile |
|---|---|---|
| ... | ... | ... |
```

- [ ] **Step 5: Commit the gap analysis**

```bash
git add docs/upgrade-gap-analysis.md
git commit -m "docs: upgrade gap analysis from discovery diagnostics

Categorizes missed upgrades as recoverable (abilLink found) or inherent
(no CmdEvent signal in stripped data).

Refs #351"
```

### Task 3: Add abilLink overrides for each recoverable upgrade

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityProfile.java`
  (for HSC_2025 tournament overrides)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java`
  (for V4_9_3 direct constants, if any)
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AbilityMappingTest.java`
  (unit tests for each new override)

**Interfaces:**
- Consumes: Gap analysis from Task 2 (recoverable abilLink→upgrade
  mappings)
- Produces: New `AbilityDispatch` entries in the override map; unit tests
  verifying each

For each recoverable upgrade from the Task 2 categorization, follow this
TDD cycle. The patterns below show the exact code shapes to use — repeat
for each discovered mapping.

- [ ] **Step 1: Write failing test for the new override**

For a **building-specific tournament abilLink** (e.g., abilLink=XXX →
SomeUpgrade at idx=0):

```java
@Test
void hsc2025_abilLink_XXX_maps_to_SomeUpgrade() {
    var m = new AbilityMapping(1, true, Race.TERRAN, AbilityProfile.HSC_2025);
    var result = m.process(fakeCmdEvent(XXX, 0, 5000, null, null, 0));
    assertThat(result).hasSize(1);
    assertThat(result.get(0)).isInstanceOf(ReplayCommand.UpgradeCommand.class);
    var uc = (ReplayCommand.UpgradeCommand) result.get(0);
    assertThat(uc.upgradeName()).isEqualTo("SomeUpgrade");
}
```

For a **unitLink-disambiguated generic abilLink** (e.g., abilLink=177,
unitLink=95 CyberneticsCore, idx=N → SomeProtossUpgrade):

```java
@Test
void hsc2025_abilLink177_cyberneticsCore_idx_N_maps_to_SomeProtossUpgrade() {
    var m = new AbilityMapping(1, true, Race.PROTOSS, AbilityProfile.HSC_2025);
    m.onSelection(selectionEvent(0, null, null,
        new int[][]{{95, 1}},    // CyberneticsCore unitLink
        new Integer[]{(1 << 18) | 1}));
    var result = m.process(fakeCmdEvent(177, 0, 5000, null, null, N));
    assertThat(result).hasSize(1);
    assertThat(result.get(0)).isInstanceOf(ReplayCommand.UpgradeCommand.class);
    var uc = (ReplayCommand.UpgradeCommand) result.get(0);
    assertThat(uc.upgradeName()).isEqualTo("SomeProtossUpgrade");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=AbilityMappingTest#hsc2025_abilLink_XXX_maps_to_SomeUpgrade -q`

Expected: FAIL — the abilLink is not yet mapped.

- [ ] **Step 3: Add the override**

For a **building-specific tournament abilLink**, add to
`AbilityProfile.buildHsc2025Overrides()`:

```java
m.put(XXX, (idx, event, loop, race, unitLink) -> {
    if (race != Race.TERRAN) return null;
    return switch (idx) {
        case 0 -> upgradeCmd(loop, "SomeUpgrade");
        default -> null;
    };
});
```

For an **additional idx case on an existing generic abilLink** (e.g.,
adding idx=N to abilLink 177's CyberneticsCore switch), add the case
to the existing `unitLink` switch inside the abilLink 177 dispatch:

```java
// Inside the unitLink==95 (CyberneticsCore) case:
case N -> upgradeCmd(loop, "SomeProtossUpgrade");
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=AbilityMappingTest#hsc2025_abilLink_XXX_maps_to_SomeUpgrade -q`

Expected: PASS

- [ ] **Step 5: Repeat Steps 1-4 for each recoverable upgrade**

Apply the same TDD cycle for every entry in the "Recoverable" section of
`docs/upgrade-gap-analysis.md`. Each override gets its own test method.

- [ ] **Step 6: Run the full accuracy report to confirm improvement**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=StrippedReplayValidationTest -q`

Expected: Accuracy improved significantly from baseline. Note the new
accuracy figure and any remaining misses.

- [ ] **Step 7: Run the full AbilityMappingTest suite**

Run: `mvn test -pl quarkmind-sc2 -Dtest=AbilityMappingTest -q`

Expected: All tests pass (existing + new).

- [ ] **Step 8: Commit all overrides and tests**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityProfile.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AbilityMappingTest.java
git commit -m "feat: add abilLink overrides for recoverable upgrade gaps

Adds N new abilLink→UpgradeType overrides discovered from diagnostic
correlation of oracle tracker events with stripped CmdEvents.
Pushes upgrade detection accuracy from 87.7% to X%.

Refs #351"
```

### Task 4: Document known gaps and enforce accuracy threshold

**Files:**
- Create: `docs/upgrade-detection-gaps.md`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayValidationTest.java`
- Remove: `docs/upgrade-gap-analysis.md` (temporary working file → final
  documentation)

**Interfaces:**
- Consumes: Inherent gap list from Task 2; final accuracy from Task 3
- Produces: Permanent gap documentation; accuracy threshold assertion

- [ ] **Step 1: Write the gap documentation**

Create `docs/upgrade-detection-gaps.md` from the "Inherent gaps" section
of the Task 2 analysis:

```markdown
# Upgrade Detection Gaps

Upgrade types that cannot be detected from stripped replay data.
These are inherent limitations — no CmdEvent is emitted for these
upgrades in the stripped replay format.

## Inherent gaps

| UpgradeType | Oracle occurrences | Root cause |
|---|---|---|
| ... | ... | No CmdEvent emitted in stripped replay data |

## Detection coverage

- **Total UpgradeTypes:** 83
- **Detected:** N (X%)
- **Inherent gaps:** M (Y%)
- **Validation dataset:** 118 oracle replays (patch 4.9.3)
```

- [ ] **Step 2: Add accuracy threshold assertion**

Add to `StrippedReplayValidationTest`, after the per-type accuracy
summary section added in Task 1:

```java
assertTrue(overallAccuracy >= 95.0,
    String.format("Upgrade detection accuracy %.1f%% below 95%% threshold", overallAccuracy));
```

Set the threshold at the achieved accuracy rounded down to the nearest
integer (aspire to 100%, assert at the level actually reached).

- [ ] **Step 3: Run the report to verify the assertion passes**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=StrippedReplayValidationTest -q`

Expected: PASS with accuracy above threshold.

- [ ] **Step 4: Run the full quarkmind-sc2 test suite**

Run: `mvn test -pl quarkmind-sc2 -q`

Expected: All tests pass.

- [ ] **Step 5: Remove temporary working file**

Delete `docs/upgrade-gap-analysis.md` — its content has been promoted
to the permanent `docs/upgrade-detection-gaps.md`.

- [ ] **Step 6: Commit documentation and threshold**

```bash
git add docs/upgrade-detection-gaps.md
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayValidationTest.java
git rm docs/upgrade-gap-analysis.md
git commit -m "feat: document upgrade detection gaps + enforce accuracy threshold

Documents inherent upgrade detection limitations (no CmdEvent signal
in stripped data). Adds accuracy assertion at X% threshold.

Refs #351"
```

---

## References

- [2026-10-04-upgrade-detection-accuracy-design.md] — design spec
- `AbilityProfile.java:50-348` — HSC_2025 override patterns
- `AbilityMapping.java:98-131` — V4_9_3 abilLink constants
- `AbilityMappingTest.java:720-728,1349-1359` — upgrade test patterns
- `AbilityDiscoveryCalibrationTest.java:72-166,445-466` — discovery diagnostics
- `StrippedReplayValidationTest.java:178-246` — divergence report
- `UpgradeType.java` — 83 upgrade enum values
- Protocol `sc2data-train-times-require-calibration.md` — calibration methodology
- GitHub #350 — unitLink-aware building-type disambiguation
- GitHub #338 — initial 78.6% accuracy baseline
- GitHub #341 — parent: Phase 1 feature extraction completeness
