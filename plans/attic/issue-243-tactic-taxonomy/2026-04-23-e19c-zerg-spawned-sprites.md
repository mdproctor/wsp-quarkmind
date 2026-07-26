# E19c: Zerg + Spawned Unit Sprites Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add canvas 2D draw functions and UNIT_MATS registrations for 9 new Zerg and spawned unit types, adding 21 new Playwright tests (smoke + spawn for all 9; elevation for 3 air units).

**Prerequisite:** E19a and E19b complete.

**Air units (FLYING_UNITS):** Overlord, Overseer, Locust (flying form). Drone, Baneling, Broodling, Infested Terran, Changeling, Auto Turret are ground units.

**Draw function conventions:**
- Canvas is 128×128px (`S=128`). All sizes proportional to S.
- Dir 0=front, 1=right, 2=back, 3=left. Use dir-3 mirror pattern for all units.
- `hexToRgba(teamColor, alpha)` available. `function` declarations only.
- **Smoke test samples pixel (64,64)** — body must produce non-zero alpha there for all 4 dirs.
- **Draw function code is NOT specified.** Write from the visual description.

**Tech Stack:** Canvas 2D API, Three.js r128, Java 21, Playwright (Chromium), JUnit 5, Maven

---

### Task 0: Create GitHub issue

- [ ] **Step 1: Create issue**

```bash
gh issue create \
  --title "E19c: Zerg + spawned unit sprites (Drone, Overlord, Overseer, Baneling, Locust, Broodling, Infested Terran, Changeling, Auto Turret)" \
  --body "Add sprites for all previously-missing Zerg and spawned multiplayer units.

## Units
Air: Overlord, Overseer, Locust
Ground: Drone, Baneling, Broodling, Infested Terran, Changeling, Auto Turret

## Acceptance criteria
- draw function for each unit (4 dirs × 2 team colours)
- UNIT_MATS registration (_F and _E)
- smokeTestDrawFn lookup entry for each
- FLYING_UNITS: Overlord, Overseer, Locust
- Playwright: smoke + spawn for all 9; elevation for 3 air units (21 tests)

Refs #83" \
  --label "enhancement"
```

Note the issue number — every commit uses `Refs #N #83`.

---

### Task 1: Drone (Zerg worker)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void droneDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawDrone', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawDrone dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void droneEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.DRONE, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Drone enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#droneDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawDrone` in `visualizer.js`**

Add `function drawDrone(ctx, S, dir, teamColor)` before `// Populated by initSpriteMaterials()`. Use dir-3 mirror pattern.

**Visual design — implement using Canvas 2D API:**
- Small insectoid Zerg worker, rounder and smaller than Zergling. Dark organic brown-green palette (~`#2a1a08`).
- Body: rounded ellipse centred at (S/2, S/2), horizontal ~S×0.22, vertical ~S×0.18. Must cover (64,64).
- Small carapace wing buds: two tiny swept-back wing stubs above the body (Drones have vestigial wings).
- Claws: two small pointed claw paths at the front, slightly lighter brown.
- Eyes: two tiny bright spots near the head in team colour, shadowBlur ~6.
- Legs: 3 short leg strokes below the body in dir 0/2; 2 in dir 1.
- Smaller and less threatening than Zergling — this is a humble worker unit.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

After the last existing entry in the smokeTestDrawFn function body:

```javascript
    if (typeof drawDrone !== 'undefined') lookup.drawDrone = drawDrone;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['DRONE_F'] = makeDirTextures(drawDrone, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['DRONE_E'] = makeDirTextures(drawDrone, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#droneDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+droneEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Drone sprite — drawDrone, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 2: Overlord (air — Zerg supply/transport)

Air unit — FLYING_UNITS + elevation test.

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void overlordDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawOverlord', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawOverlord dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void overlordEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.OVERLORD, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Overlord enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void overlordSpawnsHigherThanGroundUnit() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.MARINE, new Point2d(20, 20));
    engine.observe();
    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double marineY = ((List<?>) page.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page.close();

    orchestrator.startGame();
    Page page2 = browser.newPage();
    page2.navigate(pageUrl.toString());
    page2.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.OVERLORD, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double overlordY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(overlordY)
        .as("Overlord Y (%.3f) must be higher than Marine Y (%.3f)".formatted(overlordY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#overlordDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawOverlord` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Large bloated flying jellyfish-like organism. Dark purple-grey Zerg palette (~`#1a0a1a` body).
