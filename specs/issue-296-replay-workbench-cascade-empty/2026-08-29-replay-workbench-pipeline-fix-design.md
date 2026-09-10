# Replay Workbench Pipeline Fix — Cascade Classifier + Layered Testing

**Issue:** casehubio/quarkmind#296
**Decisions:** D1–D3 in `decisions.md`

## 1. Problem

The replay workbench shows no data across all four tabs (Pattern, Coaching, Strategy, Commentary). The cascade classifier returns empty assessments because:

1. Hand-authored Drools rules cover early-game rushes only (< 4–5 min). Counter-indication rules actively decay them after 5 min.
2. Mid-game generic rules depend on YAML taxonomy `SignatureSpec` facts (weight=0.5 per unit type). Real replay compositions often don't satisfy `minCount` thresholds for any single archetype cleanly.
3. ONNX tier has no models configured for the replay profile.
4. LLM fallback is async and requires a `ChatModel` bean.

Without `PatternAssessmentPublished` events, the entire downstream chain — strategy selection, summarisation, moment detection, commentary — produces nothing.

## 2. Fix: COMPOSITION_UNKNOWN Fallback + Weight Tuning

### 2a. New archetype: COMPOSITION_UNKNOWN

Add to `StrategyArchetype`:

```java
COMPOSITION_UNKNOWN(Race.ZERG, GamePhase.MID, ArchetypeCategory.COMPOSITION),
// Note: race is set dynamically — the enum value is a placeholder.
// The actual PatternAssessment carries the detected enemy race context.
```

Actually, `StrategyArchetype` enum values have fixed race. Since the fallback can trigger for any enemy race, add three race-specific unknowns:

```java
// Fallback — cascade returned empty with enemies visible
TERRAN_COMPOSITION_UNKNOWN(Race.TERRAN, GamePhase.MID, ArchetypeCategory.COMPOSITION),
ZERG_COMPOSITION_UNKNOWN(Race.ZERG, GamePhase.MID, ArchetypeCategory.COMPOSITION),
PROTOSS_COMPOSITION_UNKNOWN(Race.PROTOSS, GamePhase.MID, ArchetypeCategory.COMPOSITION),
```

These are real `StrategyArchetype` values. Downstream consumers see them as low-confidence assessments — no special-casing needed (D3).

### 2b. Cascade fallback logic

In `CascadingPatternClassifier.classify()`, after all three tiers produce nothing above `DISPATCH_THRESHOLD`:

```java
// After line 192 — before the final return
List<PatternAssessment> assessments = allAssessments(cumulativeConfidence, frame, AssessmentSource.DROOLS);
if (assessments.isEmpty() && enemyRace != null && hasVisibleEnemies(evidence)) {
    StrategyArchetype unknown = unknownForRace(enemyRace);
    assessments = List.of(new PatternAssessment(
        unknown, FALLBACK_CONFIDENCE, frame,
        "Enemies visible but no archetype matched — composition unrecognised",
        AssessmentSource.DROOLS));
}
return new CascadeResult(assessments, llmTriggered);
```

`FALLBACK_CONFIDENCE = 0.35` — just above `DISPATCH_THRESHOLD` (0.3). This is a constant, not configurable — the fallback is a safety net with intentionally low influence.

`enemyCount > 0` is the fallback gate. Add an `int enemyCount` parameter to `classify()` from the scouting task, which already has `enemies.size()`. Evidence markers alone are not a reliable signal — expansion events can produce markers with no enemies visible.

### 2c. Generic rule weight tuning

The current hardcoded weight of 0.5 per signature unit type (in `StrategyTaxonomy.parseSignature()` line 151) is too conservative. With two matching unit types, the multiplicative combination gives `1 - (1-0.5)*(1-0.5) = 0.75` — barely above the 0.7 drools threshold. With one matching type, 0.5 — below threshold.

**Change:** Read weight from the YAML taxonomy per unit spec, with a default of 0.5. Allow the taxonomy author to tune weights per unit type per archetype:

```yaml
ZERG_ROACH_HYDRA:
  signature:
    units:
      - type: ROACH
        minCount: 3
        weight: 0.55
      - type: HYDRALISK
        minCount: 2
        weight: 0.55
```

Update `StrategyTaxonomy.parseSignature()`:

```java
double weight = unitSpec.containsKey("weight")
    ? ((Number) unitSpec.get("weight")).doubleValue()
    : 0.5;
```

This makes tuning a YAML edit, not a code change. The tests assert ranges against replay data — tune, run calibration test, repeat.

## 3. Testing Architecture: Recorded Event Replay

### 3a. Principle

Each pipeline layer has a well-defined input/output contract. Tests work at two levels:

