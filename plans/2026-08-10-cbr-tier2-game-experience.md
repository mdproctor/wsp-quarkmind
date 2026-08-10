# CBR Tier 2 — Structured Game Experience Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #215 — Event hierarchy case enrichment
**Issue group:** #215, #217, #218, #219, #220, #221

**Goal:** Enrich the CBR case model with structured game experience data across six domains so the bot learns from how games unfold, not just who won.

**Architecture:** Extend `SC2GameCbrCase` with ~16 new features. `SC2CbrRetentionObserver` subscribes to event buses and accumulates L2/L3/L4 data during the game, then builds an enriched case at close. Three new CaseFile keys provide instrumentation data from other plugins. QA endpoints compute learning metrics on demand.

**Tech Stack:** Java 21, Quarkus, Drools (rule unit), casehub-neocortex-memory-api (CbrCase, FeatureValue, FeatureField, CbrCaseMemoryStore), JAX-RS

## Global Constraints

- All new features use simple `FeatureValue` types (string, number, stringList) — no StructVal/TimeSeries (Tier 3 territory)
- Features that may be absent (timing features where the event didn't occur) are omitted from the feature map entirely — no sentinel values
- All new CaseFile keys added to `QuarkMindCaseFile.ALL_KEYS` (if exists) or the constants class
- QA endpoints carry `@UnlessBuildProfile("prod")`
- Unit tests are plain JUnit (no `@QuarkusTest`) unless CDI context is required
- Design spec: `/Users/mdproctor/claude/public/quarkmind/specs/issue-215-cbr-tier2-game-experience/2026-08-10-cbr-tier2-game-experience-design.md`

## Verification items from light review

These must be verified during implementation:
1. **Weight normalisation** (Task 1): Confirm neocortex normalises weighted similarity sums when total weight > 1.0
2. **`FeatureField.categoricalList` / `FeatureField.text`** (Task 1): Verify these factory methods exist; fall back to `categorical` / omit if not
3. **`CbrCaseMemoryStore.scan()`** (Task 7): Verify `InMemoryCbrCaseMemoryStore` implements scan; if not, use `retrieveSimilar` with a broad query as fallback

---

### Task 1: Foundation — EnrichedGameData, buildForGameEnriched, schema extension

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/QuarkMindCaseFile.java`
- Create: `src/main/java/io/quarkmind/agent/cbr/EnrichedGameData.java`
- Modify: `src/main/java/io/quarkmind/agent/cbr/SC2GameCbrCase.java`
- Modify: `src/main/java/io/quarkmind/agent/cbr/SC2CbrSchemaRegistrar.java`
- Modify: `src/test/java/io/quarkmind/agent/cbr/SC2GameCbrCaseTest.java`
- Test: `src/test/java/io/quarkmind/agent/cbr/SC2GameCbrCaseTest.java`

**Interfaces:**
- Produces: `QuarkMindCaseFile.OPPONENT_ID`, `QuarkMindCaseFile.SCOUTING_DISPATCH_FRAME`, `QuarkMindCaseFile.CBR_INFLUENCED_SELECTION` — String constants used by Tasks 4, 3, 5
- Produces: `EnrichedGameData` record — used by Task 6 to pass enrichment data to the factory
- Produces: `SC2GameCbrCase.buildForGameEnriched(String, String, String, double, String, EnrichedGameData)` — used by Task 6

- [ ] **Step 1: Add new CaseFile key constants**

Add three new constants to `QuarkMindCaseFile.java` after line 50 (after `LLM_FALLBACK_RATIONALE`):

```java
public static final String OPPONENT_ID = "game.opponent.id";
public static final String SCOUTING_DISPATCH_FRAME = "game.scouting.dispatch.frame";
public static final String CBR_INFLUENCED_SELECTION = "agent.strategy.cbr.influenced";
```

- [ ] **Step 2: Write the failing test for buildForGameEnriched**

Add to `SC2GameCbrCaseTest.java`:

```java
@Test
void buildForGameEnriched_allFeaturesPopulated() {
    var enrichment = new EnrichedGameData(
            List.of("EARLY_MACRO", "MID_SKIRMISH"), 8, "Game progression: EARLY_MACRO -> MID_SKIRMISH", 12.5,
            3, 0.4, 0.3,
            2, 42, 0.5, 1,
            OptionalDouble.of(2.1), OptionalDouble.of(1.5), 0.82,
            "ZERG_ROACH_RUSH");
    var c = SC2GameCbrCase.buildForGameEnriched(
            "ZERG_ROACH_RUSH", "ZERG", "PvZ", 0.82, "strategy.early-pressure", enrichment);

    assertThat(c.problem()).isEqualTo("vs ZERG_ROACH_RUSH (PvZ)");
    assertThat(c.solution()).isEqualTo("strategy.early-pressure");
    assertThat(c.features()).containsEntry("enemy_archetype", FeatureValue.string("ZERG_ROACH_RUSH"));
    assertThat(c.features()).containsEntry("phase_sequence", FeatureValue.stringList(List.of("EARLY_MACRO", "MID_SKIRMISH")));
    assertThat(c.features()).containsEntry("moment_count", FeatureValue.number(8));
    assertThat(c.features()).containsEntry("battle_count", FeatureValue.number(3));
    assertThat(c.features()).containsEntry("dominance_overall", FeatureValue.number(0.3));
    assertThat(c.features()).containsEntry("expansion_count", FeatureValue.number(2));
    assertThat(c.features()).containsEntry("worker_count_final", FeatureValue.number(42));
    assertThat(c.features()).containsEntry("supply_block_count", FeatureValue.number(1));
    assertThat(c.features()).containsEntry("first_contact_minute", FeatureValue.number(2.1));
    assertThat(c.features()).containsEntry("scout_dispatch_minute", FeatureValue.number(1.5));
    assertThat(c.features()).containsEntry("opponent_id", FeatureValue.string("ZERG_ROACH_RUSH"));
}

@Test
void buildForGameEnriched_optionalTimingFeaturesOmittedWhenEmpty() {
    var enrichment = new EnrichedGameData(
            List.of("EARLY_MACRO"), 2, "", 5.0,
            0, 0.0, 0.0,
            1, 22, 0.0, 0,
            OptionalDouble.empty(), OptionalDouble.empty(), 0.5,
            "mock-opponent");
    var c = SC2GameCbrCase.buildForGameEnriched(
            "TERRAN_2RAX_MARINE", "TERRAN", "PvT", 0.5, "strategy.drools", enrichment);

    assertThat(c.features()).doesNotContainKey("first_contact_minute");
    assertThat(c.features()).doesNotContainKey("scout_dispatch_minute");
    assertThat(c.features()).containsKey("moment_count");
    assertThat(c.features()).containsKey("opponent_id");
}

@Test
void buildForGameEnriched_withOutcomePreservesEnrichment() {
    var enrichment = new EnrichedGameData(
            List.of("EARLY_MACRO"), 5, "narrative", 10.0,
            2, 0.3, 0.2,
            1, 30, 0.1, 0,
            OptionalDouble.of(3.0), OptionalDouble.empty(), 0.7,
            "ZERG_MASS_LING");
    var c = SC2GameCbrCase.buildForGameEnriched(
            "ZERG_MASS_LING", "ZERG", "PvZ", 0.7, "strategy.early-pressure", enrichment);
    var updated = (SC2GameCbrCase) c.withOutcome("WIN", 0.9);

    assertThat(updated.outcome()).isEqualTo("WIN");
    assertThat(updated.features()).containsEntry("battle_count", FeatureValue.number(2));
    assertThat(updated.features()).containsEntry("opponent_id", FeatureValue.string("ZERG_MASS_LING"));
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -Dtest=SC2GameCbrCaseTest -q`
Expected: FAIL — `EnrichedGameData` and `buildForGameEnriched` do not exist yet

- [ ] **Step 4: Create EnrichedGameData record**

Create `src/main/java/io/quarkmind/agent/cbr/EnrichedGameData.java`:

```java
package io.quarkmind.agent.cbr;

import java.util.List;
import java.util.OptionalDouble;

public record EnrichedGameData(
        List<String> phaseSequence,
        int momentCount,
        String arcNarrative,
        double gameDurationMinutes,
        int battleCount,
        double dominanceArmy,
        double dominanceOverall,
        int expansionCount,
        int workerCountFinal,
        double dominanceEconomy,
        int supplyBlockCount,
        OptionalDouble firstContactMinute,
        OptionalDouble scoutDispatchMinute,
        double archetypeConfidence,
        String opponentId
) {
    public EnrichedGameData {
        phaseSequence = List.copyOf(phaseSequence);
    }
}
```

- [ ] **Step 5: Implement buildForGameEnriched**

Add static method to `SC2GameCbrCase.java` after `buildForGame()`:

```java
public static SC2GameCbrCase buildForGameEnriched(
        String archetypeName, String raceName, String matchup,
        double assessmentConfidence, String strategyId,
        EnrichedGameData e) {
    var features = new java.util.HashMap<String, FeatureValue>();
    features.put("enemy_archetype", FeatureValue.string(archetypeName));
    features.put("enemy_race", FeatureValue.string(raceName));
    features.put("matchup", FeatureValue.string(matchup));
    features.put("assessment_confidence", FeatureValue.number(assessmentConfidence));

    features.put("phase_sequence", FeatureValue.stringList(e.phaseSequence()));
    features.put("phase_count", FeatureValue.number(e.phaseSequence().stream().distinct().count()));
    features.put("moment_count", FeatureValue.number(e.momentCount()));
    if (e.arcNarrative() != null && !e.arcNarrative().isEmpty()) {
        features.put("arc_narrative", FeatureValue.string(e.arcNarrative()));
    }
    features.put("game_duration_minutes", FeatureValue.number(e.gameDurationMinutes()));

    features.put("battle_count", FeatureValue.number(e.battleCount()));
    features.put("dominance_army", FeatureValue.number(e.dominanceArmy()));
    features.put("dominance_overall", FeatureValue.number(e.dominanceOverall()));

    features.put("expansion_count", FeatureValue.number(e.expansionCount()));
    features.put("worker_count_final", FeatureValue.number(e.workerCountFinal()));
    features.put("dominance_economy", FeatureValue.number(e.dominanceEconomy()));
    features.put("supply_block_count", FeatureValue.number(e.supplyBlockCount()));

    e.firstContactMinute().ifPresent(v -> features.put("first_contact_minute", FeatureValue.number(v)));
    e.scoutDispatchMinute().ifPresent(v -> features.put("scout_dispatch_minute", FeatureValue.number(v)));
    features.put("archetype_confidence", FeatureValue.number(e.archetypeConfidence()));

    features.put("opponent_id", FeatureValue.string(e.opponentId()));

    return new SC2GameCbrCase(
            "vs " + archetypeName + " (" + matchup + ")",
            strategyId, null, null, features);
}
```

Note: `FeatureValue.stringList()` may not exist. If compilation fails, use `FeatureValue.of(e.phaseSequence())` or fall back to `FeatureValue.string(String.join(",", e.phaseSequence()))`. Similarly, `arc_narrative` uses `FeatureValue.string()` (not `FeatureValue.text()`) as a safe fallback — semantic text matching is a Tier 2+ enhancement on the neocortex side.

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn test -Dtest=SC2GameCbrCaseTest -q`
Expected: PASS (all 10 tests including 3 new ones)

- [ ] **Step 7: Extend SC2CbrSchemaRegistrar**

Replace the strategy schema registration in `register()` with the extended schema:

```java
CbrFeatureSchema strategySchema = CbrFeatureSchema.of(
        SC2GameCbrCase.CBR_TYPE,
        // Tier 1
        FeatureField.categorical("enemy_archetype"),
        FeatureField.categorical("enemy_race"),
        FeatureField.categorical("matchup"),
        FeatureField.numeric("assessment_confidence", 0.0, 1.0),
        // #215 — Event hierarchy
        FeatureField.categoricalList("phase_sequence"),
        FeatureField.numeric("phase_count", 0, 20),
        FeatureField.numeric("moment_count", 0, 50),
        FeatureField.categorical("arc_narrative"),
        FeatureField.numeric("game_duration_minutes", 0, 30),
        // #217 — Tactical
        FeatureField.numeric("battle_count", 0, 20),
        FeatureField.numeric("dominance_army", -1.0, 1.0),
        FeatureField.numeric("dominance_overall", -1.0, 1.0),
        // #218 — Economics
        FeatureField.numeric("expansion_count", 0, 8),
        FeatureField.numeric("worker_count_final", 0, 80),
        FeatureField.numeric("dominance_economy", -1.0, 1.0),
        FeatureField.numeric("supply_block_count", 0, 20),
        // #219 — Scouting
        FeatureField.numeric("first_contact_minute", 0, 15),
        FeatureField.numeric("scout_dispatch_minute", 0, 10),
        FeatureField.numeric("archetype_confidence", 0, 1.0),
        // #220 — Opponent
        FeatureField.categorical("opponent_id")
);
```

Note: If `FeatureField.categoricalList()` does not compile, use `FeatureField.categorical("phase_sequence")` — the field still stores and retrieves, just without list-based similarity scoring. Verify at compile time and adapt.

- [ ] **Step 8: Compile and run existing tests**

Run: `mvn test -q`
Expected: All existing tests pass, schema registration includes new fields

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/agent/QuarkMindCaseFile.java src/main/java/io/quarkmind/agent/cbr/EnrichedGameData.java src/main/java/io/quarkmind/agent/cbr/SC2GameCbrCase.java src/main/java/io/quarkmind/agent/cbr/SC2CbrSchemaRegistrar.java src/test/java/io/quarkmind/agent/cbr/SC2GameCbrCaseTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#215): CBR Tier 2 foundation — EnrichedGameData, buildForGameEnriched, schema extension

Refs #215, #217, #218, #219, #220"
```

---

### Task 2: Supply block moment detection

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/summarisation/MomentDetectionRuleUnit.java`
- Modify: `src/main/java/io/quarkmind/plugin/summarisation/MomentDetectionTask.java`
- Modify: `src/main/resources/io/quarkmind/plugin/summarisation/MomentDetectionTask.drl`
- Modify: `src/test/java/io/quarkmind/plugin/summarisation/MomentDetectionTaskTest.java` (or create if absent)
- Test: verify SUPPLY_BLOCK moments are produced

**Interfaces:**
- Consumes: `GameMomentType.SUPPLY_BLOCK` (already exists in enum)
- Produces: `GameMoment(SUPPLY_BLOCK, frame, Map.of())` events on momentBus — consumed by Task 6 accumulator

- [ ] **Step 1: Write the failing test**

In the MomentDetection test class, add a test that fires rules when supply is blocked:

```java
@Test
void supplyBlock_detected() {
    var ruleUnit = new MomentDetectionRuleUnit();
    ruleUnit.setCurrentFrame(500);
    ruleUnit.setSupplyUsed(46);
    ruleUnit.setSupplyCap(46);
    // insert a dummy intel event to trigger rule evaluation
    ruleUnit.getIntelEvents().add(new ScoutingIntelPayload.ArmySize(5));

    var instance = ruleUnitInstance(ruleUnit);
    instance.fire();

    assertThat(ruleUnit.getDetectedMoments())
            .extracting(GameMoment::type)
            .contains(GameMomentType.SUPPLY_BLOCK);
}

@Test
void supplyBlock_notDetectedWhenNotBlocked() {
    var ruleUnit = new MomentDetectionRuleUnit();
    ruleUnit.setCurrentFrame(500);
    ruleUnit.setSupplyUsed(30);
    ruleUnit.setSupplyCap(46);
    ruleUnit.getIntelEvents().add(new ScoutingIntelPayload.ArmySize(5));

    var instance = ruleUnitInstance(ruleUnit);
    instance.fire();

    assertThat(ruleUnit.getDetectedMoments())
            .extracting(GameMoment::type)
            .doesNotContain(GameMomentType.SUPPLY_BLOCK);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=MomentDetectionTaskTest#supplyBlock_detected -q`
Expected: FAIL — `setSupplyUsed`/`setSupplyCap` do not exist on MomentDetectionRuleUnit

- [ ] **Step 3: Add supplyUsed and supplyCap to MomentDetectionRuleUnit**

Add fields after `previousPosture` (line 17):

```java
private int supplyUsed;
private int supplyCap;

public int getSupplyUsed() { return supplyUsed; }
public void setSupplyUsed(int supplyUsed) { this.supplyUsed = supplyUsed; }
public int getSupplyCap() { return supplyCap; }
public void setSupplyCap(int supplyCap) { this.supplyCap = supplyCap; }
```

- [ ] **Step 4: Populate supplyUsed/supplyCap in MomentDetectionTask.execute()**

In `execute(CaseContext ctx)`, after setting `previousArmyValue` and `previousPosture` on the rule unit, add:

```java
Integer supplyUsed = ctx.get(QuarkMindCaseFile.SUPPLY_USED, Integer.class).orElse(0);
Integer supplyCap = ctx.get(QuarkMindCaseFile.SUPPLY_CAP, Integer.class).orElse(0);
ruleUnit.setSupplyUsed(supplyUsed);
ruleUnit.setSupplyCap(supplyCap);
```

This requires reading the import for `QuarkMindCaseFile` — add if not present.

- [ ] **Step 5: Add Supply Block rule to MomentDetectionTask.drl**

After the "Posture Change" rule (salience 140), add:

```drl
rule "Supply Block"
    salience 130
when
    eval(supplyUsed > 0 && supplyUsed >= supplyCap)
then
    detectedMoments.add(new GameMoment(
        GameMomentType.SUPPLY_BLOCK, currentFrame, java.util.Map.of()));
end
```

- [ ] **Step 6: Add supply block dedup in MomentDetectionTask**

Add a field:

```java
private long lastSupplyBlockFrame = -1;
```

In `fireRules()`, after deduplicating FIRST_CONTACT, add supply block dedup:

```java
moments.removeIf(m -> m.type() == GameMomentType.SUPPLY_BLOCK
        && lastSupplyBlockFrame >= 0
        && m.gameFrame() - lastSupplyBlockFrame < 224);
moments.stream()
        .filter(m -> m.type() == GameMomentType.SUPPLY_BLOCK)
        .findFirst()
        .ifPresent(m -> lastSupplyBlockFrame = m.gameFrame());
```

Clear in `onGameStarted()`:

```java
lastSupplyBlockFrame = -1;
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn test -Dtest=MomentDetectionTaskTest -q`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/plugin/summarisation/MomentDetectionRuleUnit.java src/main/java/io/quarkmind/plugin/summarisation/MomentDetectionTask.java src/main/resources/io/quarkmind/plugin/summarisation/MomentDetectionTask.drl src/test/java/io/quarkmind/plugin/summarisation/
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#218): supply block moment detection — Drools rule with dedup

Refs #218"
```

---

### Task 3: Scout dispatch tracking

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java`
- Modify: existing scouting test (or add test method)

**Interfaces:**
- Consumes: `QuarkMindCaseFile.SCOUTING_DISPATCH_FRAME` (from Task 1)
- Produces: CaseFile key `game.scouting.dispatch.frame` written on first scout dispatch — consumed by Task 6

- [ ] **Step 1: Write the failing test**

```java
@Test
void scoutDispatchFrame_writtenOnFirstDispatch() {
    // Set up game state with a worker and enemy base estimate
    var ctx = new MapCaseContext(Map.of(
            QuarkMindCaseFile.GAME_FRAME, 200L,
            QuarkMindCaseFile.GAME_STATE, gameStateWithWorkers()));
    task.execute(ctx);

    assertThat(ctx.get(QuarkMindCaseFile.SCOUTING_DISPATCH_FRAME, Long.class))
            .isPresent()
            .hasValue(200L);
}

@Test
void scoutDispatchFrame_notOverwrittenOnSubsequentTicks() {
    var ctx = new MapCaseContext(Map.of(
            QuarkMindCaseFile.GAME_FRAME, 200L,
            QuarkMindCaseFile.GAME_STATE, gameStateWithWorkers()));
    task.execute(ctx);

    var ctx2 = new MapCaseContext(Map.of(
            QuarkMindCaseFile.GAME_FRAME, 400L,
            QuarkMindCaseFile.GAME_STATE, gameStateWithWorkers(),
            QuarkMindCaseFile.SCOUTING_DISPATCH_FRAME, 200L));
    task.execute(ctx2);

    assertThat(ctx2.get(QuarkMindCaseFile.SCOUTING_DISPATCH_FRAME, Long.class))
            .hasValue(200L);
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `SCOUTING_DISPATCH_FRAME` not written by execute()

- [ ] **Step 3: Add scout dispatch tracking to DroolsScoutingTask**

Add field after `lastFrame` (line 120):

```java
private boolean scoutDispatched;
```

In `maybeSendScout()` (lines 392-406), after successfully sending the scout intent, add:

```java
if (!scoutDispatched) {
    scoutDispatched = true;
}
```

In `execute()`, after `maybeSendScout()` call, write the CaseFile key:

```java
if (scoutDispatched) {
    ctx.get(QuarkMindCaseFile.SCOUTING_DISPATCH_FRAME, Long.class)
        .orElseGet(() -> {
            long frame = ctx.get(QuarkMindCaseFile.GAME_FRAME, Long.class).orElse(0L);
            ctx.set(QuarkMindCaseFile.SCOUTING_DISPATCH_FRAME, frame);
            return frame;
        });
}
```

In `resetDispatchState()` (lines 137-150), add:

```java
scoutDispatched = false;
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -Dtest=DroolsScoutingTaskTest -q` (or the relevant test class)
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java src/test/java/io/quarkmind/plugin/scouting/
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#219): scout dispatch frame tracking — CaseFile key on first dispatch

Refs #219"
```

---

### Task 4: Opponent ID plumbing

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/mock/MockStartupBean.java`
- Modify: `src/main/java/io/quarkmind/agent/AgentOrchestrator.java` (or the appropriate startup path)
- Test: verify opponent_id is written at game start

**Interfaces:**
- Consumes: `QuarkMindCaseFile.OPPONENT_ID` (from Task 1)
- Produces: CaseFile key `game.opponent.id` written at game start — consumed by Task 6

- [ ] **Step 1: Determine the game start write point**

The opponent_id needs to be written to the CaseFile at game start. The write point depends on the profile:
- Mock/emulated: the `EnemyStrategy.name()` value is known when the game starts
- SC2: deferred (write "unknown")

Use `ide_find_references` on `GameStarted` to find where game start is handled and the CaseFile is first populated. The `GameStateTranslator` or `AgentOrchestrator.startGame()` is likely the right place.

- [ ] **Step 2: Write the failing test**

```java
@Test
void opponentId_writtenAtGameStart() {
    // After startGame(), verify the CaseFile contains opponent_id
    orchestrator.startGame();
    var caseFile = orchestrator.currentCaseFile();
    assertThat(caseFile.get(QuarkMindCaseFile.OPPONENT_ID))
            .isEqualTo("mock-opponent");
}
```

The exact test setup depends on where the write happens. In mock mode, the value should be `"mock-opponent"`. In emulated mode, it should be `EnemyStrategy.name()`.

- [ ] **Step 3: Implement opponent_id write**

In the game start path (likely `GameStateTranslator.translate()` on first tick, or a `@Observes GameStarted` handler), write:

```java
caseFile.set(QuarkMindCaseFile.OPPONENT_ID, resolveOpponentId());
```

Where `resolveOpponentId()` returns:
- Emulated: injected `EnemyBehavior` → `strategy.name()` (if available via CDI)
- Mock: `"mock-opponent"` (config property or hardcoded)
- SC2: `"unknown"` (deferred to #271)

If `EnemyBehavior` is not CDI-injectable (it's created by `EmulatedGame`), use a config property:

```properties
# application.properties
%mock.quarkmind.opponent.id=mock-opponent
%emulated.quarkmind.opponent.id=${quarkmind.emulated.enemy-strategy:PROTOSS_4GATE}
%sc2.quarkmind.opponent.id=unknown
```

Then inject `@ConfigProperty(name = "quarkmind.opponent.id", defaultValue = "unknown")` and write it.

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#220): opponent_id CaseFile key — written at game start per profile

Refs #220"
```

---

### Task 5: CBR influence tracking

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/cbr/SC2StrategyRouterTask.java`
- Modify: `src/test/java/io/quarkmind/agent/cbr/SC2StrategyRouterTaskTest.java`

**Interfaces:**
- Consumes: `QuarkMindCaseFile.CBR_INFLUENCED_SELECTION` (from Task 1)
- Produces: CaseFile key `agent.strategy.cbr.influenced` (Boolean) — consumed by Task 7 (QA endpoints)

- [ ] **Step 1: Write the failing test**

Add to `SC2StrategyRouterTaskTest.java`:

```java
@Test
void cbrInfluencedSelection_trueWhenCbrChangesWinner() {
    // Set up: trust-only would select "strategy.drools",
    // but CBR experience shifts to "strategy.early-pressure"
    // ... (mock setup matching existing test patterns)

    task.execute(ctx);

    assertThat(ctx.get(QuarkMindCaseFile.CBR_INFLUENCED_SELECTION, Boolean.class))
            .isPresent()
            .hasValue(true);
}

@Test
void cbrInfluencedSelection_falseWhenCbrAgreesWithTrust() {
    // Set up: trust-only and CBR blended both select "strategy.drools"
    // ... (mock setup)

    task.execute(ctx);

    assertThat(ctx.get(QuarkMindCaseFile.CBR_INFLUENCED_SELECTION, Boolean.class))
            .isPresent()
            .hasValue(false);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=SC2StrategyRouterTaskTest#cbrInfluencedSelection_trueWhenCbrChangesWinner -q`
Expected: FAIL — CBR_INFLUENCED_SELECTION not written

- [ ] **Step 3: Implement CBR influence tracking**

In `SC2StrategyRouterTask.execute()`, after the `routingStrategy.select()` call returns the winner, compute the trust-only winner and compare:

```java
// After: String winner = selection.bindingNames().get(0);
// Compute trust-only winner for influence tracking
String trustOnlyWinner = candidates.stream()
        .max(Comparator.comparingDouble(c ->
                routingStrategy.trustScoreFor(c.bindingName())))
        .map(ImplementationCandidate::bindingName)
        .orElse(FALLBACK);
ctx.set(QuarkMindCaseFile.CBR_INFLUENCED_SELECTION, !winner.equals(trustOnlyWinner));
```

Note: `routingStrategy.trustScoreFor()` may not exist as a public method. If not, track the trust scores during the select() call. The exact implementation depends on what `SC2ImplementationRoutingStrategy` exposes. Check the class structure and adapt — the key requirement is comparing the CBR-blended winner vs what trust-only would have selected.

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -Dtest=SC2StrategyRouterTaskTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/agent/cbr/SC2StrategyRouterTask.java src/test/java/io/quarkmind/agent/cbr/SC2StrategyRouterTaskTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#221): CBR influence tracking — records whether CBR shifted strategy selection

Refs #221"
```

---

### Task 6: SC2CbrRetentionObserver — bus subscriptions and enriched case building

This is the core task. Depends on Tasks 1-5.

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserver.java`
- Modify: `src/test/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserverTest.java`

**Interfaces:**
- Consumes: `EnrichedGameData` and `SC2GameCbrCase.buildForGameEnriched()` (from Task 1)
- Consumes: `SummarisationLifecycle.phaseBus()`, `MomentBroker.momentBus()`, `SummarisationLifecycle.arcBus()`
- Consumes: `MultiFactorDominanceAssessor.assess(GameState)` → `DominanceScore`
- Consumes: CaseFile keys from Tasks 2-5 (SUPPLY_BLOCK moments, SCOUTING_DISPATCH_FRAME, OPPONENT_ID)
- Produces: Enriched `SC2GameCbrCase` stored via `cbrStore.store()`

- [ ] **Step 1: Write the failing test — bus accumulation and GameStarted reset**

Add to `SC2CbrRetentionObserverTest.java`:

```java
@Test
void accumulatesMoments() {
    observer.collectMoment(new LevelEvent<>(
            new GameMoment(GameMomentType.BATTLE_STARTED, 100, Map.of()),
            100, MomentDetectionTask.LEVEL_2));
    observer.collectMoment(new LevelEvent<>(
            new GameMoment(GameMomentType.FIRST_CONTACT, 50, Map.of()),
            50, MomentDetectionTask.LEVEL_2));

    assertThat(observer.moments()).hasSize(2);
}

@Test
void gameStarted_clearsAccumulators() {
    observer.collectMoment(new LevelEvent<>(
            new GameMoment(GameMomentType.BATTLE_STARTED, 100, Map.of()),
            100, MomentDetectionTask.LEVEL_2));
    observer.collectPhase(new LevelEvent<>(
            new TacticalPosture("MID_SKIRMISH", 100, "test"),
            100, new EventLevel("phase", 3)));

    observer.onGameStarted(new GameStarted());

    assertThat(observer.moments()).isEmpty();
    assertThat(observer.phases()).isEmpty();
    assertThat(observer.latestArc()).isNull();
}
```

- [ ] **Step 2: Write the failing test — enriched onOutcome**

```java
@Test
void onOutcome_buildsEnrichedCase() {
    // Accumulate some data
    observer.collectMoment(new LevelEvent<>(
            new GameMoment(GameMomentType.FIRST_CONTACT, 2800, Map.of()),
            2800, MomentDetectionTask.LEVEL_2));
    observer.collectMoment(new LevelEvent<>(
            new GameMoment(GameMomentType.BATTLE_STARTED, 5000, Map.of()),
            5000, MomentDetectionTask.LEVEL_2));
    observer.collectMoment(new LevelEvent<>(
            new GameMoment(GameMomentType.SUPPLY_BLOCK, 3000, Map.of()),
            3000, MomentDetectionTask.LEVEL_2));
    observer.collectPhase(new LevelEvent<>(
            new TacticalPosture("EARLY_MACRO", 0, "no combat"),
            0, new EventLevel("phase", 3)));
    observer.collectPhase(new LevelEvent<>(
            new TacticalPosture("MID_SKIRMISH", 5000, "combat"),
            5000, new EventLevel("phase", 3)));
    observer.collectArc(new LevelEvent<>(
            new GameArc("Game progression: EARLY_MACRO -> MID_SKIRMISH", 5000),
            5000, new EventLevel("arc", 4)));

    // Build CaseFile snapshot
    GameState gameState = buildGameState(2, 30, 8000); // 2 expansions, 30 workers, frame 8000
    Map<String, Object> snapshot = Map.of(
            QuarkMindCaseFile.STRATEGY_ROUTED_ARCHETYPE, "ZERG_ROACH_RUSH",
            QuarkMindCaseFile.STRATEGY_SELECTED_ID, "strategy.early-pressure",
            QuarkMindCaseFile.STRATEGY_ROUTED_CONFIDENCE, 0.85,
            QuarkMindCaseFile.GAME_STATE, gameState,
            QuarkMindCaseFile.OPPONENT_ID, "ZERG_ROACH_RUSH",
            QuarkMindCaseFile.SCOUTING_DISPATCH_FRAME, 1500L);

    // Mock dominance assessor
    when(dominanceAssessor.assess(gameState))
            .thenReturn(new DominanceScore(0.3, Map.of(
                    "economy", 0.5, "army", 0.4, "tech", 0.2, "bases", 0.1)));

    var event = buildOutcomeEvent("WIN", snapshot);
    observer.onOutcome(event);

    // Verify stored case has enrichment features
    var captor = ArgumentCaptor.forClass(SC2GameCbrCase.class);
    verify(cbrStore).store(captor.capture(), any(), any(), any(), any(), any(), any());
    var storedCase = captor.getValue();

    assertThat(storedCase.features()).containsKey("phase_sequence");
    assertThat(storedCase.features()).containsEntry("moment_count", FeatureValue.number(3));
    assertThat(storedCase.features()).containsEntry("battle_count", FeatureValue.number(1));
    assertThat(storedCase.features()).containsEntry("supply_block_count", FeatureValue.number(1));
    assertThat(storedCase.features()).containsEntry("dominance_overall", FeatureValue.number(0.3));
    assertThat(storedCase.features()).containsEntry("expansion_count", FeatureValue.number(2));
    assertThat(storedCase.features()).containsEntry("worker_count_final", FeatureValue.number(30));
    assertThat(storedCase.features()).containsEntry("opponent_id", FeatureValue.string("ZERG_ROACH_RUSH"));
    assertThat(storedCase.features()).containsKey("first_contact_minute");
    assertThat(storedCase.features()).containsKey("scout_dispatch_minute");
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -Dtest=SC2CbrRetentionObserverTest -q`
Expected: FAIL — accumulation methods and enriched onOutcome don't exist

- [ ] **Step 4: Implement bus subscriptions and accumulation**

Modify `SC2CbrRetentionObserver.java`:

Add new injections:

```java
@Inject SummarisationLifecycle summarisationLifecycle;
@Inject MomentBroker momentBroker;
@Inject MultiFactorDominanceAssessor dominanceAssessor;
```

Add accumulated state:

```java
private final List<GameMoment> moments = new CopyOnWriteArrayList<>();
private final List<TacticalPosture> phases = new CopyOnWriteArrayList<>();
private final AtomicReference<GameArc> latestArc = new AtomicReference<>();
```

Add `@PostConstruct` for bus subscriptions:

```java
@PostConstruct
void subscribeToBuses() {
    momentBroker.momentBus().subscribe(e -> true, this::collectMoment);
    summarisationLifecycle.phaseBus().subscribe(e -> true, this::collectPhase);
    summarisationLifecycle.arcBus().subscribe(e -> true, this::collectArc);
}
```

Add collector methods:

```java
void collectMoment(LevelEvent<GameMoment> event) {
    moments.add(event.payload());
}

void collectPhase(LevelEvent<TacticalPosture> event) {
    phases.add(event.payload());
}

void collectArc(LevelEvent<GameArc> event) {
    latestArc.set(event.payload());
}

List<GameMoment> moments() { return List.copyOf(moments); }
List<TacticalPosture> phases() { return List.copyOf(phases); }
GameArc latestArc() { return latestArc.get(); }
```

Add GameStarted handler:

```java
void onGameStarted(@Observes GameStarted event) {
    moments.clear();
    phases.clear();
    latestArc.set(null);
}
```

- [ ] **Step 5: Implement enriched onOutcome**

Replace the existing `onOutcome()` body. After the existing Tier 1 extraction (archetype, strategy, confidence, raceName, matchup), add:

```java
// Tier 2 enrichment
GameState gameState = (GameState) snapshot.get(QuarkMindCaseFile.GAME_STATE);
DominanceScore dominance = gameState != null
        ? dominanceAssessor.assess(gameState) : DominanceScore.NEUTRAL;

List<String> phaseSequence = phases.stream()
        .map(TacticalPosture::posture).toList();
int momentCount = moments.size();
int battleCount = (int) moments.stream()
        .filter(m -> m.type() == GameMomentType.BATTLE_STARTED).count();
int supplyBlockCount = (int) moments.stream()
        .filter(m -> m.type() == GameMomentType.SUPPLY_BLOCK).count();

OptionalDouble firstContactMinute = moments.stream()
        .filter(m -> m.type() == GameMomentType.FIRST_CONTACT)
        .mapToDouble(m -> m.gameFrame() / SC2Data.GAME_LOOPS_PER_SECOND / 60.0)
        .findFirst();

Long scoutFrame = (Long) snapshot.get(QuarkMindCaseFile.SCOUTING_DISPATCH_FRAME);
OptionalDouble scoutDispatchMinute = scoutFrame != null
        ? OptionalDouble.of(scoutFrame / SC2Data.GAME_LOOPS_PER_SECOND / 60.0)
        : OptionalDouble.empty();

int expansionCount = gameState != null
        ? (int) gameState.myBuildings().stream().filter(b -> isBase(b.type())).count() : 0;
int workerCountFinal = gameState != null
        ? (int) gameState.myUnits().stream().filter(u -> u.type().isWorker()).count() : 0;

GameArc arc = latestArc.get();
String arcNarrative = arc != null ? arc.narrative() : "";
double gameDurationMinutes = gameState != null ? gameState.gameTimeMinutes() : 0.0;
String opponentId = (String) snapshot.getOrDefault(QuarkMindCaseFile.OPPONENT_ID, "unknown");

// CBR influence tracking — read from CaseFile snapshot, include as case feature
Boolean cbrInfluenced = (Boolean) snapshot.get(QuarkMindCaseFile.CBR_INFLUENCED_SELECTION);

var enrichment = new EnrichedGameData(
        phaseSequence, momentCount, arcNarrative, gameDurationMinutes,
        battleCount, dominance.factors().getOrDefault("army", 0.0), dominance.overall(),
        expansionCount, workerCountFinal,
        dominance.factors().getOrDefault("economy", 0.0), supplyBlockCount,
        firstContactMinute, scoutDispatchMinute,
        confidence != null ? confidence : 0.0,
        opponentId);

SC2GameCbrCase cbrCase = SC2GameCbrCase.buildForGameEnriched(
        archetype, raceName, matchup,
        confidence != null ? confidence : 0.0, strategyId, enrichment);

// Add CBR influence as a case feature (not in EnrichedGameData to avoid circular dependency)
if (cbrInfluenced != null) {
    var enrichedFeatures = new java.util.HashMap<>(cbrCase.features());
    enrichedFeatures.put("cbr_influenced", FeatureValue.string(cbrInfluenced.toString()));
    cbrCase = (SC2GameCbrCase) cbrCase.withFeatures(enrichedFeatures);
}

cbrCase = (SC2GameCbrCase) cbrCase.withOutcome(event.outcomeLabel(), null);
```

Add the `isBase()` helper:

```java
private static boolean isBase(BuildingType type) {
    return type == BuildingType.NEXUS || type == BuildingType.HATCHERY
            || type == BuildingType.LAIR || type == BuildingType.HIVE
            || type == BuildingType.COMMAND_CENTER || type == BuildingType.ORBITAL_COMMAND
            || type == BuildingType.PLANETARY_FORTRESS;
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn test -Dtest=SC2CbrRetentionObserverTest -q`
Expected: PASS

- [ ] **Step 7: Run full test suite**

Run: `mvn test -q`
Expected: PASS — no regressions

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserver.java src/test/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserverTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#215): SC2CbrRetentionObserver enrichment — bus subscriptions, L2/L3/L4 accumulation, enriched case building

Refs #215, #217, #218, #219, #220"
```

---

### Task 7: QA endpoints — learning curve observability

**Files:**
- Create: `src/main/java/io/quarkmind/qa/CbrLearningCurveEndpoint.java`
- Create: `src/test/java/io/quarkmind/qa/CbrLearningCurveEndpointTest.java`

**Interfaces:**
- Consumes: `CbrCaseMemoryStore.scan()` or `retrieveSimilar()` with broad query
- Consumes: `QuarkMindCaseFile.CBR_INFLUENCED_SELECTION` (from Task 5, via stored case features or CaseFile scan)
- Produces: JSON responses on `/qa/cbr/learning-curve`, `/qa/cbr/strategy-evolution`, `/qa/cbr/case-stats`

- [ ] **Step 1: Verify CbrCaseMemoryStore.scan() availability**

Use IntelliJ to check if `InMemoryCbrCaseMemoryStore` overrides `scan()`:

```
ide_find_references on CbrCaseMemoryStore.scan
```

If `scan()` is not implemented, use a broad `retrieveSimilar()` query as fallback:
```java
var query = CbrQuery.of(tenantId, DOMAIN, Path.of("quarkmind"), SC2GameCbrCase.CBR_TYPE,
        Map.of(), 1000).withMinSimilarity(0.0);
```

- [ ] **Step 2: Write the failing test**

```java
@Test
void learningCurve_computesWinRates() {
    // Pre-load store with 5 cases: 3 wins, 2 losses
    storeCases(List.of("WIN", "WIN", "LOSS", "WIN", "LOSS"));

    var response = endpoint.learningCurve();

    assertThat(response.getStatus()).isEqualTo(200);
    var body = response.readEntity(JsonObject.class);
    assertThat(body.getInt("totalGames")).isEqualTo(5);
    assertThat(body.getJsonNumber("overallWinRate").doubleValue()).isCloseTo(0.6, within(0.01));
}

@Test
void learningCurve_emptyStoreReturnsZeros() {
    var response = endpoint.learningCurve();
    var body = response.readEntity(JsonObject.class);
    assertThat(body.getInt("totalGames")).isEqualTo(0);
    assertThat(body.getJsonNumber("overallWinRate").doubleValue()).isEqualTo(0.0);
}

@Test
void strategyEvolution_groupsByStrategy() {
    // Pre-load: 3 games with strategy A, 2 with strategy B
    storeCasesWithStrategies(Map.of("strategy.early-pressure", List.of("WIN", "WIN", "LOSS"),
                                     "strategy.drools", List.of("LOSS", "WIN")));

    var response = endpoint.strategyEvolution();
    var body = response.readEntity(JsonObject.class);
    var strategies = body.getJsonArray("strategies");
    assertThat(strategies).hasSize(2);
}

@Test
void caseStats_reportsInfluenceRate() {
    // Pre-load: 5 cases, 2 with CBR_INFLUENCED_SELECTION=true
    storeCasesWithInfluence(5, 2);

    var response = endpoint.caseStats();
    var body = response.readEntity(JsonObject.class);
    assertThat(body.getJsonNumber("retrievalInfluenceRate").doubleValue())
            .isCloseTo(0.4, within(0.01));
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -Dtest=CbrLearningCurveEndpointTest -q`
Expected: FAIL — class does not exist

- [ ] **Step 4: Implement CbrLearningCurveEndpoint**

Create `src/main/java/io/quarkmind/qa/CbrLearningCurveEndpoint.java`:

```java
package io.quarkmind.qa;

import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.*;
import io.casehub.platform.api.path.Path;
import io.quarkus.arc.profile.UnlessBuildProfile;
import io.quarkmind.agent.cbr.SC2GameCbrCase;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import java.util.*;
import java.util.stream.Collectors;

@UnlessBuildProfile("prod")
@Path("/qa/cbr")
@Produces(MediaType.APPLICATION_JSON)
@ApplicationScoped
public class CbrLearningCurveEndpoint {

    private static final MemoryDomain DOMAIN = new MemoryDomain("quarkmind");

    @Inject CbrCaseMemoryStore cbrStore;

    @GET @Path("/learning-curve")
    public Response learningCurve() {
        var cases = retrieveAllCases();
        if (cases.isEmpty()) {
            return Response.ok(Map.of("totalGames", 0, "overallWinRate", 0.0,
                    "last10WinRate", 0.0, "last20WinRate", 0.0,
                    "trend", "STABLE", "perMatchup", Map.of())).build();
        }
        // Sort by storedAt
        cases.sort(Comparator.comparing(ScoredCbrCase::storedAt));

        double overallWinRate = winRate(cases);
        double last10 = winRate(tail(cases, 10));
        double last20 = winRate(tail(cases, 20));
        double first10 = winRate(head(cases, 10));
        String trend = last10 - first10 > 0.1 ? "IMPROVING"
                : first10 - last10 > 0.1 ? "DECLINING" : "STABLE";

        var perMatchup = cases.stream()
                .collect(Collectors.groupingBy(c ->
                        c.cbrCase().features().getOrDefault("matchup",
                                FeatureValue.string("unknown")).toString()))
                .entrySet().stream()
                .collect(Collectors.toMap(Map.Entry::getKey,
                        e -> Map.of("games", e.getValue().size(),
                                "winRate", winRate(e.getValue()))));

        return Response.ok(Map.of("totalGames", cases.size(),
                "overallWinRate", overallWinRate,
                "last10WinRate", last10, "last20WinRate", last20,
                "trend", trend, "perMatchup", perMatchup)).build();
    }

    @GET @Path("/strategy-evolution")
    public Response strategyEvolution() {
        var cases = retrieveAllCases();
        var byStrategy = cases.stream()
                .collect(Collectors.groupingBy(c -> c.cbrCase().solution()));

        var strategies = byStrategy.entrySet().stream().map(e -> {
            var strategyCases = e.getValue();
            var perArchetype = strategyCases.stream()
                    .collect(Collectors.groupingBy(c ->
                            c.cbrCase().features().getOrDefault("enemy_archetype",
                                    FeatureValue.string("unknown")).toString()))
                    .entrySet().stream()
                    .collect(Collectors.toMap(Map.Entry::getKey,
                            ae -> Map.of("games", ae.getValue().size(),
                                    "winRate", winRate(ae.getValue()))));
            return Map.of("strategyId", e.getKey(),
                    "selectionRate", cases.isEmpty() ? 0.0 : (double) strategyCases.size() / cases.size(),
                    "winRate", winRate(strategyCases),
                    "games", strategyCases.size(),
                    "perArchetype", perArchetype);
        }).toList();

        return Response.ok(Map.of("strategies", strategies)).build();
    }

    @GET @Path("/case-stats")
    public Response caseStats() {
        var strategyCases = retrieveAllCases();
        // Advisory cases retrieved separately
        int tier2Count = (int) strategyCases.stream()
                .filter(c -> c.cbrCase().features().containsKey("moment_count"))
                .count();
        double tier2Coverage = strategyCases.isEmpty() ? 0.0
                : (double) tier2Count / strategyCases.size();

        long influencedCount = strategyCases.stream()
                .filter(c -> {
                    var v = c.cbrCase().features().get("cbr_influenced");
                    return v != null && "true".equals(v.toString());
                }).count();
        double influenceRate = strategyCases.isEmpty() ? 0.0
                : (double) influencedCount / strategyCases.size();

        var perOpponent = strategyCases.stream()
                .filter(c -> c.cbrCase().features().containsKey("opponent_id"))
                .collect(Collectors.groupingBy(c ->
                        c.cbrCase().features().get("opponent_id").toString(),
                        Collectors.counting()));

        return Response.ok(Map.of(
                "totalCases", strategyCases.size(),
                "tier2Coverage", tier2Coverage,
                "retrievalInfluenceRate", influenceRate,
                "perOpponent", perOpponent)).build();
    }

    private List<ScoredCbrCase<SC2GameCbrCase>> retrieveAllCases() {
        // Use retrieveSimilar with minimal filter as scan() fallback
        var query = CbrQuery.of("default", DOMAIN,
                Path.of("quarkmind", "strategy", "cases"),
                SC2GameCbrCase.CBR_TYPE, Map.of(), 1000)
                .withMinSimilarity(0.0)
                .withRetrievalMode(RetrievalMode.FEATURE_ONLY);
        return new ArrayList<>(cbrStore.retrieveSimilar(query, SC2GameCbrCase.class));
    }

    private static double winRate(List<? extends ScoredCbrCase<?>> cases) {
        if (cases.isEmpty()) return 0.0;
        long wins = cases.stream().filter(c -> "WIN".equals(c.cbrCase().outcome())).count();
        return (double) wins / cases.size();
    }

    private static <T> List<T> tail(List<T> list, int n) {
        return list.subList(Math.max(0, list.size() - n), list.size());
    }

    private static <T> List<T> head(List<T> list, int n) {
        return list.subList(0, Math.min(n, list.size()));
    }
}
```

Note: `cbr_influenced` is added as a feature by `SC2CbrRetentionObserver.onOutcome()` after building the enriched case (Task 6, Step 5). It reads `CBR_INFLUENCED_SELECTION` from the CaseFile snapshot and adds it as a string feature ("true"/"false") via `withFeatures()`.

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -Dtest=CbrLearningCurveEndpointTest -q`
Expected: PASS

- [ ] **Step 6: Run full test suite**

Run: `mvn test -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/qa/CbrLearningCurveEndpoint.java src/test/java/io/quarkmind/qa/CbrLearningCurveEndpointTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#221): CBR learning curve QA endpoints — win rate, strategy evolution, case stats

Refs #221"
```

---

### Task 8: Integration tests

**Files:**
- Modify: `src/test/java/io/quarkmind/agent/cbr/SC2CbrRetentionIT.java`
- Create: `src/test/java/io/quarkmind/qa/CbrLearningCurveIT.java`

**Interfaces:**
- Consumes: All production code from Tasks 1-7
- Produces: Full lifecycle verification

- [ ] **Step 1: Extend SC2CbrRetentionIT**

Add a test that runs a full game lifecycle and verifies the stored case contains enrichment features:

```java
@Test
void enrichedCaseStoredAfterGameLifecycle() {
    // Run several game ticks to generate moments, phases
    for (int i = 0; i < 20; i++) {
        orchestrator.gameTick();
    }
    // Stop game
    orchestrator.stopGame("WIN");

    // Verify the stored case has Tier 2 features
    var cases = cbrStore.retrieveSimilar(
            CbrQuery.of("default", DOMAIN, Path.of("quarkmind", "strategy", "cases"),
                    SC2GameCbrCase.CBR_TYPE, Map.of(), 10)
                    .withMinSimilarity(0.0),
            SC2GameCbrCase.class);
    assertThat(cases).isNotEmpty();
    var stored = cases.get(0).cbrCase();
    assertThat(stored.features()).containsKey("moment_count");
    assertThat(stored.features()).containsKey("expansion_count");
    assertThat(stored.features()).containsKey("worker_count_final");
    assertThat(stored.features()).containsKey("opponent_id");
    assertThat(stored.features()).containsKey("game_duration_minutes");
}
```

- [ ] **Step 2: Create CbrLearningCurveIT**

```java
@QuarkusTest
class CbrLearningCurveIT {

    @Inject CbrCaseMemoryStore cbrStore;

    @Test
    void learningCurveEndpoint_returnsJson() {
        given()
            .when().get("/qa/cbr/learning-curve")
            .then()
            .statusCode(200)
            .body("totalGames", is(0))
            .body("overallWinRate", is(0.0f));
    }

    @Test
    void strategyEvolutionEndpoint_returnsJson() {
        given()
            .when().get("/qa/cbr/strategy-evolution")
            .then()
            .statusCode(200)
            .body("strategies", hasSize(0));
    }

    @Test
    void caseStatsEndpoint_returnsJson() {
        given()
            .when().get("/qa/cbr/case-stats")
            .then()
            .statusCode(200)
            .body("totalCases", is(0));
    }
}
```

- [ ] **Step 3: Run integration tests**

Run: `mvn test -Dtest=SC2CbrRetentionIT,CbrLearningCurveIT -q`
Expected: PASS

- [ ] **Step 4: Run full test suite including benchmarks**

Run: `mvn test -q`
Expected: PASS — no regressions

Run: `mvn test -Pbenchmark -Dtest=GameLoopBenchmarkTest -q`
Expected: PASS — bus subscriptions add no measurable tick latency

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/test/java/io/quarkmind/agent/cbr/SC2CbrRetentionIT.java src/test/java/io/quarkmind/qa/CbrLearningCurveIT.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "test(#214): CBR Tier 2 integration tests — enriched retention lifecycle, QA endpoints

Refs #215, #221"
```
