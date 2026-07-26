# E17 Terran Sprites Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add canvas 2D draw functions and UNIT_MATS registrations for all 10 remaining Terran units, bring Terran sprite coverage from 3/13 to 13/13, add 5 flying types to FLYING_UNITS, extend ShowcaseResource with a Terran row, and update the Playwright showcase test.

**Architecture:** Each unit gets one `function drawX(ctx, S, dir, teamColor)` declaration added to `visualizer.js` before `initSpriteMaterials()`. The function is then registered as `UNIT_MATS['X_F']` and `UNIT_MATS['X_E']` inside `initSpriteMaterials()`. Air units are also added to the `FLYING_UNITS` set at the top of the file. Tests are written first (TDD): smoke test verifies the draw function produces non-transparent pixels; spawn test verifies the unit renders in the game; elevation test (air only) verifies the sprite Y is higher than a ground unit.

**Tech Stack:** Canvas 2D API, Three.js r128, Java 21, Playwright (Chromium), JUnit 5, Maven

---

### Task 0: Create GitHub issue under epic #83

**Files:** none

- [ ] **Step 1: Create issue**

```bash
gh issue create \
  --title "E17: remaining Terran sprites (10 units)" \
  --body "Complete 2D sprite coverage for all remaining Terran units.

## Units
Ground: Ghost, Cyclone, Widow Mine, Siege Tank, Thor
Air: Viking, Raven, Banshee, Liberator, Battlecruiser

## Acceptance criteria
- draw function for each unit (4 dirs × 2 team colours)
- UNIT_MATS registration for each (\_F and \_E)
- FLYING_UNITS: Viking, Raven, Banshee, Liberator, Battlecruiser
- ShowcaseResource Terran row (10 → 20 enemies)
- Playwright: smoke + spawn for all 10; elevation for 5 air; showcase count 10 → 20

Refs #83" \
  --label "enhancement"
```

Note the issue number printed — every commit in this plan uses `Refs #N` with it.

---

### Task 1: Ghost (ground infantry)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js` (add `drawGhost` before `initSpriteMaterials`, add 2 UNIT_MATS entries)
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java` (add 2 tests)

- [ ] **Step 1: Write failing smoke + spawn tests**

Add after the last Mutalisk test (around line 1285) in `VisualizerRenderTest.java`:

```java
@Test
@Tag("browser")
void ghostDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawGhost', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawGhost dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void ghostEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.GHOST, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Ghost enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#ghostDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL — `smokeTestDrawFn` returns -1 (function not found).

- [ ] **Step 3: Add `drawGhost` to `visualizer.js`**

Insert before the `// Populated by initSpriteMaterials()` comment (currently around line 1369):

```javascript
function drawGhost(ctx, S, dir, teamColor) {
  if (dir === 3) {
    ctx.save(); ctx.translate(S, 0); ctx.scale(-1, 1);
    drawGhost(ctx, S, 1, teamColor); ctx.restore(); return;
  }
  const cx = S / 2, cy = S / 2 + 2;

  if (dir === 2) {
    ctx.fillStyle = '#2a2a2a';
    ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.2, S * 0.23, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#333';
    ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.16, S * 0.14, S * 0.15, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#444';
    ctx.fillRect(cx + S * 0.06, cy - S * 0.28, S * 0.04, S * 0.22);
    ctx.fillStyle = '#1a1a1a';
    ctx.beginPath(); ctx.ellipse(cx - S * 0.08, cy + S * 0.25, S * 0.08, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(cx + S * 0.08, cy + S * 0.25, S * 0.08, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
    return;
  }
  if (dir === 1) {
    const bg = ctx.createRadialGradient(cx - S * 0.04, cy, S * 0.02, cx, cy, S * 0.22);
    bg.addColorStop(0, '#4a4a4a'); bg.addColorStop(1, '#1a1a1a');
    ctx.fillStyle = bg; ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.18, S * 0.23, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#333';
    ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.16, S * 0.14, S * 0.16, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = hexToRgba(teamColor, 0.85);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 6;
    ctx.beginPath(); ctx.ellipse(cx + S * 0.07, cy - S * 0.18, S * 0.055, S * 0.022, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
    ctx.fillStyle = '#555'; ctx.fillRect(cx + S * 0.08, cy - S * 0.04, S * 0.3, S * 0.05);
    ctx.fillStyle = '#333'; ctx.fillRect(cx + S * 0.28, cy - S * 0.02, S * 0.1, S * 0.03);
    ctx.fillStyle = '#1a1a1a';
    ctx.beginPath(); ctx.ellipse(cx, cy + S * 0.25, S * 0.1, S * 0.042, 0, 0, Math.PI * 2); ctx.fill();
    return;
  }
  // FRONT
  const bg = ctx.createRadialGradient(cx - S * 0.05, cy - S * 0.04, S * 0.03, cx, cy, S * 0.25);
  bg.addColorStop(0, '#4a4a4a'); bg.addColorStop(0.5, '#2a2a2a'); bg.addColorStop(1, '#111');
  ctx.fillStyle = bg; ctx.beginPath(); ctx.ellipse(cx, cy + S * 0.02, S * 0.2, S * 0.25, 0, 0, Math.PI * 2); ctx.fill();
  const hg = ctx.createRadialGradient(cx - S * 0.04, cy - S * 0.22, S * 0.02, cx, cy - S * 0.15, S * 0.16);
  hg.addColorStop(0, '#555'); hg.addColorStop(1, '#1a1a1a');
  ctx.fillStyle = hg; ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.15, S * 0.14, S * 0.16, 0, 0, Math.PI * 2); ctx.fill();
  ctx.fillStyle = hexToRgba(teamColor, 0.9);
  ctx.shadowColor = teamColor; ctx.shadowBlur = 8;
  ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.18, S * 0.08, S * 0.028, 0, 0, Math.PI * 2); ctx.fill();
  ctx.shadowBlur = 0;
  ctx.fillStyle = '#444'; ctx.fillRect(cx + S * 0.16, cy - S * 0.02, S * 0.25, S * 0.05);
  ctx.fillStyle = '#1a1a1a';
  ctx.beginPath(); ctx.ellipse(cx - S * 0.09, cy + S * 0.25, S * 0.08, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.ellipse(cx + S * 0.09, cy + S * 0.25, S * 0.08, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
}
```

- [ ] **Step 4: Register in `initSpriteMaterials()`**

Add after the MEDIVAC entries:

```javascript
  UNIT_MATS['GHOST_F']    = makeDirTextures(drawGhost,    TEAM_COLOR_FRIENDLY);
  UNIT_MATS['GHOST_E']    = makeDirTextures(drawGhost,    TEAM_COLOR_ENEMY);
```

