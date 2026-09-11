# Spatial Recalibration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #300 — Recalibrate spatial features against real movement data
**Issue group:** #300

**Goal:** Fix posture stickiness, remove dead event wiring, create spatial calibration test, and tune thresholds against replay data with real enemy movement.

**Architecture:** ENEMY_POSTURE caches its last classification so it survives the 3-minute unit buffer eviction. Dead `EnemyPostureClassifiedEvent` is removed. A new `SpatialCalibrationTest` measures posture persistence, timing attack detection rate, and TacticalPosture transition quality across replay datasets.

**Tech Stack:** Java 21, Quarkus, Drools, JUnit 5, AssertJ

## Global Constraints

- All spatial constants must be calibrated from replay ground truth, not estimated from formulae (protocol `sc2data-spatial-constants-require-calibration.md`)
- `UNIT_WINDOW_MS` (3 min) must not be changed — it controls build-order classification, not posture
- Pattern classification (`PatternClassificationRuleUnit`) must not be affected by posture changes
- No `@QuarkusTest` for tests that can be plain JUnit

---

## Batch 1: Posture persistence + dead event removal

### Task 1: Posture persistence fix (D1)

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/DroolsScoutingTaskTest.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/DroolsScoutingTaskIT.java`

**Interfaces:**
- Consumes: `ScoutingRuleUnit.getPostureDecisions()` (existing)
- Produces: `QuarkMindCaseFile.ENEMY_POSTURE` — now persists last classification instead of falling back to UNKNOWN

- [ ] **Step 1: Write failing test — posture persists after buffer eviction**

In `DroolsScoutingTaskTest.java`, add a test that feeds enemy units to the scouting task, then advances game time past the 3-minute eviction window with no new enemies, and asserts posture is still ALL_IN (not UNKNOWN).

This test exercises the full `execute()` method and needs the DRL rules to fire, so it belongs in `DroolsScoutingTaskIT.java` (QuarkusTest with Drools runtime).

```java
@Test
void posturePersistsAfterBufferEviction() {
    // Tick 1: see enemies near enemy base (no expansion) → ALL_IN
    var ctx1 = caseContext(List.of(enemy(200, 200)), List.of(), 100L);
    scoutingTask.execute(ctx1);
    assertThat(ctx1.getAs(QuarkMindCaseFile.ENEMY_POSTURE, String.class))
        .isEqualTo("ALL_IN");

    // Tick 2: 4 minutes later, no enemies visible → buffer evicted
    long fourMinMs = 4L * 60 * 1000;
    long fourMinFrames = (long)(fourMinMs / 1000.0 * SC2Data.GAME_LOOPS_PER_SECOND);
    var ctx2 = caseContext(List.of(), List.of(), fourMinFrames);
    scoutingTask.execute(ctx2);

    // Posture should persist as ALL_IN, not revert to UNKNOWN
    assertThat(ctx2.getAs(QuarkMindCaseFile.ENEMY_POSTURE, String.class))
        .isEqualTo("ALL_IN");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DroolsScoutingTaskIT#posturePersistsAfterBufferEviction -q`
Expected: FAIL — posture is "UNKNOWN" instead of "ALL_IN"

- [ ] **Step 3: Write failing test — posture transitions ALL_IN → MACRO after eviction**

```java
@Test
void postureTransitionsToMacroAfterEviction() {
    // Tick 1: enemy near base, no expansion → ALL_IN
    var ctx1 = caseContext(List.of(enemy(200, 200)), List.of(), 100L);
    scoutingTask.execute(ctx1);
    assertThat(ctx1.getAs(QuarkMindCaseFile.ENEMY_POSTURE, String.class))
        .isEqualTo("ALL_IN");

    // Tick 2: 4 minutes later, enemy far from their base (expansion signal)
    long fourMinFrames = (long)(4L * 60 * 1000 / 1000.0 * SC2Data.GAME_LOOPS_PER_SECOND);
    var ctx2 = caseContext(
        List.of(new Unit("e-exp", UnitType.ZEALOT, new Point2d(100, 100), 100, 100, 50, 50, 0, 0)),
        List.of(), fourMinFrames);
    scoutingTask.execute(ctx2);

    // Expansion detected → MACRO overrides cached ALL_IN
    assertThat(ctx2.getAs(QuarkMindCaseFile.ENEMY_POSTURE, String.class))
        .isEqualTo("MACRO");
}
```

- [ ] **Step 4: Write failing test — posture resets on game restart**

```java
@Test
void postureResetsOnGameRestart() {
    // Tick 1: classify as ALL_IN
    var ctx1 = caseContext(List.of(enemy(200, 200)), List.of(), 100L);
    scoutingTask.execute(ctx1);
    assertThat(ctx1.getAs(QuarkMindCaseFile.ENEMY_POSTURE, String.class))
        .isEqualTo("ALL_IN");

    // Reset (simulates game restart)
    scoutingTask.resetDispatchState();
    sessionManager.reset();

    // Tick 2: no enemies → should be UNKNOWN (fresh game)
    var ctx2 = caseContext(List.of(), List.of(), 100L);
    scoutingTask.execute(ctx2);
    assertThat(ctx2.getAs(QuarkMindCaseFile.ENEMY_POSTURE, String.class))
        .isEqualTo("UNKNOWN");
}
```

- [ ] **Step 5: Implement posture persistence**

In `DroolsScoutingTask.java`, add a `cachedPosture` field and update the posture resolution logic.

Add field after line 100 (near other volatile fields):
```java
volatile String cachedPosture = "UNKNOWN";
```

Replace lines 249-251 (the posture resolution):
```java
// Before:
String posture = data != null && !data.getPostureDecisions().isEmpty()
    ? data.getPostureDecisions().get(0) : "UNKNOWN";

// After:
if (data != null && !data.getPostureDecisions().isEmpty()) {
    cachedPosture = data.getPostureDecisions().get(0);
}
String posture = cachedPosture;
```

Add reset to `resetDispatchState()` (line ~130):
```java
cachedPosture = "UNKNOWN";
```

- [ ] **Step 6: Run all three new tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DroolsScoutingTaskIT#posturePersistsAfterBufferEviction+postureTransitionsToMacroAfterEviction+postureResetsOnGameRestart -q`
Expected: PASS

- [ ] **Step 7: Run existing scouting tests to verify no regressions**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DroolsScoutingTaskIT -q`
Expected: ALL PASS — existing `postureUnknownWhenNoEnemiesEverSeen` should still pass (UNKNOWN is correct pre-classification)

Run: `mvn test -pl quarkmind-sc2 -Dtest=DroolsScoutingTaskTest -q`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java
git add quarkmind-sc2/src/test/java/io/quarkmind/plugin/DroolsScoutingTaskIT.java
git commit -m "fix: posture persists after buffer eviction — conclusion survives evidence window

ENEMY_POSTURE now caches the last DRL classification (ALL_IN or MACRO)
instead of falling back to UNKNOWN when the 3-minute unit buffer is
evicted. UNKNOWN means 'never classified', not 'evidence expired'.

Refs #300"
```

### Task 2: Remove dead EnemyPostureClassifiedEvent (D3)

**Files:**
- Delete: `quarkmind-sc2/src/main/java/io/quarkmind/agent/EnemyPostureClassifiedEvent.java` (use `ide_refactor_safe_delete`)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/DroolsScoutingTaskTest.java`

**Interfaces:**
- Consumes: nothing new
- Produces: nothing — this is a removal

- [ ] **Step 1: Remove the `postureClassified` field and `fire()` call from DroolsScoutingTask**

In `DroolsScoutingTask.java`:
- Remove import: `import io.quarkmind.agent.EnemyPostureClassifiedEvent;` (line 17)
- Remove field: `@Inject Event<EnemyPostureClassifiedEvent> postureClassified;` (line 74)
- Remove the `postureClassified.fire()` block (lines 276-278):
```java
// Remove this block:
if (!"UNKNOWN".equals(posture)) {
    postureClassified.fire(new EnemyPostureClassifiedEvent(posture));
}
```

- [ ] **Step 2: Remove mock setup from DroolsScoutingTaskTest**

In `DroolsScoutingTaskTest.java`, remove line 62:
```java
// Remove:
task.postureClassified = mock(jakarta.enterprise.event.Event.class);
```

- [ ] **Step 3: Delete EnemyPostureClassifiedEvent.java**

Use `ide_refactor_safe_delete` on `EnemyPostureClassifiedEvent.java` to verify no other consumers exist, then delete.

- [ ] **Step 4: Run full test suite to verify no breakage**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: ALL PASS — no consumers of the event exist

- [ ] **Step 5: Commit**

```bash
git add -A quarkmind-sc2/src/
git commit -m "chore: remove dead EnemyPostureClassifiedEvent

Zero @Observes consumers exist. The StrategyTrustObserver mentioned in
ARC42STORIES.MD was never implemented — strategy routing uses
ScoutingIntelBroker + TacticalPosture instead.

Refs #300"
```

## Batch 2: Calibration test + threshold tuning

### Task 3: SpatialCalibrationTest (D4)

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/SpatialCalibrationTest.java`

**Interfaces:**
- Consumes: `ReplaySimulatedGame`, `IEM10JsonSimulatedGame` (existing replay infrastructure from `ScoutingCalibrationTest`)
- Consumes: `ScoutingSessionManager` (to read posture, army-near-base events)
- Produces: printed report with spatial metrics per replay and aggregate stats

- [ ] **Step 1: Create SpatialCalibrationTest**

Follow the pattern from `ScoutingCalibrationTest`. This test drives replays through `SimulatedGame`, feeds each tick's enemy units to a `ScoutingSessionManager`, reads posture decisions via the DRL, and collects metrics.

```java
package io.quarkmind.plugin.scouting;

import io.quarkmind.domain.GameState;
import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.Unit;
import io.quarkmind.sc2.mock.IEM10JsonSimulatedGame;
import io.quarkmind.sc2.mock.ReplaySimulatedGame;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("benchmark")
class SpatialCalibrationTest {

    private static final Path AI_ARENA_DIR = Path.of("replays/aiarena_protoss");
    private static final Path IEM10_ZIP    = Path.of("replays/2016_IEM_10_Taipei.zip");
    private static final int TICKS_FULL_GAME = 900; // ~10 min at SC2 Faster speed

    record SpatialMetrics(
        String replayName,
        String matchup,
        int totalFramesAfterFirstClassification,
        int unknownFramesAfterFirstClassification,
        int armyNearBaseEventCount,
        int postureTransitions
    ) {
        double unknownRate() {
            return totalFramesAfterFirstClassification == 0 ? 0.0
                : (double) unknownFramesAfterFirstClassification / totalFramesAfterFirstClassification;
        }
    }

    @Test
    void spatialMetrics() throws IOException {
        Map<String, List<SpatialMetrics>> metricsByMatchup = new LinkedHashMap<>();
        metricsByMatchup.put("PvT", new ArrayList<>());
        metricsByMatchup.put("PvZ", new ArrayList<>());
        metricsByMatchup.put("PvP", new ArrayList<>());

        // AI Arena replays
        List<Path> replayFiles = Files.list(AI_ARENA_DIR)
            .filter(p -> p.toString().endsWith(".SC2Replay"))
            .sorted()
            .collect(Collectors.toList());

        for (Path replay : replayFiles) {
            try {
                ReplaySimulatedGame game = new ReplaySimulatedGame(replay, 1);
                SpatialMetrics m = measureReplay(game, replay.getFileName().toString(), "PvP");
                metricsByMatchup.get("PvP").add(m);
            } catch (IllegalArgumentException e) {
                // skip unloadable replays
            }
        }

        // IEM10 replays
        List<IEM10JsonSimulatedGame> iem10Games = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP);
        for (IEM10JsonSimulatedGame game : iem10Games) {
            SpatialMetrics m = measureReplay(game, game.name(), game.matchup());
            metricsByMatchup.get(game.matchup()).add(m);
        }

        String report = buildReport(metricsByMatchup);
        System.out.println(report);

        // Sanity assertion: at least some replays loaded
        int total = metricsByMatchup.values().stream().mapToInt(List::size).sum();
        assertThat(total).as("Total replays loaded").isGreaterThan(0);
    }

    private SpatialMetrics measureReplay(Object game, String name, String matchup) {
        ScoutingSessionManager mgr = new ScoutingSessionManager();
        boolean firstClassified = false;
        int totalAfter = 0;
        int unknownAfter = 0;
        int armyNearBase = 0;
        int postureTransitions = 0;
        String lastPosture = "UNKNOWN";

        Point2d ourNexus = new Point2d(8, 8);
        Point2d estimatedEnemyBase = new Point2d(224, 224);

        for (int tick = 0; tick < TICKS_FULL_GAME; tick++) {
            GameState state;
            if (game instanceof ReplaySimulatedGame rsg) {
                rsg.tick();
                state = rsg.snapshot();
            } else {
                IEM10JsonSimulatedGame iem = (IEM10JsonSimulatedGame) game;
                iem.tick();
                state = iem.snapshot();
            }

            List<Unit> enemies = state.enemyUnits();
            long gameTimeMs = (long)(tick * 1000.0 / 22.4) * 1000;

            int prevArmyBuffer = mgr.armyBufferSize();
            mgr.processFrame(enemies, gameTimeMs, ourNexus, estimatedEnemyBase);
            mgr.evict(gameTimeMs);

            if (mgr.armyBufferSize() > prevArmyBuffer) {
                armyNearBase++;
            }

            // Determine posture from buffer state
            boolean hasUnits = mgr.unitBufferSize() > 0;
            boolean hasExpansions = mgr.expansionBufferSize() > 0;
            String posture;
            if (hasUnits && !hasExpansions) {
                posture = "ALL_IN";
            } else if (hasExpansions) {
                posture = "MACRO";
            } else {
                posture = "UNKNOWN";
            }

            if (!posture.equals("UNKNOWN") && !firstClassified) {
                firstClassified = true;
            }

            if (firstClassified) {
                totalAfter++;
                if (posture.equals("UNKNOWN")) {
                    unknownAfter++;
                }
            }

            if (!posture.equals(lastPosture)) {
                postureTransitions++;
                lastPosture = posture;
            }
        }

        return new SpatialMetrics(name, matchup, totalAfter, unknownAfter,
            armyNearBase, postureTransitions);
    }

    private String buildReport(Map<String, List<SpatialMetrics>> metricsByMatchup) {
        StringBuilder sb = new StringBuilder();
        sb.append("\n=== SPATIAL CALIBRATION REPORT ===\n\n");

        for (var entry : metricsByMatchup.entrySet()) {
            String matchup = entry.getKey();
            List<SpatialMetrics> metrics = entry.getValue();
            if (metrics.isEmpty()) continue;

            sb.append(String.format("--- %s (%d replays) ---\n", matchup, metrics.size()));
            sb.append(String.format("%-40s %8s %8s %8s %8s\n",
                "Replay", "Unk%", "ArmyEvt", "PostTrn", "Frames"));

            for (SpatialMetrics m : metrics) {
                sb.append(String.format("%-40s %7.1f%% %8d %8d %8d\n",
                    truncate(m.replayName(), 40),
                    m.unknownRate() * 100,
                    m.armyNearBaseEventCount(),
                    m.postureTransitions(),
                    m.totalFramesAfterFirstClassification()));
            }

            double avgUnknown = metrics.stream().mapToDouble(SpatialMetrics::unknownRate).average().orElse(0);
            double avgArmy = metrics.stream().mapToDouble(SpatialMetrics::armyNearBaseEventCount).average().orElse(0);
            double avgTransitions = metrics.stream().mapToDouble(SpatialMetrics::postureTransitions).average().orElse(0);

            sb.append(String.format("\n  Avg UNKNOWN rate: %.1f%%\n", avgUnknown * 100));
            sb.append(String.format("  Avg army-near-base events: %.1f\n", avgArmy));
            sb.append(String.format("  Avg posture transitions: %.1f\n\n", avgTransitions));
        }

        return sb.toString();
    }

    private static String truncate(String s, int max) {
        return s.length() <= max ? s : s.substring(0, max - 3) + "...";
    }
}
```

- [ ] **Step 2: Run the test**

Run: `mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=SpatialCalibrationTest -q`
Expected: PASS — prints spatial metrics report. Review the UNKNOWN rate (should be high before D1 fix is applied, demonstrating the problem).

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/SpatialCalibrationTest.java
git commit -m "test: SpatialCalibrationTest — replay-driven spatial metrics

Measures posture UNKNOWN rate, army-near-base event count, and posture
transition count across AI Arena + IEM10 replay datasets. Prints per-replay
and per-matchup aggregate statistics.

Refs #300"
```

### Task 4: Run calibration suite and tune thresholds

**Files:**
- Modify (if needed): `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/ScoutingSessionManager.java`
- Modify: `ARC42STORIES.MD`
- Modify: `docs/protocols/strategy-attack-under-unknown-posture.md`

**Interfaces:**
- Consumes: calibration test output
- Produces: tuned thresholds, updated docs

- [ ] **Step 1: Run the full calibration suite**

Run: `mvn test -pl quarkmind-sc2 -Pbenchmark -q`

This runs `ScoutingCalibrationTest`, `MapControlCalibrationTest`, `PatternClassificationCalibrationTest`, `ExpansionLocationCalibrationTest`, and `SpatialCalibrationTest`. All should pass. Review the spatial metrics report.

- [ ] **Step 2: Assess threshold changes based on data**

Review the SpatialCalibrationTest output:
- If UNKNOWN rate is near zero → D1 fix is working, no threshold change needed
- If army-near-base events are too frequent/infrequent → consider adjusting `NEAR_BASE_DISTANCE` (30f), `MIN_ARMY_NEAR_BASE` (3), or `ARMY_WINDOW_MS` (10s)
- If posture transitions are very low → the system isn't detecting movement changes; may need follow-up on `GamePhaseSummariser` sensitivity

If adjustments are needed, update the constants in `ScoutingSessionManager.java` and re-run calibration to verify.

- [ ] **Step 3: Update protocol — strategy-attack-under-unknown-posture.md**

The protocol documents UNKNOWN posture as a known edge case. With D1, UNKNOWN after first classification is eliminated. Update the protocol to reflect this:

- Section "Why UNKNOWN can mean two different things" → note that post-eviction UNKNOWN is now prevented by posture persistence
- Section "Why ATTACK under UNKNOWN is acceptable" → note that UNKNOWN is now pre-contact only
- Section "Known edge case" → mark as resolved by #300

- [ ] **Step 4: Update ARC42STORIES.MD §9**

Remove the reference to `StrategyTrustObserver` consuming `EnemyPostureClassifiedEvent`. The text at line ~836 mentions:

> "At the first mid-game checkpoint (when DroolsScoutingTask fires EnemyPostureClassifiedEvent...)"

Update to reflect that posture updates flow through `ScoutingIntelBroker` and the TacticalPosture cascade instead of a CDI event observer.

- [ ] **Step 5: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/ScoutingSessionManager.java
git add ARC42STORIES.MD
git add docs/protocols/strategy-attack-under-unknown-posture.md
git commit -m "docs: update posture protocol and ARC42 for persistence fix

- Protocol: mark post-eviction UNKNOWN as resolved by #300
- ARC42 §9: replace StrategyTrustObserver reference with ScoutingIntelBroker
  + TacticalPosture cascade (matches actual implementation)
- Thresholds: [adjusted/unchanged based on calibration data]

Refs #300"
```

## References

- [2026-09-11-spatial-recalibration-design.md] — design spec this plan implements
- [DroolsScoutingTask.java:249-251] — posture resolution code (fix target)
- [ScoutingSessionManager.java:30-96] — buffer and threshold definitions
- [EnemyPostureClassifiedEvent.java] — dead event (removal target)
- [ScoutingCalibrationTest.java] — calibration test pattern to follow
- [DroolsScoutingTaskIT.java] — integration test fixture for posture tests
- Protocol `sc2data-spatial-constants-require-calibration.md` — calibration methodology
- Protocol `strategy-attack-under-unknown-posture.md` — posture semantics (update target)
- GitHub #298 — UnitPositions tracker events (upstream fix)
- GitHub #299 — Replay smoke tests (regression guard)
