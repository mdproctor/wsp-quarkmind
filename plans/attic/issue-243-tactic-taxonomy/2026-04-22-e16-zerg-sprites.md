# E16 Zerg Sprites — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add cartoon draw functions for Zergling, Roach, Hydralisk, and Mutalisk; register them in UNIT_MATS; update ShowcaseResource to seed all three races.

**Architecture:** Four new `function drawXxx(ctx, S, dir, teamColor)` declarations added after `drawMedivac` in `visualizer.js`. Each is registered in `initSpriteMaterials()` with `_F`/`_E` keys. Mutalisk joins `FLYING_UNITS`. All four added to `smokeTestDrawFn` lookup. Palette: carapace `#2a0a3a`, flesh `#5c1a6e`, plate `#8b3a9e`, eyes `#ffe066`, teamColor bio-sac glow via radial gradient + shadowBlur.

**Tech Stack:** Canvas 2D API, Three.js r128, Playwright + `@QuarkusTest`.

**Spec:** `docs/superpowers/specs/2026-04-22-e16-zerg-sprites-design.md`
**Epic:** #83

---

## File Map

| File | Change |
|------|--------|
| `src/main/resources/META-INF/resources/visualizer.js` | 4 draw fns, FLYING_UNITS, initSpriteMaterials, smokeTestDrawFn |
| `src/main/java/io/quarkmind/qa/ShowcaseResource.java` | Add Zerg units to seeded state |
| `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java` | Smoke + spawn tests per unit; update unitMats coverage test |

No Java domain changes — `UnitType` already has all four values.

---

## Task 1: Create GitHub Issue

- [ ] **Create issue under epic #83**

```bash
gh issue create \
  --title "E16: Zerg sprites (Zergling, Roach, Hydralisk, Mutalisk)" \
  --body "## Summary
Add cartoon draw functions for Zergling, Roach, Hydralisk, and Mutalisk.
Purple organic palette, yellow eyes, teamColor bio-sac glow.
Mutalisk joins FLYING_UNITS. ShowcaseResource updated to show all three races.
Refs #83

## Acceptance criteria
- drawZergling, drawRoach, drawHydralisk, drawMutalisk implemented — all 4 dirs × 2 team colours
- UNIT_MATS entries: ZERGLING_F/E, ROACH_F/E, HYDRALISK_F/E, MUTALISK_F/E (total 22 keys)
- FLYING_UNITS includes MUTALISK
- ShowcaseResource seeds Zerg row alongside Protoss/Terran rows
- All Playwright smoke + spawn tests green" \
  --label enhancement
```

- [ ] **Note the issue number** — every commit references it as `Refs #N`. Replace `#N` throughout this plan with the real number.

---

## Task 2: Zergling

**Per-unit TDD order:** extend smokeTestDrawFn lookup → write failing test → implement → register → run → commit.

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Extend `smokeTestDrawFn` lookup** — in `visualizer.js` at line 108, after the `drawMedivac` entry, add:

```js
    if (typeof drawZergling  !== 'undefined') lookup.drawZergling  = drawZergling;
```

- [ ] **Write failing smoke + spawn tests** — add to `VisualizerRenderTest.java`:

```java
@Test
@Tag("browser")
void zerglingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawZergling', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue())
            .as("drawZergling dir=" + dir + " team=" + color)
            .isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void zerglingEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.ZERGLING, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Zergling enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Run both tests to confirm they fail**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#zerglingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+zerglingEnemySpawnsAndRendersInVisualizer" -q
```

Expected: smoke test fails with `-1`; spawn test fails (ZERGLING_E not in UNIT_MATS → falls back to UNKNOWN_E, but spawn test still passes since UNKNOWN_E exists). Actually the spawn test may pass already — focus is on the smoke test returning -1.

- [ ] **Add `drawZergling` function** to `visualizer.js` after `drawMedivac` (after line 1011):

