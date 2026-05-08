# E18b: Zerg Sprites Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add canvas 2D draw functions and UNIT_MATS registrations for all 9 remaining Zerg units, add 3 flying types to FLYING_UNITS, and add 21 new Playwright tests (smoke + spawn for all 9; elevation for 3 air units).

**Prerequisite:** E18a (Protoss sprites) should be complete. FLYING_UNITS already includes `'OBSERVER'`, `'VOID_RAY'`, `'CARRIER'` from E18a. This plan adds `'BROOD_LORD'`, `'CORRUPTOR'`, `'VIPER'`.

**Architecture:** Each unit gets one `function drawX(ctx, S, dir, teamColor)` added to `visualizer.js` before `initSpriteMaterials()`. The function is registered as `UNIT_MATS['X_F']` and `UNIT_MATS['X_E']` inside `initSpriteMaterials()`, and added to the `smokeTestDrawFn` manual lookup table (in the `smokeTestDrawFn` function body — look for the block of `if (typeof drawX !== 'undefined') lookup.drawX = drawX;` lines, after the last E18a entry). Air units are added to the `FLYING_UNITS` set at the top of the file. Tests use TDD: smoke test verifies alpha > 0 for all 8 dir/team combos; spawn test verifies unit renders; elevation test (air only) verifies Y > marine Y + 0.3.

**Draw function conventions:**
- Canvas is 128×128px (`S=128`). All sizes are proportional to S.
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
- `hexToRgba(teamColor, alpha)` is available for team-coloured fills with transparency.
- **Draw function code is NOT specified in this plan.** Write Canvas 2D implementations based on the visual description in each task. See `docs/superpowers/plans/2026-04-23-e17-terran-sprites.md` for Canvas 2D drawing patterns and style reference.
- Every draw function must be a `function` declaration (not an arrow function).

**Tech Stack:** Canvas 2D API, Three.js r128, Java 21, Playwright (Chromium), JUnit 5, Maven

---

### Task 0: Create GitHub issue under epic #83

**Files:** none

- [ ] **Step 1: Create issue**

```bash
gh issue create \
  --title "E18b: Zerg sprites (9 units)" \
  --body "Complete 2D sprite coverage for all remaining Zerg units.

## Units
Ground: Ravager, Infestor, Lurker, Swarm Host, Queen, Ultralisk
Air: Corruptor, Viper, Brood Lord

## Acceptance criteria
- draw function for each unit (4 dirs × 2 team colours)
- UNIT_MATS registration for each (_F and _E)
- smokeTestDrawFn lookup entry for each
- FLYING_UNITS: Brood Lord, Corruptor, Viper (added to E18a set)
- Playwright: smoke + spawn for all 9; elevation for 3 air units (21 tests total)

Refs #83" \
  --label "enhancement"
```

Note the issue number printed — every commit in this plan uses `Refs #N` with it.

---

### Task 1: Ravager (evolved Roach)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

Add at the end of `VisualizerRenderTest.java` (before the closing `}`):

```java
@Test
@Tag("browser")
void ravagerDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawRavager', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawRavager dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void ravagerEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.RAVAGER, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Ravager enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#ravagerDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL — `smokeTestDrawFn` returns -1 (function not registered).

- [ ] **Step 3: Implement `drawRavager` in `visualizer.js`**

Add `function drawRavager(ctx, S, dir, teamColor)` before the `// Populated by initSpriteMaterials()` comment.

**Visual design — implement using Canvas 2D API:**
- Evolved Roach: larger and more angular body, dark brownish-green Zerg palette (~`#2a1a0a` base, `#3a2a10` mid).
- Body: broad rounded rect or ellipse with a pronounced hump, wider than the Roach silhouette.
- Legs: 4 short angled legs visible from dir 0/2; 2 visible from dir 1. Dark grey-brown.
- Bile cannon: a raised protrusion on the back/top — a rounded dome shape with a wide circular muzzle opening at the top. Dir 0/2: cannon centred on top. Dir 1: cannon raised on the back, barrel angled slightly forward.
- Team colour on cannon muzzle: bright filled ellipse at the muzzle tip, shadowBlur ~10.
- Bio-sacs: 2–3 small rounded blobs on each flank of the body, slightly darker than the body, with team colour glow on each.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

In `visualizer.js`, in the `smokeTestDrawFn` function body after the last E18a entry, add:

```javascript
    if (typeof drawRavager !== 'undefined') lookup.drawRavager = drawRavager;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['RAVAGER_F'] = makeDirTextures(drawRavager, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['RAVAGER_E'] = makeDirTextures(drawRavager, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#ravagerDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+ravagerEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Ravager sprite — drawRavager, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 2: Infestor (fungal organism)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void infestorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawInfestor', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawInfestor dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void infestorEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.INFESTOR, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Infestor enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#infestorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawInfestor` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Rounded organic body, wider than tall, very low to ground. Dark purple-green Zerg palette (~`#1a0a1a` body, `#2a1a10` mid).
- Body: wide filled ellipse centred at (S/2, S/2+10), horizontal radius ~S×0.3, vertical ~S×0.2, irregular organic feel.
- Tentacle protrusions: 4–5 wavy stroke paths (ctx.bezierCurveTo or quadraticCurveTo) extending from the body sides and rear, curling outward. Dark purple-grey, lineWidth ~S×0.03.
- Infested glow: a radial gradient in the body centre — team colour at opacity 0.5 at centre, fading outward. Combined with shadowBlur ~12 on a central filled ellipse for the glow source.
- Dir 1 (side): body shows as a lower, wider profile; tentacles visible from front and rear.
- Dir 2 (back): tentacles more prominent, glow still visible through body.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawInfestor !== 'undefined') lookup.drawInfestor = drawInfestor;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['INFESTOR_F'] = makeDirTextures(drawInfestor, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['INFESTOR_E'] = makeDirTextures(drawInfestor, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#infestorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+infestorEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Infestor sprite — drawInfestor, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 3: Lurker (burrowed surface pose)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void lurkerDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawLurker', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawLurker dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void lurkerEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.LURKER, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Lurker enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#lurkerDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawLurker` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Always burrowed — no body visible above ground. Dir 0/2/1 all show the same burrowed surface view (dir 3 still mirrors dir 1 per convention, but the view is essentially unchanged).
- Ground mound: wide filled ellipse near canvas centre, earth-brown colour (~`#3a2a10`), horizontal radius ~S×0.3, vertical ~S×0.14, centred at (S/2, S/2+10). Textured with a slightly darker ellipse overlay.
- 5–6 spines erupting from the mound: each spine is a thin pointed triangle path (3-point polygon), angled at irregular intervals across the mound. Spine bodies in dark organic grey-brown (~`#1a1a10`), widest at base (~S×0.04), tapering to a point.
- Team colour on spine tips: a small bright filled circle at each spine apex, shadowBlur ~8.
- A faint ridge line along the mound top (stroke in slightly lighter brown) to give depth.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawLurker !== 'undefined') lookup.drawLurker = drawLurker;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['LURKER_F'] = makeDirTextures(drawLurker, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['LURKER_E'] = makeDirTextures(drawLurker, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#lurkerDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+lurkerEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Lurker sprite — drawLurker, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 4: Swarm Host (armoured beetle)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void swarmHostDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawSwarmHost', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawSwarmHost dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void swarmHostEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.SWARM_HOST, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Swarm Host enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#swarmHostDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawSwarmHost` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Large armoured beetle, dark khaki-green Zerg palette (~`#2a3010` carapace, `#1a2008` underside).
- Carapace (top shell): wide rounded oval fill (horizontal ~S×0.36, vertical ~S×0.22), centred at (S/2, S/2+4). A ridge line stroke across the centre of the carapace for detail.
- Legs: 3 legs per side in dir 0/2 (6 total) — short angled line strokes from body underside; 2 visible in dir 1.
- Spawn vents: 2–3 oval openings on the carapace, arranged in a row. Each vent is a dark ellipse with team colour glow radiating from within (use shadowBlur ~12 on a small filled ellipse inside each vent).
- Dir 1 (side): shows the raised carapace profile; vents visible on top surface; legs visible below.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawSwarmHost !== 'undefined') lookup.drawSwarmHost = drawSwarmHost;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['SWARM_HOST_F'] = makeDirTextures(drawSwarmHost, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['SWARM_HOST_E'] = makeDirTextures(drawSwarmHost, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#swarmHostDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+swarmHostEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Swarm Host sprite — drawSwarmHost, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 5: Queen (tall Zerg unit)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void queenDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawQueen', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawQueen dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void queenEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.QUEEN, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Queen enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#queenDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawQueen` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Tall, slender Zerg queen. Ground unit (despite wings) — NOT in FLYING_UNITS. Dark purple-black palette (~`#1a0a2a`).
- Upper body/head: elongated narrow ellipse centred at (S/2, S/2-8).
- Wing appendages folded close: two swept-back triangle fills, one on each side of the upper body, angled backward (like pinched wings at rest). Slightly darker than the body.
- Tentacle arm extensions: 2 wavy line strokes (bezierCurveTo) extending downward from the lower body, curling outward at the tips.
- Leg section: narrow lower body ellipse, slightly wider than the torso.
- Bio-sacs on body flanks: 2 rounded filled ellipses, each with team colour glow (small filled circle inside, shadowBlur ~10).
- Dir 1 (side): narrow profile; one wing visible; arms trail behind.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawQueen !== 'undefined') lookup.drawQueen = drawQueen;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['QUEEN_F'] = makeDirTextures(drawQueen, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['QUEEN_E'] = makeDirTextures(drawQueen, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#queenDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+queenEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Queen sprite — drawQueen, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 6: Ultralisk (massive tank unit)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void ultralisksDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawUltralisk', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawUltralisk dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void ultralisksEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.ULTRALISK, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Ultralisk enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#ultralisksDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawUltralisk` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Massive unit — nearly fills the canvas. Very wide stance. Dark charcoal-brown Zerg palette (~`#1a1008` body).
- Carapace body: large rounded rect fill nearly spanning the canvas, heavy and low.
- Kaiser blade scythes: two large curved arc paths (ctx.arc with a thick lineWidth ~S×0.04, or a filled crescent shape), one rising from each shoulder. Blades arc upward and outward above the body, reaching ~S×0.35 above body top. Dark bone-grey fill with team colour glow on the inner blade edge.
- Bio-sac clusters on body flanks: 2–3 small ellipse groups on each side, team colour glow (shadowBlur ~10).
- Legs: 4 short heavy legs below the body in dir 0/2; 2 in dir 1.
- Dir 1 (side): shows one full blade arc rising dramatically above the body; the body profile shows a forward lean.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawUltralisk !== 'undefined') lookup.drawUltralisk = drawUltralisk;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['ULTRALISK_F'] = makeDirTextures(drawUltralisk, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['ULTRALISK_E'] = makeDirTextures(drawUltralisk, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#ultralisksDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+ultralisksEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Ultralisk sprite — drawUltralisk, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 7: Corruptor (air — flying spore ball)

Air unit — FLYING_UNITS update + elevation test.

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void corruptorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawCorruptor', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawCorruptor dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void corruptorEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.CORRUPTOR, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Corruptor enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void corruptorSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.CORRUPTOR, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double corruptorY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(corruptorY)
        .as("Corruptor Y (%.3f) must be higher than Marine Y (%.3f)".formatted(corruptorY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#corruptorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawCorruptor` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Flying spore ball, nearly circular. Dark purple-grey Zerg palette (~`#1a0a1a`).
- Body: large filled circle (radius ~S×0.26), centred at (S/2, S/2). Surface bumps: 5–6 small filled ellipse blobs distributed irregularly around the body edge, slightly darker, to give a lumpy organic surface.
- Tentacle clusters: 3–4 short wavy stroke paths (quadraticCurveTo) hanging below the body, lineWidth ~S×0.025, dark purple.
- Corruption spore glow: 2–3 small bright patches on the body surface in team colour, each as a filled circle with shadowBlur ~12, giving a bioluminescent spore effect.
- Dir 1 (side): same nearly circular profile; tentacles shift to one side; bumps on visible half.

- [ ] **Step 4: Update `FLYING_UNITS` and add to `smokeTestDrawFn` lookup table**

Update `FLYING_UNITS` to add `'CORRUPTOR'` (assumes E18a is done and includes OBSERVER, VOID_RAY, CARRIER):

```javascript
const FLYING_UNITS = new Set([
  'MEDIVAC', 'MUTALISK',
  'VIKING', 'RAVEN', 'BANSHEE', 'LIBERATOR', 'BATTLECRUISER',
  'OBSERVER', 'VOID_RAY', 'CARRIER',
  'CORRUPTOR'
]);
```

Then add to `smokeTestDrawFn`:

```javascript
    if (typeof drawCorruptor !== 'undefined') lookup.drawCorruptor = drawCorruptor;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['CORRUPTOR_F'] = makeDirTextures(drawCorruptor, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['CORRUPTOR_E'] = makeDirTextures(drawCorruptor, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#corruptorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+corruptorEnemySpawnsAndRendersInVisualizer+corruptorSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all three tests).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Corruptor sprite — drawCorruptor, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 8: Viper (air — flying Zerg caster)

Air unit — FLYING_UNITS update + elevation test.

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void viperDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawViper', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawViper dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void viperEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.VIPER, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Viper enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void viperSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.VIPER, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double viperY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(viperY)
        .as("Viper Y (%.3f) must be higher than Marine Y (%.3f)".formatted(viperY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#viperDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawViper` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Flying serpent with an elongated eel/snake body. Dark teal-purple Zerg palette (~`#0a1a1a`).
- Dir 0 (front — snake head toward viewer):
  - Head: slightly wider rounded ellipse at top-centre.
  - Body: S-curve sinuous path tapering to a point at the bottom — use bezierCurveTo to create the S-shape. Width starts at ~S×0.14 at head, tapers to a point.
  - Abduct claw at head: two small curved hook paths flanking the head, in team colour with shadowBlur ~12.
- Dir 1 (side — full serpentine body visible):
  - Elongated sinuous profile across the canvas width, head at left, tapering tail at right.
  - Body drawn as a bezierCurveTo path with varying width (thickest ~S×0.12 at head, tapering to point at tail).
  - Abduct claw at the head-left: two curved hooks extending left, team colour.
- Dir 2 (back): snake from above, tail pointing toward viewer; claw hooks less prominent.

- [ ] **Step 4: Update `FLYING_UNITS` and add to `smokeTestDrawFn` lookup table**

Update `FLYING_UNITS` to add `'VIPER'`:

```javascript
const FLYING_UNITS = new Set([
  'MEDIVAC', 'MUTALISK',
  'VIKING', 'RAVEN', 'BANSHEE', 'LIBERATOR', 'BATTLECRUISER',
  'OBSERVER', 'VOID_RAY', 'CARRIER',
  'CORRUPTOR', 'VIPER'
]);
```

Then add to `smokeTestDrawFn`:

```javascript
    if (typeof drawViper !== 'undefined') lookup.drawViper = drawViper;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['VIPER_F'] = makeDirTextures(drawViper, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['VIPER_E'] = makeDirTextures(drawViper, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#viperDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+viperEnemySpawnsAndRendersInVisualizer+viperSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all three tests).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Viper sprite — drawViper, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 9: Brood Lord (air — massive flying unit)

Air unit — FLYING_UNITS update to final E18b state + elevation test.

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void broodLordDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawBroodLord', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawBroodLord dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void broodLordEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.BROOD_LORD, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Brood Lord enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void broodLordSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.BROOD_LORD, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double broodLordY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(broodLordY)
        .as("Brood Lord Y (%.3f) must be higher than Marine Y (%.3f)".formatted(broodLordY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#broodLordDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawBroodLord` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Large flying unit. Dark charcoal-purple Zerg palette (~`#1a0a0a`).
- Dir 0/2 (top-down):
  - Two large swept-back wing fills, one each side: triangle/crescent paths extending from centre, angled backward (~S×0.44 span tip to tip). Wings slightly darker than body.
  - Central body: rounded oval fill between wings.
  - Broodling sacs hanging below body: 2–3 teardrop-shaped filled paths centred below the body oval, dark greenish-black. Team colour glow on each sac: small bright ellipse inside each sac, shadowBlur ~10.
- Dir 1 (side):
  - Wings visible as swept-back triangle fills behind the body.
  - Body profile in foreground.
  - Sacs hanging below body, team colour glow visible.

- [ ] **Step 4: Update `FLYING_UNITS` to E18b final state and add to `smokeTestDrawFn` lookup table**

Update `FLYING_UNITS` to the complete E18 final state:

```javascript
const FLYING_UNITS = new Set([
  'MEDIVAC', 'MUTALISK',
  'VIKING', 'RAVEN', 'BANSHEE', 'LIBERATOR', 'BATTLECRUISER',
  'OBSERVER', 'VOID_RAY', 'CARRIER',
  'BROOD_LORD', 'CORRUPTOR', 'VIPER'
]);
```

Then add to `smokeTestDrawFn`:

```javascript
    if (typeof drawBroodLord !== 'undefined') lookup.drawBroodLord = drawBroodLord;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['BROOD_LORD_F'] = makeDirTextures(drawBroodLord, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['BROOD_LORD_E'] = makeDirTextures(drawBroodLord, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#broodLordDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+broodLordEnemySpawnsAndRendersInVisualizer+broodLordSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all three tests).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Brood Lord sprite — drawBroodLord, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 10: Full Playwright suite + close issue

- [ ] **Step 1: Run full Playwright suite**

```bash
mvn test -Pplaywright -q
```

Expected: all browser-tagged tests pass. If any smoke test returns -1, the draw function is either missing from the `smokeTestDrawFn` lookup table or the function name doesn't match the `function` declaration exactly.

- [ ] **Step 2: Run full non-Playwright test suite**

```bash
mvn test -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Close issue**

```bash
gh issue close N --comment "All 9 Zerg sprites complete. FLYING_UNITS updated with Brood Lord, Corruptor, Viper. E18 complete — full sprite coverage for all 43 unit types. 36 Playwright tests added."
```

(Replace `N` with the issue number from Task 0.)