- Body: large rounded blob ellipse centred at (S/2, S/2), horizontal ~S×0.38, vertical ~S×0.3. Must cover (64,64).
- Tentacle clusters: 6–8 tentacle strokes hanging below the body in dir 0/2, shorter at the sides, longer in the centre. Wavy using quadraticCurveTo. Dark purple.
- Dir 1 (side): wide blob profile, tentacles hang below.
- Bio-sacs: 2–3 rounded bumps on the body surface, slightly lighter than body.
- Eyes: two small team colour glowing dots near the front (top), shadowBlur ~8.
- Overall: the most passive-looking Zerg unit — a slow drifting gas bag.

- [ ] **Step 4: Update FLYING_UNITS and add to lookup**

Add `'OVERLORD'` to FLYING_UNITS. Add:

```javascript
    if (typeof drawOverlord !== 'undefined') lookup.drawOverlord = drawOverlord;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['OVERLORD_F'] = makeDirTextures(drawOverlord, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['OVERLORD_E'] = makeDirTextures(drawOverlord, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#overlordDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+overlordEnemySpawnsAndRendersInVisualizer+overlordSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all 3).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Overlord sprite — drawOverlord, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 3: Overseer (air — evolved Overlord with detection)

Air unit — FLYING_UNITS + elevation test.

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void overseerDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawOverseer', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawOverseer dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void overseerEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.OVERSEER, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Overseer enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void overseerSpawnsHigherThanGroundUnit() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.MARINE, new Point2d(20, 20));
    engine.observe();
    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double marineY = ((List<?>) page.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page.close();

    orchestrator.startGame();
    Page page2 = browser.newPage();
    page2.navigate(pageUrl.toString());
    page2.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.OVERSEER, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double overseerY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(overseerY)
        .as("Overseer Y (%.3f) must be higher than Marine Y (%.3f)".formatted(overseerY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#overseerDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawOverseer` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Evolved Overlord with detection capability. Similar blob shape but more evolved/angular than Overlord.