```js
function drawZergling(ctx, S, dir, teamColor) {
  // dir=3 is mirror of dir=1
  if (dir === 3) {
    ctx.save(); ctx.translate(S, 0); ctx.scale(-1, 1);
    drawZergling(ctx, S, 1, teamColor);
    ctx.restore(); return;
  }
  const cx = S / 2, cy = S * 0.52;

  if (dir === 0 || dir === 2) {
    // Legs drawn first (behind body)
    ctx.strokeStyle = '#5c1a6e'; ctx.lineWidth = 3;
    [[-0.28, 0.14], [-0.14, 0.2], [0.14, 0.2], [0.28, 0.14]].forEach(([dx, dy]) => {
      ctx.beginPath();
      ctx.moveTo(cx + dx * S * 0.6, cy + S * 0.08);
      ctx.lineTo(cx + dx * S, cy + dy * S + S * 0.14);
      ctx.stroke();
    });
    // Carapace body
    ctx.fillStyle = '#2a0a3a';
    ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.22, S * 0.2, 0, 0, Math.PI * 2); ctx.fill();
    // Dorsal plates
    ctx.fillStyle = '#8b3a9e';
    [[-0.08, -0.08], [0, -0.12], [0.08, -0.08]].forEach(([dx, dy]) => {
      ctx.beginPath(); ctx.ellipse(cx + dx * S, cy + dy * S, S * 0.055, S * 0.04, 0, 0, Math.PI * 2); ctx.fill();
    });
    if (dir === 0) {
      // Flesh belly
      ctx.fillStyle = '#5c1a6e';
      ctx.beginPath(); ctx.ellipse(cx, cy + S * 0.06, S * 0.13, S * 0.1, 0, 0, Math.PI * 2); ctx.fill();
      // Scythe blades
      ctx.strokeStyle = '#8b3a9e'; ctx.lineWidth = 3;
      ctx.beginPath();
      ctx.moveTo(cx - S * 0.14, cy - S * 0.1);
      ctx.quadraticCurveTo(cx - S * 0.34, cy - S * 0.22, cx - S * 0.28, cy - S * 0.38);
      ctx.stroke();
      ctx.beginPath();
      ctx.moveTo(cx + S * 0.14, cy - S * 0.1);
      ctx.quadraticCurveTo(cx + S * 0.34, cy - S * 0.22, cx + S * 0.28, cy - S * 0.38);
      ctx.stroke();
      // Head
      ctx.fillStyle = '#2a0a3a';
      ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.22, S * 0.12, S * 0.11, 0, 0, Math.PI * 2); ctx.fill();
      // Eyes
      ctx.fillStyle = '#ffe066';
      ctx.beginPath(); ctx.arc(cx - S * 0.05, cy - S * 0.24, S * 0.03, 0, Math.PI * 2); ctx.fill();
      ctx.beginPath(); ctx.arc(cx + S * 0.05, cy - S * 0.24, S * 0.03, 0, Math.PI * 2); ctx.fill();
      // Belly bio-sac
      const g = ctx.createRadialGradient(cx, cy + S * 0.09, 2, cx, cy + S * 0.09, S * 0.09);
      g.addColorStop(0, teamColor + 'ff'); g.addColorStop(0.55, teamColor + '88'); g.addColorStop(1, teamColor + '00');
      ctx.fillStyle = g; ctx.shadowColor = teamColor; ctx.shadowBlur = 8;
      ctx.beginPath(); ctx.ellipse(cx, cy + S * 0.09, S * 0.08, S * 0.07, 0, 0, Math.PI * 2); ctx.fill();
      // Shoulder sacs
      [cx - S * 0.18, cx + S * 0.18].forEach(x => {
        const g2 = ctx.createRadialGradient(x, cy, 1, x, cy, S * 0.05);
        g2.addColorStop(0, teamColor + 'cc'); g2.addColorStop(1, teamColor + '00');
        ctx.fillStyle = g2; ctx.beginPath(); ctx.arc(x, cy, S * 0.045, 0, Math.PI * 2); ctx.fill();
      });
      ctx.shadowBlur = 0;
    } else {
      // Back: carapace highlight, back of head
      ctx.fillStyle = '#8b3a9e';
      ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.05, S * 0.15, S * 0.1, 0, 0, Math.PI * 2); ctx.fill();
      ctx.fillStyle = '#2a0a3a';
      ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.22, S * 0.11, S * 0.1, 0, 0, Math.PI * 2); ctx.fill();
      ctx.fillStyle = '#8b3a9e';
      ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.24, S * 0.07, S * 0.05, 0, 0, Math.PI * 2); ctx.fill();
    }
  } else {
    // dir=1: right side profile
    ctx.fillStyle = '#2a0a3a';
    ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.28, S * 0.17, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#8b3a9e';
    ctx.beginPath(); ctx.ellipse(cx - S * 0.04, cy - S * 0.1, S * 0.2, S * 0.07, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#5c1a6e';
    ctx.beginPath(); ctx.ellipse(cx + S * 0.04, cy + S * 0.05, S * 0.16, S * 0.07, 0, 0, Math.PI * 2); ctx.fill();
    // Legs
    ctx.strokeStyle = '#5c1a6e'; ctx.lineWidth = 3;
    [[cx - S * 0.1, cy + S * 0.1, cx - S * 0.2, cy + S * 0.28],
     [cx + S * 0.08, cy + S * 0.1, cx + S * 0.16, cy + S * 0.28]].forEach(([x1, y1, x2, y2]) => {
      ctx.beginPath(); ctx.moveTo(x1, y1); ctx.lineTo(x2, y2); ctx.stroke();
    });
    // Head
    ctx.fillStyle = '#2a0a3a';
    ctx.beginPath(); ctx.ellipse(cx - S * 0.2, cy - S * 0.04, S * 0.1, S * 0.09, 0, 0, Math.PI * 2); ctx.fill();
    // Blade
    ctx.strokeStyle = '#8b3a9e'; ctx.lineWidth = 3;
    ctx.beginPath();
    ctx.moveTo(cx - S * 0.22, cy - S * 0.09);
    ctx.quadraticCurveTo(cx - S * 0.4, cy - S * 0.22, cx - S * 0.32, cy - S * 0.36);
    ctx.stroke();
    // Eye
    ctx.fillStyle = '#ffe066';
    ctx.beginPath(); ctx.arc(cx - S * 0.24, cy - S * 0.07, S * 0.028, 0, Math.PI * 2); ctx.fill();
    // Belly bio-sac
    const g = ctx.createRadialGradient(cx + S * 0.06, cy + S * 0.05, 1, cx + S * 0.06, cy + S * 0.05, S * 0.07);
    g.addColorStop(0, teamColor + 'ff'); g.addColorStop(0.5, teamColor + '77'); g.addColorStop(1, teamColor + '00');
    ctx.fillStyle = g; ctx.shadowColor = teamColor; ctx.shadowBlur = 7;
    ctx.beginPath(); ctx.ellipse(cx + S * 0.06, cy + S * 0.05, S * 0.06, S * 0.055, 0, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
  }
}
```

