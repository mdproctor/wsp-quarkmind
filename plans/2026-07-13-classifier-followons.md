# Classifier Follow-ons Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #236 — test: replay classification accuracy >= 70% acceptance criterion
**Issue group:** #236, #238

**Goal:** Validate the Drools CEP pattern classifier against replay data with accuracy assertions, and update Commentator/Advisory LLM prompts to reason about PATTERN_ASSESSMENT intel.

**Architecture:** Two independent deliverables on one branch. #236 adds a `@QuarkusTest @Tag("benchmark")` test that runs the full classification pipeline (ScoutingSessionManager → PatternClassificationRuleUnit → PatternClassifier) against replay datasets and asserts ≥ 70% accuracy for rush/air-threat archetypes. #238 updates system prompt builders and user message builders in CommentaryWorkerFactory and AdvisoryWorkerFactory to reference and format PATTERN_ASSESSMENT data.

**Tech Stack:** Drools Rule Units, Quarkus CDI, LangChain4j ChatModel, JUnit 5, AssertJ

## Global Constraints

- `@QuarkusTest` only for tests that need CDI (Drools rule unit injection). Plain JUnit otherwise.
- `@Tag("benchmark")` tests run via `mvn test -Pbenchmark`, excluded from default surefire run.
- IntelliJ MCP required for all source file operations. Use `project_path=/Users/mdproctor/claude/casehub/quarkmind`.
- All commits reference issue numbers: `Refs #236` or `Refs #238`.

---

### Task 1: PatternClassificationCalibrationTest — ground truth labeler and test skeleton

**Files:**
- Create: `src/test/java/io/quarkmind/plugin/scouting/PatternClassificationCalibrationTest.java`

**Interfaces:**
- Consumes: `IEM10JsonSimulatedGame.enumerate(Path)`, `ReplaySimulatedGame(Path, int)`, `GameState.enemyUnits()`, `EnemyArchetype`, `UnitType`
- Produces: `deriveGroundTruth(Map<UnitType, Long>, double gameTimeMin)` → `EnemyArchetype` or `null`

- [ ] **Step 1: Write the failing test — ground truth derivation**