1. **Isolation** — replay committed fixtures through a single layer, assert output
2. **Composition** — incrementally stack layers (L1, L1+L2, L1+L2+L3, ...) using the output of each layer as input to the next

Fixtures are captured from real replay runs and committed to git. They are the calibration artifacts — when weights change, re-run the capture tool, update fixtures, re-run tests.

### 3b. Layer boundaries

| Layer | Class | Input type | Output type |
|-------|-------|-----------|-------------|
| L1: Drools evidence | `PatternClassificationRuleUnit` + DRL rules | `ScoutingSessionManager` state (unitEvents, expansionEvents, gameTime) | `List<EvidenceMarker>`, `List<ConfidenceRevision>` |
| L2: Cascade routing | `CascadingPatternClassifier` | `List<EvidenceMarker>`, `List<ConfidenceRevision>`, `StrategyFeatures`, `Race`, frame, enemyCount | `CascadeResult` (assessments, llmTriggered) |
| L3: Scouting publish | `DroolsScoutingTask` (lines 296–328) | `CascadeResult`, broker subscription state, dispatch flags | `PatternAssessmentPublished` CDI event (fired or not) |
| L4: Workbench enricher | `WorkbenchEnricher` | `PatternAssessmentPublished` event | `WorkbenchEvent` → `WorkbenchBroadcaster.broadcast()` |
| L5: WebSocket delivery | `WorkbenchBroadcaster` | `WorkbenchEvent` | JSON text on WebSocket |

### 3c. Fixture format

JSON files committed to `quarkmind-sc2/src/test/resources/fixtures/pipeline/<replay-slug>/`:

```
fixtures/pipeline/nothing-4720936/
  l1-input.json      — frames[] with unitEvents, expansionEvents, gameTime per tick
  l1-output.json     — frames[] with evidenceMarkers[], revisions[] per tick
  l2-input.json      — frames[] with evidence, revisions, features, race, frame
  l2-output.json     — frames[] with assessments[], llmTriggered per tick
  metadata.json      — replay name, player, race matchup, total frames
```

Each fixture file is an array of frame snapshots. Not every tick — sample at meaningful intervals (every 336 frames / 15 seconds of game time, plus on-change for evidence). The fixture capture tool decides sampling.

### 3d. Fixture capture tool

A new test utility `PipelineFixtureCapture` that:

1. Runs a replay through the scouting pipeline (like `PatternClassificationCalibrationTest`)
2. At each sampled frame, serialises the input and output of each layer boundary
3. Writes the fixture files to `src/test/resources/fixtures/pipeline/<slug>/`

This is a diagnostic test (`@Tag("diagnostic")`) — run manually to regenerate fixtures after tuning. Not part of the default test suite.

### 3e. Isolation tests (plain JUnit, fast)

**L1 isolation — `DroolsEvidenceReplayTest`:**
- Loads `l1-input.json` fixtures
- For each frame: populates `PatternClassificationRuleUnit` from fixture data, fires rules, collects evidence markers
- Asserts: evidence markers match `l1-output.json` (archetype, weight within ±0.05)
- Proves: Drools rules produce expected evidence for known game states

**L2 isolation — `CascadeReplayTest`:**
- Loads `l2-input.json` fixtures (or `l1-output.json` as cascade input)
- Constructs `CascadingPatternClassifier` (no CDI — test constructor)
- For each frame: calls `classify()` with fixture evidence markers
- Asserts: assessments are non-empty when enemies are visible, archetype and confidence within expected ranges
- Proves: cascade routing logic produces assessments from known evidence

**L3 isolation — `ScoutingPublishReplayTest`:**
- Loads `l2-output.json` fixtures
- Constructs `DroolsScoutingTask` publish logic in isolation (extract the publish decision into a testable static method or delegate)
- Asserts: `PatternAssessmentPublished` would fire for frames where assessments changed
- Proves: publish gating (change detection, dispatch flags) works correctly

**L4 isolation — `WorkbenchEnricherReplayTest`:**
- Constructs `WorkbenchEnricher` with a mock `WorkbenchBroadcaster`
- Fires `PatternAssessmentPublished` events from fixture data
- Asserts: `WorkbenchEvent` with type "pattern" is broadcast
- Proves: CDI event observation and enrichment work

### 3f. Composition tests (plain JUnit, fast)

**L1+L2 — `DroolsCascadeCompositionTest`:**
- Loads `l1-input.json`
- Runs Drools rules → feeds output directly to cascade → asserts assessments
- Proves: Drools evidence is sufficient to produce cascade assessments for known replays
- Key regression test: "given this replay at frame N, at least one assessment exists"

**L1+L2+L3 — `ScoutingPipelineCompositionTest`:**
- Loads `l1-input.json`
- Runs Drools → cascade → scouting publish logic
- Asserts: `PatternAssessmentPublished` fires at expected frames
- Proves: the full scouting pipeline produces CDI events from raw unit data

