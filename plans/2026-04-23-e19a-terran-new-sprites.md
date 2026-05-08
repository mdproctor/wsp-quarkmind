# E19a: New Terran Sprites Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add canvas 2D draw functions and UNIT_MATS registrations for 7 new Terran unit types, adding 14 new Playwright tests (smoke + spawn for all 7).

**Architecture:** Each unit gets one `function drawX(ctx, S, dir, teamColor)` added to `visualizer.js` before `initSpriteMaterials()`. The function is registered as `UNIT_MATS['X_F']` and `UNIT_MATS['X_E']` inside `initSpriteMaterials()`, and added to the `smokeTestDrawFn` manual lookup table in the `smokeTestDrawFn` function body. Tests use TDD: smoke test verifies alpha > 0 for all 8 dir/team combos; spawn test verifies unit renders.

**Air units (FLYING_UNITS):** None of the 7 units in this plan are air units. Do NOT modify FLYING_UNITS.

**Draw function conventions:**
- Canvas is 128×128px (`S=128`). All sizes proportional to S.
- Dir 0=front, 1=right, 2=back, 3=left. Every function uses the dir-3 mirror pattern:
```javascript
function drawX(ctx, S, dir, teamColor) {
  if (dir === 3) {
    ctx.save(); ctx.translate(S, 0); ctx.scale(-1, 1);
    drawX(ctx, S, 1, teamColor); ctx.restore(); return;
  }
  // implement dirs 0, 1, 2
}
```
- `hexToRgba(teamColor, alpha)` available. Every draw function must be a `function` declaration (not arrow fn).
- **Smoke test samples pixel (64,64)** — body MUST produce non-zero alpha there for all 4 dirs.
- **Draw function code is NOT specified.** Write Canvas 2D implementations from the visual description. See `docs/superpowers/plans/2026-04-23-e17-terran-sprites.md` for style reference.

**Tech Stack:** Canvas 2D API, Three.js r128, Java 21, Playwright (Chromium), JUnit 5, Maven

---

### Task 0: Create GitHub issue

- [ ] **Step 1: Create issue**

```bash
gh issue create \
  --title "E19a: new Terran sprites (SCV, Reaper, Hellion, Hellbat, MULE, Viking Assault, Liberator AG)" \
  --body "Add sprites for all previously-missing Terran multiplayer units.

## Units
SCV, Reaper, Hellion, Hellbat, MULE, Viking Assault (ground form), Liberator AG (defender mode)

## Acceptance criteria
- draw function for each unit (4 dirs × 2 team colours)
- UNIT_MATS registration (_F and _E)
- smokeTestDrawFn lookup entry for each
- Playwright: smoke + spawn for all 7 (14 tests)

Refs #83" \
  --label "enhancement"
```

Note the issue number — every commit uses `Refs #N #83`.

---

### Task 1: SCV (worker)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

Add at the end of `VisualizerRenderTest.java` (before the closing `}`):

```java
@Test
@Tag("browser")
void scvDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawSCV', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawSCV dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void scvEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.SCV, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one SCV enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#scvDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawSCV` in `visualizer.js`**

Add `function drawSCV(ctx, S, dir, teamColor)` before `// Populated by initSpriteMaterials()`. Use dir-3 mirror pattern.

**Visual design — implement using Canvas 2D API:**
- Stocky industrial worker suit, Terran grey-gunmetal palette (~`#4a4a4a` body, `#333` limbs).
- Body: rounded rectangular torso centred at (S/2, S/2). Must cover (64,64).
- Helmet: rounded dome on top with a small visor slit in team colour, shadowBlur ~6.
- Tool arm: dir 1 (side) — a welding arm extends to the right: a thick rect with a bright spark/flame tip in team colour. Dir 0/2 (front/back) — arm visible at shoulder level, tool tip as a small circle.
- Legs: two short rectangular leg fills below the torso.
- Backpack: small rectangular protrusion on the back (dir 2 shows it clearly, dir 0 mostly hidden).
- Colour accent: team colour on visor and tool tip only.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

In the `smokeTestDrawFn` function body, after the last existing entry, add:

```javascript
    if (typeof drawSCV !== 'undefined') lookup.drawSCV = drawSCV;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['SCV_F'] = makeDirTextures(drawSCV, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['SCV_E'] = makeDirTextures(drawSCV, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#scvDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+scvEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: SCV sprite — drawSCV, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 2: Reaper (jet-pack infantry)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void reaperDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawReaper', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawReaper dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void reaperEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.REAPER, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Reaper enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#reaperDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawReaper` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Light infantry suit, darker and sleeker than Marine. Dark grey (~`#2a2a2a`) body.
- Body: slimmer than Marine torso, slightly hunched forward.
- Jet pack: the defining feature — two compact thruster pods on the back, visible clearly in dir 1 (side) and dir 2 (back). Dir 1: two small rectangular nozzles extending rearward-upward. Dir 2: two round nozzle openings visible. Team colour thruster glow at the nozzle exits, shadowBlur ~10.
- Dual pistols: dir 1 shows two small rectangular gun barrels extending forward from both hands. Dir 0: pistols shown as small dark circles at the sides.
- Helmet: rounded with a dark visor, no team colour visor (Reapers are darker than Marines).
- Team colour: only on thruster glow.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawReaper !== 'undefined') lookup.drawReaper = drawReaper;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['REAPER_F'] = makeDirTextures(drawReaper, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['REAPER_E'] = makeDirTextures(drawReaper, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#reaperDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+reaperEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Reaper sprite — drawReaper, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 3: Hellion (wheeled flamethrower)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void hellionDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawHellion', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawHellion dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void hellionEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.HELLION, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Hellion enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#hellionDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawHellion` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Fast wheeled vehicle, very wide and low. Dark gunmetal with orange-red accent (~`#3a3a3a` body).
- Body: elongated low chassis — in dir 1 (side) it's very long (almost full canvas width) and low; in dir 0/2 (front/back) it's wide but shallow.
- Wheels: 4 visible in dir 1 as dark filled circles below chassis; 2 per side in dir 0/2 as ellipses.
- Flamethrower barrel: at the front. Dir 1: long barrel extending to the right (~S×0.3). Dir 0: barrel seen tip-on as a circular muzzle. Flame jet: team colour glow at the muzzle tip, shadowBlur ~12 — the fire effect is the key accent.
- Driver cockpit: small raised section near the rear in dir 1.
- Colour: team colour only on the flame/muzzle glow.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawHellion !== 'undefined') lookup.drawHellion = drawHellion;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['HELLION_F'] = makeDirTextures(drawHellion, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['HELLION_E'] = makeDirTextures(drawHellion, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#hellionDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+hellionEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Hellion sprite — drawHellion, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 4: Hellbat (transformed bipedal mech)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void hellbatDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawHellbat', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawHellbat dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void hellbatEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.HELLBAT, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Hellbat enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#hellbatDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawHellbat` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Bipedal mech walker — Hellion transformed. Very different from Hellion: upright, two legs, wide shoulders. Dark gunmetal body.
- Body: wide squat torso centred near (S/2, S/2), similar width to Thor but shorter and heavier.
- Two legs: thick rectangular leg fills below, splayed outward.
- Flamethrower arms: two arm-mounted flamethrowers extending outward from the shoulders. In dir 0 (front): two arm barrels extending left and right ending in flame nozzles. In dir 1 (side): one visible arm with barrel extending forward.
- Flame nozzles: team colour glow at each nozzle end, shadowBlur ~12 — these are the key accent.
- Armoured carapace on chest: a slightly raised darker rectangular plate.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawHellbat !== 'undefined') lookup.drawHellbat = drawHellbat;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['HELLBAT_F'] = makeDirTextures(drawHellbat, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['HELLBAT_E'] = makeDirTextures(drawHellbat, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#hellbatDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+hellbatEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Hellbat sprite — drawHellbat, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 5: MULE (calldown construction robot)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void muleDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawMULE', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawMULE dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void muleEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.MULE, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one MULE enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#muleDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawMULE` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Stocky four-legged industrial robot, heavier and boxier than SCV. Dark grey-blue (~`#3a3a4a`).
- Body: large square/rectangular torso centred at (S/2, S/2). Must cover (64,64).
- Four legs: short mechanical leg fills at the four body corners, angled downward — unlike bipedal SCV.
- Mining arm: a large articulated arm on top with a drill/mining head. Dir 1: arm extends upward-right with a drill tip. Dir 0/2: arm visible from above as an L-shape.
- Sensor dome: small dome on the torso top with team colour sensor glow, shadowBlur ~8.
- Industrial utility feel: more boxy and mechanical than SCV, less human-shaped.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawMULE !== 'undefined') lookup.drawMULE = drawMULE;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['MULE_F'] = makeDirTextures(drawMULE, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['MULE_E'] = makeDirTextures(drawMULE, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#muleDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+muleEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: MULE sprite — drawMULE, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 6: Viking Assault (ground bipedal mech)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void vikingAssaultDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawVikingAssault', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawVikingAssault dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void vikingAssaultEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.VIKING_ASSAULT, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Viking Assault enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#vikingAssaultDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawVikingAssault` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Viking landed as bipedal ground mech — completely different from the air Viking (drawViking). Same body but now upright on legs with wings folded back.
- Body: the Viking fuselage is now the torso, upright. Cockpit at the top (dome with team colour visor).
- Legs: two mechanical legs extending downward from the body — like an AT-ST walker. Wider stance in dir 0/2, side profile in dir 1.
- Wings: folded back behind the body — in dir 1 visible as swept-back shapes behind the fuselage. In dir 0/2, wing tips visible at the sides.
- Autocannons: two barrel extensions at the shoulder level, pointing forward. Dir 0: two small circles (barrel tips). Dir 1: one barrel extending forward-right.
- Team colour: visor and sensor, same Terran blue-grey palette as `drawViking`.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawVikingAssault !== 'undefined') lookup.drawVikingAssault = drawVikingAssault;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['VIKING_ASSAULT_F'] = makeDirTextures(drawVikingAssault, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['VIKING_ASSAULT_E'] = makeDirTextures(drawVikingAssault, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#vikingAssaultDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+vikingAssaultEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Viking Assault sprite — drawVikingAssault, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 7: Liberator AG (defender/siege mode)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void liberatorAGDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawLiberatorAG', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawLiberatorAG dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void liberatorAGEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.LIBERATOR_AG, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Liberator AG enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#liberatorAGDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawLiberatorAG` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Liberator in AG (Anti-Ground) defender mode — hovering stationary, wider deployed stance than the mobile Liberator (`drawLiberator`). Same base shape but with a targeting circle and angled weapon pods deployed downward.
- Body: same fuselage as Liberator but with wings spread wider — in dir 0/2 wings extend almost to canvas edge (~S×0.48 span).
- Targeting circle: a circular stroke ring below the hull (in dir 0/2), team colour with shadowBlur ~10. This is the defining visual of AG mode — a zone marker. Radius ~S×0.22.
- Weapon pods angled down: the weapon pods point downward rather than sideways, visible in dir 1 as downward-pointing barrels.
- Engine glow: same as Liberator but weapons glow is team colour on the downward-pointing barrel tips.
- Dir 1 (side): fuselage profile, two downward weapon barrels visible, no circle visible.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawLiberatorAG !== 'undefined') lookup.drawLiberatorAG = drawLiberatorAG;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['LIBERATOR_AG_F'] = makeDirTextures(drawLiberatorAG, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['LIBERATOR_AG_E'] = makeDirTextures(drawLiberatorAG, TEAM_COLOR_ENEMY);
```

Note: `LIBERATOR_AG` is an air unit — add `'LIBERATOR_AG'` to the `FLYING_UNITS` set (it hovers stationary but is still airborne).

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#liberatorAGDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+liberatorAGEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Liberator AG sprite — drawLiberatorAG, UNIT_MATS, FLYING_UNITS, smoke + spawn tests Refs #N #83"
```

---

### Task 8: Full suite + close issue

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
gh issue close N --comment "All 7 new Terran sprites complete. 14 Playwright tests added. Refs #83"
```
