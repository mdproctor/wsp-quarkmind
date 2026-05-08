# E18a: Protoss Sprites Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add canvas 2D draw functions and UNIT_MATS registrations for all 11 remaining Protoss units, add 3 flying types to FLYING_UNITS, and add 25 new Playwright tests (smoke + spawn for all 11; elevation for 3 air units).

**Architecture:** Each unit gets one `function drawX(ctx, S, dir, teamColor)` added to `visualizer.js` before `initSpriteMaterials()`. The function is registered as `UNIT_MATS['X_F']` and `UNIT_MATS['X_E']` inside `initSpriteMaterials()`, and added to the `smokeTestDrawFn` manual lookup table (lines 98–130 of the same file — after the `drawBattlecruiser` line). Air units are added to the `FLYING_UNITS` set at the top of the file. Tests use TDD: smoke test verifies alpha > 0 for all 8 dir/team combos; spawn test verifies unit renders; elevation test (air only) verifies Y > marine Y + 0.3.

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
  --title "E18a: Protoss sprites (11 units)" \
  --body "Complete 2D sprite coverage for all remaining Protoss units.

## Units
Ground: Sentry, Adept, Dark Templar, High Templar, Disruptor, Immortal, Archon, Colossus
Air: Observer, Void Ray, Carrier

## Acceptance criteria
- draw function for each unit (4 dirs × 2 team colours)
- UNIT_MATS registration for each (_F and _E)
- smokeTestDrawFn lookup entry for each
- FLYING_UNITS: Observer, Void Ray, Carrier
- Playwright: smoke + spawn for all 11; elevation for 3 air units (25 tests total)

Refs #83" \
  --label "enhancement"
```

Note the issue number printed — every commit in this plan uses `Refs #N` with it.

---

### Task 1: Sentry (hovering ground unit)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

Add at the end of `VisualizerRenderTest.java` (before the closing `}`):

```java
@Test
@Tag("browser")
void sentryDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawSentry', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawSentry dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void sentryEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.SENTRY, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Sentry enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#sentryDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL — `smokeTestDrawFn` returns -1 (function not registered).

- [ ] **Step 3: Implement `drawSentry` in `visualizer.js`**

Add `function drawSentry(ctx, S, dir, teamColor)` before the `// Populated by initSpriteMaterials()` comment. Use the dir-3 mirror pattern from the plan header.

**Visual design — implement using Canvas 2D API:**
- Hovering unit, centre at ~(S/2, S/2+8). Mostly similar across dirs.
- Body: small rounded filled ellipse, dark blue-grey (~`#1a1a3a`), horizontal radius ~S×0.18, vertical ~S×0.15.
- Equatorial ring: thin stroke ellipse (lineWidth ~S×0.025) in team colour with shadowBlur ~8, radius ~S×0.22. For dir 1 (side), squash ring to perspective ellipse (horizontal ~S×0.22, vertical ~S×0.08).
- Shield projector hub: small darker filled ellipse above the body top.
- Three emitter dots equidistant around the ring (at 0°, 120°, 240°): tiny filled circles in team colour, shadowBlur ~10.
- Dir 2 (back): hub hidden behind body; emitters shifted slightly.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

In `visualizer.js`, after the `drawBattlecruiser` line in the `smokeTestDrawFn` function body (around line 122), add:

```javascript
    if (typeof drawSentry !== 'undefined') lookup.drawSentry = drawSentry;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

Inside `initSpriteMaterials()`, add:

```javascript
  UNIT_MATS['SENTRY_F'] = makeDirTextures(drawSentry, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['SENTRY_E'] = makeDirTextures(drawSentry, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#sentryDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+sentryEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Sentry sprite — drawSentry, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 2: Adept (streamlined warrior)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

Add at the end of `VisualizerRenderTest.java` (before the closing `}`):

```java
@Test
@Tag("browser")
void adeptDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawAdept', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawAdept dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void adeptEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.ADEPT, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Adept enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#adeptDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawAdept` in `visualizer.js`**

Add `function drawAdept(ctx, S, dir, teamColor)` before the `// Populated by initSpriteMaterials()` comment.

**Visual design — implement using Canvas 2D API:**
- Upright humanoid warrior, Protoss gold palette (~`#3a3020` base, `#5a5030` mid).
- Body: upper torso ellipse centred ~(S/2, S/2), lower leg section slightly wider ellipse below it.
- Psi-lance blade: extends from the right side. Dir 1 (side): long diagonal rect path angled upward-right, blade extends ~S×0.3 from shoulder. Dir 0/2 (front/back): blade shown tip-on as a small pointed diamond or circle at right side.
- Blade fill: team colour with shadowBlur ~10.
- Eye strip: thin horizontal filled rect across the helm/visor, team colour, shadowBlur ~6.
- Dir 2 (back): no blade visible; carapace ridge drawn as a vertical line down the back.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawAdept !== 'undefined') lookup.drawAdept = drawAdept;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['ADEPT_F'] = makeDirTextures(drawAdept, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['ADEPT_E'] = makeDirTextures(drawAdept, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#adeptDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+adeptEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Adept sprite — drawAdept, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 3: Dark Templar (cloaked warrior)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void darkTemplarDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawDarkTemplar', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawDarkTemplar dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void darkTemplarEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.DARK_TEMPLAR, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Dark Templar enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#darkTemplarDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawDarkTemplar` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Similar silhouette to Adept but narrower, very dark body (~`#0a0a1a`, near-black).
- Body: two ellipses (torso + lower), darker and more sinister than Adept. Use a dark fill (not fully opaque if possible to suggest partial cloaking — `hexToRgba('#0a0a1a', 0.85)`).
- Warp blade: curved path on one side (a crescent or curved diagonal); blade body is dark, blade edge is pure team colour with shadowBlur ~14. Dir 1 shows full blade curve; dir 0/2 shows a shorter angled slash.
- Eyes: two small bright filled circles in team colour with shadowBlur ~12, high up on head.
- Subtle body outline stroke in dark purple (~`#1a0a2a`), lineWidth ~S×0.015.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawDarkTemplar !== 'undefined') lookup.drawDarkTemplar = drawDarkTemplar;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['DARK_TEMPLAR_F'] = makeDirTextures(drawDarkTemplar, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['DARK_TEMPLAR_E'] = makeDirTextures(drawDarkTemplar, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#darkTemplarDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+darkTemplarEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Dark Templar sprite — drawDarkTemplar, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 4: High Templar (robed psion)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void highTemplarDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawHighTemplar', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawHighTemplar dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void highTemplarEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.HIGH_TEMPLAR, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one High Templar enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#highTemplarDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawHighTemplar` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Robed figure, Protoss gold-blue palette (~`#2a2a40` robe, `#5a5030` trim).
- Robe: wide triangular/trapezoidal filled path — narrow at the top (shoulders ~S×0.2 wide), wider at the hem (bottom ~S×0.38 wide), giving a flowing robe silhouette.
- Upper body: small rounded ellipse for the torso/head above the robe.
- Staff: thin vertical rect (width ~S×0.02) running through the body, extending above the head and below the hem.
- Psionic energy: at each hand position (lower body sides), draw 3–4 small arc strokes (ctx.arc, not full ellipse) radiating outward in team colour with shadowBlur ~16. Suggests energy crackling around the hands.
- Dir 1 (side): robe visible as a thinner trapezoid profile; staff extends up-left; energy visible at the front arm position.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawHighTemplar !== 'undefined') lookup.drawHighTemplar = drawHighTemplar;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['HIGH_TEMPLAR_F'] = makeDirTextures(drawHighTemplar, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['HIGH_TEMPLAR_E'] = makeDirTextures(drawHighTemplar, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#highTemplarDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+highTemplarEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: High Templar sprite — drawHighTemplar, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 5: Disruptor (floating energy sphere)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void disruptorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawDisruptor', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawDisruptor dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void disruptorEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.DISRUPTOR, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Disruptor enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#disruptorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawDisruptor` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Floating energy sphere: identical from all 4 directions (direction is irrelevant, but still handle dir-3 mirror for consistency).
- Main sphere: large filled circle centred at (S/2, S/2+4), radius ~S×0.28, using `createRadialGradient` from centre — team colour at 0% opacity 0 (pure glow) to near-black at outer edge. Or: dark base sphere first, then energy glow on top.
- Energy arcs across surface: 4–5 stroke arcs (ctx.beginPath, ctx.arc with start/end angle) distributed across the sphere, in team colour, lineWidth ~S×0.022, shadowBlur ~12.
- Central core: small bright filled circle (radius ~S×0.07) in team colour, shadowBlur ~20.
- Subtle floating shadow: dark semi-transparent ellipse below the sphere centre, radius ~S×0.2 horizontal, ~S×0.06 vertical.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawDisruptor !== 'undefined') lookup.drawDisruptor = drawDisruptor;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['DISRUPTOR_F'] = makeDirTextures(drawDisruptor, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['DISRUPTOR_E'] = makeDirTextures(drawDisruptor, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#disruptorDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+disruptorEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Disruptor sprite — drawDisruptor, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 6: Immortal (heavy walker)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void immortalDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawImmortal', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawImmortal dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void immortalEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.IMMORTAL, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Immortal enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#immortalDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawImmortal` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Heavy walker, dark gunmetal palette (~`#2a3040` base), wider than tall.
- Two chunky legs: thick rectangular fills below the body, angled slightly outward. Dir 0/2: legs spread wide; dir 1: two legs visible in profile.
- Hardened shield plates: two large overlapping rounded rect or ellipse fills at the shoulders/sides, slightly lighter grey, creating a broad armoured silhouette.
- Assault cannon arm: dir 1 shows the full cannon — a long horizontal rect extending to the right (~S×0.35 long, ~S×0.08 tall) with a flared muzzle end. Dir 0 (front): cannon seen tip-on as a small circle/ellipse at right side.
- Energy cell at cannon muzzle/tip: bright filled ellipse in team colour, shadowBlur ~10.
- Body torso: large rounded rect between shoulders, raised above legs.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawImmortal !== 'undefined') lookup.drawImmortal = drawImmortal;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['IMMORTAL_F'] = makeDirTextures(drawImmortal, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['IMMORTAL_E'] = makeDirTextures(drawImmortal, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#immortalDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+immortalEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Immortal sprite — drawImmortal, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 7: Archon (pure energy being)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void archonDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawArchon', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawArchon dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void archonEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.ARCHON, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Archon enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#archonDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawArchon` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- No solid body — pure energy. Team colour is the dominant colour throughout.
- Two overlapping thick ring strokes (lineWidth ~S×0.055), both centred near (S/2, S/2): outer ring radius ~S×0.26, inner ring radius ~S×0.20, both in team colour with shadowBlur ~18. Offset the two ring centres slightly (~S×0.04 apart) to overlap rather than concentric.
- Arc discharges: 5–6 short arc paths (ctx.arc, partial arcs) radiating just beyond the outer ring edge, in team colour, lineWidth ~S×0.02.
- Central glowing core: bright filled circle radius ~S×0.08 at canvas centre, team colour, shadowBlur ~22.
- All 4 directions nearly identical — dir 1 may offset the two ring centres slightly more horizontally to suggest a side angle.
- No dark background fill needed — the energy IS the sprite.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawArchon !== 'undefined') lookup.drawArchon = drawArchon;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['ARCHON_F'] = makeDirTextures(drawArchon, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['ARCHON_E'] = makeDirTextures(drawArchon, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#archonDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+archonEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Archon sprite — drawArchon, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 8: Colossus (massive 4-legged walker)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void colossusDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawColossus', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawColossus dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void colossusEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.COLOSSUS, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Colossus enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#colossusDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawColossus` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Widest ground sprite — 4 long legs span almost the full canvas width. Protoss bronze-gold palette (~`#4a3820` hull, `#2a2010` legs).
- Dir 0/2 (front/back — widest view):
  - 4 legs as thin angled rect fills, spread from ~S×0.08 to ~S×0.92 horizontally, extending from a central hip point down to the canvas bottom.
  - Two tall stilt supports rising from the hip centre up to the body (~S×0.4 height).
  - Body hull: large rounded rect at the top, ~S×0.5 wide, ~S×0.2 tall.
  - Twin thermal lance arrays: two horizontal rect strips on the top face of the hull, symmetrically placed.
  - Lance tips: bright team colour dots at the outer ends of each lance, shadowBlur ~10.
- Dir 1 (side — tallest view):
  - 2 visible legs in profile, very long, extending the full canvas height.
  - Body hull elevated at top — the extreme height is the key visual.
  - One thermal lance visible in profile extending to the right.

- [ ] **Step 4: Add to `smokeTestDrawFn` lookup table**

```javascript
    if (typeof drawColossus !== 'undefined') lookup.drawColossus = drawColossus;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['COLOSSUS_F'] = makeDirTextures(drawColossus, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['COLOSSUS_E'] = makeDirTextures(drawColossus, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#colossusDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+colossusEnemySpawnsAndRendersInVisualizer" -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Colossus sprite — drawColossus, UNIT_MATS, smoke + spawn tests Refs #N #83"
```

---

### Task 9: Observer (air — cloaking disc)

Air unit — also adds FLYING_UNITS and requires an elevation test.

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void observerDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawObserver', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawObserver dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void observerEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.OBSERVER, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Observer enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void observerSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.OBSERVER, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double observerY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(observerY)
        .as("Observer Y (%.3f) must be higher than Marine Y (%.3f)".formatted(observerY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#observerDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawObserver` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Small cloaking disc, roughly 2/3 the size of other air units.
- Body: flattened filled ellipse centred at (S/2, S/2), horizontal radius ~S×0.22, vertical ~S×0.12, very dark (~`#0a0a1a`).
- Sensor arrays: 4 short rectangular protrusions at equal spacing around the disc edge (top, bottom, left, right), dark grey (~`#2a2a3a`), length ~S×0.08, width ~S×0.025.
- Scan emitters: two small bright filled circles at the left and right sides, team colour, radius ~S×0.04, shadowBlur ~10.
- Outline stroke: dark grey, lineWidth ~S×0.02.
- Dir 1 (side): disc seen edge-on — draw as a very thin horizontal ellipse (horizontal ~S×0.22, vertical ~S×0.04) with two emitter dots at left and right tips.

- [ ] **Step 4: Update `FLYING_UNITS` and add to `smokeTestDrawFn` lookup table**

In `visualizer.js`, update the `FLYING_UNITS` set (near the top of the file) to add `'OBSERVER'`:

```javascript
const FLYING_UNITS = new Set([
  'MEDIVAC', 'MUTALISK',
  'VIKING', 'RAVEN', 'BANSHEE', 'LIBERATOR', 'BATTLECRUISER',
  'OBSERVER'
]);
```

Then in the `smokeTestDrawFn` function body, add:

```javascript
    if (typeof drawObserver !== 'undefined') lookup.drawObserver = drawObserver;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['OBSERVER_F'] = makeDirTextures(drawObserver, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['OBSERVER_E'] = makeDirTextures(drawObserver, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#observerDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+observerEnemySpawnsAndRendersInVisualizer+observerSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all three tests).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Observer sprite — drawObserver, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 10: Void Ray (air — angular warship)

Air unit — FLYING_UNITS update + elevation test.

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void voidRayDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawVoidRay', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawVoidRay dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void voidRayEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.VOID_RAY, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Void Ray enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void voidRaySpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.VOID_RAY, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double voidRayY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(voidRayY)
        .as("Void Ray Y (%.3f) must be higher than Marine Y (%.3f)".formatted(voidRayY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#voidRayDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawVoidRay` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Large angular warship, dark blue-grey palette (~`#1a2840`).
- Dir 0 (front — bow pointing toward viewer):
  - Diamond/arrowhead shape — pointed front at top-centre, swept-back wings extending left and right, narrowing at the stern.
  - Prismatic beam emitter at the bow tip: bright team colour filled circle, shadowBlur ~16.
  - Hull panelling: stroke lines along the wing surfaces, slightly lighter grey.
- Dir 2 (back — engine view): similar silhouette but reversed; engine glow ports at the rear as team colour dots.
- Dir 1 (side profile):
  - Elongated blade-like profile, pointed at front-left, engine glow cluster at rear-right.
  - Nacelle visible below the wing — a slight hull swell in the lower profile.
  - Team colour at both bow tip and engine glow.

- [ ] **Step 4: Update `FLYING_UNITS` and add to `smokeTestDrawFn` lookup table**

Update `FLYING_UNITS` to add `'VOID_RAY'`:

```javascript
const FLYING_UNITS = new Set([
  'MEDIVAC', 'MUTALISK',
  'VIKING', 'RAVEN', 'BANSHEE', 'LIBERATOR', 'BATTLECRUISER',
  'OBSERVER', 'VOID_RAY'
]);
```

Then add to `smokeTestDrawFn`:

```javascript
    if (typeof drawVoidRay !== 'undefined') lookup.drawVoidRay = drawVoidRay;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['VOID_RAY_F'] = makeDirTextures(drawVoidRay, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['VOID_RAY_E'] = makeDirTextures(drawVoidRay, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#voidRayDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+voidRayEnemySpawnsAndRendersInVisualizer+voidRaySpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all three tests).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Void Ray sprite — drawVoidRay, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 11: Carrier (air — Protoss capital ship)

Air unit — FLYING_UNITS update to final E18a state + elevation test.

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void carrierDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawCarrier', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawCarrier dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void carrierEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.CARRIER, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Carrier enemy must render").isEqualTo(1);
    page.close();
}

@Test
@Tag("browser")
void carrierSpawnsHigherThanGroundUnit() throws Exception {
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

    simulatedGame.spawnEnemyUnit(UnitType.CARRIER, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    double carrierY = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList().get(0);
    page2.close();

    assertThat(carrierY)
        .as("Carrier Y (%.3f) must be higher than Marine Y (%.3f)".formatted(carrierY, marineY))
        .isGreaterThan(marineY + 0.3);
}
```

- [ ] **Step 2: Run to verify failure**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#carrierDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Step 3: Implement `drawCarrier` in `visualizer.js`**

**Visual design — implement using Canvas 2D API:**
- Largest Protoss capital ship — wide oval hull approaching canvas edges. Dark navy palette (~`#1a1a30`).
- Dir 0/2 (top-down):
  - Wide filled ellipse hull (horizontal ~S×0.44, vertical ~S×0.3).
  - Central hangar bay: darker rectangular fill in the belly (~S×0.28 wide, ~S×0.14 tall), centred.
  - Interceptor silhouettes inside bay: 3–4 tiny diamond shapes (~S×0.04 each) arranged in the bay area, dark grey.
  - Running lights: 4 small team colour filled circles at hull corners (top-left, top-right, bottom-left, bottom-right), shadowBlur ~8.
  - Two rear engine ports: slightly larger team colour ellipses at the stern, shadowBlur ~10.
- Dir 1 (side):
  - Broad, flat profile — wide horizontal ellipse, height ~S×0.16.
  - Hull detail: a ridge line along the middle.
  - Rear engine glow: 2–3 team colour dots at the right end, shadowBlur ~10.

- [ ] **Step 4: Update `FLYING_UNITS` to E18a final state and add to `smokeTestDrawFn` lookup table**

Update `FLYING_UNITS` to the complete E18a state:

```javascript
const FLYING_UNITS = new Set([
  'MEDIVAC', 'MUTALISK',
  'VIKING', 'RAVEN', 'BANSHEE', 'LIBERATOR', 'BATTLECRUISER',
  'OBSERVER', 'VOID_RAY', 'CARRIER'
]);
```

Then add to `smokeTestDrawFn`:

```javascript
    if (typeof drawCarrier !== 'undefined') lookup.drawCarrier = drawCarrier;
```

- [ ] **Step 5: Register in `initSpriteMaterials()`**

```javascript
  UNIT_MATS['CARRIER_F'] = makeDirTextures(drawCarrier, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['CARRIER_E'] = makeDirTextures(drawCarrier, TEAM_COLOR_ENEMY);
```

- [ ] **Step 6: Run to verify pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#carrierDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+carrierEnemySpawnsAndRendersInVisualizer+carrierSpawnsHigherThanGroundUnit" -q
```

Expected: PASS (all three tests).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: Carrier sprite — drawCarrier, UNIT_MATS, FLYING_UNITS, smoke + spawn + elevation tests Refs #N #83"
```

---

### Task 12: Full Playwright suite + close issue

- [ ] **Step 1: Run full Playwright suite**

```bash
mvn test -Pplaywright -q
```

Expected: all browser-tagged tests pass. If any smoke test returns -1, the draw function is either missing from the `smokeTestDrawFn` lookup table or the function name in the test doesn't match the `function` declaration exactly.

- [ ] **Step 2: Run full non-Playwright test suite**

```bash
mvn test -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Close issue**

```bash
gh issue close N --comment "All 11 Protoss sprites complete. FLYING_UNITS updated with Observer, Void Ray, Carrier. 40 Playwright tests added."
```

(Replace `N` with the issue number from Task 0.)