```java
package io.quarkmind.plugin.scouting;

import io.quarkmind.domain.EnemyArchetype;
import io.quarkmind.domain.UnitType;
import org.junit.jupiter.api.Test;

import java.util.EnumMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class PatternClassificationCalibrationTest {

    @Test
    void groundTruth_roachRush() {
        var counts = new EnumMap<UnitType, Long>(UnitType.class);
        counts.put(UnitType.ROACH, 5L);
        assertThat(deriveGroundTruth(counts, 3.0)).isEqualTo(EnemyArchetype.ZERG_ROACH_RUSH);
    }

    @Test
    void groundTruth_marineRush() {
        var counts = new EnumMap<UnitType, Long>(UnitType.class);
        counts.put(UnitType.MARINE, 6L);
        assertThat(deriveGroundTruth(counts, 3.0)).isEqualTo(EnemyArchetype.TERRAN_MARINE_RUSH);
    }

    @Test
    void groundTruth_noPattern() {
        var counts = new EnumMap<UnitType, Long>(UnitType.class);
        counts.put(UnitType.ZEALOT, 1L);
        assertThat(deriveGroundTruth(counts, 3.0)).isNull();
    }

    @Test
    void groundTruth_zerglingRush() {
        var counts = new EnumMap<UnitType, Long>(UnitType.class);
        counts.put(UnitType.ZERGLING, 7L);
        assertThat(deriveGroundTruth(counts, 3.0)).isEqualTo(EnemyArchetype.ZERG_ZERGLING_RUSH);
    }

    @Test
    void groundTruth_gatewayRush() {
        var counts = new EnumMap<UnitType, Long>(UnitType.class);
        counts.put(UnitType.STALKER, 2L);
        counts.put(UnitType.ZEALOT, 3L);
        assertThat(deriveGroundTruth(counts, 4.0)).isEqualTo(EnemyArchetype.PROTOSS_GATEWAY_RUSH);
    }

    @Test
    void groundTruth_bansheeHarass() {
        var counts = new EnumMap<UnitType, Long>(UnitType.class);
        counts.put(UnitType.BANSHEE, 1L);
        assertThat(deriveGroundTruth(counts, 6.0)).isEqualTo(EnemyArchetype.TERRAN_BANSHEE_HARASS);
    }

    @Test
    void groundTruth_bioTiming_marinesLate() {
        var counts = new EnumMap<UnitType, Long>(UnitType.class);
        counts.put(UnitType.MARINE, 7L);
        assertThat(deriveGroundTruth(counts, 5.0)).isEqualTo(EnemyArchetype.TERRAN_BIO_TIMING);
    }

    @Test
    void groundTruth_mechPush() {
        var counts = new EnumMap<UnitType, Long>(UnitType.class);
        counts.put(UnitType.SIEGE_TANK, 3L);
        assertThat(deriveGroundTruth(counts, 6.0)).isEqualTo(EnemyArchetype.TERRAN_MECH_PUSH);
    }

    @Test
    void groundTruth_precedence_marineRushBeforeBioTiming() {
        var counts = new EnumMap<UnitType, Long>(UnitType.class);
        counts.put(UnitType.MARINE, 6L);
        // At 3.0 min (< 4.0), marines >= 5 → MARINE_RUSH takes precedence over BIO_TIMING
        assertThat(deriveGroundTruth(counts, 3.0)).isEqualTo(EnemyArchetype.TERRAN_MARINE_RUSH);
    }

    static EnemyArchetype deriveGroundTruth(Map<UnitType, Long> counts, double gameTimeMin) {
        // Evaluated top-down — first match wins
        long marines = counts.getOrDefault(UnitType.MARINE, 0L);
        long roaches = counts.getOrDefault(UnitType.ROACH, 0L);
        long zerglings = counts.getOrDefault(UnitType.ZERGLING, 0L);
        long stalkers = counts.getOrDefault(UnitType.STALKER, 0L);
        long zealots = counts.getOrDefault(UnitType.ZEALOT, 0L);
        long siegeTanks = counts.getOrDefault(UnitType.SIEGE_TANK, 0L);
        long banshees = counts.getOrDefault(UnitType.BANSHEE, 0L);

        if (marines >= 5 && gameTimeMin < 4.0) return EnemyArchetype.TERRAN_MARINE_RUSH;
        if (banshees >= 1 && gameTimeMin < 8.0) return EnemyArchetype.TERRAN_BANSHEE_HARASS;
        if (zerglings >= 6 && gameTimeMin < 4.0) return EnemyArchetype.ZERG_ZERGLING_RUSH;
        if (roaches >= 4 && gameTimeMin < 5.0) return EnemyArchetype.ZERG_ROACH_RUSH;
        if (stalkers + zealots >= 4 && gameTimeMin < 5.0) return EnemyArchetype.PROTOSS_GATEWAY_RUSH;
        if (marines >= 6 && gameTimeMin >= 4.0) return EnemyArchetype.TERRAN_BIO_TIMING;
        if (siegeTanks >= 2 && gameTimeMin >= 5.0) return EnemyArchetype.TERRAN_MECH_PUSH;
        return null;
    }
}
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `mvn test -Dtest=PatternClassificationCalibrationTest -q`
Expected: All 9 tests PASS (deriveGroundTruth is implemented inline as a static method)

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/test/java/io/quarkmind/plugin/scouting/PatternClassificationCalibrationTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "test(#236): ground truth labeler for pattern classification calibration

Refs #236"
```

---

### Task 2: PatternClassificationCalibrationTest — replay classification pipeline with accuracy assertions

**Files:**
- Modify: `src/test/java/io/quarkmind/plugin/scouting/PatternClassificationCalibrationTest.java`

**Interfaces:**
- Consumes: `deriveGroundTruth()` from Task 1, `ScoutingSessionManager()`, `ScoutingSessionManager.processFrame(List<Unit>, long, Point2d, Point2d)`, `ScoutingSessionManager.evict(long)`, `ScoutingSessionManager.buildPatternRuleUnit(double)`, `PatternClassifier.computeAllConfidences(List<EvidenceMarker>)`, `PatternClassifier.mergeCumulative(EnumMap, Map)`, `PatternClassifier.topAssessment(EnumMap, long)`, `RuleUnit<PatternClassificationRuleUnit>` (injected)
- Produces: `calibrateClassificationAccuracy()` test method — asserts ≥ 70% rush/air-threat accuracy

- [ ] **Step 1: Convert test to @QuarkusTest, add the calibration test method**

