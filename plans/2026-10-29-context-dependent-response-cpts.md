# Context-Dependent Response CPTs Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #469 — Phase 5: Context-dependent response CPTs
**Issue group:** #469

**Goal:** Replace the uniformly tiled response CPT with context-dependent values learned from 149K labelled replays, so strategy recommendations adapt to the player's economy/army/tech state.

**Architecture:** Extend `PriorExportPipeline` with a `ContextResponseOutcome` record and `learnContextResponsePriors()` using hierarchical smoothing against the Phase 4 unconditional prior. Add `buildContextResponseCpt()` to `SC2HypothesisNetworkBuilder` that populates different CPT rows per context combination. Wire through `HypothesisTrackerFusion.initialize()` with backward-compatible fallback. Generate real data via a report test that processes reconstituted replay JSON at 3-min and 5-min checkpoints.

**Tech Stack:** Java 21, JUnit 5, Jackson JSON, Quarkus (test profile only for calibration)

## Global Constraints

- Context variable states: economy=behind/even/ahead, army=smaller/even/larger, tech=behind/even/ahead
- Discretization thresholds match `DroolsScoutingTask.deriveGameStateContext()`: income ratio 0.8/1.2, army ratio 0.7/1.3, tech count comparison
- Tech buildings: 24 types from `DroolsScoutingTask.TECH_BUILDINGS`
- Context key format: `"<economy>:<army>:<tech>"` e.g. `"behind:even:ahead"`
- 27 context combinations: economy × army × tech (3 × 3 × 3)
- Combination index: `e * 9 + a * 3 + t` where e/a/t ∈ {0,1,2}
- State ordering: economy=[behind,even,ahead], army=[smaller,even,larger], tech=[behind,even,ahead]
- Hierarchical smoothing: `P_smoothed = (n / (n + α)) * P_cell + (α / (n + α)) * P_marginal`, default α=10.0
- Checkpoints: game loop 4032 (3-min) and 6720 (5-min)
- PlayerStats values are ×1000 encoded in tracker events (divide by 1000)
- Laplace smoothing constant: 1.0 (existing `PriorExportPipeline.SMOOTHING`)
- CPT floor/ceiling: 0.01/0.99 (existing `PriorExportPipeline.CPT_FLOOR/CPT_CEILING`)

---

## Batch 1: Learning Pipeline

### Task 1: ContextResponseOutcome and discretizeContext

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/PriorExportPipeline.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/PriorExportPipelineTest.java`

**Interfaces:**
- Consumes: existing `PriorExportPipeline.SMOOTHING` constant
- Produces:
  - `PriorExportPipeline.ContextResponseOutcome` record with fields `(String ourStrategy, String opponentStrategy, String matchup, boolean won, String economyState, String armyState, String techState)`
  - `PriorExportPipeline.discretizeContext(double ourIncome, double enemyIncome, int ourArmyCount, int enemyArmyCount, long ourTechCount, long enemyTechCount)` → `Map<String, String>` with keys `economyState`, `armyState`, `techState`
  - `PriorExportPipeline.contextKey(String economyState, String armyState, String techState)` → `String` formatted as `"economy:army:tech"`

- [ ] **Step 1: Write failing tests for discretizeContext**

```java
// In PriorExportPipelineTest.java

@Test
void discretizeContext_economyBehind() {
    // income ratio = 700/1000 = 0.7 < 0.8 → behind
    var ctx = PriorExportPipeline.discretizeContext(700, 1000, 10, 10, 2, 2);
    assertEquals("behind", ctx.get("economyState"));
    assertEquals("even", ctx.get("armyState"));
    assertEquals("even", ctx.get("techState"));
}

@Test
void discretizeContext_economyAhead() {
    // income ratio = 1300/1000 = 1.3 > 1.2 → ahead
    var ctx = PriorExportPipeline.discretizeContext(1300, 1000, 10, 10, 2, 2);
    assertEquals("ahead", ctx.get("economyState"));
}

@Test
void discretizeContext_economyEven() {
    // income ratio = 1000/1000 = 1.0, within [0.8, 1.2] → even
    var ctx = PriorExportPipeline.discretizeContext(1000, 1000, 10, 10, 2, 2);
    assertEquals("even", ctx.get("economyState"));
}

@Test
void discretizeContext_armySmaller() {
    // army ratio = 5/10 = 0.5 < 0.7 → smaller
    var ctx = PriorExportPipeline.discretizeContext(1000, 1000, 5, 10, 2, 2);
    assertEquals("smaller", ctx.get("armyState"));
}

@Test
void discretizeContext_armyLarger() {
    // army ratio = 15/10 = 1.5 > 1.3 → larger
    var ctx = PriorExportPipeline.discretizeContext(1000, 1000, 15, 10, 2, 2);
    assertEquals("larger", ctx.get("armyState"));
}

@Test
void discretizeContext_techBehind() {
    var ctx = PriorExportPipeline.discretizeContext(1000, 1000, 10, 10, 1, 3);
    assertEquals("behind", ctx.get("techState"));
}

@Test
void discretizeContext_techAhead() {
    var ctx = PriorExportPipeline.discretizeContext(1000, 1000, 10, 10, 4, 2);
    assertEquals("ahead", ctx.get("techState"));
}

@Test
void discretizeContext_zeroEnemyIncome_defaultsEven() {
    var ctx = PriorExportPipeline.discretizeContext(1000, 0, 10, 10, 2, 2);
    assertEquals("even", ctx.get("economyState"));
}

@Test
void discretizeContext_zeroEnemyArmy_defaultsEven() {
    var ctx = PriorExportPipeline.discretizeContext(1000, 1000, 10, 0, 2, 2);
    assertEquals("even", ctx.get("armyState"));
}