**Full stack — `WorkbenchPipelineIT` (`@QuarkusTest`):**
- Boots CDI, runs game ticks with replay data
- Asserts: `WorkbenchBroadcaster` has non-null `latestPatternSnapshot()` after N ticks
- This is the existing failing test from the issue — it passes when all layers work

### 3g. Calibration tests (diagnostic, replay-driven)

Extend existing `CascadeValidationCalibrationTest` pattern:

**`PipelineCalibrationTest` (`@Tag("diagnostic")`):**
- Runs the default replay (`Nothing_4720936.SC2Replay`) through the full scouting pipeline
- At each sampled frame, logs: evidence markers, cumulative confidences, cascade result, whether assessment was published
- Asserts calibration ranges:
  - "After enemies are first visible, cascade returns non-empty within 5 ticks"
  - "At least 1 assessment with confidence >= 0.35 by frame 3000"
  - "COMPOSITION_UNKNOWN fallback fires no more than 30% of ticks with enemies visible" (i.e., real classifications dominate)
- This test is the tuning feedback loop — run it, read the output, adjust YAML weights, re-run

## 4. Changes Summary

### New files

| File | What |
|------|------|
| `EvidenceMarker` — no change | Input fixture type for L2 |
| `fixtures/pipeline/nothing-4720936/*.json` | Committed test fixtures |
| `DroolsEvidenceReplayTest` | L1 isolation test |
| `CascadeReplayTest` | L2 isolation test |
| `ScoutingPublishReplayTest` | L3 isolation test |
| `WorkbenchEnricherReplayTest` | L4 isolation test |
| `DroolsCascadeCompositionTest` | L1+L2 composition test |
| `ScoutingPipelineCompositionTest` | L1+L2+L3 composition test |
| `PipelineFixtureCapture` | Diagnostic fixture generator |
| `PipelineCalibrationTest` | Calibration / tuning feedback test |

### Modified files

| File | Change |
|------|--------|
| `StrategyArchetype.java` | Add `TERRAN_COMPOSITION_UNKNOWN`, `ZERG_COMPOSITION_UNKNOWN`, `PROTOSS_COMPOSITION_UNKNOWN` |
| `CascadingPatternClassifier.java` | Add `enemyCount` param to `classify()`, fallback logic, `FALLBACK_CONFIDENCE` constant |
| `DroolsScoutingTask.java` | Pass `enemies.size()` to cascade `classify()` |
| `StrategyTaxonomy.java` | Read per-unit weight from YAML (default 0.5) |
| `strategy-taxonomy.yaml` | Tune weights for mid-game archetypes after calibration |
| `CascadingPatternClassifierTest.java` | Add fallback tests |
| `WorkbenchPipelineIT.java` | Fix the existing failing test |
| All `switch` expressions on `StrategyArchetype` | Add cases for new enum values |

### Not changed

- `WorkbenchBroadcaster`, `WorkbenchEnricher`, `WorkbenchSocket` — the pipeline plumbing is correct; the problem is that the cascade produces no input for it
- Commentary pipeline — no changes needed; it will start receiving events once assessments flow
- ONNX tier — out of scope; the fallback handles the gap

## 5. Scope Boundaries

**In scope:**
- Cascade fallback (COMPOSITION_UNKNOWN)
- Per-unit weight in YAML taxonomy
- Layered test infrastructure (isolation + composition)
- Fixture capture tool
- Calibration test
- Fix `WorkbenchPipelineIT`

**Out of scope:**
- ONNX model configuration — separate concern, requires neocortex models
- LLM fallback wiring for replay — requires ChatModel bean configuration, follow-up issue
- New hand-authored Drools rules for mid-game — the generic rules with tuned weights cover mid-game; hand-authored rules are a calibration refinement, not a structural fix
- Workbench rendering changes — the UI is correct; it just needs data

## References

- `CascadingPatternClassifier.java` — cascade routing logic, DISPATCH_THRESHOLD, allAssessments
- `DroolsScoutingTask.java:296-328` — cascade invocation and publish
- `PatternClassification.drl` — evidence rules (hand-authored + generic)
- `StrategyTaxonomy.java:138-154` — signature spec parsing, hardcoded weight
- `strategy-taxonomy.yaml:1303` — ZERG_ROACH_HYDRA signature example
- `StrategyArchetype.java` — enum with race/phase/category
- `WorkbenchBroadcaster.java` — event broadcast and snapshot
- `WorkbenchEnricher.java` — CDI event observer
- `PatternClassificationCalibrationTest` — existing calibration pattern
- `CascadeValidationCalibrationTest` — existing cascade calibration
- Issue #212 — three-tier cascade design spec
- Issue #289 — workbench blocks-ui design spec