The class needs `@QuarkusTest @Tag("benchmark")` and injection of `RuleUnit<PatternClassificationRuleUnit>`. Add the main calibration test that runs the full pipeline.

Convert the class annotation to `@QuarkusTest @Tag("benchmark")` and add the injected field and calibration method. The existing ground truth unit tests remain (they don't need CDI, but running inside @QuarkusTest is fine).

```java
// Add to class: annotations and injection
@QuarkusTest
@Tag("benchmark")
class PatternClassificationCalibrationTest {

    @Inject RuleUnit<PatternClassificationRuleUnit> ruleUnit;

    private static final Path AI_ARENA_DIR = Path.of("replays/aiarena_protoss");
    private static final Path IEM10_ZIP    = Path.of("replays/2016_IEM_10_Taipei.zip");
    private static final int TICKS_3MIN = 183;
    private static final double GAME_TIME_3MIN = 3.0;

    @Test
    void calibrateClassificationAccuracy() throws IOException {
        int rushCorrect = 0, rushTotal = 0;
        int airCorrect = 0, airTotal = 0;
        int totalGames = 0, labelledGames = 0;
        StringBuilder report = new StringBuilder();
        report.append("=== Pattern Classification Accuracy — 3-min mark ===\n\n");

        // --- AI Arena (PvP) ---
        List<Path> replayFiles = Files.list(AI_ARENA_DIR)
            .filter(p -> p.toString().endsWith(".SC2Replay"))
            .sorted().toList();

        for (Path replay : replayFiles) {
            try {
                ReplaySimulatedGame game = new ReplaySimulatedGame(replay, 1);
                var result = classifyGame(game, "PvP", replay.getFileName().toString());
                totalGames++;
                if (result != null) {
                    labelledGames++;
                    report.append(result.reportLine()).append("\n");
                    if (result.isRush()) { rushTotal++; if (result.correct()) rushCorrect++; }
                    if (result.isAirThreat()) { airTotal++; if (result.correct()) airCorrect++; }
                }
            } catch (IllegalArgumentException e) { /* skip unparseable */ }
        }

        // --- IEM10 ---
        List<IEM10JsonSimulatedGame> iem10Games = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP);
        for (IEM10JsonSimulatedGame game : iem10Games) {
            var result = classifyGame(game, game.matchup(), game.replayName());
            totalGames++;
            if (result != null) {
                labelledGames++;
                report.append(result.reportLine()).append("\n");
                if (result.isRush()) { rushTotal++; if (result.correct()) rushCorrect++; }
                if (result.isAirThreat()) { airTotal++; if (result.correct()) airCorrect++; }
            }
        }

        double rushAccuracy = rushTotal > 0 ? (double) rushCorrect / rushTotal : 1.0;
        double airAccuracy = airTotal > 0 ? (double) airCorrect / airTotal : 1.0;

        report.append("\n=== Summary ===\n");
        report.append(String.format("Total games: %d, labelled: %d%n", totalGames, labelledGames));
        report.append(String.format("Rush accuracy: %d/%d = %.1f%%%n", rushCorrect, rushTotal, rushAccuracy * 100));
        report.append(String.format("Air threat accuracy: %d/%d = %.1f%%%n", airCorrect, airTotal, airAccuracy * 100));
        System.out.println(report);

        assertThat(rushTotal).as("Rush sample size").isGreaterThan(0);
        assertThat(rushAccuracy).as("Rush archetype accuracy").isGreaterThanOrEqualTo(0.7);
        if (airTotal > 0) {
            assertThat(airAccuracy).as("Air threat accuracy").isGreaterThanOrEqualTo(0.7);
        }
    }

    private ClassificationResult classifyGame(SimulatedGame game, String matchup, String gameName) {
        ScoutingSessionManager sessionManager = new ScoutingSessionManager();
        EnumMap<EnemyArchetype, Double> cumulative = new EnumMap<>(EnemyArchetype.class);
        Point2d ourNexus = new Point2d(30, 30);
        Point2d enemyBase = new Point2d(120, 120);

        for (int tick = 0; tick < TICKS_3MIN; tick++) {
            game.tick();
            GameState state = game.snapshot();
            long gameTimeMs = (long) (tick * (22.0 / 22.4) * 1000);

            sessionManager.processFrame(state.enemyUnits(), gameTimeMs, ourNexus, enemyBase);
            sessionManager.evict(gameTimeMs);

            PatternClassificationRuleUnit data = sessionManager.buildPatternRuleUnit(tick * (22.0 / 22.4) / 60.0);
            try (RuleUnitInstance<PatternClassificationRuleUnit> instance = ruleUnit.createInstance(data)) {
                instance.fire();
            }
            var tickConf = PatternClassifier.computeAllConfidences(data.getEvidence());
            PatternClassifier.mergeCumulative(cumulative, tickConf);
        }

        // Ground truth from final unit counts
        GameState finalState = game.snapshot();
        Map<UnitType, Long> counts = new EnumMap<>(UnitType.class);
        for (var unit : finalState.enemyUnits()) {
            counts.merge(unit.type(), 1L, Long::sum);
        }
        EnemyArchetype groundTruth = deriveGroundTruth(counts, GAME_TIME_3MIN);
        if (groundTruth == null) return null;

        var topOpt = PatternClassifier.topAssessment(cumulative, TICKS_3MIN);
        EnemyArchetype predicted = topOpt.map(EnemyPatternAssessment::archetype).orElse(null);
        boolean correct = groundTruth == predicted;

        return new ClassificationResult(matchup, gameName, groundTruth, predicted, correct,
            topOpt.map(EnemyPatternAssessment::confidence).orElse(0.0));
    }

    private record ClassificationResult(String matchup, String gameName,
                                         EnemyArchetype groundTruth, EnemyArchetype predicted,
                                         boolean correct, double confidence) {
        boolean isRush() {
            return groundTruth == EnemyArchetype.TERRAN_MARINE_RUSH
                || groundTruth == EnemyArchetype.ZERG_ZERGLING_RUSH
                || groundTruth == EnemyArchetype.ZERG_ROACH_RUSH
                || groundTruth == EnemyArchetype.PROTOSS_GATEWAY_RUSH;
        }
        boolean isAirThreat() {
            return groundTruth == EnemyArchetype.TERRAN_BANSHEE_HARASS;
        }
        String reportLine() {
            return String.format("  %-4s %-40s truth=%-24s pred=%-24s conf=%.2f %s",
                matchup, gameName, groundTruth, predicted, confidence, correct ? "✓" : "✗");
        }
    }

    // deriveGroundTruth() already exists from Task 1
}
```

- [ ] **Step 2: Add required imports**

```java
import io.quarkmind.domain.EnemyPatternAssessment;
import io.quarkmind.domain.GameState;
import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.mock.IEM10JsonSimulatedGame;
import io.quarkmind.sc2.mock.ReplaySimulatedGame;
import io.quarkmind.sc2.mock.SimulatedGame;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.drools.ruleunits.api.RuleUnit;
import org.drools.ruleunits.api.RuleUnitInstance;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.EnumMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
```

- [ ] **Step 3: Run the calibration test**

Run: `mvn test -Pbenchmark -Dtest=PatternClassificationCalibrationTest#calibrateClassificationAccuracy -q`
Expected: PASS with ≥ 70% rush accuracy. Review the printed report — if accuracy is below 70%, the Drools rules or ground truth thresholds need tuning.

- [ ] **Step 4: If accuracy < 70%, tune ground truth thresholds**

The ground truth derivation uses independent thresholds from the Drools rules. If accuracy is low, review the report to identify systematic mismatches:
- If Drools classifies correctly but ground truth labels disagree → adjust ground truth thresholds
- If ground truth is correct but Drools misses → the Drools rules need improvement (separate issue)

Do NOT adjust Drools rules to match ground truth — the ground truth should be the simpler, more lenient check.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/test/java/io/quarkmind/plugin/scouting/PatternClassificationCalibrationTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "test(#236): replay classification accuracy calibration with ≥ 70% assertions

Runs PatternClassificationRuleUnit against AI Arena + IEM10 replays,
asserts ≥ 70% accuracy for rush and air-threat archetypes at 3-min mark.

Refs #236"
```

---

### Task 3: Update CommentaryWorkerFactory prompts for PATTERN_ASSESSMENT

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/commentary/CommentaryWorkerFactory.java`
- Modify: `src/test/java/io/quarkmind/plugin/commentary/CommentaryWorkerFactoryTest.java`

**Interfaces:**
- Consumes: `EnemyPatternAssessment(archetype, confidence, detectedAtFrame, rationale)`, `QuarkMindCaseFile.COMMENTARY_TRIGGER`, `QuarkMindCaseFile.COMMENTARY_NARRATIVE_TRIGGER`
- Produces: Updated `buildReactiveSystemPrompt()`, `buildNarrativeSystemPrompt()`, `buildReactiveUserMessage()`, `buildNarrativeUserMessage()`

- [ ] **Step 1: Write failing tests for system prompt updates**

Add to `CommentaryWorkerFactoryTest.java`:

```java
@Test
void reactiveSystemPrompt_containsPatternAssessmentGuidance() {
    AgentDescriptor descriptor = buildReactiveDescriptor(
        "claude:commentator-energetic@v1", "Energetic Commentator", "bold");
    String prompt = CommentaryWorkerFactory.buildReactiveSystemPrompt(descriptor);
    assertThat(prompt).contains("PATTERN_ASSESSMENT");
    assertThat(prompt).contains("archetype");
    assertThat(prompt).contains("confidence");
}

@Test
void narrativeSystemPrompt_containsPatternAssessmentGuidance() {
    AgentDescriptor descriptor = buildNarrativeDescriptor(
        "claude:narrator-dramatic@v1", "Dramatic Narrator", "flexible");
    String prompt = CommentaryWorkerFactory.buildNarrativeSystemPrompt(descriptor);
    assertThat(prompt).contains("PATTERN_ASSESSMENT");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=CommentaryWorkerFactoryTest#reactiveSystemPrompt_containsPatternAssessmentGuidance+narrativeSystemPrompt_containsPatternAssessmentGuidance -q`
Expected: FAIL — prompts do not yet mention PATTERN_ASSESSMENT

- [ ] **Step 3: Update buildReactiveSystemPrompt**

Use `ide_replace_member` on `CommentaryWorkerFactory.buildReactiveSystemPrompt` to add PATTERN_ASSESSMENT guidance after the disposition block:

```java
static String buildReactiveSystemPrompt(AgentDescriptor descriptor) {
    AgentDisposition disposition = descriptor.disposition();
    StringBuilder sb = new StringBuilder();
    sb.append("You are a StarCraft II play-by-play commentator.\n");
    sb.append("Your name is: ").append(descriptor.name()).append("\n\n");
    sb.append("Behavioural disposition:\n");
    if (disposition != null) {
        appendTrait(sb, "Risk appetite", disposition.riskAppetite());
        appendTrait(sb, "Rule following", disposition.ruleFollowing());
        appendTrait(sb, "Social orientation", disposition.socialOrient());
        appendTrait(sb, "Autonomy", disposition.autonomy());
        appendTrait(sb, "Conflict mode", disposition.conflictMode());
    }
    sb.append("\nIntel types you may receive:\n");
    sb.append("- PATTERN_ASSESSMENT: enemy strategy classification with archetype name ");
    sb.append("(e.g. ZERG_ROACH_RUSH) and confidence score (0.0–1.0). ");
    sb.append("When present, call out the classification naturally.\n\n");
    sb.append("React with energy to the game moment. Keep it to 1-2 sentences.\n");
    sb.append("Your response should be plain text commentary (no labels or structure).\n");
    return sb.toString();
}
```

- [ ] **Step 4: Update buildNarrativeSystemPrompt**

Use `ide_replace_member` on `CommentaryWorkerFactory.buildNarrativeSystemPrompt`:

```java
static String buildNarrativeSystemPrompt(AgentDescriptor descriptor) {
    AgentDisposition disposition = descriptor.disposition();
    StringBuilder sb = new StringBuilder();
    sb.append("You are a StarCraft II color commentator.\n");
    sb.append("Your name is: ").append(descriptor.name()).append("\n\n");
    sb.append("Behavioural disposition:\n");
    if (disposition != null) {
        appendTrait(sb, "Risk appetite", disposition.riskAppetite());
        appendTrait(sb, "Rule following", disposition.ruleFollowing());
        appendTrait(sb, "Social orientation", disposition.socialOrient());
        appendTrait(sb, "Autonomy", disposition.autonomy());
        appendTrait(sb, "Conflict mode", disposition.conflictMode());
    }
    sb.append("\nIntel types you may receive:\n");
    sb.append("- PATTERN_ASSESSMENT: enemy strategy classification with archetype name ");
    sb.append("and confidence score (0.0–1.0). ");
    sb.append("When present, weave the strategic implications into your narrative.\n\n");
    sb.append("Narrate the strategic arc. Do NOT repeat moments just announced.\n");
    sb.append("Provide context and analysis. Keep it to 2-3 sentences.\n");
    sb.append("Your response should be plain text commentary (no labels or structure).\n");
    return sb.toString();
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -Dtest=CommentaryWorkerFactoryTest -q`
Expected: ALL tests PASS including the two new ones

- [ ] **Step 6: Write failing tests for user message pattern data formatting**

Add to `CommentaryWorkerFactoryTest.java`:

```java
@Test
void reactiveUserMessage_includesPatternAssessment_whenPresent() {
    String msg = CommentaryWorkerFactory.buildReactiveUserMessage(
        Map.of(QuarkMindCaseFile.COMMENTARY_TRIGGER, Map.of(
            "gameFrame", 1500L,
            "patternAssessment", Map.of(
                "archetype", "ZERG_ROACH_RUSH",
                "confidence", 0.72))));
    assertThat(msg).contains("ENEMY PATTERN: ZERG_ROACH_RUSH");
    assertThat(msg).contains("0.72");
}

@Test
void reactiveUserMessage_omitsPatternAssessment_whenAbsent() {
    String msg = CommentaryWorkerFactory.buildReactiveUserMessage(
        Map.of(QuarkMindCaseFile.COMMENTARY_TRIGGER, Map.of("gameFrame", 1500L)));
    assertThat(msg).doesNotContain("ENEMY PATTERN");
}

@Test
void narrativeUserMessage_includesPatternAssessment_whenPresent() {
    String msg = CommentaryWorkerFactory.buildNarrativeUserMessage(
        Map.of(QuarkMindCaseFile.COMMENTARY_NARRATIVE_TRIGGER, Map.of(
            "patternAssessment", Map.of(
                "archetype", "TERRAN_MARINE_RUSH",
                "confidence", 0.85))));
    assertThat(msg).contains("ENEMY PATTERN: TERRAN_MARINE_RUSH");
    assertThat(msg).contains("0.85");
}
```

- [ ] **Step 7: Run tests to verify they fail**

Run: `mvn test -Dtest=CommentaryWorkerFactoryTest#reactiveUserMessage_includesPatternAssessment_whenPresent+reactiveUserMessage_omitsPatternAssessment_whenAbsent+narrativeUserMessage_includesPatternAssessment_whenPresent -q`
Expected: FAIL — user message builders don't extract pattern data yet

- [ ] **Step 8: Update buildReactiveUserMessage to extract pattern assessment**

Use `ide_replace_member` on `CommentaryWorkerFactory.buildReactiveUserMessage`. Add pattern assessment extraction after the game state block:

```java
static String buildReactiveUserMessage(Map<String, Object> input) {
    StringBuilder sb = new StringBuilder();
    Object trigger = input.get(QuarkMindCaseFile.COMMENTARY_TRIGGER);
    if (trigger instanceof Map<?, ?> triggerMap) {
        Object momentTypes = triggerMap.get("momentTypes");
        if (momentTypes != null) {
            sb.append("MOMENT: ").append(momentTypes).append("\n\n");
        }

        sb.append("Game state:\n");
        Object frame = triggerMap.get("gameFrame");
        if (frame != null) {
            sb.append("- Frame: ").append(frame).append("\n");
        }
        Object minerals = triggerMap.get("minerals");
        if (minerals != null) {
            sb.append("- Minerals: ").append(minerals).append("\n");
        }
        Object supplyUsed = triggerMap.get("supplyUsed");
        if (supplyUsed != null) {
            sb.append("- Supply: ").append(supplyUsed).append("\n");
        }
        Object army = triggerMap.get("army");
        if (army != null) {
            sb.append("- Army size: ").append(army).append("\n");
        }

        appendPatternAssessment(sb, triggerMap);
    }

    sb.append("\nProvide your immediate commentary on this moment.");
    return sb.toString();
}
```

- [ ] **Step 9: Update buildNarrativeUserMessage to extract pattern assessment**

Use `ide_replace_member` on `CommentaryWorkerFactory.buildNarrativeUserMessage`:

```java
static String buildNarrativeUserMessage(Map<String, Object> input) {
    StringBuilder sb = new StringBuilder();
    Object trigger = input.get(QuarkMindCaseFile.COMMENTARY_NARRATIVE_TRIGGER);
    if (trigger instanceof Map<?, ?> triggerMap) {
        Object moments = triggerMap.get("moments");
        if (moments != null) {
            sb.append("Recent moments: ").append(moments).append("\n\n");
        }

        Object frame = triggerMap.get("gameFrame");
        if (frame != null) {
            sb.append("Frame: ").append(frame).append("\n");
        }

        appendPatternAssessment(sb, triggerMap);
    }

    sb.append("\nProvide contextual narration of the strategic arc.");
    return sb.toString();
}
```

- [ ] **Step 10: Add appendPatternAssessment helper**

Use `ide_insert_member` after `buildNarrativeUserMessage`:

```java
private static void appendPatternAssessment(StringBuilder sb, Map<?, ?> triggerMap) {
    Object pattern = triggerMap.get("patternAssessment");
    if (pattern instanceof Map<?, ?> patternMap) {
        Object archetype = patternMap.get("archetype");
        Object confidence = patternMap.get("confidence");
        if (archetype != null) {
            sb.append("\nENEMY PATTERN: ").append(archetype);
            if (confidence != null) {
                sb.append(" (confidence: ").append(confidence).append(")");
            }
            sb.append("\n");
        }
    }
}
```

- [ ] **Step 11: Run all CommentaryWorkerFactory tests**

Run: `mvn test -Dtest=CommentaryWorkerFactoryTest -q`
Expected: ALL tests PASS

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/plugin/commentary/CommentaryWorkerFactory.java src/test/java/io/quarkmind/plugin/commentary/CommentaryWorkerFactoryTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#238): update Commentator prompts for PATTERN_ASSESSMENT intel

System prompts now reference PATTERN_ASSESSMENT archetype + confidence.
User messages extract and format pattern data when present in triggers.

Refs #238"
```

---

### Task 4: Update AdvisoryWorkerFactory prompts for PATTERN_ASSESSMENT

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/advisory/AdvisoryWorkerFactory.java`
- Modify: `src/test/java/io/quarkmind/plugin/advisory/AdvisoryWorkerFactoryTest.java`

**Interfaces:**
- Consumes: `AdvisoryWorkerFactory.buildSystemPrompt(AgentDescriptor, String)`, `AdvisoryWorkerFactory.buildUserMessage(String, Map)`
- Produces: Updated system prompt with PATTERN_ASSESSMENT reference, updated user message with pattern data formatting

- [ ] **Step 1: Write failing tests**

Add to `AdvisoryWorkerFactoryTest.java`:

```java
@Test
void system_prompt_references_pattern_assessment() {
    AgentDescriptor descriptor = buildDescriptor(
        "claude:crisis-aggressive@v1", "Aggressive Crisis Advisor",
        "crisis-response", "advisory-crisis",
        AgentDisposition.builder()
            .riskAppetite("bold").ruleFollowing("flexible")
            .socialOrient("collaborative").autonomy("semi-autonomous")
            .conflictMode("collaborate").delegation(false).build());
    String prompt = AdvisoryWorkerFactory.buildSystemPrompt(descriptor, "crisis");
    assertThat(prompt).contains("PATTERN_ASSESSMENT");
    assertThat(prompt).contains("archetype");
}

@Test
void user_message_includes_pattern_assessment_when_present() {
    String msg = AdvisoryWorkerFactory.buildUserMessage("crisis",
        Map.of("game.advisory.trigger.crisis", Map.of(
            "event", "TIMING_ALERT",
            "patternAssessment", Map.of(
                "archetype", "ZERG_ROACH_RUSH",
                "confidence", 0.72)),
            "game.frame", 3000));
    assertThat(msg).contains("Enemy pattern classification: ZERG_ROACH_RUSH");
    assertThat(msg).contains("0.72");
}

@Test
void user_message_omits_pattern_assessment_when_absent() {
    String msg = AdvisoryWorkerFactory.buildUserMessage("crisis",
        Map.of("game.advisory.trigger.crisis", "TIMING_ALERT", "game.frame", 3000));
    assertThat(msg).doesNotContain("Enemy pattern classification");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=AdvisoryWorkerFactoryTest#system_prompt_references_pattern_assessment+user_message_includes_pattern_assessment_when_present+user_message_omits_pattern_assessment_when_absent -q`
Expected: FAIL

- [ ] **Step 3: Update buildSystemPrompt**

Use `ide_replace_member` on `AdvisoryWorkerFactory.buildSystemPrompt`:

```java
static String buildSystemPrompt(AgentDescriptor descriptor, String role) {
    AgentDisposition disposition = descriptor.disposition();
    StringBuilder sb = new StringBuilder();
    sb.append("You are a StarCraft II ").append(role).append(" advisor.\n");
    sb.append("Your name is: ").append(descriptor.name()).append("\n\n");
    sb.append("Behavioural disposition:\n");
    if (disposition != null) {
        appendTrait(sb, "Risk appetite", disposition.riskAppetite());
        appendTrait(sb, "Rule following", disposition.ruleFollowing());
        appendTrait(sb, "Social orientation", disposition.socialOrient());
        appendTrait(sb, "Autonomy", disposition.autonomy());
        appendTrait(sb, "Conflict mode", disposition.conflictMode());
    }
    sb.append("\nIntel types you may receive:\n");
    sb.append("- PATTERN_ASSESSMENT: enemy strategy classification with archetype name ");
    sb.append("and confidence score (0.0–1.0). ");
    sb.append("Factor the classified intent into your recommendation — ");
    sb.append("a high-confidence rush classification should increase urgency.\n\n");
    sb.append("Respond with:\n");
    sb.append("RECOMMENDATION: <your recommendation>\n");
    sb.append("REASONING: <your reasoning>\n");
    sb.append("CONFIDENCE: <0.0 to 1.0>\n");
    return sb.toString();
}
```

- [ ] **Step 4: Update buildUserMessage to extract pattern assessment from trigger**

Use `ide_replace_member` on `AdvisoryWorkerFactory.buildUserMessage`:

```java
static String buildUserMessage(String role, Map<String, Object> input) {
    StringBuilder sb = new StringBuilder();
    String triggerKey = "game.advisory.trigger." + role;
    Object trigger = input.get(triggerKey);
    if (trigger != null) {
        if (trigger instanceof Map<?, ?> triggerMap) {
            Object event = triggerMap.get("event");
            if (event != null) {
                sb.append("TRIGGER EVENT: ").append(event).append("\n");
            } else {
                sb.append("TRIGGER EVENT: ").append(trigger).append("\n");
            }
            Object pattern = triggerMap.get("patternAssessment");
            if (pattern instanceof Map<?, ?> patternMap) {
                Object archetype = patternMap.get("archetype");
                Object confidence = patternMap.get("confidence");
                if (archetype != null) {
                    sb.append("Enemy pattern classification: ").append(archetype);
                    if (confidence != null) {
                        sb.append(" (confidence: ").append(confidence).append(")");
                    }
                    sb.append("\n");
                }
            }
        } else {
            sb.append("TRIGGER EVENT: ").append(trigger).append("\n");
        }
        sb.append("\n");
    }

    sb.append("Current game state:\n");
    Object frame = input.get("game.frame");
    if (frame != null) {
        sb.append("- Game frame: ").append(frame).append("\n");
    }
    Object minerals = input.get("game.resources.minerals");
    if (minerals != null) {
        sb.append("- Minerals: ").append(minerals).append("\n");
    }
    Object vespene = input.get("game.resources.vespene");
    if (vespene != null) {
        sb.append("- Vespene: ").append(vespene).append("\n");
    }
    Object supplyUsed = input.get("game.resources.supply.used");
    if (supplyUsed != null) {
        sb.append("- Supply used: ").append(supplyUsed).append("\n");
    }
    Object supplyCap = input.get("game.resources.supply.cap");
    if (supplyCap != null) {
        sb.append("- Supply cap: ").append(supplyCap).append("\n");
    }

    sb.append("\nProvide your advisory recommendation for this ").append(role).append(" situation.");
    return sb.toString();
}
```

- [ ] **Step 5: Run all AdvisoryWorkerFactory tests**

Run: `mvn test -Dtest=AdvisoryWorkerFactoryTest -q`
Expected: ALL tests PASS

- [ ] **Step 6: Run full test suite to check for regressions**

Run: `mvn test -q`
Expected: ALL tests PASS, zero regressions

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/plugin/advisory/AdvisoryWorkerFactory.java src/test/java/io/quarkmind/plugin/advisory/AdvisoryWorkerFactoryTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#238): update Advisory prompts for PATTERN_ASSESSMENT intel

System prompts now reference PATTERN_ASSESSMENT archetype + confidence.
User messages extract and format pattern data from trigger maps.

Refs #238"
```