@Test
void contextKey_formatsCorrectly() {
    assertEquals("behind:even:ahead",
                 PriorExportPipeline.contextKey("behind", "even", "ahead"));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PriorExportPipelineTest -q`
Expected: compilation failures — `discretizeContext` and `contextKey` not defined

- [ ] **Step 3: Implement ContextResponseOutcome, discretizeContext, contextKey**

Add to `PriorExportPipeline.java` after the existing `ResponseOutcome` record:

```java
public record ContextResponseOutcome(
        String ourStrategy,
        String opponentStrategy,
        String matchup,
        boolean won,
        String economyState,
        String armyState,
        String techState
) {}

public static Map<String, String> discretizeContext(
        double ourIncome, double enemyIncome,
        int ourArmyCount, int enemyArmyCount,
        long ourTechCount, long enemyTechCount) {
    Map<String, String> context = new LinkedHashMap<>();

    double incomeRatio = enemyIncome > 0 ? ourIncome / enemyIncome : 1.0;
    context.put("economyState",
                incomeRatio < 0.8 ? "behind" : incomeRatio > 1.2 ? "ahead" : "even");

    double armyRatio = enemyArmyCount > 0 ? (double) ourArmyCount / enemyArmyCount : 1.0;
    context.put("armyState",
                armyRatio < 0.7 ? "smaller" : armyRatio > 1.3 ? "larger" : "even");

    context.put("techState",
                ourTechCount < enemyTechCount ? "behind"
                : ourTechCount > enemyTechCount ? "ahead" : "even");

    return context;
}

public static String contextKey(String economyState, String armyState, String techState) {
    return economyState + ":" + armyState + ":" + techState;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PriorExportPipelineTest -q`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/PriorExportPipeline.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/PriorExportPipelineTest.java
git commit -m "feat(#469): add ContextResponseOutcome and discretizeContext

Refs #469"
```

### Task 2: learnContextResponsePriors with hierarchical smoothing

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/PriorExportPipeline.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/PriorExportPipelineTest.java`

**Interfaces:**
- Consumes: `ContextResponseOutcome` from Task 1, `contextKey()` from Task 1, existing `learnResponsePriors()` output as marginal
- Produces:
  - `PriorExportPipeline.learnContextResponsePriors(List<ContextResponseOutcome> outcomes, Set<String> allOurArchetypes, Map<String, Map<String, Double>> marginalPriors, double alpha)` → `Map<String, Map<String, Map<String, Double>>>` (opponent → contextKey → strategy → probability)
  - `PriorExportPipeline.exportContextResponsePriorsToJson(Path outputDir, String matchup, Map<String, Map<String, Map<String, Double>>> contextPriors, double alpha)` → void

- [ ] **Step 1: Write failing tests**

```java
// In PriorExportPipelineTest.java

@Test
void learnContextResponsePriors_dataRichCell_divergesFromMarginal() {
    var outcomes = new ArrayList<PriorExportPipeline.ContextResponseOutcome>();
    // 100 wins of GATEWAY_RUSH vs MARINE_RUSH when economy=behind, army=even, tech=even
    for (int i = 0; i < 100; i++) {
        outcomes.add(new PriorExportPipeline.ContextResponseOutcome(
                "GATEWAY_RUSH", "MARINE_RUSH", "vs_terran", true,
                "behind", "even", "even"));
    }
    // 10 wins of MACRO vs MARINE_RUSH in same context
    for (int i = 0; i < 10; i++) {
        outcomes.add(new PriorExportPipeline.ContextResponseOutcome(
                "MACRO", "MARINE_RUSH", "vs_terran", true,
                "behind", "even", "even"));
    }
    var allOur = Set.of("GATEWAY_RUSH", "MACRO", "CANNON_RUSH");
    var marginal = Map.of("MARINE_RUSH",
            Map.of("GATEWAY_RUSH", 0.33, "MACRO", 0.34, "CANNON_RUSH", 0.33));

    var result = PriorExportPipeline.learnContextResponsePriors(outcomes, allOur, marginal, 10.0);

    var cell = result.get("MARINE_RUSH").get("behind:even:even");
    assertTrue(cell.get("GATEWAY_RUSH") > cell.get("MACRO"),
               "GATEWAY_RUSH should dominate in data-rich cell");
    assertTrue(cell.get("GATEWAY_RUSH") > marginal.get("MARINE_RUSH").get("GATEWAY_RUSH"),
               "Data-rich cell should diverge from marginal");
    assertEquals(1.0, cell.values().stream().mapToDouble(d -> d).sum(), 0.01);
}

@Test
void learnContextResponsePriors_sparseCell_regressesTowardMarginal() {
    var outcomes = List.of(
            new PriorExportPipeline.ContextResponseOutcome(
                    "CANNON_RUSH", "MARINE_RUSH", "vs_terran", true,
                    "ahead", "larger", "ahead"));
    var allOur = Set.of("GATEWAY_RUSH", "MACRO", "CANNON_RUSH");
    var marginal = Map.of("MARINE_RUSH",
            Map.of("GATEWAY_RUSH", 0.5, "MACRO", 0.4, "CANNON_RUSH", 0.1));

    var result = PriorExportPipeline.learnContextResponsePriors(outcomes, allOur, marginal, 10.0);

    var cell = result.get("MARINE_RUSH").get("ahead:larger:ahead");
    // With n=1 and alpha=10, marginal weight = 10/11 ≈ 91%
    // GATEWAY_RUSH should be close to marginal (0.5) even though no cell data supports it
    assertTrue(cell.get("GATEWAY_RUSH") > 0.3,
               "Sparse cell should regress toward marginal where GATEWAY_RUSH=0.5");
}

@Test
void learnContextResponsePriors_emptyCell_notPresent() {
    var outcomes = List.of(
            new PriorExportPipeline.ContextResponseOutcome(
                    "GATEWAY_RUSH", "MARINE_RUSH", "vs_terran", true,
                    "even", "even", "even"));
    var allOur = Set.of("GATEWAY_RUSH", "MACRO");
    var marginal = Map.of("MARINE_RUSH",
            Map.of("GATEWAY_RUSH", 0.6, "MACRO", 0.4));

    var result = PriorExportPipeline.learnContextResponsePriors(outcomes, allOur, marginal, 10.0);

    // "behind:smaller:behind" has no outcomes — should not be present
    assertFalse(result.get("MARINE_RUSH").containsKey("behind:smaller:behind"));
}

@Test
void learnContextResponsePriors_lossesExcluded() {
    var outcomes = List.of(
            new PriorExportPipeline.ContextResponseOutcome(
                    "GATEWAY_RUSH", "MARINE_RUSH", "vs_terran", true,
                    "even", "even", "even"),
            new PriorExportPipeline.ContextResponseOutcome(
                    "CANNON_RUSH", "MARINE_RUSH", "vs_terran", false,
                    "even", "even", "even"));
    var allOur = Set.of("GATEWAY_RUSH", "CANNON_RUSH");
    var marginal = Map.of("MARINE_RUSH",
            Map.of("GATEWAY_RUSH", 0.5, "CANNON_RUSH", 0.5));

    var result = PriorExportPipeline.learnContextResponsePriors(outcomes, allOur, marginal, 10.0);

    var cell = result.get("MARINE_RUSH").get("even:even:even");
    // Only the win should count
    assertTrue(cell.get("GATEWAY_RUSH") > cell.get("CANNON_RUSH"));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PriorExportPipelineTest -q`
Expected: compilation failures — `learnContextResponsePriors` not defined

- [ ] **Step 3: Implement learnContextResponsePriors and exportContextResponsePriorsToJson**

Add to `PriorExportPipeline.java`:

```java
public static Map<String, Map<String, Map<String, Double>>> learnContextResponsePriors(
        List<ContextResponseOutcome> outcomes, Set<String> allOurArchetypes,
        Map<String, Map<String, Double>> marginalPriors, double alpha) {

    var wins = outcomes.stream().filter(ContextResponseOutcome::won).toList();

    // Group wins by (opponent, contextKey)
    Map<String, Map<String, List<ContextResponseOutcome>>> byOpponentAndContext = new LinkedHashMap<>();
    for (var w : wins) {
        String ctxKey = contextKey(w.economyState(), w.armyState(), w.techState());
        byOpponentAndContext
                .computeIfAbsent(w.opponentStrategy(), k -> new LinkedHashMap<>())
                .computeIfAbsent(ctxKey, k -> new ArrayList<>())
                .add(w);
    }

    Map<String, Map<String, Map<String, Double>>> result = new LinkedHashMap<>();
    for (var opEntry : byOpponentAndContext.entrySet()) {
        String opponent = opEntry.getKey();
        Map<String, Double> marginal = marginalPriors.getOrDefault(opponent, Map.of());

        Map<String, Map<String, Double>> contextMap = new LinkedHashMap<>();
        for (var ctxEntry : opEntry.getValue().entrySet()) {
            String ctxKey = ctxEntry.getKey();
            List<ContextResponseOutcome> cellWins = ctxEntry.getValue();

            // Cell-specific counts with Laplace smoothing
            Map<String, Double> counts = new LinkedHashMap<>();
            for (String arch : allOurArchetypes) {
                counts.put(arch, SMOOTHING);
            }
            for (var w : cellWins) {
                counts.merge(w.ourStrategy(), 1.0, Double::sum);
            }
            double cellTotal = counts.values().stream().mapToDouble(d -> d).sum();
            Map<String, Double> pCell = new LinkedHashMap<>();
            for (var e : counts.entrySet()) {
                pCell.put(e.getKey(), e.getValue() / cellTotal);
            }

            // Hierarchical blend: (n/(n+α)) * P_cell + (α/(n+α)) * P_marginal
            int n = cellWins.size();
            double cellWeight = (double) n / (n + alpha);
            double marginalWeight = alpha / (n + alpha);

            Map<String, Double> blended = new LinkedHashMap<>();
            for (String arch : allOurArchetypes) {
                double pc = pCell.getOrDefault(arch, 1.0 / allOurArchetypes.size());
                double pm = marginal.getOrDefault(arch, 1.0 / allOurArchetypes.size());
                blended.put(arch, cellWeight * pc + marginalWeight * pm);
            }

            // Re-normalize
            double sum = blended.values().stream().mapToDouble(d -> d).sum();
            blended.replaceAll((k, v) -> v / sum);

            contextMap.put(ctxKey, blended);
        }
        result.put(opponent, contextMap);
    }
    return result;
}

public static void exportContextResponsePriorsToJson(
        Path outputDir, String matchup,
        Map<String, Map<String, Map<String, Double>>> contextPriors,
        double alpha) throws IOException {
    Files.createDirectories(outputDir);
    Map<String, Object> export = new LinkedHashMap<>();
    export.put("alpha", alpha);
    export.put("priors", contextPriors);
    MAPPER.writerWithDefaultPrettyPrinter().writeValue(
            outputDir.resolve("response-cpts-context-" + matchup + ".json").toFile(),
            export);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PriorExportPipelineTest -q`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/PriorExportPipeline.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/PriorExportPipelineTest.java
git commit -m "feat(#469): hierarchical smoothing for context-dependent response CPTs

Refs #469"
```

---

## Batch 2: Network Integration

### Task 3: buildContextResponseCpt and loadContextResponsePriors

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/SC2HypothesisNetworkBuilder.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/HypothesisPriorLoader.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/SC2HypothesisNetworkBuilderTest.java`

**Interfaces:**
- Consumes: `PriorExportPipeline.contextKey()` from Task 1
- Produces:
  - `HypothesisPriorLoader.loadContextResponsePriors(Race enemyRace)` → `Map<String, Map<String, Map<String, Double>>>` (opponent → contextKey → strategy → probability). Returns `null` if the resource file doesn't exist (fallback to tiled CPT).
  - `SC2HypothesisNetworkBuilder.buildContextResponseCpt(String[] enemyArchetypes, String[] ourStrategies, Map<String, Map<String, Map<String, Double>>> contextPriors, Map<String, Map<String, Double>> fallbackPriors, List<HypothesisPriorLoader.ExpertOverride> expertOverrides)` → `double[]`

- [ ] **Step 1: Write failing tests**

```java
// In SC2HypothesisNetworkBuilderTest.java

@Test
void buildContextResponseCpt_differentContexts_differentRows() {
    String[] enemies = {"MARINE_RUSH", "FAST_EXPAND"};
    String[] ours = {"GATEWAY_RUSH", "MACRO"};

    // Context priors: behind:even:even → GATEWAY_RUSH heavy; ahead:even:even → MACRO heavy
    var contextPriors = Map.of(
            "MARINE_RUSH", Map.of(
                    "behind:even:even", Map.of("GATEWAY_RUSH", 0.8, "MACRO", 0.2),
                    "ahead:even:even", Map.of("GATEWAY_RUSH", 0.3, "MACRO", 0.7)),
            "FAST_EXPAND", Map.of(
                    "even:even:even", Map.of("GATEWAY_RUSH", 0.5, "MACRO", 0.5)));
    var fallback = Map.of(
            "MARINE_RUSH", Map.of("GATEWAY_RUSH", 0.5, "MACRO", 0.5),
            "FAST_EXPAND", Map.of("GATEWAY_RUSH", 0.5, "MACRO", 0.5));

    double[] cpt = SC2HypothesisNetworkBuilder.buildContextResponseCpt(
            enemies, ours, contextPriors, fallback, List.of());

    // behind:even:even = index e=0 * 9 + a=1 * 3 + t=1 = 4 → but behind=0, even=1
    // economy behind=0, army even=1, tech even=1 → 0*9 + 1*3 + 1 = 4
    int behindIdx = (0 * 27 + 4) * 2; // enemy=0 (MARINE_RUSH), ctx=4, 2 strategies
    // economy ahead=2, army even=1, tech even=1 → 2*9 + 1*3 + 1 = 22
    int aheadIdx = (0 * 27 + 22) * 2;

    // Behind context should have GATEWAY_RUSH (index 0) > MACRO (index 1)
    assertTrue(cpt[behindIdx] > cpt[behindIdx + 1],
               "behind context: GATEWAY_RUSH should dominate");
    // Ahead context should have MACRO (index 1) > GATEWAY_RUSH (index 0)
    assertTrue(cpt[aheadIdx + 1] > cpt[aheadIdx],
               "ahead context: MACRO should dominate");
    // The two rows should be different
    assertNotEquals(cpt[behindIdx], cpt[aheadIdx]);
}

@Test
void buildContextResponseCpt_missingContext_fallsBackToUnconditional() {
    String[] enemies = {"MARINE_RUSH"};
    String[] ours = {"GATEWAY_RUSH", "MACRO"};

    // Only one context key provided — all others should fall back
    var contextPriors = Map.of(
            "MARINE_RUSH", Map.of(
                    "even:even:even", Map.of("GATEWAY_RUSH", 0.9, "MACRO", 0.1)));
    var fallback = Map.of(
            "MARINE_RUSH", Map.of("GATEWAY_RUSH", 0.4, "MACRO", 0.6));

    double[] cpt = SC2HypothesisNetworkBuilder.buildContextResponseCpt(
            enemies, ours, contextPriors, fallback, List.of());

    // even:even:even = 1*9 + 1*3 + 1 = 13
    int evenIdx = 13 * 2;
    // behind:smaller:behind = 0*9 + 0*3 + 0 = 0
    int fallbackIdx = 0 * 2;

    // Even context should reflect context priors (GATEWAY_RUSH dominant)
    assertTrue(cpt[evenIdx] > cpt[evenIdx + 1]);
    // Fallback context should reflect fallback priors (MACRO dominant)
    assertTrue(cpt[fallbackIdx + 1] > cpt[fallbackIdx]);
}

@Test
void buildContextResponseCpt_expertOverrides_applied() {
    String[] enemies = {"CANNON_RUSH"};
    String[] ours = {"GATEWAY_RUSH", "MACRO"};

    var contextPriors = Map.of(
            "CANNON_RUSH", Map.of(
                    "even:even:even", Map.of("GATEWAY_RUSH", 0.9, "MACRO", 0.1)));
    var fallback = Map.of(
            "CANNON_RUSH", Map.of("GATEWAY_RUSH", 0.5, "MACRO", 0.5));
    var overrides = List.of(
            new HypothesisPriorLoader.ExpertOverride("CANNON_RUSH", "MACRO", 3.0, 0.05));

    double[] cpt = SC2HypothesisNetworkBuilder.buildContextResponseCpt(
            enemies, ours, contextPriors, fallback, overrides);

    // even:even:even = 13
    int evenIdx = 13 * 2;
    // Expert override boosts MACRO — even though context prior has 0.1 for MACRO,
    // the 3x multiplier should shift it
    assertTrue(cpt[evenIdx + 1] > 0.1,
               "Expert override should boost MACRO above context prior");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2HypothesisNetworkBuilderTest -q`
Expected: compilation failures — `buildContextResponseCpt` and `loadContextResponsePriors` not defined

- [ ] **Step 3: Implement loadContextResponsePriors in HypothesisPriorLoader**

Add to `HypothesisPriorLoader.java`:

```java
@SuppressWarnings("unchecked")
public static Map<String, Map<String, Map<String, Double>>> loadContextResponsePriors(Race enemyRace) {
    String matchup = "vs_" + enemyRace.name().toLowerCase();
    String path    = "priors/response-cpts-context-" + matchup + ".json";
    try (InputStream is = Thread.currentThread().getContextClassLoader().getResourceAsStream(path)) {
        if (is == null) {return null;}
        Map<String, Object> root = MAPPER.readValue(is, new TypeReference<>() {});
        Map<String, Map<String, Map<String, Double>>> priors =
                MAPPER.convertValue(root.get("priors"), new TypeReference<>() {});
        return priors;
    } catch (IOException e) {
        return null;
    }
}
```

- [ ] **Step 4: Implement buildContextResponseCpt in SC2HypothesisNetworkBuilder**

Add to `SC2HypothesisNetworkBuilder.java` after `buildTiledResponseCpt()`:

```java
static double[] buildContextResponseCpt(
        String[] enemyArchetypes, String[] ourStrategies,
        Map<String, Map<String, Map<String, Double>>> contextPriors,
        Map<String, Map<String, Double>> fallbackPriors,
        List<HypothesisPriorLoader.ExpertOverride> expertOverrides) {

    String[] economyStates = {"behind", "even", "ahead"};
    String[] armyStates    = {"smaller", "even", "larger"};
    String[] techStates    = {"behind", "even", "ahead"};
    int      contextCombinations = 27;
    double[] cpt = new double[enemyArchetypes.length * contextCombinations * ourStrategies.length];

    for (int e = 0; e < enemyArchetypes.length; e++) {
        String enemy = enemyArchetypes[e];
        Map<String, Map<String, Double>> enemyContextMap =
                contextPriors != null ? contextPriors.getOrDefault(enemy, Map.of()) : Map.of();
        Map<String, Double> fallback = fallbackPriors.getOrDefault(enemy, Map.of());

        List<HypothesisPriorLoader.ExpertOverride> matching = expertOverrides == null
                ? List.of()
                : expertOverrides.stream()
                        .filter(ov -> ov.whenOpponent().equals(enemy)).toList();

        for (int ei = 0; ei < 3; ei++) {
            for (int ai = 0; ai < 3; ai++) {
                for (int ti = 0; ti < 3; ti++) {
                    int ctxIdx = ei * 9 + ai * 3 + ti;
                    String ctxKey = PriorExportPipeline.contextKey(
                            economyStates[ei], armyStates[ai], techStates[ti]);

                    Map<String, Double> source = enemyContextMap.getOrDefault(ctxKey, fallback);
                    if (source.isEmpty()) {source = fallback;}

                    Map<String, Double> priorDist = new LinkedHashMap<>();
                    for (String our : ourStrategies) {
                        priorDist.put(our, source.getOrDefault(our, 1.0 / ourStrategies.length));
                    }
                    double sum = priorDist.values().stream().mapToDouble(d -> d).sum();
                    priorDist.replaceAll((k, v) -> v / sum);

                    if (!matching.isEmpty()) {
                        priorDist = HypothesisPriorLoader.applyExpertOverrides(priorDist, matching);
                    }

                    double[] row = new double[ourStrategies.length];
                    for (int s = 0; s < ourStrategies.length; s++) {
                        row[s] = Math.max(0.01, Math.min(0.99,
                                priorDist.getOrDefault(ourStrategies[s], 1.0 / ourStrategies.length)));
                    }
                    double rowSum = 0;
                    for (double v : row) {rowSum += v;}
                    for (int s = 0; s < row.length; s++) {row[s] /= rowSum;}

                    int offset = (e * contextCombinations + ctxIdx) * ourStrategies.length;
                    System.arraycopy(row, 0, cpt, offset, row.length);
                }
            }
        }
    }
    return cpt;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2HypothesisNetworkBuilderTest -q`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/SC2HypothesisNetworkBuilder.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/HypothesisPriorLoader.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/SC2HypothesisNetworkBuilderTest.java
git commit -m "feat(#469): context-dependent CPT builder and loader

Refs #469"
```

### Task 4: Wire context CPTs through HypothesisTrackerFusion

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/SC2HypothesisNetworkBuilder.java` (add overloaded `buildTracker`)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/HypothesisTrackerFusion.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/SC2HypothesisNetworkBuilderTest.java`

**Interfaces:**
- Consumes: `buildContextResponseCpt()` from Task 3, `loadContextResponsePriors()` from Task 3
- Produces:
  - `SC2HypothesisNetworkBuilder.buildTracker()` uses `buildContextResponseCpt()` when context priors are non-null, falls back to `buildTiledResponseCpt()` otherwise
  - `HypothesisTrackerFusion.initialize()` loads context priors and passes them through

- [ ] **Step 1: Write failing test for buildTracker with context priors**

```java
// In SC2HypothesisNetworkBuilderTest.java

@Test
void buildTracker_withContextPriors_differentContextsProduceDifferentMarginals() {
    Map<String, Double> priors = Map.of("MARINE_RUSH", 0.6, "FAST_EXPAND", 0.4);
    var obsCpts = new HypothesisPriorLoader.ObservationCpts(Map.of(), Map.of());
    Map<String, Map<String, Double>> responsePriors = Map.of(
            "MARINE_RUSH", Map.of("GATEWAY_RUSH", 0.5, "MACRO", 0.5),
            "FAST_EXPAND", Map.of("GATEWAY_RUSH", 0.5, "MACRO", 0.5));
    var contextPriors = Map.of(
            "MARINE_RUSH", Map.of(
                    "behind:even:even", Map.of("GATEWAY_RUSH", 0.9, "MACRO", 0.1),
                    "ahead:even:even", Map.of("GATEWAY_RUSH", 0.2, "MACRO", 0.8)),
            "FAST_EXPAND", Map.of(
                    "behind:even:even", Map.of("GATEWAY_RUSH", 0.4, "MACRO", 0.6),
                    "ahead:even:even", Map.of("GATEWAY_RUSH", 0.6, "MACRO", 0.4)));

    HypothesisTracker tracker = SC2HypothesisNetworkBuilder.buildTracker(
            Race.TERRAN, GamePhase.EARLY, Race.PROTOSS,
            priors, List.of(), Map.of(), obsCpts,
            responsePriors, List.of(), contextPriors);

    assertNotNull(tracker);

    // Set behind context and check marginal
    tracker.setPerTickEvidence("response", "our_economy_state", new double[]{1.0, 0.0, 0.0});
    tracker.setPerTickEvidence("response", "our_army_state", new double[]{0.0, 1.0, 0.0});
    tracker.setPerTickEvidence("response", "our_tech_state", new double[]{0.0, 1.0, 0.0});
    double[] behindMarginal = tracker.marginalize("MARINE_RUSH", "response", "our_strategy");

    // Set ahead context and check marginal
    tracker.setPerTickEvidence("response", "our_economy_state", new double[]{0.0, 0.0, 1.0});
    double[] aheadMarginal = tracker.marginalize("MARINE_RUSH", "response", "our_strategy");

    assertNotEquals(behindMarginal[0], aheadMarginal[0], 0.01,
                    "Behind vs ahead should produce different response marginals");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2HypothesisNetworkBuilderTest#buildTracker_withContextPriors_differentContextsProduceDifferentMarginals -q`
Expected: compilation failure — overloaded `buildTracker` not defined

- [ ] **Step 3: Add overloaded buildTracker accepting context priors**

Add to `SC2HypothesisNetworkBuilder.java` after the existing `buildTracker()` method:

```java
public static HypothesisTracker buildTracker(
        Race enemyRace, GamePhase phase, Race ourRace,
        Map<String, Double> learnedPriors,
        List<String> activeDetectorNames,
        Map<String, Map<String, Double>> detectorCpts,
        HypothesisPriorLoader.ObservationCpts observationCpts,
        Map<String, Map<String, Double>> responsePriors,
        List<HypothesisPriorLoader.ExpertOverride> expertOverrides,
        Map<String, Map<String, Map<String, Double>>> contextResponsePriors) {

    String[] archetypes = filteredArchetypes(enemyRace, phase, learnedPriors);
    JunctionTree intelTree = buildIntelTree(archetypes, learnedPriors,
                                            activeDetectorNames, detectorCpts, observationCpts);

    var compBuilder = CompositionBuilder.create("sc2-scouting")
                                        .slot("intel", intelTree);

    if (responsePriors != null && !responsePriors.isEmpty()) {
        String[] ourStrategies = filteredOurArchetypes(responsePriors);
        double[] enemyPriors   = normalizedPriors(archetypes, learnedPriors);

        double[] responseCpt;
        if (contextResponsePriors != null && !contextResponsePriors.isEmpty()) {
            responseCpt = buildContextResponseCpt(archetypes, ourStrategies,
                    contextResponsePriors, responsePriors, expertOverrides);
        } else {
            responseCpt = buildTiledResponseCpt(archetypes, ourStrategies,
                    responsePriors, expertOverrides);
        }

        var responseBuilder = BayesNetworkBuilder.create("response");
        responseBuilder.variable("enemy_strategy", archetypes).prior(enemyPriors);
        responseBuilder.variable("our_economy_state", "behind", "even", "ahead")
                       .prior(1.0 / 3, 1.0 / 3, 1.0 / 3);
        responseBuilder.variable("our_army_state", "smaller", "even", "larger")
                       .prior(1.0 / 3, 1.0 / 3, 1.0 / 3);
        responseBuilder.variable("our_tech_state", "behind", "even", "ahead")
                       .prior(1.0 / 3, 1.0 / 3, 1.0 / 3);
        responseBuilder.variable("our_strategy", ourStrategies)
                       .given("enemy_strategy", "our_economy_state",
                              "our_army_state", "our_tech_state")
                       .cpt(responseCpt);
        JunctionTree responseTree = responseBuilder.buildJunctionTree();

        compBuilder.slot("response", responseTree);
        compBuilder.link("intel", "enemy_strategy").to("response", "enemy_strategy");
    }

    NetworkComposition composition = compBuilder.build();
    return buildHypothesisTracker(composition, archetypes, learnedPriors);
}
```

- [ ] **Step 4: Wire HypothesisTrackerFusion.initialize() to load and pass context priors**

Modify `HypothesisTrackerFusion.initialize()` — after the existing response prior loading (lines 106-113), add context prior loading and call the new overload:

Change the tracker construction block (lines 115-127) from:

```java
if (rp != null && !rp.isEmpty()) {
    opponentTracker     = SC2HypothesisNetworkBuilder.buildTracker(
            enemyRace, phase, ourRace != null ? ourRace : Race.PROTOSS,
            priors, sorted, detectorCpts, obsCpts, rp, eo);
```

To:

```java
if (rp != null && !rp.isEmpty()) {
    var contextRp = HypothesisPriorLoader.loadContextResponsePriors(enemyRace);
    opponentTracker     = SC2HypothesisNetworkBuilder.buildTracker(
            enemyRace, phase, ourRace != null ? ourRace : Race.PROTOSS,
            priors, sorted, detectorCpts, obsCpts, rp, eo, contextRp);
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2HypothesisNetworkBuilderTest -q`
Expected: all tests PASS

- [ ] **Step 6: Run full test suite to verify no regressions**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all tests PASS — backward compatible since `loadContextResponsePriors()` returns `null` when no context CPT files exist, and the new `buildTracker` overload falls back to `buildTiledResponseCpt()` when `contextResponsePriors` is null.

- [ ] **Step 7: Commit**

```
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/SC2HypothesisNetworkBuilder.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/HypothesisTrackerFusion.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/SC2HypothesisNetworkBuilderTest.java
git commit -m "feat(#469): wire context-dependent CPTs through network builder and fusion

Refs #469"
```

---

## Batch 3: Data Generation and Calibration

### Task 5: ContextResponsePriorExportTest and calibration assertion

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ContextResponsePriorExportTest.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/ResponseSelectionCalibrationTest.java`

**Interfaces:**
- Consumes: `PriorExportPipeline.ContextResponseOutcome`, `discretizeContext()`, `learnContextResponsePriors()`, `exportContextResponsePriorsToJson()`, `learnResponsePriors()` from Tasks 1-2
- Produces:
  - `src/main/resources/priors/response-cpts-context-vs_protoss.json`
  - `src/main/resources/priors/response-cpts-context-vs_terran.json`
  - `src/main/resources/priors/response-cpts-context-vs_zerg.json`
  - Updated calibration assertion in `ResponseSelectionCalibrationTest`

- [ ] **Step 1: Create ContextResponsePriorExportTest**

The test reads all reconstituted replay JSON files and their `.label.json` sidecars, extracts game state at 3-min and 5-min checkpoints, builds `ContextResponseOutcome` records, learns context-dependent CPTs with hierarchical smoothing, and exports JSON.

```java
package io.quarkmind.sc2.replay;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.quarkmind.sc2.replay.PriorExportPipeline.ContextResponseOutcome;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.stream.Stream;

import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertTrue;

@Tag("report")
class ContextResponsePriorExportTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final Path RECONSTITUTED_DIR =
            Path.of("../quarkmind-classifier/data/reconstituted");
    private static final Path OUTPUT_DIR =
            Path.of("src/main/resources/priors");
    private static final int CHECKPOINT_3MIN = 4032;
    private static final int CHECKPOINT_5MIN = 6720;
    private static final double ALPHA = 10.0;

    private static final Set<String> TECH_BUILDING_NAMES = Set.of(
            "CyberneticsCore", "TwilightCouncil", "RoboticsFacility", "Stargate",
            "DarkShrine", "TemplarArchives", "FleetBeacon", "RoboticsBay", "Forge",
            "EngineeringBay", "Armory", "GhostAcademy", "Factory", "Starport", "FusionCore",
            "SpawningPool", "EvolutionChamber", "RoachWarren", "BanelingNest",
            "HydraliskDen", "LurkerDenMP", "InfestationPit", "Spire",
            "GreaterSpire", "UltraliskCavern");

    @Test
    void exportContextResponsePriors() throws Exception {
        assertTrue(Files.isDirectory(RECONSTITUTED_DIR),
                   "Reconstituted data required at " + RECONSTITUTED_DIR);

        Map<String, List<ContextResponseOutcome>> byMatchup = new LinkedHashMap<>();
        Map<String, Set<String>> allOurByMatchup = new LinkedHashMap<>();
        int totalOutcomes = 0;
        int totalReplays = 0;

        for (Path datasetDir : listSubDirs(RECONSTITUTED_DIR)) {
            List<Path> replayFiles = listJsonFiles(datasetDir);
            for (Path replayFile : replayFiles) {
                Path labelFile = Path.of(replayFile.toString()
                        .replace(".json", ".label.json"));
                if (!Files.exists(labelFile)) continue;

                totalReplays++;
                JsonNode replay = MAPPER.readTree(replayFile.toFile());
                JsonNode labels = MAPPER.readTree(labelFile.toFile());
                JsonNode perspectives = labels.get("perspectives");
                if (perspectives == null) continue;

                JsonNode header = replay.get("header");
                if (header == null) continue;
                int totalLoops = header.has("elapsedGameLoops")
                        ? header.get("elapsedGameLoops").asInt() : 0;

                JsonNode events = replay.get("trackerEvents");
                if (events == null || !events.isArray()) continue;

                JsonNode toonMap = replay.get("ToonPlayerDescMap");
                if (toonMap == null) continue;

                for (var pEntry : perspectives.fields()) {
                    String pid = pEntry.getKey();
                    JsonNode pLabel = pEntry.getValue();
                    String matchup = pLabel.has("matchup") ? pLabel.get("matchup").asText() : null;
                    String coarseLabel = pLabel.has("coarseLabel") ? pLabel.get("coarseLabel").asText() : null;
                    if (matchup == null || coarseLabel == null) continue;

                    JsonNode playerInfo = toonMap.get(pid);
                    if (playerInfo == null) continue;
                    String result = playerInfo.has("result") ? playerInfo.get("result").asText() : "";
                    boolean won = "Win".equals(result);

                    int playerId = Integer.parseInt(pid);
                    int enemyId = playerId == 1 ? 2 : 1;

                    // Get enemy label for opponent strategy
                    JsonNode enemyPerspective = perspectives.get(String.valueOf(enemyId));
                    if (enemyPerspective == null) continue;
                    String enemyLabel = enemyPerspective.has("coarseLabel")
                            ? enemyPerspective.get("coarseLabel").asText() : null;
                    if (enemyLabel == null) continue;

                    for (int checkpoint : new int[]{CHECKPOINT_3MIN, CHECKPOINT_5MIN}) {
                        if (totalLoops < checkpoint) continue;

                        var snapshot = extractGameStateAtCheckpoint(
                                events, playerId, enemyId, checkpoint);
                        if (snapshot == null) continue;

                        var ctx = PriorExportPipeline.discretizeContext(
                                snapshot.ourIncome, snapshot.enemyIncome,
                                snapshot.ourArmyCount, snapshot.enemyArmyCount,
                                snapshot.ourTechCount, snapshot.enemyTechCount);

                        var outcome = new ContextResponseOutcome(
                                coarseLabel, enemyLabel, matchup, won,
                                ctx.get("economyState"), ctx.get("armyState"), ctx.get("techState"));
                        byMatchup.computeIfAbsent(matchup, k -> new ArrayList<>()).add(outcome);
                        allOurByMatchup.computeIfAbsent(matchup, k -> new LinkedHashSet<>())
                                .add(coarseLabel);
                        totalOutcomes++;
                    }
                }
            }
        }

        System.out.printf("Processed %d replays, %d outcomes across %d matchups%n",
                          totalReplays, totalOutcomes, byMatchup.size());

        for (var entry : byMatchup.entrySet()) {
            String matchup = entry.getKey();
            List<ContextResponseOutcome> outcomes = entry.getValue();
            Set<String> allOur = allOurByMatchup.get(matchup);

            // Compute marginal (unconditional) priors for hierarchical smoothing
            var marginalOutcomes = outcomes.stream()
                    .map(o -> new PriorExportPipeline.ResponseOutcome(
                            o.ourStrategy(), o.opponentStrategy(), o.matchup(), o.won()))
                    .toList();
            var marginalPriors = PriorExportPipeline.learnResponsePriors(marginalOutcomes, allOur);

            var contextPriors = PriorExportPipeline.learnContextResponsePriors(
                    outcomes, allOur, marginalPriors, ALPHA);

            PriorExportPipeline.exportContextResponsePriorsToJson(
                    OUTPUT_DIR, matchup, contextPriors, ALPHA);

            // Statistics
            long totalWins = outcomes.stream().filter(ContextResponseOutcome::won).count();
            int opponentCount = contextPriors.size();
            int totalCells = contextPriors.values().stream()
                    .mapToInt(m -> m.size()).sum();
            System.out.printf("  %s: %d outcomes, %d wins, %d opponents, %d context cells%n",
                              matchup, outcomes.size(), totalWins, opponentCount, totalCells);
        }

        assertFalse(byMatchup.isEmpty(), "Should have produced at least one matchup");
    }

    record GameStateSnapshot(
            double ourIncome, double enemyIncome,
            int ourArmyCount, int enemyArmyCount,
            long ourTechCount, long enemyTechCount) {}

    static GameStateSnapshot extractGameStateAtCheckpoint(
            JsonNode events, int playerId, int enemyId, int checkpointLoop) {

        double ourIncome = 0, enemyIncome = 0;
        boolean foundOurStats = false, foundEnemyStats = false;

        // Track units alive at checkpoint
        Map<Integer, String> aliveUnits = new LinkedHashMap<>(); // tag → unitType
        Map<Integer, Integer> unitOwner = new LinkedHashMap<>(); // tag → controlPlayerId

        for (JsonNode event : events) {
            int loop = event.get("loop").asInt();
            if (loop > checkpointLoop) break;

            String evtType = event.has("evtTypeName") ? event.get("evtTypeName").asText() : "";

            switch (evtType) {
                case "PlayerStats" -> {
                    int cpid = event.has("controlPlayerId") ? event.get("controlPlayerId").asInt() : 0;
                    JsonNode stats = event.get("stats");
                    if (stats == null) continue;
                    double minRate = stats.has("scoreValueMineralsCollectionRate")
                            ? stats.get("scoreValueMineralsCollectionRate").asDouble() / 1000.0 : 0;
                    double vesRate = stats.has("scoreValueVespeneCollectionRate")
                            ? stats.get("scoreValueVespeneCollectionRate").asDouble() / 1000.0 : 0;
                    double income = minRate + vesRate;
                    if (cpid == playerId) {
                        ourIncome = income;
                        foundOurStats = true;
                    } else if (cpid == enemyId) {
                        enemyIncome = income;
                        foundEnemyStats = true;
                    }
                }
                case "UnitBorn", "UnitInit" -> {
                    int cpid = event.has("controlPlayerId") ? event.get("controlPlayerId").asInt() : 0;
                    if (cpid != playerId && cpid != enemyId) continue;
                    String unitType = event.has("unitTypeName") ? event.get("unitTypeName").asText() : "";
                    int tag = event.has("unitTagIndex") ? event.get("unitTagIndex").asInt() : -1;
                    if (tag >= 0) {
                        aliveUnits.put(tag, unitType);
                        unitOwner.put(tag, cpid);
                    }
                }
                case "UnitDied" -> {
                    int tag = event.has("unitTagIndex") ? event.get("unitTagIndex").asInt() : -1;
                    aliveUnits.remove(tag);
                    unitOwner.remove(tag);
                }
            }
        }

        if (!foundOurStats && !foundEnemyStats) return null;

        int ourArmy = 0, enemyArmy = 0;
        long ourTech = 0, enemyTech = 0;
        for (var entry : aliveUnits.entrySet()) {
            int tag = entry.getKey();
            String unitType = entry.getValue();
            int owner = unitOwner.getOrDefault(tag, 0);
            if (owner == playerId) {
                ourArmy++;
                if (TECH_BUILDING_NAMES.contains(unitType)) ourTech++;
            } else if (owner == enemyId) {
                enemyArmy++;
                if (TECH_BUILDING_NAMES.contains(unitType)) enemyTech++;
            }
        }

        return new GameStateSnapshot(ourIncome, enemyIncome, ourArmy, enemyArmy, ourTech, enemyTech);
    }

    private static List<Path> listSubDirs(Path dir) throws IOException {
        try (Stream<Path> s = Files.list(dir)) {
            return s.filter(Files::isDirectory).sorted().toList();
        }
    }

    private static List<Path> listJsonFiles(Path dir) throws IOException {
        try (Stream<Path> s = Files.walk(dir)) {
            return s.filter(p -> p.toString().endsWith(".json")
                             && !p.toString().endsWith(".label.json"))
                    .sorted().toList();
        }
    }
}
```

- [ ] **Step 2: Update ResponseSelectionCalibrationTest to assert context shift**

In `ResponseSelectionCalibrationTest.responseSelection_contextVariables_shiftRecommendation()`, replace the print-only section (lines 149-154) with an assertion:

```java
System.out.println("=== Context Variable Sensitivity (Phase 5 — context-dependent CPTs) ===");
System.out.println("Behind context: " +
                   (responseBehind.isEmpty() ? "—" : responseBehind.getFirst().archetype()));
System.out.println("Ahead context:  " +
                   (responseAhead.isEmpty() ? "—" : responseAhead.getFirst().archetype()));

if (!responseBehind.isEmpty() && !responseAhead.isEmpty()) {
    boolean anyDifferent = false;
    for (int i = 0; i < Math.min(responseBehind.size(), responseAhead.size()); i++) {
        if (Math.abs(responseBehind.get(i).confidence() - responseAhead.get(i).confidence()) > 0.001) {
            anyDifferent = true;
            break;
        }
    }
    assertTrue(anyDifferent,
               "Context-dependent CPTs should produce different response distributions for behind vs ahead");
}
```

But wait — this test uses `syntheticResponsePriors()` not real priors. The assertion should work with the real loaded priors once the context CPT JSON files exist. For the test to be meaningful, it needs to load real response priors instead of synthetic ones.

Change the test to use loaded priors when context CPT files exist:

```java
@Test
void responseSelection_contextVariables_shiftRecommendation() {
    Race      enemyRace = Race.TERRAN;
    GamePhase phase     = GamePhase.EARLY;

    var priors    = HypothesisPriorLoader.loadOpponentPriors(phase, enemyRace);
    var obsCpts   = HypothesisPriorLoader.loadObservationCpts(phase, enemyRace);
    var overrides = HypothesisPriorLoader.loadExpertOverrides(enemyRace);
    var allActive = registry.getActiveDetectors(phase, enemyRace);
    var allKeys   = new HashSet<String>();
    for (var def : allActive) {allKeys.add(def.key(enemyRace));}
    var detCpts = buildDetectorCpts(allKeys, enemyRace);

    var responsePriors = HypothesisPriorLoader.loadResponsePriors(enemyRace);
    if (responsePriors.isEmpty()) {
        responsePriors = syntheticResponsePriors(priors);
    }

    var contextPriors = HypothesisPriorLoader.loadContextResponsePriors(enemyRace);

    var fusion = new HypothesisTrackerFusion(
            registry, priors, detCpts, obsCpts, Map.of(), null,
            responsePriors, overrides);
    fusion.initialize(List.copyOf(allKeys), phase, enemyRace);

    var behind = Map.of(
            "our_economy_state", "behind",
            "our_army_state", "smaller",
            "our_tech_state", "behind");
    var ahead = Map.of(
            "our_economy_state", "ahead",
            "our_army_state", "larger",
            "our_tech_state", "ahead");

    var responseBehind = fusion.selectResponse(Race.PROTOSS, behind);
    var responseAhead  = fusion.selectResponse(Race.PROTOSS, ahead);

    System.out.println("=== Context Variable Sensitivity ===");
    System.out.println("Behind context: " +
                       (responseBehind.isEmpty() ? "—" : responseBehind.getFirst().archetype()));
    System.out.println("Ahead context:  " +
                       (responseAhead.isEmpty() ? "—" : responseAhead.getFirst().archetype()));
    System.out.println("Context CPTs loaded: " + (contextPriors != null));

    if (contextPriors != null && !responseBehind.isEmpty() && !responseAhead.isEmpty()) {
        boolean anyDifferent = false;
        for (int i = 0; i < Math.min(responseBehind.size(), responseAhead.size()); i++) {
            if (Math.abs(responseBehind.get(i).confidence() - responseAhead.get(i).confidence()) > 0.001) {
                anyDifferent = true;
                break;
            }
        }
        assertTrue(anyDifferent,
                   "Context-dependent CPTs should produce different distributions for behind vs ahead");
    }
}
```

- [ ] **Step 3: Run the report test to generate context CPT JSON files**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=ContextResponsePriorExportTest -q`
Expected: JSON files written to `src/main/resources/priors/response-cpts-context-vs_*.json`

- [ ] **Step 4: Run the calibration test to verify assertion passes**

Run: `mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=ResponseSelectionCalibrationTest#responseSelection_contextVariables_shiftRecommendation -q`
Expected: PASS with different response distributions for behind vs ahead contexts

- [ ] **Step 5: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all tests PASS

- [ ] **Step 6: Register ContextResponsePriorExportTest in CLAUDE.md**

Add to the "Replay divergence report" section under the existing report tests:
```
- `ContextResponsePriorExportTest` — processes reconstituted replays at 3-min and 5-min checkpoints; extracts game state context (economy/army/tech); learns context-dependent response CPTs with hierarchical smoothing (α=10.0); writes response-cpts-context-vs_*.json
```

- [ ] **Step 7: Commit**

```
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ContextResponsePriorExportTest.java quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/ResponseSelectionCalibrationTest.java quarkmind-sc2/src/main/resources/priors/response-cpts-context-vs_protoss.json quarkmind-sc2/src/main/resources/priors/response-cpts-context-vs_terran.json quarkmind-sc2/src/main/resources/priors/response-cpts-context-vs_zerg.json CLAUDE.md
git commit -m "feat(#469): context-dependent response CPT generation and calibration assertion

Closes #469"
```

## References

- [2026-10-29-context-dependent-response-cpts-design.md] — design spec this plan implements
- [SC2HypothesisNetworkBuilder.java:194-235] — existing `buildTiledResponseCpt()`
- [PriorExportPipeline.java:30-60] — existing `ResponseOutcome` and `learnResponsePriors()`
- [HypothesisPriorLoader.java:76-85] — existing `loadResponsePriors()`
- [HypothesisTrackerFusion.java:106-127] — response prior loading and tracker construction
- [DroolsScoutingTask.java:492-506] — `TECH_BUILDINGS` set
- [DroolsScoutingTask.java:680-707] — `deriveGameStateContext()` thresholds
- [ResponseSelectionCalibrationTest.java:119-155] — Phase 4 context sensitivity test
- [GitHub #469] — Phase 5: Context-dependent response CPTs