- Body: slightly smaller than Overlord (~S×0.32 horizontal), more elongated and less perfectly round. Same dark purple palette but with more defined edges.
- Detection eye cluster: 3–4 large compound eye protrusions on the front/top of the body — these are the key visual difference from Overlord. Each eye is a distinct rounded ellipse protruding from the body surface. Team colour glow on the eyes, shadowBlur ~10.
- Fewer tentacles than Overlord (it's evolved past the transport phase): 3–4 shorter tentacles.
- Overall more alert and predatory look than the passive Overlord.

- [ ] **Step 4: Update FLYING_UNITS and add to lookup**

Add `'OVERSEER'` to FLYING_UNITS. Add:

```javascript
    if (typeof drawOverseer !== 'undefined') lookup.drawOverseer = drawOverseer;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['OVERSEER_F'] = makeDirTextures(drawOverseer, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['OVERSEER_E'] = makeDirTextures(drawOverseer, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#overseerDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+overseerEnemySpawnsAndRendersInVisualizer+overseerSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all 3).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Overseer sprite — drawOverseer, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 4: Baneling (acid bomb Zergling morph)

Ground unit — smoke + spawn only.

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void banelingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawBaneling', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawBaneling dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void banelingEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.BANELING, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Baneling enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#banelingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawBaneling` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Small rolling acid bomb — a spherical creature with short legs. Dark green-yellow palette (~`#1a2a08` body, `#4a6a10` acid sac).
- Body: roughly spherical — large central filled circle centred at (S/2, S/2+4), radius ~S×0.24. Must cover (64,64).
- Acid sac: the defining feature — a large bulging acid gland on the body top/front. Draw as a brighter rounded filled ellipse on the body surface in acid green (~`#5a8a10`), slightly offset from centre.
- Legs: 4 very short stubby leg strokes below the body, barely visible.
- Eyes: two small bright spots, team colour, shadowBlur ~8.
- Acid drip: a small team colour drop at the bottom of the body, suggesting the acid is leaking.
- All 4 dirs similar (nearly spherical) — dir 1 shows the round profile from the side.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawBaneling !== 'undefined') lookup.drawBaneling = drawBaneling;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['BANELING_F'] = makeDirTextures(drawBaneling, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['BANELING_E'] = makeDirTextures(drawBaneling, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#banelingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+banelingEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Baneling sprite — drawBaneling, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 5: Locust (air — Swarm Host spawn)

Air unit — FLYING_UNITS + elevation test.

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void locustDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawLocust', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawLocust dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void locustEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.LOCUST, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Locust enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void locustSpawnsHigherThanGroundUnit() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.MARINE, new Point2d(20, 20));
    engine.observe();
    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double marineY = ((List<?>) page.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page.close();

    orchestrator.startGame();
    Page page2 = browser.newPage();
    page2.navigate(pageUrl.toString());
    page2.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.LOCUST, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double locustY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(locustY)
        .as("Locust Y (%.3f) must be higher than Marine Y (%.3f)".formatted(locustY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#locustDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawLocust` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Small insectoid flying unit spawned by Swarm Host. Angular, sharp, predatory. Dark olive-brown palette (~`#1a1a08`).
- Body: narrow elongated ellipse centred at canvas centre (~S×0.18 wide, ~S×0.12 tall). Small but clearly visible.
- Wings: two pairs of insect-like wings — pointed angular fills extending above-left and above-right, giving an angular dragonfly silhouette.
- Mandibles: two short pointed paths at the front in team colour, shadowBlur ~6.
- Dir 1 (side): side profile shows wings angled up, narrow body profile, mandibles at left.
- Fast aggressive appearance — sharp angles throughout.

- [ ] **Step 4: Update FLYING_UNITS and add to lookup**

Add `'LOCUST'` to FLYING_UNITS. Add:

```javascript
    if (typeof drawLocust !== 'undefined') lookup.drawLocust = drawLocust;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['LOCUST_F'] = makeDirTextures(drawLocust, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['LOCUST_E'] = makeDirTextures(drawLocust, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#locustDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+locustEnemySpawnsAndRendersInVisualizer+locustSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all 3).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Locust sprite — drawLocust, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 6: Broodling (tiny Brood Lord spawn)

Ground unit — smoke + spawn only.

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void broodlingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawBroodling', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawBroodling dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void broodlingEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.BROODLING, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Broodling enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#broodlingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawBroodling` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Tiny spider-like creature, the smallest Zerg unit. 6 legs radiating from a small body. Very dark (~`#0a0a0a` body).
- Body: small rounded ellipse centred at canvas centre, ~S×0.14 radius. Must produce non-zero alpha at (64,64) — body is centred there.
- Legs: 6 thin line strokes radiating out from the body at evenly-spaced angles (60° apart). Thin and spidery, length ~S×0.2. Dark grey.
- Mandibles: 2 short curved paths at the front, slightly lighter.
- Eyes: 2 tiny team colour dots, shadowBlur ~4.
- Very simple design — this is a tiny transient unit.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawBroodling !== 'undefined') lookup.drawBroodling = drawBroodling;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['BROODLING_F'] = makeDirTextures(drawBroodling, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['BROODLING_E'] = makeDirTextures(drawBroodling, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#broodlingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+broodlingEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Broodling sprite — drawBroodling, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 7: Infested Terran (Infestor spawn)

Ground unit — smoke + spawn only.

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void infestedTerranDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawInfestedTerran', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawInfestedTerran dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void infestedTerranEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.INFESTED_TERRAN, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Infested Terran enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#infestedTerranDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawInfestedTerran` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- A zombie Marine — a Terran infantry soldier consumed by Zerg infestation. Hybrid look: Terran armour shape but organic Zerg growths.
- Body: similar silhouette to Marine (rounded torso + helmet) but with Zerg corruption visible.
- Infestation: organic tendrils/growths extending from the body — 3–4 wavy green protrusions from the torso sides and back. Acid green (~`#3a6a08`).
- Armour: same general Marine shape but darker and degraded (~`#1a2a1a` instead of CMC blue-grey).
- Helmet: cracked/damaged visor, no team colour visor. Instead a Zerg eye glow peering through in team colour, shadowBlur ~8.
- One arm holding a grenade/weapon stub.
- Overall: recognisably humanoid but clearly wrong.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawInfestedTerran !== 'undefined') lookup.drawInfestedTerran = drawInfestedTerran;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['INFESTED_TERRAN_F'] = makeDirTextures(drawInfestedTerran, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['INFESTED_TERRAN_E'] = makeDirTextures(drawInfestedTerran, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#infestedTerranDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+infestedTerranEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Infested Terran sprite — drawInfestedTerran, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 8: Changeling (Overseer shapeshifter)

Ground unit — smoke + spawn only. Renders as a neutral/morphing blob.

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void changelingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawChangeling', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawChangeling dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void changelingEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.CHANGELING, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Changeling enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#changelingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawChangeling` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- A shapeshifting spy unit — render as an amorphous blob with undefined edges, suggesting it hasn't fully settled into a form. Dark Zerg palette with a distinctive shimmering quality.
- Body: irregular blob shape — NOT a perfect ellipse. Use a series of bezier curves to create an amoeba-like outline, roughly centred at (S/2, S/2). Must cover (64,64).
- Shimmer effect: the body fill uses a radial gradient from team colour (opacity 0.4) at centre to near-transparent at edge, with a dark organic background. Suggests the shapeshifting nature.
- Shifting pseudopods: 3–4 rounded protrusions from the body edge, slightly extended, as if the form is still settling.
- Eyes: two small team colour dots, positioned slightly off-centre, shadowBlur ~8.
- All 4 dirs similar (amorphous) — slight rotation of the protrusions per direction.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawChangeling !== 'undefined') lookup.drawChangeling = drawChangeling;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['CHANGELING_F'] = makeDirTextures(drawChangeling, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['CHANGELING_E'] = makeDirTextures(drawChangeling, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#changelingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+changelingEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Changeling sprite — drawChangeling, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 9: Auto Turret (Raven deployed weapon)

Ground unit — smoke + spawn only. Terran deployed structure.

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void autoTurretDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawAutoTurret', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawAutoTurret dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void autoTurretEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.AUTO_TURRET, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Auto Turret enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#autoTurretDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawAutoTurret` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Deployed gun turret, fully stationary. Military dark grey Terran palette (~`#3a3a3a` base, `#2a2a2a` mount).
- Base/mount: a flat hexagonal or octagonal mounting plate at canvas bottom-centre (~S×0.26 wide, ~S×0.08 tall), dark grey.
- Turret body: raised rounded box on the mount, centred at (S/2, S/2+4). Must cover (64,64).
- Twin autocannon barrels: two parallel barrel rects extending upward from the turret body. In dir 0 (front): barrels extend upward. In dir 1 (side): barrels extend to the right.
- Sensor/tracker: small dome on top of the turret with team colour sensor dot, shadowBlur ~8.
- Direction-agnostic (turret can rotate) — all 4 dirs nearly identical, only slight barrel angle change.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawAutoTurret !== 'undefined') lookup.drawAutoTurret = drawAutoTurret;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['AUTO_TURRET_F'] = makeDirTextures(drawAutoTurret, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['AUTO_TURRET_E'] = makeDirTextures(drawAutoTurret, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#autoTurretDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+autoTurretEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Auto Turret sprite — drawAutoTurret, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 10: Full suite + close issue

- [ ] **Step 1: Run full Playwright suite**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest" -q
```

Expected: all browser-tagged tests pass, 0 failures.

- [ ] **Step 2: Run non-Playwright suite**

```bash
mvn test -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Close issue**

```bash
gh issue close N --comment "All 9 Zerg + spawned unit sprites complete. 21 Playwright tests added. E19 complete — all multiplayer units now have sprites. Refs #83"
```