- [ ] **Register ZERGLING in `initSpriteMaterials`** — add after the MEDIVAC lines (line 1028):

```js
  UNIT_MATS['ZERGLING_F']  = makeDirTextures(drawZergling,  TEAM_COLOR_FRIENDLY);
  UNIT_MATS['ZERGLING_E']  = makeDirTextures(drawZergling,  TEAM_COLOR_ENEMY);
```

- [ ] **Run both tests to confirm they pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#zerglingDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+zerglingEnemySpawnsAndRendersInVisualizer" -q
```

Expected: both PASS.

- [ ] **Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat(e16): drawZergling — quadrupedal with scythe blades and belly bio-sac Refs #N Refs #83"
```

---

## Task 3: Roach

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Extend `smokeTestDrawFn` lookup** — add after the `drawZergling` entry:

```js
    if (typeof drawRoach     !== 'undefined') lookup.drawRoach     = drawRoach;
```

- [ ] **Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void roachDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawRoach', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawRoach dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void roachEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.ROACH, new Point2d(22, 22));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Roach enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Run to confirm smoke test fails with -1**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#roachDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL — returns -1.

- [ ] **Add `drawRoach` function** after `drawZergling`:

```js
function drawRoach(ctx, S, dir, teamColor) {
  if (dir === 3) {
    ctx.save(); ctx.translate(S, 0); ctx.scale(-1, 1);
    drawRoach(ctx, S, 1, teamColor);
    ctx.restore(); return;
  }
  const cx = S / 2, cy = S * 0.55;

  if (dir === 0 || dir === 2) {
    // Legs behind body
    ctx.strokeStyle = '#5c1a6e'; ctx.lineWidth = 3.5;
    [[-0.3, 0.12], [-0.18, 0.18], [-0.06, 0.2], [0.06, 0.2], [0.18, 0.18], [0.3, 0.12]].forEach(([dx, dy]) => {
      ctx.beginPath();
      ctx.moveTo(cx + dx * S * 0.7, cy + S * 0.04);
      ctx.lineTo(cx + dx * S, cy + dy * S + S * 0.08);
      ctx.stroke();
    });
    // Wide low body
    ctx.fillStyle = '#2a0a3a';
    ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.34, S * 0.22, 0, 0, Math.PI * 2); ctx.fill();
    // Armour plates
    ctx.fillStyle = '#8b3a9e';
    ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.06, S * 0.28, S * 0.14, 0, 0, Math.PI * 2); ctx.fill();
    if (dir === 0) {
      // Head — heavy jaw
      ctx.fillStyle = '#2a0a3a';
      ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.26, S * 0.18, S * 0.14, 0, 0, Math.PI * 2); ctx.fill();
      ctx.fillStyle = '#5c1a6e';
      ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.16, S * 0.14, S * 0.07, 0, 0, Math.PI * 2); ctx.fill();
      // Eyes
      ctx.fillStyle = '#ffe066';
      ctx.beginPath(); ctx.arc(cx - S * 0.1, cy - S * 0.3, S * 0.035, 0, Math.PI * 2); ctx.fill();
      ctx.beginPath(); ctx.arc(cx + S * 0.1, cy - S * 0.3, S * 0.035, 0, Math.PI * 2); ctx.fill();
    } else {
      // Back of head
      ctx.fillStyle = '#2a0a3a';
      ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.24, S * 0.16, S * 0.12, 0, 0, Math.PI * 2); ctx.fill();
    }
    // Acid gland sacs (3 on carapace ridge)
    [[cx - S * 0.12, cy - S * 0.06], [cx, cy - S * 0.1], [cx + S * 0.12, cy - S * 0.06]].forEach(([x, y]) => {
      const g = ctx.createRadialGradient(x, y, 1, x, y, S * 0.065);
      g.addColorStop(0, teamColor + 'ff'); g.addColorStop(0.5, teamColor + '88'); g.addColorStop(1, teamColor + '00');
      ctx.fillStyle = g; ctx.shadowColor = teamColor; ctx.shadowBlur = 8;
      ctx.beginPath(); ctx.ellipse(x, y, S * 0.058, S * 0.05, 0, 0, Math.PI * 2); ctx.fill();
    });
    ctx.shadowBlur = 0;
  } else {
    // dir=1: right side profile — hunched low
    ctx.strokeStyle = '#5c1a6e'; ctx.lineWidth = 3.5;
    [[cx - S * 0.15, cy + S * 0.06, cx - S * 0.3, cy + S * 0.22],
     [cx,            cy + S * 0.1,  cx,            cy + S * 0.28],
     [cx + S * 0.15, cy + S * 0.06, cx + S * 0.28, cy + S * 0.22]].forEach(([x1, y1, x2, y2]) => {
      ctx.beginPath(); ctx.moveTo(x1, y1); ctx.lineTo(x2, y2); ctx.stroke();
    });
    ctx.fillStyle = '#2a0a3a';
    ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.32, S * 0.2, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#8b3a9e';
    ctx.beginPath(); ctx.ellipse(cx - S * 0.04, cy - S * 0.08, S * 0.26, S * 0.12, 0, 0, Math.PI * 2); ctx.fill();
    // Head
    ctx.fillStyle = '#2a0a3a';
    ctx.beginPath(); ctx.ellipse(cx - S * 0.22, cy - S * 0.1, S * 0.12, S * 0.1, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#5c1a6e';
    ctx.beginPath(); ctx.ellipse(cx - S * 0.22, cy - S * 0.04, S * 0.1, S * 0.055, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#ffe066';
    ctx.beginPath(); ctx.arc(cx - S * 0.28, cy - S * 0.14, S * 0.03, 0, Math.PI * 2); ctx.fill();
    // Two visible sacs
    [[cx - S * 0.06, cy - S * 0.09], [cx + S * 0.1, cy - S * 0.09]].forEach(([x, y]) => {
      const g = ctx.createRadialGradient(x, y, 1, x, y, S * 0.062);
      g.addColorStop(0, teamColor + 'ff'); g.addColorStop(0.5, teamColor + '77'); g.addColorStop(1, teamColor + '00');
      ctx.fillStyle = g; ctx.shadowColor = teamColor; ctx.shadowBlur = 7;
      ctx.beginPath(); ctx.ellipse(x, y, S * 0.055, S * 0.048, 0, 0, Math.PI * 2); ctx.fill();
    });
    ctx.shadowBlur = 0;
  }
}
```

