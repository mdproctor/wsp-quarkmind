# SC2 Playbook Delivery Type — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #384 — Emulator reconstitution pipeline
**Issue group:** #384

**Goal:** Enable SC2 build-order playbooks that drive EmulatedGame via
CaseHub's YAML playbook system, with inline assertions for physics
calibration.

**Architecture:** CDI-based `SC2DeliveryHandler` translates playbook step
actions (`train`, `build`, `research`, `morph`, `assert`) into
`IntentQueue` calls. `SC2PlaybookRunner` owns the tick loop — it
compiles YAML via `PlaybookCompiler`, then drives tick → observe →
evaluate → dispatch. The same playbook runs against `%emulated` or
`%sc2` by profile-switching the `SC2Engine` bean.

**Tech Stack:** Quarkus CDI, CaseHub yaml-core + pages-playbook (parser
only), EmulatedGame, IntentQueue

## Global Constraints

- Dependency: `casehub-pages-playbook` (parser/compiler), NOT
  `casehub-pages-playbook-runtime` (executor is quarkmind's own)
- Playbook YAML files in `src/test/resources/playbooks/`
- Tests: `@QuarkusTest` with `%test` profile for emulated execution
- No `delivery:` key in SC2 playbooks — action-key-as-step-type
- Commit references #384

---

## Batch 1: Delivery handler and runner core

### Task 1: Add pages-playbook dependency and SC2DeliveryHandler

**Files:**
- Modify: `quarkmind-sc2/pom.xml` — add `casehub-pages-playbook` dependency
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/playbook/SC2DeliveryHandler.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/SC2DeliveryHandlerTest.java`

**Interfaces:**
- Consumes: `DeliveryHandler` SPI from `casehub-pages-playbook`, `EmulatedGame.applyIntent(Intent)`
- Produces: `SC2DeliveryHandler.execute(String stepName, Map<String, Object> data, DeliveryContext ctx)` — returns `StepOutcome`

- [ ] **Step 1: Write failing test — train action produces TrainIntent**

```java
@Test
void trainAction_producesTrainIntent() {
    var game = new EmulatedGame();
    game.reset();
    var handler = new SC2DeliveryHandler(game);

    var data = Map.<String, Object>of("action", "train", "train", "PROBE");
    StepOutcome outcome = handler.execute("train-0", data, null);

    assertThat(outcome.success()).isTrue();
    assertThat(game.snapshot().myUnits().stream()
        .filter(u -> u.type() == UnitType.PROBE).count()).isGreaterThan(12);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2DeliveryHandlerTest#trainAction_producesTrainIntent -q`
Expected: FAIL — class not found

- [ ] **Step 3: Add pages-playbook dependency to pom.xml**

Add to `quarkmind-sc2/pom.xml` dependencies:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-pages-playbook</artifactId>
</dependency>
```

- [ ] **Step 4: Implement SC2DeliveryHandler**

Use `ide_create_file`:

```java
package io.quarkmind.sc2.playbook;

import io.casehub.pages.playbook.DeliveryContext;
import io.casehub.pages.playbook.DeliveryHandler;
import io.casehub.pages.playbook.StepOutcome;
import io.quarkmind.domain.*;
import io.quarkmind.sc2.emulated.EmulatedGame;
import io.quarkmind.sc2.intent.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;

@ApplicationScoped
public class SC2DeliveryHandler implements DeliveryHandler {

    private final EmulatedGame game;

    @Inject
    public SC2DeliveryHandler(EmulatedGame game) {
        this.game = game;
    }

    // Plain constructor for unit tests
    SC2DeliveryHandler() { this.game = null; }

    @Override
    public String name() { return "sc2"; }

    @Override
    public StepOutcome execute(String stepName, Map<String, Object> data,
                               DeliveryContext ctx) {
        String action = (String) data.get("action");
        return switch (action) {
            case "train"    -> doTrain(stepName, data);
            case "build"    -> doBuild(stepName, data);
            case "research" -> doResearch(stepName, data);
            case "morph"    -> doMorph(stepName, data);
            case "assert"   -> doAssert(stepName, data);
            default -> StepOutcome.fail(stepName, "Unknown SC2 action: " + action);
        };
    }

    private StepOutcome doTrain(String stepName, Map<String, Object> data) {
        UnitType unit = UnitType.valueOf((String) data.get("train"));
        String tag = "r-playbook-" + unit.name().toLowerCase();
        game.applyIntent(new TrainIntent(tag, unit));
        return StepOutcome.ok(stepName, Map.of("unit", unit.name()));
    }

    private StepOutcome doBuild(String stepName, Map<String, Object> data) {
        BuildingType bt = BuildingType.valueOf((String) data.get("build"));
        game.applyIntent(new BuildIntent("r-playbook", bt, new Point2d(30, 30)));
        return StepOutcome.ok(stepName, Map.of("building", bt.name()));
    }

    private StepOutcome doResearch(String stepName, Map<String, Object> data) {
        UpgradeType ut = UpgradeType.valueOf((String) data.get("research"));
        game.applyIntent(new ResearchIntent("r-playbook", ut));
        return StepOutcome.ok(stepName, Map.of("upgrade", ut.name()));
    }

    private StepOutcome doMorph(String stepName, Map<String, Object> data) {
        String target = (String) data.get("morph");
        String source = (String) data.get("source");
        game.applyIntent(new MorphIntent("r-playbook", source, target));
        return StepOutcome.ok(stepName, Map.of("morph", target));
    }

    @SuppressWarnings("unchecked")
    private StepOutcome doAssert(String stepName, Map<String, Object> data) {
        var expect = (Map<String, Object>) data.get("expect");
        if (expect == null) return StepOutcome.ok(stepName, Map.of());
        GameState state = game.snapshot();
        var errors = new java.util.ArrayList<String>();

        if (expect.containsKey("units")) {
            var unitExpect = (Map<String, Map<String, Integer>>) expect.get("units");
            for (var entry : unitExpect.entrySet()) {
                UnitType ut = UnitType.valueOf(entry.getKey());
                int actual = (int) state.myUnits().stream()
                    .filter(u -> u.type() == ut).count();
                var bounds = entry.getValue();
                if (bounds.containsKey("min") && actual < bounds.get("min")) {
                    errors.add(ut + " expected >=" + bounds.get("min") + " but was " + actual);
                }
                if (bounds.containsKey("max") && actual > bounds.get("max")) {
                    errors.add(ut + " expected <=" + bounds.get("max") + " but was " + actual);
                }
            }
        }

        if (expect.containsKey("minerals")) {
            var mineralBounds = (Map<String, Integer>) expect.get("minerals");
            int actual = (int) state.minerals();
            if (mineralBounds.containsKey("min") && actual < mineralBounds.get("min")) {
                errors.add("minerals expected >=" + mineralBounds.get("min") + " but was " + actual);
            }
        }

        if (!errors.isEmpty()) {
            String msg = stepName + " checkpoint failed: " + String.join("; ", errors);
            throw new AssertionError(msg);
        }
        return StepOutcome.ok(stepName, Map.of("passed", true));
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2DeliveryHandlerTest -q`
Expected: PASS

- [ ] **Step 6: Add tests for build, research, morph, assert actions**

```java
@Test
void buildAction_createsBuildIntent() {
    var handler = new SC2DeliveryHandler(game);
    var data = Map.<String, Object>of("action", "build", "build", "PYLON");
    StepOutcome outcome = handler.execute("build-0", data, null);
    assertThat(outcome.success()).isTrue();
}

@Test
void assertAction_passesWhenConditionMet() {
    var handler = new SC2DeliveryHandler(game);
    var expect = Map.<String, Object>of("units",
        Map.of("PROBE", Map.of("min", 12)));
    var data = Map.<String, Object>of("action", "assert", "expect", expect);
    StepOutcome outcome = handler.execute("assert-0", data, null);
    assertThat(outcome.success()).isTrue();
}

@Test
void assertAction_throwsWhenConditionFails() {
    var handler = new SC2DeliveryHandler(game);
    var expect = Map.<String, Object>of("units",
        Map.of("PROBE", Map.of("min", 100)));
    var data = Map.<String, Object>of("action", "assert", "expect", expect);
    assertThatThrownBy(() -> handler.execute("assert-0", data, null))
        .isInstanceOf(AssertionError.class)
        .hasMessageContaining("expected >=100");
}
```

- [ ] **Step 7: Run all tests, commit**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2DeliveryHandlerTest -q`
Expected: PASS

```bash
git add quarkmind-sc2/pom.xml quarkmind-sc2/src/main/java/io/quarkmind/sc2/playbook/ quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/
git commit -m "feat: SC2DeliveryHandler — train/build/research/morph/assert actions Refs #384"
```

### Task 2: SC2PlaybookRunner — tick-driven playbook execution

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/playbook/SC2PlaybookRunner.java`
- Create: `quarkmind-sc2/src/test/resources/playbooks/test-train-probes.yaml`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/SC2PlaybookRunnerTest.java`

**Interfaces:**
- Consumes: `SC2DeliveryHandler.execute()`, `PlaybookCompiler.compile()`, `EmulatedGame`
- Produces: `SC2PlaybookRunner.execute(String yamlPath)` — runs playbook to completion or assertion failure

- [ ] **Step 1: Write minimal test playbook YAML**

Create `src/test/resources/playbooks/test-train-probes.yaml`:
```yaml
scenario: test-train-probes
schema: sc2
meta:
  race: PROTOSS
  duration: 1m

steps:
  - train: PROBE

  - assert:
    at: 30s
    expect:
      units:
        PROBE: { min: 13 }
```

- [ ] **Step 2: Write failing test**

```java
@Test
void executePlaybook_trainsProbeAndAsserts() {
    var runner = new SC2PlaybookRunner(new SC2DeliveryHandler(game), game);
    runner.execute("playbooks/test-train-probes.yaml");
    // assert step inside playbook validates
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2PlaybookRunnerTest -q`
Expected: FAIL — class not found

- [ ] **Step 4: Implement SC2PlaybookRunner**

```java
package io.quarkmind.sc2.playbook;

import io.casehub.pages.playbook.CompactStep;
import io.casehub.pages.playbook.PlaybookCompiler;
import io.casehub.pages.playbook.PlaybookEnvelope;
import io.quarkmind.domain.SC2Data;
import io.quarkmind.sc2.emulated.EmulatedGame;
import io.quarkmind.sc2.emulated.RaceModelFactory;
import io.quarkmind.domain.Race;

import java.io.IOException;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class SC2PlaybookRunner {

    private final SC2DeliveryHandler handler;
    private final EmulatedGame game;

    private static final int TICKS_PER_MINUTE =
        (int) (60 * SC2Data.GAME_LOOPS_PER_SECOND / SC2Data.LOOPS_PER_TICK);

    public SC2PlaybookRunner(SC2DeliveryHandler handler, EmulatedGame game) {
        this.handler = handler;
        this.game = game;
    }

    public void execute(String resourcePath) {
        String yaml = loadResource(resourcePath);
        var envelope = PlaybookCompiler.compile(yaml, Map.of());

        Race race = resolveRace(envelope);
        game.setPlayerRaceModel(RaceModelFactory.forRace(race));
        game.reset();

        int durationTicks = resolveDuration(envelope);
        List<CompactStep> steps = envelope.allSteps();

        for (int tick = 0; tick < durationTicks; tick++) {
            game.tick();
            long gameLoop = (long) tick * SC2Data.LOOPS_PER_TICK;

            for (CompactStep step : steps) {
                if (shouldFire(step, tick, gameLoop)) {
                    Map<String, Object> data = new HashMap<>(step.params());
                    data.put("action", step.action());
                    handler.execute(step.action() + "-" + tick, data, null);
                }
            }
        }
    }

    private boolean shouldFire(CompactStep step, int tick, long gameLoop) {
        Object atValue = step.params().get("at");
        if (atValue == null) return tick == 0;
        if (atValue instanceof String s) {
            int targetTick = parseTimeTicks(s);
            return tick == targetTick;
        }
        return false;
    }

    static int parseTimeTicks(String timeStr) {
        timeStr = timeStr.trim();
        if (timeStr.endsWith("m")) {
            int minutes = Integer.parseInt(timeStr.replace("m", "").trim());
            return minutes * TICKS_PER_MINUTE;
        }
        if (timeStr.endsWith("s")) {
            int seconds = Integer.parseInt(timeStr.replace("s", "").trim());
            return (int) (seconds * SC2Data.GAME_LOOPS_PER_SECOND / SC2Data.LOOPS_PER_TICK);
        }
        return Integer.parseInt(timeStr);
    }

    private Race resolveRace(PlaybookEnvelope envelope) {
        if (envelope.meta() != null && envelope.meta().labels() != null) {
            // meta.race is stored in labels for now
        }
        return Race.PROTOSS; // default
    }

    private int resolveDuration(PlaybookEnvelope envelope) {
        return 5 * TICKS_PER_MINUTE; // default 5 minutes
    }

    private String loadResource(String path) {
        try (InputStream is = getClass().getClassLoader().getResourceAsStream(path)) {
            if (is == null) throw new IllegalArgumentException("Playbook not found: " + path);
            return new String(is.readAllBytes(), StandardCharsets.UTF_8);
        } catch (IOException e) {
            throw new RuntimeException("Failed to load playbook: " + path, e);
        }
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2PlaybookRunnerTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/playbook/SC2PlaybookRunner.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/SC2PlaybookRunnerTest.java quarkmind-sc2/src/test/resources/playbooks/
git commit -m "feat: SC2PlaybookRunner — tick-driven playbook execution Refs #384"
```

## Batch 2: Economy calibration playbooks

### Task 3: Economy-only calibration playbooks + integration test

**Files:**
- Create: `quarkmind-sc2/src/test/resources/playbooks/economy-only-protoss.yaml`
- Create: `quarkmind-sc2/src/test/resources/playbooks/economy-only-terran.yaml`
- Create: `quarkmind-sc2/src/test/resources/playbooks/economy-only-zerg.yaml`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/SC2PlaybookCalibrationIT.java`

**Interfaces:**
- Consumes: `SC2PlaybookRunner.execute()`, `SC2DeliveryHandler`
- Produces: Three calibration playbooks + `@QuarkusTest` integration test

- [ ] **Step 1: Write economy-only-protoss.yaml**

```yaml
scenario: economy-only-protoss
schema: sc2
meta:
  race: PROTOSS
  duration: 5m

steps:
  - train: PROBE
    loop: continuous
    priority: background

  - build: PYLON
    at: 14 supply

  - assert:
    at: 1m
    expect:
      units:
        PROBE: { min: 16 }

  - assert:
    at: 3m
    expect:
      units:
        PROBE: { min: 30 }

  - assert:
    at: 5m
    expect:
      units:
        PROBE: { min: 44 }
```

- [ ] **Step 2: Write economy-only-terran.yaml**

```yaml
scenario: economy-only-terran
schema: sc2
meta:
  race: TERRAN
  duration: 5m

steps:
  - train: SCV
    loop: continuous
    priority: background

  - build: SUPPLY_DEPOT
    at: 14 supply

  - assert:
    at: 1m
    expect:
      units:
        SCV: { min: 16 }

  - assert:
    at: 3m
    expect:
      units:
        SCV: { min: 30 }

  - assert:
    at: 5m
    expect:
      units:
        SCV: { min: 44 }
```

- [ ] **Step 3: Write economy-only-zerg.yaml**

```yaml
scenario: economy-only-zerg
schema: sc2
meta:
  race: ZERG
  duration: 5m

steps:
  - train: DRONE
    loop: continuous
    priority: background

  - train: OVERLORD
    at: 13 supply

  - train: QUEEN
    at: 14 supply

  - assert:
    at: 1m
    expect:
      units:
        DRONE: { min: 16 }

  - assert:
    at: 3m
    expect:
      units:
        DRONE: { min: 30 }
        QUEEN: { min: 1 }

  - assert:
    at: 5m
    expect:
      units:
        DRONE: { min: 40 }
```

- [ ] **Step 4: Write integration test**

```java
package io.quarkmind.sc2.playbook;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

@QuarkusTest
class SC2PlaybookCalibrationIT {

    @Inject SC2PlaybookRunner runner;

    @Test
    void economyOnlyProtoss() {
        runner.execute("playbooks/economy-only-protoss.yaml");
    }

    @Test
    void economyOnlyTerran() {
        runner.execute("playbooks/economy-only-terran.yaml");
    }

    @Test
    void economyOnlyZerg() {
        runner.execute("playbooks/economy-only-zerg.yaml");
    }
}
```

- [ ] **Step 5: Run integration tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2PlaybookCalibrationIT -q`
Expected: Tests run — assertions may fail where physics constants need calibration. Each failure is a specific constant to fix.

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/test/resources/playbooks/ quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/SC2PlaybookCalibrationIT.java
git commit -m "feat: economy-only calibration playbooks for P/T/Z Refs #384"
```

## References

- [2026-10-09-sc2-playbook-delivery-type-design.md] — design spec
- [D8-D11 in decisions.md] — execution env, tick sync, vocabulary, assertions
- [DeliveryHandler.java] — Pages SPI (`execute(stepName, data, ctx)`)
- [PlaybookCompiler.java] — YAML parser/compiler
- [PlaybookContentParser.java:153] — action-key-as-step-type parser
- [EmulatedGame.java:264-280] — `applyIntent` dispatch
- [Intent records] — TrainIntent, BuildIntent, ResearchIntent, MorphIntent, MuleCalldownIntent
- [Platform #562] — at:, on-complete:, priority:/resource:
- [Platform #563] — loop:, cancel:, background:
- [GitHub #384] — parent issue