- [ ] **Step 5: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#ghostDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+ghostEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Ghost sprite — drawGhost, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 2: Cyclone (hover vehicle)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing tests**

```java
@Test
@Tag("browser")
void cycloneDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawCyclone', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawCyclone dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void cycloneEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.CYCLONE, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Cyclone enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#cycloneDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Add `drawCyclone`**

```javascript
function drawCyclone(ctx, S, dir, teamColor) {
  if (dir === 3) {
    ctx.save(); ctx.translate(S, 0); ctx.scale(-1, 1);
    drawCyclone(ctx, S, 1, teamColor); ctx.restore(); return;
  }
  const cx = S / 2, cy = S / 2 + 4;

  if (dir === 1) {
    // Side — low hover chassis
    const cg = ctx.createLinearGradient(cx - S * 0.34, cy, cx + S * 0.34, cy);
    cg.addColorStop(0, '#2a3a4a'); cg.addColorStop(0.5, '#3a5060'); cg.addColorStop(1, '#2a3a4a');
    ctx.fillStyle = cg;
    ctx.beginPath(); ctx.ellipse(cx, cy + S * 0.06, S * 0.32, S * 0.12, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#1a2a3a';
    ctx.beginPath(); ctx.ellipse(cx, cy + S * 0.16, S * 0.1, S * 0.05, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#3a4a5a';
    ctx.beginPath(); ctx.ellipse(cx - S * 0.06, cy - S * 0.04, S * 0.12, S * 0.1, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#555'; ctx.fillRect(cx - S * 0.04, cy - S * 0.1, S * 0.25, S * 0.04);
    ctx.fillStyle = hexToRgba(teamColor, 0.7);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 8;
    ctx.beginPath(); ctx.ellipse(cx - S * 0.08, cy - S * 0.04, S * 0.055, S * 0.045, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
  } else {
    // FRONT / BACK
    const cg = ctx.createLinearGradient(cx - S * 0.36, cy, cx + S * 0.36, cy);
    cg.addColorStop(0, '#2a3a4a'); cg.addColorStop(0.5, '#3a5060'); cg.addColorStop(1, '#2a3a4a');
    ctx.fillStyle = cg;
    ctx.beginPath(); ctx.ellipse(cx, cy + S * 0.06, S * 0.36, S * 0.14, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#1a2a3a';
    [-0.26, 0.26].forEach(dx => {
      ctx.beginPath(); ctx.ellipse(cx + dx * S, cy + S * 0.14, S * 0.07, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
    });
    ctx.fillStyle = '#3a4a5a';
    ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.06, S * 0.14, S * 0.12, 0, 0, Math.PI * 2); ctx.fill();
    const armLen = dir === 0 ? -S * 0.22 : S * 0.22;
    ctx.fillStyle = '#555'; ctx.fillRect(cx - S * 0.025, cy - S * 0.12, S * 0.05, armLen);
    ctx.fillStyle = '#666';
    ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.12 + armLen, S * 0.06, S * 0.06, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = hexToRgba(teamColor, 0.7);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 8;
    ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.06, S * 0.07, S * 0.055, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
  }
}
```

- [ ] **Step 4: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['CYCLONE_F']  = makeDirTextures(drawCyclone,  TEAM_COLOR_FRIENDLY);
  UNIT_MATS['CYCLONE_E']  = makeDirTextures(drawCyclone,  TEAM_COLOR_ENEMY);
```

- [ ] **Step 5: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#cycloneDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+cycloneEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Cyclone sprite — drawCyclone, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 3: Widow Mine (deployed)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing tests**

```java
@Test
@Tag("browser")
void widowMineDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawWidowMine', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawWidowMine dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void widowMineEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.WIDOW_MINE, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Widow Mine enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#widowMineDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Add `drawWidowMine`**

```javascript
function drawWidowMine(ctx, S, dir, teamColor) {
  if (dir === 3) {
    ctx.save(); ctx.translate(S, 0); ctx.scale(-1, 1);
    drawWidowMine(ctx, S, 1, teamColor); ctx.restore(); return;
  }
  const cx = S / 2, cy = S / 2 + 4;

  // Legs
  ctx.strokeStyle = '#445566'; ctx.lineWidth = S * 0.035;
  if (dir === 1) {
    [[-0.55 * Math.PI, 0.28], [0.15 * Math.PI, 0.28]].forEach(([a, r]) => {
      ctx.beginPath(); ctx.moveTo(cx, cy + S * 0.04);
      ctx.lineTo(cx + Math.cos(a) * S * r, cy + S * 0.16 + Math.sin(a) * S * 0.06); ctx.stroke();
    });
  } else {
    [[-0.28, 0.2], [0, 0.24], [0.28, 0.2]].forEach(([dx, dy]) => {
      ctx.beginPath(); ctx.moveTo(cx, cy + S * 0.04);
      ctx.lineTo(cx + dx * S, cy + dy * S); ctx.stroke();
    });
  }

  // Body
  const bg = ctx.createRadialGradient(cx - S * 0.06, cy - S * 0.06, S * 0.02, cx, cy, S * 0.2);
  bg.addColorStop(0, '#556677'); bg.addColorStop(0.6, '#334455'); bg.addColorStop(1, '#1a2a33');
  ctx.fillStyle = bg; ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.19, S * 0.17, 0, 0, Math.PI * 2); ctx.fill();

  // Targeting spike
  ctx.fillStyle = '#3a4a55';
  ctx.beginPath();
  ctx.moveTo(cx - S * 0.03, cy - S * 0.12); ctx.lineTo(cx + S * 0.03, cy - S * 0.12);
  ctx.lineTo(cx + S * 0.015, cy - S * 0.36); ctx.lineTo(cx - S * 0.015, cy - S * 0.36);
  ctx.closePath(); ctx.fill();

  // Sensor eye
  ctx.fillStyle = teamColor;
  ctx.shadowColor = teamColor; ctx.shadowBlur = 10;
  ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.34, S * 0.04, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
  ctx.shadowBlur = 0;

  ctx.strokeStyle = '#667788'; ctx.lineWidth = S * 0.02;
  ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.19, S * 0.17, 0, 0, Math.PI * 2); ctx.stroke();
}
```

- [ ] **Step 4: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['WIDOW_MINE_F'] = makeDirTextures(drawWidowMine, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['WIDOW_MINE_E'] = makeDirTextures(drawWidowMine, TEAM_COLOR_ENEMY);
```

- [ ] **Step 5: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#widowMineDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+widowMineEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Widow Mine sprite — drawWidowMine, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 4: Siege Tank (mobile mode)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing tests**

```java
@Test
@Tag("browser")
void siegeTankDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawSiegeTank', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawSiegeTank dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void siegeTankEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.SIEGE_TANK, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Siege Tank enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#siegeTankDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Add `drawSiegeTank`**

```javascript
function drawSiegeTank(ctx, S, dir, teamColor) {
  if (dir === 3) {
    ctx.save(); ctx.translate(S, 0); ctx.scale(-1, 1);
    drawSiegeTank(ctx, S, 1, teamColor); ctx.restore(); return;
  }
  const cx = S / 2, cy = S / 2 + 6;

  if (dir === 1) {
    // Tracks
    ctx.fillStyle = '#1a1a1a';
    ctx.fillRect(cx - S * 0.36, cy + S * 0.06, S * 0.72, S * 0.14);
    ctx.fillStyle = '#2a2a2a';
    ctx.fillRect(cx - S * 0.34, cy + S * 0.08, S * 0.68, S * 0.1);
    ctx.fillStyle = '#333';
    for (let i = 0; i < 5; i++) {
      ctx.beginPath(); ctx.ellipse(cx - S * 0.28 + i * S * 0.14, cy + S * 0.13, S * 0.03, S * 0.025, 0, 0, Math.PI * 2); ctx.fill();
    }
    // Body
    const bg = ctx.createLinearGradient(cx, cy - S * 0.14, cx, cy + S * 0.06);
    bg.addColorStop(0, '#5a6a4a'); bg.addColorStop(1, '#3a4a32');
    ctx.fillStyle = bg; ctx.fillRect(cx - S * 0.28, cy - S * 0.12, S * 0.56, S * 0.2);
    // Turret
    ctx.fillStyle = '#4a5a3a';
    ctx.beginPath(); ctx.ellipse(cx - S * 0.06, cy - S * 0.14, S * 0.16, S * 0.12, 0, 0, Math.PI * 2); ctx.fill();
    // Barrel (side — full length)
    ctx.fillStyle = '#3a4a2a'; ctx.fillRect(cx - S * 0.06, cy - S * 0.17, S * 0.42, S * 0.06);
    ctx.fillStyle = '#2a3a22'; ctx.fillRect(cx + S * 0.32, cy - S * 0.16, S * 0.06, S * 0.04);
    // Cockpit
    ctx.fillStyle = hexToRgba(teamColor, 0.65);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 6;
    ctx.beginPath(); ctx.ellipse(cx - S * 0.08, cy - S * 0.14, S * 0.05, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
  } else if (dir === 2) {
    // Rear
    ctx.fillStyle = '#1a1a1a'; ctx.fillRect(cx - S * 0.32, cy + S * 0.06, S * 0.64, S * 0.14);
    const bg = ctx.createLinearGradient(cx, cy - S * 0.14, cx, cy + S * 0.06);
    bg.addColorStop(0, '#4a5a3a'); bg.addColorStop(1, '#2a3a22');
    ctx.fillStyle = bg; ctx.fillRect(cx - S * 0.3, cy - S * 0.1, S * 0.6, S * 0.18);
    ctx.fillStyle = '#2a2a2a';
    [-0.14, 0.14].forEach(dx => { ctx.fillRect(cx + dx * S - S * 0.03, cy - S * 0.08, S * 0.06, S * 0.12); });
    ctx.fillStyle = '#4a5a3a';
    ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.12, S * 0.14, S * 0.1, 0, 0, Math.PI * 2); ctx.fill();
  } else {
    // FRONT — barrel foreshortened
    ctx.fillStyle = '#1a1a1a'; ctx.fillRect(cx - S * 0.34, cy + S * 0.06, S * 0.68, S * 0.14);
    const bg = ctx.createLinearGradient(cx, cy - S * 0.14, cx, cy + S * 0.06);
    bg.addColorStop(0, '#5a6a4a'); bg.addColorStop(1, '#3a4a32');
    ctx.fillStyle = bg; ctx.fillRect(cx - S * 0.3, cy - S * 0.1, S * 0.6, S * 0.18);
    ctx.fillStyle = '#4a5a3a';
    ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.14, S * 0.18, S * 0.13, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#2a3a22';
    ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.14, S * 0.07, S * 0.055, 0, 0, Math.PI * 2); ctx.fill();
    ctx.strokeStyle = '#3a4a2a'; ctx.lineWidth = S * 0.015;
    ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.14, S * 0.09, S * 0.07, 0, 0, Math.PI * 2); ctx.stroke();
    ctx.fillStyle = hexToRgba(teamColor, 0.65);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 6;
    ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.04, S * 0.05, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
  }
}
```

- [ ] **Step 4: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['SIEGE_TANK_F'] = makeDirTextures(drawSiegeTank, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['SIEGE_TANK_E'] = makeDirTextures(drawSiegeTank, TEAM_COLOR_ENEMY);
```

- [ ] **Step 5: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#siegeTankDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+siegeTankEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Siege Tank sprite (mobile mode) — drawSiegeTank, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 5: Thor (massive walker)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing tests**

```java
@Test
@Tag("browser")
void thorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawThor', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawThor dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void thorEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.THOR, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Thor enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#thorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Add `drawThor`**

```javascript
function drawThor(ctx, S, dir, teamColor) {
  if (dir === 3) {
    ctx.save(); ctx.translate(S, 0); ctx.scale(-1, 1);
    drawThor(ctx, S, 1, teamColor); ctx.restore(); return;
  }
  const cx = S / 2, cy = S / 2 + 2;

  if (dir === 1) {
    // Legs (side: 2 visible)
    ctx.fillStyle = '#2a3040';
    [[-0.12, 0.22], [0.08, 0.22]].forEach(([dx, dy]) => {
      ctx.fillRect(cx + dx * S, cy + S * 0.04, S * 0.12, dy * S);
    });
    ctx.fillStyle = '#1a2030';
    ctx.beginPath(); ctx.ellipse(cx, cy + S * 0.28, S * 0.26, S * 0.06, 0, 0, Math.PI * 2); ctx.fill();
    // Body
    const bg = ctx.createLinearGradient(cx, cy - S * 0.2, cx, cy + S * 0.04);
    bg.addColorStop(0, '#4a5060'); bg.addColorStop(1, '#2a3040');
    ctx.fillStyle = bg; ctx.fillRect(cx - S * 0.2, cy - S * 0.18, S * 0.4, S * 0.22);
    // Shoulder cannon
    ctx.fillStyle = '#3a4050';
    ctx.fillRect(cx + S * 0.18, cy - S * 0.24, S * 0.14, S * 0.12);
    ctx.fillStyle = '#555'; ctx.fillRect(cx + S * 0.26, cy - S * 0.2, S * 0.08, S * 0.06);
    // Core reactor
    ctx.fillStyle = hexToRgba(teamColor, 0.85);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 12;
    ctx.beginPath(); ctx.ellipse(cx - S * 0.04, cy - S * 0.07, S * 0.08, S * 0.07, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
  } else {
    // FRONT / BACK — 4 legs
    ctx.fillStyle = '#2a3040';
    [-0.28, 0.28].forEach(dx => {
      ctx.fillRect(cx + dx * S - S * 0.06, cy + S * 0.04, S * 0.12, S * 0.22);
    });
    ctx.fillStyle = '#1a2030';
    ctx.beginPath(); ctx.ellipse(cx, cy + S * 0.26, S * 0.36, S * 0.07, 0, 0, Math.PI * 2); ctx.fill();
    // Body
    const bg = ctx.createRadialGradient(cx - S * 0.06, cy - S * 0.06, S * 0.04, cx, cy, S * 0.32);
    bg.addColorStop(0, '#505566'); bg.addColorStop(1, '#1e2434');
    ctx.fillStyle = bg; ctx.fillRect(cx - S * 0.24, cy - S * 0.2, S * 0.48, S * 0.26);
    // Shoulders
    ctx.fillStyle = '#3a4050';
    [-0.28, 0.28].forEach(dx => { ctx.fillRect(cx + dx * S - S * 0.06, cy - S * 0.26, S * 0.14, S * 0.12); });
    // Shoulder cannons
    ctx.fillStyle = '#556';
    if (dir === 0) {
      [-0.28, 0.28].forEach(dx => {
        ctx.beginPath(); ctx.ellipse(cx + dx * S, cy - S * 0.22, S * 0.05, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
      });
      ctx.fillStyle = '#445';
      [-0.2, 0.2].forEach(dx => { ctx.fillRect(cx + dx * S - S * 0.02, cy - S * 0.12, S * 0.04, S * 0.2); });
    } else {
      [-0.28, 0.28].forEach(dx => { ctx.fillRect(cx + dx * S - S * 0.04, cy - S * 0.3, S * 0.08, S * 0.08); });
    }
    // Core reactor
    ctx.fillStyle = hexToRgba(teamColor, 0.85);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 14;
    ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.07, S * 0.1, S * 0.09, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
  }
}
```

- [ ] **Step 4: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['THOR_F']     = makeDirTextures(drawThor,     TEAM_COLOR_FRIENDLY);
  UNIT_MATS['THOR_E']     = makeDirTextures(drawThor,     TEAM_COLOR_ENEMY);
```

- [ ] **Step 5: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#thorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+thorEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Thor sprite — drawThor, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 6: Viking (air)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js` (add draw fn, UNIT_MATS, FLYING_UNITS)
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing tests**

```java
@Test
@Tag("browser")
void vikingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawViking', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawViking dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void vikingEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.VIKING, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Viking enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void vikingSpawnsHigherThanGroundUnit() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.MARINE, new Point2d(20, 20));
    engine.observe();
    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    List<Double> marineYs = ((List<?>) page.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList();
    double marineY = marineYs.get(0);
    page.close();

    orchestrator.startGame();
    Page page2 = browser.newPage();
    page2.navigate(pageUrl.toString());
    page2.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.VIKING, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    List<Double> vikingYs = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList();
    double vikingY = vikingYs.get(0);
    page2.close();

    assertThat(vikingY)
        .as("Viking Y (%.3f) must be higher than Marine Y (%.3f)".formatted(vikingY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#vikingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Add `drawViking` to `visualizer.js`**

```javascript
function drawViking(ctx, S, dir, teamColor) {
  if (dir === 3) {
    ctx.save(); ctx.translate(S, 0); ctx.scale(-1, 1);
    drawViking(ctx, S, 1, teamColor); ctx.restore(); return;
  }
  const cx = S / 2, cy = S / 2;

  if (dir === 0 || dir === 2) {
    // Wings
    ctx.fillStyle = '#3a4a5a';
    ctx.beginPath();
    ctx.moveTo(cx - S * 0.04, cy - S * 0.04); ctx.lineTo(cx - S * 0.44, cy + S * 0.06);
    ctx.lineTo(cx - S * 0.4, cy + S * 0.16); ctx.lineTo(cx - S * 0.06, cy + S * 0.06); ctx.closePath(); ctx.fill();
    ctx.beginPath();
    ctx.moveTo(cx + S * 0.04, cy - S * 0.04); ctx.lineTo(cx + S * 0.44, cy + S * 0.06);
    ctx.lineTo(cx + S * 0.4, cy + S * 0.16); ctx.lineTo(cx + S * 0.06, cy + S * 0.06); ctx.closePath(); ctx.fill();
    // Engine nacelles
    ctx.fillStyle = '#2a3a4a';
    [-0.4, 0.4].forEach(dx => {
      ctx.beginPath(); ctx.ellipse(cx + dx * S, cy + S * 0.1, S * 0.06, S * 0.1, 0, 0, Math.PI * 2); ctx.fill();
    });
    // Engine glow
    ctx.fillStyle = hexToRgba(teamColor, 0.8);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 10;
    const exhaustY = dir === 0 ? S * 0.18 : S * 0.02;
    [-0.4, 0.4].forEach(dx => {
      ctx.beginPath(); ctx.ellipse(cx + dx * S, cy + exhaustY, S * 0.04, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
    });
    ctx.shadowBlur = 0;
    // Fuselage
    ctx.fillStyle = '#4a5a6a';
    ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.07, S * 0.22, 0, 0, Math.PI * 2); ctx.fill();
    // Cockpit
    ctx.fillStyle = hexToRgba(teamColor, 0.65);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 6;
    const cockpitY = dir === 0 ? cy - S * 0.12 : cy - S * 0.06;
    ctx.beginPath(); ctx.ellipse(cx, cockpitY, S * 0.05, S * 0.045, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
  } else {
    // SIDE profile
    ctx.fillStyle = '#3a4a5a';
    ctx.beginPath();
    ctx.moveTo(cx - S * 0.28, cy + S * 0.04); ctx.lineTo(cx + S * 0.28, cy - S * 0.04);
    ctx.lineTo(cx + S * 0.28, cy + S * 0.1); ctx.lineTo(cx - S * 0.28, cy + S * 0.1); ctx.closePath(); ctx.fill();
    // Wing (top-down angle)
    ctx.beginPath();
    ctx.moveTo(cx - S * 0.04, cy); ctx.lineTo(cx + S * 0.04, cy - S * 0.02);
    ctx.lineTo(cx + S * 0.02, cy - S * 0.26); ctx.lineTo(cx - S * 0.04, cy - S * 0.22); ctx.closePath(); ctx.fill();
    // Engine nacelle
    ctx.fillStyle = '#2a3a4a';
    ctx.beginPath(); ctx.ellipse(cx + S * 0.02, cy - S * 0.22, S * 0.055, S * 0.06, -0.2, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = hexToRgba(teamColor, 0.7);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 8;
    ctx.beginPath(); ctx.ellipse(cx + S * 0.04, cy - S * 0.25, S * 0.035, S * 0.035, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
    // Cockpit
    ctx.fillStyle = '#4a5a6a';
    ctx.beginPath(); ctx.ellipse(cx - S * 0.16, cy + S * 0.02, S * 0.06, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = hexToRgba(teamColor, 0.65);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 5;
    ctx.beginPath(); ctx.ellipse(cx - S * 0.18, cy + S * 0.01, S * 0.035, S * 0.026, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
  }
}
```

- [ ] **Step 4: Update `FLYING_UNITS` and register in `initSpriteMaterials()`**

Change line 6:
```javascript
const FLYING_UNITS = new Set(['MEDIVAC', 'MUTALISK', 'VIKING']);
```

Add UNIT_MATS entries:
```javascript
  UNIT_MATS['VIKING_F']   = makeDirTextures(drawViking,   TEAM_COLOR_FRIENDLY);
  UNIT_MATS['VIKING_E']   = makeDirTextures(drawViking,   TEAM_COLOR_ENEMY);
```

- [ ] **Step 5: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#vikingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+vikingEnemySpawnsAndRendersInVisualizer+vikingSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all three tests).

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Viking sprite — drawViking, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 7: Raven (air)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing tests**

```java
@Test
@Tag("browser")
void ravenDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawRaven', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawRaven dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void ravenEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.RAVEN, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Raven enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void ravenSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.RAVEN, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double ravenY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(ravenY).as("Raven Y (%.3f) must be higher than Marine Y (%.3f)".formatted(ravenY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#ravenDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Add `drawRaven`**

```javascript
function drawRaven(ctx, S, dir, teamColor) {
  if (dir === 3) {
    ctx.save(); ctx.translate(S, 0); ctx.scale(-1, 1);
    drawRaven(ctx, S, 1, teamColor); ctx.restore(); return;
  }
  const cx = S / 2, cy = S / 2;

  // Outer sensor ring
  ctx.strokeStyle = '#556677'; ctx.lineWidth = S * 0.03;
  ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.36, S * 0.3, 0, 0, Math.PI * 2); ctx.stroke();

  // Engine glow ring
  ctx.strokeStyle = hexToRgba(teamColor, 0.5);
  ctx.shadowColor = teamColor; ctx.shadowBlur = 12;
  ctx.lineWidth = S * 0.025;
  ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.32, S * 0.26, 0, 0, Math.PI * 2); ctx.stroke();
  ctx.shadowBlur = 0;

  // Body disc
  const bg = ctx.createRadialGradient(cx - S * 0.08, cy - S * 0.08, S * 0.03, cx, cy, S * 0.24);
  bg.addColorStop(0, '#5a6a7a'); bg.addColorStop(0.6, '#3a4a5a'); bg.addColorStop(1, '#1a2a3a');
  ctx.fillStyle = bg; ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.24, S * 0.2, 0, 0, Math.PI * 2); ctx.fill();

  // Sensor pods (slight offset per direction)
  const offset = (dir === 0 || dir === 2) ? 0 : S * 0.04;
  ctx.fillStyle = '#667788';
  [-S * 0.16, S * 0.16].forEach(dx => {
    ctx.beginPath(); ctx.ellipse(cx + dx + offset, cy + S * 0.04, S * 0.055, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
  });

  // Central eye
  ctx.fillStyle = hexToRgba(teamColor, 0.9);
  ctx.shadowColor = teamColor; ctx.shadowBlur = 10;
  ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.07, S * 0.06, 0, 0, Math.PI * 2); ctx.fill();
  ctx.shadowBlur = 0;
}
```

- [ ] **Step 4: Update `FLYING_UNITS` and register**

```javascript
const FLYING_UNITS = new Set(['MEDIVAC', 'MUTALISK', 'VIKING', 'RAVEN']);
```

```javascript
  UNIT_MATS['RAVEN_F']    = makeDirTextures(drawRaven,    TEAM_COLOR_FRIENDLY);
  UNIT_MATS['RAVEN_E']    = makeDirTextures(drawRaven,    TEAM_COLOR_ENEMY);
```

- [ ] **Step 5: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#ravenDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+ravenEnemySpawnsAndRendersInVisualizer+ravenSpawnsHigherThanGroundUnit" -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Raven sprite — drawRaven, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 8: Banshee (air)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing tests**

```java
@Test
@Tag("browser")
void bansheeDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawBanshee', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawBanshee dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void bansheeEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.BANSHEE, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Banshee enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void bansheeSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.BANSHEE, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double bansheeY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(bansheeY).as("Banshee Y (%.3f) must be higher than Marine Y (%.3f)".formatted(bansheeY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#bansheeDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Add `drawBanshee`**

```javascript
function drawBanshee(ctx, S, dir, teamColor) {
  if (dir === 3) {
    ctx.save(); ctx.translate(S, 0); ctx.scale(-1, 1);
    drawBanshee(ctx, S, 1, teamColor); ctx.restore(); return;
  }
  const cx = S / 2, cy = S / 2;

  if (dir === 0 || dir === 2) {
    // Engine pods
    ctx.fillStyle = '#2a3340';
    [-0.24, 0.24].forEach(dx => {
      ctx.beginPath(); ctx.ellipse(cx + dx * S, cy + S * 0.04, S * 0.1, S * 0.18, 0, 0, Math.PI * 2); ctx.fill();
    });
    // Rotor blur
    ctx.strokeStyle = hexToRgba(teamColor, 0.35); ctx.lineWidth = S * 0.02;
    [-0.24, 0.24].forEach(dx => {
      ctx.beginPath(); ctx.ellipse(cx + dx * S, cy + S * 0.04, S * 0.18, S * 0.06, 0, 0, Math.PI * 2); ctx.stroke();
    });
    // Engine glow
    ctx.fillStyle = hexToRgba(teamColor, 0.75);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 10;
    const exhaustY = dir === 0 ? S * 0.2 : -S * 0.12;
    [-0.24, 0.24].forEach(dx => {
      ctx.beginPath(); ctx.ellipse(cx + dx * S, cy + exhaustY, S * 0.05, S * 0.05, 0, 0, Math.PI * 2); ctx.fill();
    });
    ctx.shadowBlur = 0;
    // Stub wings
    ctx.fillStyle = '#3a4050';
    ctx.beginPath();
    ctx.moveTo(cx - S * 0.06, cy - S * 0.04); ctx.lineTo(cx - S * 0.26, cy + S * 0.06);
    ctx.lineTo(cx - S * 0.24, cy + S * 0.12); ctx.lineTo(cx - S * 0.06, cy + S * 0.04); ctx.closePath(); ctx.fill();
    ctx.beginPath();
    ctx.moveTo(cx + S * 0.06, cy - S * 0.04); ctx.lineTo(cx + S * 0.26, cy + S * 0.06);
    ctx.lineTo(cx + S * 0.24, cy + S * 0.12); ctx.lineTo(cx + S * 0.06, cy + S * 0.04); ctx.closePath(); ctx.fill();
    // Fuselage
    const fg = ctx.createLinearGradient(cx, cy - S * 0.32, cx, cy + S * 0.1);
    fg.addColorStop(0, '#4a5060'); fg.addColorStop(1, '#2a3040');
    ctx.fillStyle = fg; ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.1, S * 0.09, S * 0.24, 0, 0, Math.PI * 2); ctx.fill();
    // Cockpit
    ctx.fillStyle = hexToRgba(teamColor, 0.7);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 8;
    const cockY = dir === 0 ? cy - S * 0.28 : cy + S * 0.06;
    ctx.beginPath(); ctx.ellipse(cx, cockY, S * 0.055, S * 0.05, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
  } else {
    // SIDE
    const fg = ctx.createLinearGradient(cx - S * 0.3, cy, cx + S * 0.3, cy);
    fg.addColorStop(0, '#2a3040'); fg.addColorStop(0.5, '#4a5060'); fg.addColorStop(1, '#2a3040');
    ctx.fillStyle = fg; ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.32, S * 0.1, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#2a3340';
    ctx.beginPath(); ctx.ellipse(cx + S * 0.16, cy + S * 0.04, S * 0.1, S * 0.12, 0.3, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = hexToRgba(teamColor, 0.7);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 9;
    ctx.beginPath(); ctx.ellipse(cx + S * 0.24, cy + S * 0.04, S * 0.04, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
    ctx.fillStyle = hexToRgba(teamColor, 0.65);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 6;
    ctx.beginPath(); ctx.ellipse(cx - S * 0.24, cy - S * 0.02, S * 0.055, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
  }
}
```

- [ ] **Step 4: Update `FLYING_UNITS` and register**

```javascript
const FLYING_UNITS = new Set(['MEDIVAC', 'MUTALISK', 'VIKING', 'RAVEN', 'BANSHEE']);
```

```javascript
  UNIT_MATS['BANSHEE_F']  = makeDirTextures(drawBanshee,  TEAM_COLOR_FRIENDLY);
  UNIT_MATS['BANSHEE_E']  = makeDirTextures(drawBanshee,  TEAM_COLOR_ENEMY);
```

- [ ] **Step 5: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#bansheeDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+bansheeEnemySpawnsAndRendersInVisualizer+bansheeSpawnsHigherThanGroundUnit" -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Banshee sprite — drawBanshee, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 9: Liberator (air)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing tests**

```java
@Test
@Tag("browser")
void liberatorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawLiberator', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawLiberator dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void liberatorEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.LIBERATOR, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Liberator enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void liberatorSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.LIBERATOR, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double liberatorY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(liberatorY).as("Liberator Y (%.3f) must be higher than Marine Y (%.3f)".formatted(liberatorY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#liberatorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Add `drawLiberator`**

```javascript
function drawLiberator(ctx, S, dir, teamColor) {
  if (dir === 3) {
    ctx.save(); ctx.translate(S, 0); ctx.scale(-1, 1);
    drawLiberator(ctx, S, 1, teamColor); ctx.restore(); return;
  }
  const cx = S / 2, cy = S / 2;

  if (dir === 0 || dir === 2) {
    // Wings
    ctx.fillStyle = '#2a3848';
    [-1, 1].forEach(side => {
      ctx.beginPath();
      ctx.moveTo(cx + side * S * 0.06, cy + S * 0.04);
      ctx.lineTo(cx + side * S * 0.44, cy + S * 0.1);
      ctx.lineTo(cx + side * S * 0.44, cy + S * 0.2);
      ctx.lineTo(cx + side * S * 0.04, cy + S * 0.1);
      ctx.closePath(); ctx.fill();
    });
    // Weapon pods
    ctx.fillStyle = '#1a2838';
    [-0.38, 0.38].forEach(dx => {
      ctx.beginPath(); ctx.ellipse(cx + dx * S, cy + S * 0.14, S * 0.07, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
    });
    // Engine wash
    ctx.fillStyle = hexToRgba(teamColor, 0.6);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 12;
    const exhaustY = dir === 0 ? S * 0.19 : S * 0.08;
    [-0.38, 0.38].forEach(dx => {
      ctx.beginPath(); ctx.ellipse(cx + dx * S, cy + exhaustY, S * 0.04, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
    });
    ctx.shadowBlur = 0;
    // Body
    const bg = ctx.createRadialGradient(cx, cy - S * 0.08, S * 0.04, cx, cy, S * 0.18);
    bg.addColorStop(0, '#5a6878'); bg.addColorStop(1, '#2a3848');
    ctx.fillStyle = bg; ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.04, S * 0.14, S * 0.3, 0, 0, Math.PI * 2); ctx.fill();
    // Cockpit
    ctx.fillStyle = hexToRgba(teamColor, 0.7);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 8;
    const cockY = dir === 0 ? cy - S * 0.28 : cy + S * 0.22;
    ctx.beginPath(); ctx.ellipse(cx, cockY, S * 0.06, S * 0.055, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
  } else {
    // SIDE
    const bg = ctx.createLinearGradient(cx - S * 0.34, cy, cx + S * 0.34, cy);
    bg.addColorStop(0, '#1a2838'); bg.addColorStop(0.4, '#4a5868'); bg.addColorStop(1, '#1a2838');
    ctx.fillStyle = bg; ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.34, S * 0.12, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#2a3848';
    ctx.beginPath();
    ctx.moveTo(cx - S * 0.04, cy - S * 0.06); ctx.lineTo(cx + S * 0.04, cy - S * 0.06);
    ctx.lineTo(cx + S * 0.06, cy - S * 0.3); ctx.lineTo(cx - S * 0.04, cy - S * 0.26); ctx.closePath(); ctx.fill();
    ctx.fillStyle = '#1a2838';
    ctx.beginPath(); ctx.ellipse(cx + S * 0.02, cy - S * 0.28, S * 0.06, S * 0.05, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = hexToRgba(teamColor, 0.65);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 9;
    ctx.beginPath(); ctx.ellipse(cx + S * 0.05, cy - S * 0.31, S * 0.04, S * 0.035, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
    ctx.fillStyle = hexToRgba(teamColor, 0.65);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 6;
    ctx.beginPath(); ctx.ellipse(cx - S * 0.28, cy - S * 0.03, S * 0.055, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
  }
}
```

- [ ] **Step 4: Update `FLYING_UNITS` and register**

```javascript
const FLYING_UNITS = new Set(['MEDIVAC', 'MUTALISK', 'VIKING', 'RAVEN', 'BANSHEE', 'LIBERATOR']);
```

```javascript
  UNIT_MATS['LIBERATOR_F'] = makeDirTextures(drawLiberator, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['LIBERATOR_E'] = makeDirTextures(drawLiberator, TEAM_COLOR_ENEMY);
```

- [ ] **Step 5: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#liberatorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+liberatorEnemySpawnsAndRendersInVisualizer+liberatorSpawnsHigherThanGroundUnit" -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Liberator sprite — drawLiberator, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 10: Battlecruiser (air)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing tests**

```java
@Test
@Tag("browser")
void battlecruiserDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawBattlecruiser', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawBattlecruiser dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void battlecruiserEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.BATTLECRUISER, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Battlecruiser enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void battlecruiserSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.BATTLECRUISER, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double bcY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(bcY).as("Battlecruiser Y (%.3f) must be higher than Marine Y (%.3f)".formatted(bcY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#battlecruiserDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Add `drawBattlecruiser`**

```javascript
function drawBattlecruiser(ctx, S, dir, teamColor) {
  if (dir === 3) {
    ctx.save(); ctx.translate(S, 0); ctx.scale(-1, 1);
    drawBattlecruiser(ctx, S, 1, teamColor); ctx.restore(); return;
  }
  const cx = S / 2, cy = S / 2;

  if (dir === 0 || dir === 2) {
    // Hull
    const hg = ctx.createRadialGradient(cx - S * 0.08, cy - S * 0.06, S * 0.04, cx, cy, S * 0.44);
    hg.addColorStop(0, '#6a7480'); hg.addColorStop(0.5, '#3a4450'); hg.addColorStop(1, '#1a2430');
    ctx.fillStyle = hg; ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.42, S * 0.32, 0, 0, Math.PI * 2); ctx.fill();
    // Hull plates
    ctx.strokeStyle = '#2a3440'; ctx.lineWidth = S * 0.015;
    [-0.22, 0, 0.22].forEach(dx => {
      ctx.beginPath(); ctx.moveTo(cx + dx * S, cy - S * 0.32); ctx.lineTo(cx + dx * S, cy + S * 0.32); ctx.stroke();
    });
    // Gun turrets
    ctx.fillStyle = '#2a3440';
    [[-0.28, -0.18], [0.28, -0.18], [-0.28, 0.18], [0.28, 0.18]].forEach(([dx, dy]) => {
      ctx.beginPath(); ctx.ellipse(cx + dx * S, cy + dy * S, S * 0.06, S * 0.055, 0, 0, Math.PI * 2); ctx.fill();
      ctx.fillStyle = '#1a2430'; ctx.fillRect(cx + dx * S - S * 0.01, cy + dy * S - S * 0.1, S * 0.02, S * 0.1);
      ctx.fillStyle = '#2a3440';
    });
    if (dir === 0) {
      ctx.fillStyle = '#1a2430';
      ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.28, S * 0.05, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
      ctx.strokeStyle = '#3a4450'; ctx.lineWidth = S * 0.02;
      ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.28, S * 0.07, S * 0.06, 0, 0, Math.PI * 2); ctx.stroke();
    }
    // Running lights
    ctx.fillStyle = hexToRgba(teamColor, 0.8);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 10;
    [[-0.36, 0], [0.36, 0], [0, -0.3], [0, 0.3]].forEach(([dx, dy]) => {
      ctx.beginPath(); ctx.ellipse(cx + dx * S, cy + dy * S, S * 0.04, S * 0.035, 0, 0, Math.PI * 2); ctx.fill();
    });
    ctx.shadowBlur = 0;
    // Bridge
    const bg = ctx.createRadialGradient(cx - S * 0.04, cy - S * 0.06, S * 0.02, cx, cy, S * 0.14);
    bg.addColorStop(0, '#8a9aaa'); bg.addColorStop(1, '#4a5a6a');
    ctx.fillStyle = bg; ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.1, S * 0.09, 0, 0, Math.PI * 2); ctx.fill();
  } else {
    // SIDE — elongated hull
    const hg = ctx.createLinearGradient(cx - S * 0.46, cy, cx + S * 0.46, cy);
    hg.addColorStop(0, '#1a2430'); hg.addColorStop(0.2, '#4a5460');
    hg.addColorStop(0.5, '#5a6470'); hg.addColorStop(0.8, '#4a5460'); hg.addColorStop(1, '#1a2430');
    ctx.fillStyle = hg;
    ctx.beginPath();
    ctx.moveTo(cx - S * 0.46, cy + S * 0.04);
    ctx.quadraticCurveTo(cx - S * 0.44, cy - S * 0.12, cx - S * 0.36, cy - S * 0.16);
    ctx.lineTo(cx + S * 0.2, cy - S * 0.16);
    ctx.quadraticCurveTo(cx + S * 0.44, cy - S * 0.12, cx + S * 0.46, cy + S * 0.02);
    ctx.quadraticCurveTo(cx + S * 0.44, cy + S * 0.14, cx + S * 0.36, cy + S * 0.16);
    ctx.lineTo(cx - S * 0.36, cy + S * 0.16);
    ctx.quadraticCurveTo(cx - S * 0.44, cy + S * 0.14, cx - S * 0.46, cy + S * 0.04);
    ctx.closePath(); ctx.fill();
    // Hull plating
    ctx.strokeStyle = '#2a3440'; ctx.lineWidth = S * 0.012;
    [-S * 0.16, 0, S * 0.16].forEach(dx => {
      ctx.beginPath(); ctx.moveTo(cx + dx, cy - S * 0.14); ctx.lineTo(cx + dx, cy + S * 0.14); ctx.stroke();
    });
    // Yamato cannon port at bow
    ctx.fillStyle = '#111';
    ctx.beginPath(); ctx.ellipse(cx - S * 0.44, cy - S * 0.04, S * 0.04, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
    // Turrets
    ctx.fillStyle = '#2a3440';
    [-0.24, 0, 0.2].forEach(dx => {
      ctx.beginPath(); ctx.ellipse(cx + dx * S, cy - S * 0.18, S * 0.05, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
    });
    // Bridge
    ctx.fillStyle = '#6a7a8a';
    ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.14, S * 0.1, S * 0.06, 0, 0, Math.PI * 2); ctx.fill();
    // Running lights
    ctx.fillStyle = hexToRgba(teamColor, 0.8);
    ctx.shadowColor = teamColor; ctx.shadowBlur = 10;
    [-0.34, 0, 0.34].forEach(dx => {
      ctx.beginPath(); ctx.ellipse(cx + dx * S, cy + S * 0.14, S * 0.03, S * 0.028, 0, 0, Math.PI * 2); ctx.fill();
    });
    ctx.shadowBlur = 0;
  }
}
```

- [ ] **Step 4: Update `FLYING_UNITS` to final state and register**

```javascript
const FLYING_UNITS = new Set(['MEDIVAC', 'MUTALISK', 'VIKING', 'RAVEN', 'BANSHEE', 'LIBERATOR', 'BATTLECRUISER']);
```

```javascript
  UNIT_MATS['BATTLECRUISER_F'] = makeDirTextures(drawBattlecruiser, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['BATTLECRUISER_E'] = makeDirTextures(drawBattlecruiser, TEAM_COLOR_ENEMY);
```

- [ ] **Step 5: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#battlecruiserDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+battlecruiserEnemySpawnsAndRendersInVisualizer+battlecruiserSpawnsHigherThanGroundUnit" -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Battlecruiser sprite — drawBattlecruiser, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 11: ShowcaseResource — Terran row

**Files:**
- Modify: `src/main/java/io/quarkmind/qa/ShowcaseResource.java`

Add a Terran ground column (x=16, y=9–12) and air row (y=8, x=10–14). All positions are within 9 tiles of Nexus at (8,8).

- [ ] **Step 1: Update `ShowcaseResource.java`**

Replace the `@POST seedShowcase()` body with:

```java
@POST
@Consumes(MediaType.WILDCARD)
public Response seedShowcase() {
    simulatedGame.reset();

    // Row 1 (y=11): Protoss ground
    simulatedGame.spawnEnemyUnit(UnitType.PROBE,    new Point2d(10, 11));
    simulatedGame.spawnEnemyUnit(UnitType.ZEALOT,   new Point2d(12, 11));
    simulatedGame.spawnEnemyUnit(UnitType.STALKER,  new Point2d(14, 11));

    // Row 2 (y=13): original Terran ground/air
    simulatedGame.spawnEnemyUnit(UnitType.MARINE,   new Point2d(10, 13));
    simulatedGame.spawnEnemyUnit(UnitType.MARAUDER, new Point2d(12, 13));
    simulatedGame.spawnEnemyUnit(UnitType.MEDIVAC,  new Point2d(14, 13));

    // Row 3 (y=14-15): Zerg
    simulatedGame.spawnEnemyUnit(UnitType.ZERGLING,  new Point2d(10, 15));
    simulatedGame.spawnEnemyUnit(UnitType.ROACH,     new Point2d(12, 15));
    simulatedGame.spawnEnemyUnit(UnitType.HYDRALISK, new Point2d(10, 14));
    simulatedGame.spawnEnemyUnit(UnitType.MUTALISK,  new Point2d(12, 14));

    // Column (x=16, y=9-13): new Terran ground
    simulatedGame.spawnEnemyUnit(UnitType.GHOST,      new Point2d(16, 9));
    simulatedGame.spawnEnemyUnit(UnitType.CYCLONE,    new Point2d(16, 10));
    simulatedGame.spawnEnemyUnit(UnitType.WIDOW_MINE, new Point2d(16, 11));
    simulatedGame.spawnEnemyUnit(UnitType.SIEGE_TANK, new Point2d(16, 12));
    simulatedGame.spawnEnemyUnit(UnitType.THOR,       new Point2d(15, 12));

    // Row (y=8, x=10-14): new Terran air
    simulatedGame.spawnEnemyUnit(UnitType.VIKING,       new Point2d(10, 8));
    simulatedGame.spawnEnemyUnit(UnitType.RAVEN,        new Point2d(11, 8));
    simulatedGame.spawnEnemyUnit(UnitType.BANSHEE,      new Point2d(12, 8));
    simulatedGame.spawnEnemyUnit(UnitType.LIBERATOR,    new Point2d(13, 8));
    simulatedGame.spawnEnemyUnit(UnitType.BATTLECRUISER, new Point2d(14, 8));

    engine.observe();

    return Response.ok(Map.of(
        "status",  "showcase seeded",
        "enemies", "20 units: Protoss(3) + Terran-orig(3) + Zerg(4) + Terran-new-ground(5) + Terran-new-air(5)"
    )).build();
}
```

- [ ] **Step 2: Compile to catch errors**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/io/quarkmind/qa/ShowcaseResource.java
git commit -m "feat: showcase Terran row — 10 → 20 seeded enemies Refs #N #83"
```

---

### Task 12: Update showcase Playwright test (10 → 20)

**Files:**
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Update `showcaseRendersAllUnitsAboveTerrainSurface`**

Find the two occurrences of `10` in the showcase test and update:

```java
// Change:
page.waitForFunction("() => window.__test.enemyCount() >= 10",
// To:
page.waitForFunction("() => window.__test.enemyCount() >= 20",

// Change:
assertThat(count).as("all 10 showcase enemy units must render").isEqualTo(10);
// To:
assertThat(count).as("all 20 showcase enemy units must render").isEqualTo(20);
```

- [ ] **Step 2: Run showcase test**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#showcaseRendersAllUnitsAboveTerrainSurface -q
```

Expected: PASS — 20 enemies render, all above terrain surface, all within map bounds.

- [ ] **Step 3: Commit**

```bash
git add src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "test: update showcase assertion 10 → 20 enemies Refs #N #83"
```

---

### Task 13: Full Playwright suite + close issue

- [ ] **Step 1: Run full Playwright suite**

```bash
mvn test -Pplaywright -q
```

Expected: all browser-tagged tests pass. If any smoke test returns -1, the draw function has a syntax error or name mismatch — check the function name in `smokeTestDrawFn` call matches the `function` declaration exactly.

- [ ] **Step 2: Close issue**

```bash
gh issue close N --comment "All 10 Terran sprites complete. FLYING_UNITS updated. Showcase extended to 20 enemies. 26 Playwright tests added/updated."
```

- [ ] **Step 3: Final build check**

```bash
mvn test -q
```

Expected: BUILD SUCCESS (non-Playwright tests).
