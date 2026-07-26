# E19b: New Protoss Sprites Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add canvas 2D draw functions and UNIT_MATS registrations for 8 new Protoss unit types, adding 19 new Playwright tests (smoke + spawn for all 8; elevation for 4 air units).

**Prerequisite:** E19a complete.

**Air units (FLYING_UNITS):** Phoenix, Oracle, Tempest, Mothership, Warp Prism, Warp Prism Phasing, Interceptor must be added. Adept Phase Shift is NOT a flying unit.

**Draw function conventions:**
- Canvas is 128×128px (`S=128`). All sizes proportional to S.
- Dir 0=front, 1=right, 2=back, 3=left. Use dir-3 mirror pattern for all units.
- `hexToRgba(teamColor, alpha)` available. `function` declarations only (not arrow fns).
- **Smoke test samples pixel (64,64)** — body must produce non-zero alpha there for all 4 dirs.
- **Draw function code is NOT specified.** Write from the visual description. See `docs/superpowers/plans/2026-04-23-e17-terran-sprites.md` for Canvas 2D patterns.

**Tech Stack:** Canvas 2D API, Three.js r128, Java 21, Playwright (Chromium), JUnit 5, Maven

---

### Task 0: Create GitHub issue

- [ ] **Step 1: Create issue**

```bash
gh issue create \
  --title "E19b: new Protoss sprites (Phoenix, Oracle, Tempest, Mothership, Warp Prism, Warp Prism Phasing, Interceptor, Adept Phase Shift)" \
  --body "Add sprites for all previously-missing Protoss multiplayer units.

## Units
Air: Phoenix, Oracle, Tempest, Mothership, Warp Prism, Warp Prism Phasing, Interceptor
Ground/ability: Adept Phase Shift

## Acceptance criteria
- draw function for each unit (4 dirs × 2 team colours)
- UNIT_MATS registration (_F and _E)
- smokeTestDrawFn lookup entry for each
- FLYING_UNITS: Phoenix, Oracle, Tempest, Mothership, Warp Prism, Warp Prism Phasing, Interceptor
- Playwright: smoke + spawn for all 8; elevation for 7 air units (19 tests)

Refs #83" \
  --label "enhancement"
```

Note the issue number — every commit uses `Refs #N #83`.

---