- [ ] **Register ROACH in `initSpriteMaterials`** — add after the ZERGLING lines:

```js
  UNIT_MATS['ROACH_F']     = makeDirTextures(drawRoach,     TEAM_COLOR_FRIENDLY);
  UNIT_MATS['ROACH_E']     = makeDirTextures(drawRoach,     TEAM_COLOR_ENEMY);
```

- [ ] **Run both tests to confirm they pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#roachDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+roachEnemySpawnsAndRendersInVisualizer" -q
```

Expected: both PASS.

- [ ] **Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat(e16): drawRoach — low armoured body with acid gland sacs Refs #N Refs #83"
```

---

## Task 4: Hydralisk

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Extend `smokeTestDrawFn` lookup** — add after `drawRoach` entry:

```js
    if (typeof drawHydralisk !== 'undefined') lookup.drawHydralisk = drawHydralisk;
```

- [ ] **Write failing smoke + spawn tests**

```java
@Test
@Tag("browser")
void hydraliskDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawHydralisk', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawHydralisk dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void hydraliskEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.HYDRALISK, new Point2d(24, 24));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Hydralisk enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Run to confirm smoke test fails with -1**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#hydraliskDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL.

- [ ] **Add `drawHydralisk` function** after `drawRoach`:

```js
function drawHydralisk(ctx, S, dir, teamColor) {
  if (dir === 3) {
    ctx.save(); ctx.translate(S, 0); ctx.scale(-1, 1);
    drawHydralisk(ctx, S, 1, teamColor);
    ctx.restore(); return;
  }
  const cx = S / 2, cy = S * 0.5;

  if (dir === 0 || dir === 2) {
    // Snake lower body (drawn first)
    ctx.strokeStyle = '#2a0a3a'; ctx.lineWidth = 18;
    ctx.beginPath();
    ctx.moveTo(cx - S * 0.06, cy + S * 0.44);
    ctx.quadraticCurveTo(cx + S * 0.08, cy + S * 0.28, cx, cy + S * 0.12);
    ctx.stroke();
    ctx.strokeStyle = '#5c1a6e'; ctx.lineWidth = 10;
    ctx.beginPath();
    ctx.moveTo(cx - S * 0.06, cy + S * 0.44);
    ctx.quadraticCurveTo(cx + S * 0.08, cy + S * 0.28, cx, cy + S * 0.12);
    ctx.stroke();
    // Cobra frill
    ctx.fillStyle = '#5c1a6e';
    if (dir === 0) {
      ctx.beginPath();
      ctx.moveTo(cx, cy - S * 0.04);
      ctx.quadraticCurveTo(cx - S * 0.32, cy - S * 0.02, cx - S * 0.26, cy + S * 0.14);
      ctx.lineTo(cx - S * 0.14, cy + S * 0.12); ctx.closePath(); ctx.fill();
      ctx.beginPath();
      ctx.moveTo(cx, cy - S * 0.04);
      ctx.quadraticCurveTo(cx + S * 0.32, cy - S * 0.02, cx + S * 0.26, cy + S * 0.14);
      ctx.lineTo(cx + S * 0.14, cy + S * 0.12); ctx.closePath(); ctx.fill();
    } else {
      ctx.beginPath();
      ctx.moveTo(cx, cy);
      ctx.quadraticCurveTo(cx - S * 0.28, cy - S * 0.04, cx - S * 0.22, cy + S * 0.12);
      ctx.lineTo(cx - S * 0.12, cy + S * 0.1); ctx.closePath(); ctx.fill();
      ctx.beginPath();
      ctx.moveTo(cx, cy);
      ctx.quadraticCurveTo(cx + S * 0.28, cy - S * 0.04, cx + S * 0.22, cy + S * 0.12);
      ctx.lineTo(cx + S * 0.12, cy + S * 0.1); ctx.closePath(); ctx.fill();
    }
    // Torso
    ctx.fillStyle = '#2a0a3a';
    ctx.beginPath(); ctx.ellipse(cx, cy + S * 0.06, S * 0.18, S * 0.2, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#8b3a9e';
    ctx.beginPath(); ctx.ellipse(cx, cy + S * 0.02, S * 0.12, S * 0.12, 0, 0, Math.PI * 2); ctx.fill();
    // Head
    ctx.fillStyle = '#2a0a3a';
    ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.12, S * 0.13, S * 0.12, 0, 0, Math.PI * 2); ctx.fill();
    if (dir === 0) {
      ctx.fillStyle = '#ffe066';
      ctx.beginPath(); ctx.arc(cx - S * 0.055, cy - S * 0.13, S * 0.035, 0, Math.PI * 2); ctx.fill();
      ctx.beginPath(); ctx.arc(cx + S * 0.055, cy - S * 0.13, S * 0.035, 0, Math.PI * 2); ctx.fill();
    }
    // Throat sac
    const g = ctx.createRadialGradient(cx, cy + S * 0.01, 2, cx, cy + S * 0.01, S * 0.085);
    g.addColorStop(0, teamColor + 'ff'); g.addColorStop(0.55, teamColor + '88'); g.addColorStop(1, teamColor + '00');
    ctx.fillStyle = g; ctx.shadowColor = teamColor; ctx.shadowBlur = 9;
    ctx.beginPath(); ctx.ellipse(cx, cy + S * 0.01, S * 0.075, S * 0.07, 0, 0, Math.PI * 2); ctx.fill();
    // Shoulder pustules
    [cx - S * 0.16, cx + S * 0.16].forEach(x => {
      const g2 = ctx.createRadialGradient(x, cy + S * 0.06, 1, x, cy + S * 0.06, S * 0.055);
      g2.addColorStop(0, teamColor + 'dd'); g2.addColorStop(1, teamColor + '00');
      ctx.fillStyle = g2; ctx.beginPath(); ctx.arc(x, cy + S * 0.06, S * 0.05, 0, Math.PI * 2); ctx.fill();
    });
    ctx.shadowBlur = 0;
  } else {
    // dir=1: side profile
    ctx.strokeStyle = '#2a0a3a'; ctx.lineWidth = 16;
    ctx.beginPath();
    ctx.moveTo(cx + S * 0.08, cy + S * 0.44);
    ctx.quadraticCurveTo(cx + S * 0.14, cy + S * 0.28, cx, cy + S * 0.14);
    ctx.stroke();
    ctx.strokeStyle = '#5c1a6e'; ctx.lineWidth = 8;
    ctx.beginPath();
    ctx.moveTo(cx + S * 0.08, cy + S * 0.44);
    ctx.quadraticCurveTo(cx + S * 0.14, cy + S * 0.28, cx, cy + S * 0.14);
    ctx.stroke();
    // Side frill
    ctx.fillStyle = '#5c1a6e';
    ctx.beginPath();
    ctx.moveTo(cx, cy - S * 0.02);
    ctx.quadraticCurveTo(cx + S * 0.28, cy - S * 0.1, cx + S * 0.24, cy + S * 0.14);
    ctx.lineTo(cx + S * 0.12, cy + S * 0.12); ctx.closePath(); ctx.fill();
    // Torso
    ctx.fillStyle = '#2a0a3a';
    ctx.beginPath(); ctx.ellipse(cx, cy + S * 0.06, S * 0.16, S * 0.2, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#8b3a9e';
    ctx.beginPath(); ctx.ellipse(cx - S * 0.02, cy + S * 0.02, S * 0.1, S * 0.12, 0, 0, Math.PI * 2); ctx.fill();
    // Head
    ctx.fillStyle = '#2a0a3a';
    ctx.beginPath(); ctx.ellipse(cx - S * 0.04, cy - S * 0.12, S * 0.12, S * 0.11, 0, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#ffe066';
    ctx.beginPath(); ctx.arc(cx - S * 0.1, cy - S * 0.14, S * 0.032, 0, Math.PI * 2); ctx.fill();
    // Throat sac
    const g = ctx.createRadialGradient(cx, cy + S * 0.01, 2, cx, cy + S * 0.01, S * 0.08);
    g.addColorStop(0, teamColor + 'ff'); g.addColorStop(0.5, teamColor + '88'); g.addColorStop(1, teamColor + '00');
    ctx.fillStyle = g; ctx.shadowColor = teamColor; ctx.shadowBlur = 8;
    ctx.beginPath(); ctx.ellipse(cx, cy + S * 0.01, S * 0.07, S * 0.065, 0, 0, Math.PI * 2); ctx.fill();
    // One shoulder pustule
    const g2 = ctx.createRadialGradient(cx - S * 0.14, cy + S * 0.08, 1, cx - S * 0.14, cy + S * 0.08, S * 0.05);
    g2.addColorStop(0, teamColor + 'dd'); g2.addColorStop(1, teamColor + '00');
    ctx.fillStyle = g2; ctx.beginPath(); ctx.arc(cx - S * 0.14, cy + S * 0.08, S * 0.045, 0, Math.PI * 2); ctx.fill();
    ctx.shadowBlur = 0;
  }
}
```

- [ ] **Register HYDRALISK in `initSpriteMaterials`** — after ROACH lines:

```js
  UNIT_MATS['HYDRALISK_F'] = makeDirTextures(drawHydralisk, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['HYDRALISK_E'] = makeDirTextures(drawHydralisk, TEAM_COLOR_ENEMY);
```

- [ ] **Run both tests**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#hydraliskDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+hydraliskEnemySpawnsAndRendersInVisualizer" -q
```

Expected: both PASS.

- [ ] **Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat(e16): drawHydralisk — upright serpentine with cobra hood and throat sac Refs #N Refs #83"
```

---

## Task 5: Mutalisk + FLYING_UNITS

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Extend `smokeTestDrawFn` lookup** — add after `drawHydralisk` entry:

```js
    if (typeof drawMutalisk  !== 'undefined') lookup.drawMutalisk  = drawMutalisk;
```

- [ ] **Write failing smoke + spawn + elevation tests**

```java
@Test
@Tag("browser")
void mutaliskDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawMutalisk', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue()).as("drawMutalisk dir=" + dir + " team=" + color).isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void mutaliskEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.MUTALISK, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Mutalisk enemy must render").isEqualTo(1);
    page.close();
}

/**
 * Mutalisk must spawn higher than a ground Zergling.
 * TILE=0.7: ground Y = 0.7*0.65 = 0.455; flying Y = 0.7*1.5 = 1.05.
 */
@Test
@Tag("browser")
void mutaliskSpawnsHigherThanGroundUnit() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.ZERGLING, new Point2d(20, 20));
    engine.observe();
    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    List<Double> zerglingYs = ((List<?>) page.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList();
    double zerglingY = zerglingYs.get(0);
    page.close();

    orchestrator.startGame();
    Page page2 = browser.newPage();
    page2.navigate(pageUrl.toString());
    page2.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.MUTALISK, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    List<Double> mutaliskYs = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList();
    double mutaliskY = mutaliskYs.get(0);
    page2.close();

    assertThat(mutaliskY)
        .as("Mutalisk Y (%.3f) must be higher than Zergling Y (%.3f)".formatted(mutaliskY, zerglingY))
        .isGreaterThan(zerglingY + 0.3);
}
```

- [ ] **Run smoke test to confirm it fails**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#mutaliskDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams -q
```

Expected: FAIL — returns -1.

- [ ] **Update `FLYING_UNITS`** — at line 6 of `visualizer.js`, replace:

```js
const FLYING_UNITS = new Set(['MEDIVAC']);
```

With:

```js
const FLYING_UNITS = new Set(['MEDIVAC', 'MUTALISK']);
```

- [ ] **Add `drawMutalisk` function** after `drawHydralisk`:

```js
function drawMutalisk(ctx, S, dir, teamColor) {
  // All 4 directions show the manta silhouette — slight rotation per dir for variety
  const cx = S / 2, cy = S / 2;
  const angle = dir === 0 ? 0 : dir === 1 ? Math.PI * 0.06 : dir === 2 ? Math.PI * 0.1 : -Math.PI * 0.06;
  ctx.save();
  ctx.translate(cx, cy); ctx.rotate(angle); ctx.translate(-cx, -cy);

  // Wing membranes (behind body)
  ctx.fillStyle = '#5c1a6e';
  ctx.beginPath();
  ctx.moveTo(cx, cy + S * 0.02);
  ctx.quadraticCurveTo(cx - S * 0.44, cy - S * 0.1, cx - S * 0.46, cy + S * 0.14);
  ctx.quadraticCurveTo(cx - S * 0.28, cy + S * 0.2, cx, cy + S * 0.1);
  ctx.closePath(); ctx.fill();
  ctx.beginPath();
  ctx.moveTo(cx, cy + S * 0.02);
  ctx.quadraticCurveTo(cx + S * 0.44, cy - S * 0.1, cx + S * 0.46, cy + S * 0.14);
  ctx.quadraticCurveTo(cx + S * 0.28, cy + S * 0.2, cx, cy + S * 0.1);
  ctx.closePath(); ctx.fill();
  // Wing veins
  ctx.strokeStyle = '#8b3a9e'; ctx.lineWidth = 1.5; ctx.globalAlpha = 0.5;
  [[cx, cy + S * 0.06, cx - S * 0.36, cy + S * 0.14],
   [cx, cy + S * 0.06, cx - S * 0.24, cy - S * 0.06],
   [cx, cy + S * 0.06, cx + S * 0.36, cy + S * 0.14],
   [cx, cy + S * 0.06, cx + S * 0.24, cy - S * 0.06]].forEach(([x1, y1, x2, y2]) => {
    ctx.beginPath(); ctx.moveTo(x1, y1); ctx.lineTo(x2, y2); ctx.stroke();
  });
  ctx.globalAlpha = 1;
  // Central body
  ctx.fillStyle = '#2a0a3a';
  ctx.beginPath(); ctx.ellipse(cx, cy + S * 0.04, S * 0.1, S * 0.16, 0, 0, Math.PI * 2); ctx.fill();
  // Tail
  ctx.strokeStyle = '#2a0a3a'; ctx.lineWidth = 8;
  ctx.beginPath();
  ctx.moveTo(cx, cy + S * 0.18);
  ctx.quadraticCurveTo(cx + S * 0.06, cy + S * 0.32, cx + S * 0.04, cy + S * 0.44);
  ctx.stroke();
  ctx.strokeStyle = '#5c1a6e'; ctx.lineWidth = 4;
  ctx.beginPath();
  ctx.moveTo(cx, cy + S * 0.18);
  ctx.quadraticCurveTo(cx + S * 0.06, cy + S * 0.32, cx + S * 0.04, cy + S * 0.44);
  ctx.stroke();
  // Head
  ctx.fillStyle = '#8b3a9e';
  ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.12, S * 0.09, S * 0.09, 0, 0, Math.PI * 2); ctx.fill();
  ctx.fillStyle = '#2a0a3a';
  ctx.beginPath(); ctx.ellipse(cx, cy - S * 0.12, S * 0.07, S * 0.07, 0, 0, Math.PI * 2); ctx.fill();
  // Eyes
  ctx.fillStyle = '#ffe066';
  ctx.beginPath(); ctx.arc(cx - S * 0.035, cy - S * 0.14, S * 0.028, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.arc(cx + S * 0.035, cy - S * 0.14, S * 0.028, 0, Math.PI * 2); ctx.fill();
  // Wing-joint bio-sacs
  [[cx - S * 0.18, cy + S * 0.06], [cx + S * 0.18, cy + S * 0.06]].forEach(([x, y]) => {
    const g = ctx.createRadialGradient(x, y, 1, x, y, S * 0.062);
    g.addColorStop(0, teamColor + 'ff'); g.addColorStop(0.5, teamColor + '88'); g.addColorStop(1, teamColor + '00');
    ctx.fillStyle = g; ctx.shadowColor = teamColor; ctx.shadowBlur = 9;
    ctx.beginPath(); ctx.ellipse(x, y, S * 0.055, S * 0.05, 0, 0, Math.PI * 2); ctx.fill();
  });
  ctx.shadowBlur = 0;
  ctx.restore();
}
```

- [ ] **Register MUTALISK in `initSpriteMaterials`** — after HYDRALISK lines:

```js
  UNIT_MATS['MUTALISK_F']  = makeDirTextures(drawMutalisk,  TEAM_COLOR_FRIENDLY);
  UNIT_MATS['MUTALISK_E']  = makeDirTextures(drawMutalisk,  TEAM_COLOR_ENEMY);
```

- [ ] **Run all three Mutalisk tests**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#mutaliskDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+mutaliskEnemySpawnsAndRendersInVisualizer+mutaliskSpawnsHigherThanGroundUnit" -q
```

Expected: all PASS.

- [ ] **Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat(e16): drawMutalisk + FLYING_UNITS — manta wings with wing-joint bio-sacs Refs #N Refs #83"
```

---

## Task 6: Update Coverage Test + ShowcaseResource

**Files:**
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`
- Modify: `src/main/java/io/quarkmind/qa/ShowcaseResource.java`

- [ ] **Update `unitMatsCoversAllSevenTypesWithBothTeams`** — add the 8 Zerg keys to the existing `contains()` assertion (line ~961 in `VisualizerRenderTest.java`):

Replace:
```java
    assertThat(keys).contains(
        "PROBE_F",    "PROBE_E",
        "ZEALOT_F",   "ZEALOT_E",
        "STALKER_F",  "STALKER_E",
        "MARINE_F",   "MARINE_E",
        "MARAUDER_F", "MARAUDER_E",
        "MEDIVAC_F",  "MEDIVAC_E",
        "UNKNOWN_F",  "UNKNOWN_E"
    );
```

With:
```java
    assertThat(keys).contains(
        "PROBE_F",     "PROBE_E",
        "ZEALOT_F",    "ZEALOT_E",
        "STALKER_F",   "STALKER_E",
        "MARINE_F",    "MARINE_E",
        "MARAUDER_F",  "MARAUDER_E",
        "MEDIVAC_F",   "MEDIVAC_E",
        "ZERGLING_F",  "ZERGLING_E",
        "ROACH_F",     "ROACH_E",
        "HYDRALISK_F", "HYDRALISK_E",
        "MUTALISK_F",  "MUTALISK_E",
        "UNKNOWN_F",   "UNKNOWN_E"
    );
```

Also update the Javadoc line from `"all 7 unit types × 2 teams"` to `"all 11 unit types × 2 teams"`.

- [ ] **Run the coverage test to confirm it passes**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#unitMatsCoversAllSevenTypesWithBothTeams -q
```

Expected: PASS.

- [ ] **Update `ShowcaseResource.java`** — expand Row 3 to show all four Zerg units:

Replace:
```java
        // Row 3 (y=20): Zergling fallback blob
        simulatedGame.spawnEnemyUnit(UnitType.ZERGLING, new Point2d(16, 20));
```

With:
```java
        // Row 3 (y=20): Zerg enemies — now with proper sprites
        simulatedGame.spawnEnemyUnit(UnitType.ZERGLING,  new Point2d(10, 20));
        simulatedGame.spawnEnemyUnit(UnitType.ROACH,     new Point2d(14, 20));
        simulatedGame.spawnEnemyUnit(UnitType.HYDRALISK, new Point2d(18, 20));
        simulatedGame.spawnEnemyUnit(UnitType.MUTALISK,  new Point2d(22, 20));  // floats higher
```

Also update the Javadoc comment on the class:
```java
 * Shows all 11 unit types (Protoss + Terran + Zerg) in enemy team colour.
```

And update the response JSON:
```java
            "enemies", "Row1(y=12): Probe/Zealot/Stalker | Row2(y=16): Marine/Marauder/Medivac | Row3(y=20): Zergling/Roach/Hydralisk/Mutalisk"
```

- [ ] **Run standard test suite to confirm no regressions**

```bash
mvn test -q
```

Expected: all tests pass.

- [ ] **Commit**

```bash
git add src/test/java/io/quarkmind/qa/VisualizerRenderTest.java \
        src/main/java/io/quarkmind/qa/ShowcaseResource.java
git commit -m "feat(e16): update coverage test + ShowcaseResource — all three races seeded Refs #N Refs #83"
```

---

## Task 7: Full Regression

- [ ] **Run the complete Playwright suite**

```bash
mvn test -Pplaywright -q
```

Expected: all tests pass (existing + ~12 new Zerg tests).

- [ ] **Run the standard suite**

```bash
mvn test -q
```

Expected: all tests pass.

- [ ] **Manual showcase verification** — start emulated mode, open the browser, seed the showcase, confirm all three races are visible and Mutalisk floats higher:

```bash
mvn quarkus:dev -Dquarkus.profile=emulated
# In a second terminal:
curl -X POST http://localhost:8080/sc2/showcase
# Open http://localhost:8080/visualizer.html
```

- [ ] **If any test fails:** investigate before marking complete. Do not `@Disabled` without understanding the failure.

- [ ] **Close the issue**

```bash
gh issue close N --comment "All 4 Zerg draw functions implemented and tested. Closes #N Refs #83"
```

---

## Completion Checklist

- [ ] 22 `UNIT_MATS` keys registered (Protoss×6 + Terran×6 + Zerg×8 + Unknown×2)
- [ ] `FLYING_UNITS` includes `'MUTALISK'`
- [ ] All 4 draw functions use `function` declarations (not arrow fns) — required for `typeof` hoisting in `smokeTestDrawFn`
- [ ] All 4 draw functions produce non-zero alpha at centre pixel (64,64) for all 4 dirs × 2 teams
- [ ] `ShowcaseResource` seeds 10 enemy units across 3 rows
- [ ] Every commit references `#N` and `#83`
- [ ] `mvn test -q` green
- [ ] `mvn test -Pplaywright -q` green