### Task 1: Phoenix (air — Protoss interceptor)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void phoenixDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawPhoenix', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawPhoenix dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void phoenixEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.PHOENIX, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Phoenix enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void phoenixSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.PHOENIX, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double phoenixY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(phoenixY)
        .as("Phoenix Y (%.3f) must be higher than Marine Y (%.3f)".formatted(phoenixY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#phoenixDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawPhoenix` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Sleek, fast-looking Protoss air superiority fighter. Bright gold-blue Protoss palette (~`#2a2a50` body, `#6a5a20` gold accents).
- Dir 0/2 (top-down): pointed diamond shape. Two swept-forward wings angling upward-forward from the body centre. The wings have a slight upward curve (phoenix-like). Body is a narrow central spine.
- Dir 1 (side): sleek profile, pointed at front, engine glow at rear. Wings visible above as forward-swept shapes.
- Ion cannon: team colour glow at the nose tip (forward weapon), shadowBlur ~14.
- Engine glow: team colour at the rear, shadowBlur ~10.
- Overall: one of the most elegant Protoss designs — thin, fast-looking, angular wings.

- [ ] **Step 4: Update FLYING_UNITS and add to `smokeTestDrawFn` lookup**

Add `'PHOENIX'` to the FLYING_UNITS set. Then in the smokeTestDrawFn lookup (after last existing entry):

```javascript
    if (typeof drawPhoenix !== 'undefined') lookup.drawPhoenix = drawPhoenix;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['PHOENIX_F'] = makeDirTextures(drawPhoenix, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['PHOENIX_E'] = makeDirTextures(drawPhoenix, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#phoenixDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+phoenixEnemySpawnsAndRendersInVisualizer+phoenixSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all 3).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Phoenix sprite — drawPhoenix, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 2: Oracle (air — Protoss harass)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void oracleDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawOracle', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawOracle dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void oracleEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.ORACLE, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Oracle enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void oracleSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.ORACLE, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double oracleY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(oracleY)
        .as("Oracle Y (%.3f) must be higher than Marine Y (%.3f)".formatted(oracleY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#oracleDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawOracle` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Hovering Protoss psionic ship, slim and elegant. Gold-blue palette.
- Dir 0/2: elongated oval/lens shape — narrower than Carrier, wider than Phoenix. Two small swept-back fins. Central hull with a glowing psionic cannon projector at the front.
- Psionic cannon: a long barrel extending forward from the hull centre, tipped with a bright team colour glow, shadowBlur ~14. This is the defining feature — long weapon barrel on a slim hull.
- Engine glow at rear: team colour, shadowBlur ~8.
- Dir 1 (side): thin blade-like profile, long cannon barrel extends forward, engine at rear.
- Overall: longer and more slender than most Protoss ships.

- [ ] **Step 4: Update FLYING_UNITS and add to `smokeTestDrawFn` lookup**

Add `'ORACLE'` to FLYING_UNITS. Add to lookup:

```javascript
    if (typeof drawOracle !== 'undefined') lookup.drawOracle = drawOracle;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['ORACLE_F'] = makeDirTextures(drawOracle, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['ORACLE_E'] = makeDirTextures(drawOracle, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#oracleDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+oracleEnemySpawnsAndRendersInVisualizer+oracleSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all 3).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Oracle sprite — drawOracle, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 3: Tempest (air — Protoss capital ship)

Air unit — FLYING_UNITS + elevation test.

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void tempestDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawTempest', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawTempest dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void tempestEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.TEMPEST, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Tempest enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void tempestSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.TEMPEST, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double tempestY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(tempestY)
        .as("Tempest Y (%.3f) must be higher than Marine Y (%.3f)".formatted(tempestY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#tempestDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawTempest` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Large angular Protoss battleship, heavy long-range artillery. Dark blue-gold palette.
- Dir 0/2: wide arrowhead/chevron shape — broad at the rear, tapering to a point at the front. Two large wing extensions swept backward. Massive long-range cannon extends far forward from the hull centre (longer barrel than Oracle). Team colour at the cannon tip.
- Dir 1 (side): long elongated profile, the large cannon visible extending to the front, engine at rear.
- Hull: larger and more imposing than Void Ray but more angular — military capital ship.
- Team colour: cannon tip and running lights.

- [ ] **Step 4: Update FLYING_UNITS and add to lookup**

Add `'TEMPEST'` to FLYING_UNITS. Add:

```javascript
    if (typeof drawTempest !== 'undefined') lookup.drawTempest = drawTempest;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['TEMPEST_F'] = makeDirTextures(drawTempest, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['TEMPEST_E'] = makeDirTextures(drawTempest, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#tempestDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+tempestEnemySpawnsAndRendersInVisualizer+tempestSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all 3).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Tempest sprite — drawTempest, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 4: Mothership (air — massive Protoss flagship)

Air unit — FLYING_UNITS + elevation test.

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void mothershipDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawMothership', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawMothership dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void mothershipEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.MOTHERSHIP, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Mothership enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void mothershipSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.MOTHERSHIP, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double mothershipY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(mothershipY)
        .as("Mothership Y (%.3f) must be higher than Marine Y (%.3f)".formatted(mothershipY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#mothershipDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawMothership` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Enormous circular flagship — the largest Protoss unit. Takes up most of the canvas.
- Dir 0/2: large filled circle (~S×0.42 radius), centred at canvas centre. Dark blue-gold Protoss hull. A vortex/spiral energy pattern in the centre in team colour (multiple arc strokes spiralling inward). Three large engine nacelle pods at 120° spacing around the outer edge. Outer ring of ornamental spires (small protrusions at regular intervals).
- Dir 1 (side): saucer/disc profile — wide flat oval, the characteristic flying saucer shape. Engine pods visible at sides.
- Team colour: vortex core and engine glow, shadowBlur ~16.

- [ ] **Step 4: Update FLYING_UNITS and add to lookup**

Add `'MOTHERSHIP'` to FLYING_UNITS. Add:

```javascript
    if (typeof drawMothership !== 'undefined') lookup.drawMothership = drawMothership;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['MOTHERSHIP_F'] = makeDirTextures(drawMothership, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['MOTHERSHIP_E'] = makeDirTextures(drawMothership, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#mothershipDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+mothershipEnemySpawnsAndRendersInVisualizer+mothershipSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all 3).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Mothership sprite — drawMothership, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 5: Warp Prism (air — Protoss transport)

Air unit — FLYING_UNITS + elevation test.

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void warpPrismDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawWarpPrism', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawWarpPrism dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void warpPrismEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.WARP_PRISM, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Warp Prism enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void warpPrismSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.WARP_PRISM, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double warpPrismY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(warpPrismY)
        .as("Warp Prism Y (%.3f) must be higher than Marine Y (%.3f)".formatted(warpPrismY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#warpPrismDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawWarpPrism` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Geometric angular transport, Protoss gold-blue palette. Very angular/crystalline aesthetic.
- Dir 0/2: octagonal/diamond body. Large crystal prism shape — a hexagonal or octagonal filled polygon, with faceted edges. Two small thruster pods at the bottom corners. A glowing warp crystal core in the centre (team colour, shadowBlur ~14).
- Dir 1 (side): faceted diamond profile, taller than wide. Thrusters at bottom. Crystal glow from core visible as a team colour ellipse in the centre.
- Overall: the most geometrically precise Protoss unit — hard angles, no curves.

- [ ] **Step 4: Update FLYING_UNITS and add to lookup**

Add `'WARP_PRISM'` to FLYING_UNITS. Add:

```javascript
    if (typeof drawWarpPrism !== 'undefined') lookup.drawWarpPrism = drawWarpPrism;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['WARP_PRISM_F'] = makeDirTextures(drawWarpPrism, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['WARP_PRISM_E'] = makeDirTextures(drawWarpPrism, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#warpPrismDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+warpPrismEnemySpawnsAndRendersInVisualizer+warpPrismSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all 3).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Warp Prism sprite — drawWarpPrism, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 6: Warp Prism Phasing (air — deployed warp field mode)

Air unit — FLYING_UNITS + elevation test.

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void warpPrismPhasingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawWarpPrismPhasing', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawWarpPrismPhasing dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void warpPrismPhasingEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.WARP_PRISM_PHASING, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Warp Prism Phasing enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void warpPrismPhasingSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.WARP_PRISM_PHASING, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double warpPrismPhasingY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(warpPrismPhasingY)
        .as("Warp Prism Phasing Y (%.3f) must be higher than Marine Y (%.3f)".formatted(warpPrismPhasingY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#warpPrismPhasingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawWarpPrismPhasing` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Same base shape as Warp Prism (octagonal geometric crystal) but deployed: the crystal has opened/expanded and a warp field beacon radiates from it.
- Body: same octagonal polygon as Warp Prism but slightly wider spread, with slight transparency (alpha ~0.7 on the body fill).
- Warp field beacon: a large outer ring stroke in team colour, radius ~S×0.35, heavier shadowBlur ~18. This ring represents the warp-in zone.
- Crystal core: brighter than normal Warp Prism, more intense team colour glow, shadowBlur ~20.
- Landing struts: 3–4 small diagonal strokes extending from the lower body corners (it's hovering but deployed).
- Overall: recognisably the same unit as Warp Prism but with the beacon ring as the key visual difference.

- [ ] **Step 4: Update FLYING_UNITS and add to lookup**

Add `'WARP_PRISM_PHASING'` to FLYING_UNITS. Add:

```javascript
    if (typeof drawWarpPrismPhasing !== 'undefined') lookup.drawWarpPrismPhasing = drawWarpPrismPhasing;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['WARP_PRISM_PHASING_F'] = makeDirTextures(drawWarpPrismPhasing, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['WARP_PRISM_PHASING_E'] = makeDirTextures(drawWarpPrismPhasing, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#warpPrismPhasingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+warpPrismPhasingEnemySpawnsAndRendersInVisualizer+warpPrismPhasingSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all 3).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Warp Prism Phasing sprite — drawWarpPrismPhasing, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 7: Interceptor (air — Carrier fighter drone)

Air unit — FLYING_UNITS + elevation test. Very small unit.

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void interceptorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawInterceptor', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawInterceptor dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void interceptorEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.INTERCEPTOR, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Interceptor enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void interceptorSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.INTERCEPTOR, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double interceptorY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(interceptorY)
        .as("Interceptor Y (%.3f) must be higher than Marine Y (%.3f)".formatted(interceptorY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#interceptorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawInterceptor` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Tiny fighter drone, miniaturised Phoenix-like silhouette. The smallest Protoss flying unit.
- Same overall shape as Phoenix but at ~60% scale, centred at canvas centre (so it still covers (64,64)).
- Dir 0/2: small swept-wing diamond shape with a tiny pointed nose. Ion cannon tip in team colour, shadowBlur ~8.
- Dir 1 (side): tiny blade profile. Engine glow at rear.
- Keep it visually distinct from Phoenix by being noticeably smaller and simpler — fewer details.

- [ ] **Step 4: Update FLYING_UNITS and add to lookup**

Add `'INTERCEPTOR'` to FLYING_UNITS. Add:

```javascript
    if (typeof drawInterceptor !== 'undefined') lookup.drawInterceptor = drawInterceptor;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['INTERCEPTOR_F'] = makeDirTextures(drawInterceptor, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['INTERCEPTOR_E'] = makeDirTextures(drawInterceptor, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#interceptorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+interceptorEnemySpawnsAndRendersInVisualizer+interceptorSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all 3).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Interceptor sprite — drawInterceptor, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 8: Adept Phase Shift (ground — ability shade unit)

Ground unit — NOT in FLYING_UNITS. Smoke + spawn only.

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void adeptPhaseShiftDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawAdeptPhaseShift', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawAdeptPhaseShift dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void adeptPhaseShiftEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.ADEPT_PHASE_SHIFT, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Adept Phase Shift enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#adeptPhaseShiftDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawAdeptPhaseShift` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- A ghostly psionic shade — a translucent copy of the Adept. The shade left behind when the Adept uses its Phase Shift ability.
- Same silhouette as `drawAdept` (streamlined warrior with psi-lance) but rendered semi-transparently (~0.35–0.45 alpha on all fills) to suggest a ghost/projection.
- All fills use `hexToRgba(teamColor, 0.40)` or similarly low alpha — team colour is the dominant colour since the body is almost transparent.
- Body outline stroke in team colour, lineWidth ~S×0.02, semi-transparent.
- Psi-lance blade: also in team colour but more prominent than the body (~0.65 alpha) — the blade is the most solid part.
- No hard fills — the entire unit looks like a hologram.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawAdeptPhaseShift !== 'undefined') lookup.drawAdeptPhaseShift = drawAdeptPhaseShift;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['ADEPT_PHASE_SHIFT_F'] = makeDirTextures(drawAdeptPhaseShift, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['ADEPT_PHASE_SHIFT_E'] = makeDirTextures(drawAdeptPhaseShift, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#adeptPhaseShiftDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+adeptPhaseShiftEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Adept Phase Shift sprite — drawAdeptPhaseShift, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 9: Full suite + close issue

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
gh issue close N --comment "All 8 new Protoss sprites complete. 19 Playwright tests added. Refs #83"
```
