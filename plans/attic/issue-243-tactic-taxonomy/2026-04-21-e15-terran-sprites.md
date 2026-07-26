# E15 Terran Sprites + Team Colour Decal System — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add cartoon sprites for Marine, Marauder, and Medivac; retrofit all existing Protoss draw functions with SC2-style team-colour decals so friendly/enemy is always visually distinct regardless of which race the player controls.

**Architecture:** `makeDirTextures(drawFn, teamColor, size=128)` generates 4 directional `SpriteMaterial` arrays per unit per team. `UNIT_MATS` is keyed by `'TYPE_F'`/`'TYPE_E'`. Dispatch in `syncUnitLayer` looks up the key, falling back to `UNKNOWN_F`/`UNKNOWN_E`. Medivac gets a Y-offset via a `FLYING_UNITS` set. All new hooks are exposed through `window.__test` before writing any tests.

**Tech Stack:** Three.js r128 (client-side), Canvas 2D API, Playwright + Java `@QuarkusTest` (E2E), SimulatedGame test helpers.

**Spec:** `docs/superpowers/specs/2026-04-21-e15-terran-sprites-design.md`  
**Epic:** #83

---

## File Map

| File | Change |
|------|--------|
| `src/main/resources/META-INF/resources/visualizer.js` | All JS changes — constants, API, draw functions, dispatch |
| `src/main/resources/META-INF/resources/visualizer.html` | Add MARAUDER + MEDIVAC to config dropdown |
| `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java` | All new Playwright tests |
| `CLAUDE.md` | Document new `window.__test` hooks |

No Java source changes — `UnitType.MARINE/MARAUDER/MEDIVAC` already exist; `EmulatedConfig` already accepts any `UnitType` string.

---

## Task 1: Create GitHub Issue

**Files:** none

- [ ] **Create issue under epic #83**

```bash
gh issue create \
  --title "E15-1: Terran sprites (Marine, Marauder, Medivac) + team colour decal system" \
  --body "## Summary
Add cartoon sprites for Marine, Marauder, and Medivac.
Retrofit all Protoss draw functions with SC2-style team-colour decals.
Refs #83

## Acceptance criteria
- drawMarine, drawMarauder, drawMedivac implemented — all 4 directions × 2 team colours
- Existing drawProbe, drawZealot, drawStalker, drawEnemy use teamColor in decal zones
- UNIT_MATS keyed as TYPE_F / TYPE_E — unknown types fall back to UNKNOWN_F / UNKNOWN_E
- Medivac spawns at TILE * 1.5 height (flying)
- Config panel dropdown includes MARAUDER and MEDIVAC
- All Playwright tests green" \
  --label enhancement
```

- [ ] **Save the issue number** — every commit will reference it as `Refs #<N>`. The rest of this plan uses `#N` as a placeholder — replace with the real number.

---

## Task 2: Add `window.__test` Testability Hooks

All subsequent tests depend on these hooks. Add them before writing any failing tests.

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js` (lines 39–88, the `window.__test` block)
- Modify: `CLAUDE.md` (window.__test API table)

- [ ] **Add three new hooks to `window.__test`**

In `visualizer.js`, extend the `window.__test` object (currently ends with `worldToScreen`). Add these three entries after the existing `worldToScreen` entry:

```js
  unitMatsKeys: () => Object.keys(UNIT_MATS),

  allEnemyWorldY: () => Array.from(enemySprites.values()).map(sp => sp.position.y),

  smokeTestDrawFn: (name, dir, teamColor) => {
    // typeof on an undeclared identifier returns 'undefined' — safe even before fn is defined
    const lookup = {};
    if (typeof drawProbe    !== 'undefined') lookup.drawProbe    = drawProbe;
    if (typeof drawZealot   !== 'undefined') lookup.drawZealot   = drawZealot;
    if (typeof drawStalker  !== 'undefined') lookup.drawStalker  = drawStalker;
    if (typeof drawEnemy    !== 'undefined') lookup.drawEnemy    = drawEnemy;
    if (typeof drawMarine   !== 'undefined') lookup.drawMarine   = drawMarine;
    if (typeof drawMarauder !== 'undefined') lookup.drawMarauder = drawMarauder;
    if (typeof drawMedivac  !== 'undefined') lookup.drawMedivac  = drawMedivac;
    const fn = lookup[name];
    if (!fn) return -1;
    const c = document.createElement('canvas');
    c.width = c.height = 128;
    fn(c.getContext('2d'), 128, dir, teamColor);
    return c.getContext('2d').getImageData(64, 64, 1, 1).data[3]; // alpha at centre
  },
```

- [ ] **Update CLAUDE.md** — add three lines to the `window.__test` table in the Playwright section:

```
- `unitMatsKeys()` — array of all registered UNIT_MATS keys (e.g. `'MARINE_F'`, `'MARINE_E'`)
- `allEnemyWorldY()` — array of Three.js world Y positions for all current enemy sprites
- `smokeTestDrawFn(name, dir, teamColor)` — invokes named draw fn on a temp 128×128 canvas, returns centre pixel alpha (0–255), or -1 if function not found
```

- [ ] **Build and run existing tests to confirm no regressions**

```bash
mvn test -q
```

Expected: all existing tests pass (no JS or Java changes that affect behaviour).

- [ ] **Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js CLAUDE.md
git commit -m "test(e15): add window.__test hooks — unitMatsKeys, allEnemyWorldY, smokeTestDrawFn Refs #N Refs #83"
```

---

## Task 3: Team Colour Constants + `makeDirTextures` API + `UNIT_MATS` Dispatch

This task changes the dispatch plumbing. Existing draw functions silently ignore the new `teamColor` argument in JS (extra args are always safe), so existing sprite behaviour is unchanged. The failing test proves the new key format is wired up correctly.

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Write the failing Playwright test**

Add this test to `VisualizerRenderTest.java`:

```java
/**
 * UNIT_MATS must use TYPE_F / TYPE_E key format after Task 3 refactor.
 * Existing Protoss units and the UNKNOWN fallback must all be registered.
 */
@Test
@Tag("browser")
void unitMatsCoversAllRegisteredTypes() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    @SuppressWarnings("unchecked")
    List<String> keys = (List<String>) page.evaluate("() => window.__test.unitMatsKeys()");

    assertThat(keys).contains(
        "PROBE_F", "PROBE_E",
        "ZEALOT_F", "ZEALOT_E",
        "STALKER_F", "STALKER_E",
        "UNKNOWN_F", "UNKNOWN_E"
    );
    page.close();
}
```

- [ ] **Run it to confirm it fails**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#unitMatsCoversAllRegisteredTypes -q
```

Expected: FAIL — `PROBE_F` not found (current keys are `PROBE`, `ZEALOT`, `STALKER`).

- [ ] **Add team colour constants** at the top of `visualizer.js`, after the existing `const TILE = 0.7;` line:

```js
const TEAM_COLOR_FRIENDLY = '#4488ff';
const TEAM_COLOR_ENEMY    = '#ff4422';
```

- [ ] **Add `hexToRgba` helper** — needed by draw functions for alpha-transparent versions of teamColor. Add this immediately before `makeDirTextures`:

```js
function hexToRgba(hex, a) {
  const r = parseInt(hex.slice(1,3), 16);
  const g = parseInt(hex.slice(3,5), 16);
  const b = parseInt(hex.slice(5,7), 16);
  return `rgba(${r},${g},${b},${a})`;
}
```

- [ ] **Update `makeDirTextures` signature** — add `teamColor` as second parameter:

Replace:
```js
function makeDirTextures(drawFn, size = 128) {
  return [0, 1, 2, 3].map(dir => {
    const c = document.createElement('canvas');
    c.width = c.height = size;
    drawFn(c.getContext('2d'), size, dir);
```

With:
```js
function makeDirTextures(drawFn, teamColor, size = 128) {
  return [0, 1, 2, 3].map(dir => {
    const c = document.createElement('canvas');
    c.width = c.height = size;
    drawFn(c.getContext('2d'), size, dir, teamColor);
```

- [ ] **Remove the old module-level material variables** (line ~705):

Remove this entire block:
```js
let probeMats, zealotMats, stalkerMats, enemyMats;
```

- [ ] **Rewrite `initSpriteMaterials`** — replace the entire function and `populateUnitMats`:

Remove the old `initSpriteMaterials`, `populateUnitMats` call inside it, and `populateUnitMats` function entirely. Replace with:

```js
function initSpriteMaterials() {
  UNIT_MATS['PROBE_F']    = makeDirTextures(drawProbe,   TEAM_COLOR_FRIENDLY);
  UNIT_MATS['PROBE_E']    = makeDirTextures(drawProbe,   TEAM_COLOR_ENEMY);
  UNIT_MATS['ZEALOT_F']   = makeDirTextures(drawZealot,  TEAM_COLOR_FRIENDLY);
  UNIT_MATS['ZEALOT_E']   = makeDirTextures(drawZealot,  TEAM_COLOR_ENEMY);
  UNIT_MATS['STALKER_F']  = makeDirTextures(drawStalker, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['STALKER_E']  = makeDirTextures(drawStalker, TEAM_COLOR_ENEMY);
  UNIT_MATS['UNKNOWN_F']  = makeDirTextures(drawEnemy,   TEAM_COLOR_FRIENDLY);
  UNIT_MATS['UNKNOWN_E']  = makeDirTextures(drawEnemy,   TEAM_COLOR_ENEMY);
}
```

- [ ] **Update the dispatch in `syncUnitLayer`** — replace line ~409:

Replace:
```js
      const mats = isEnemy ? enemyMats : (UNIT_MATS[u.type] ?? enemyMats);
```

With:
```js
      const key  = u.type + (isEnemy ? '_E' : '_F');
      const mats = UNIT_MATS[key] ?? UNIT_MATS['UNKNOWN_' + (isEnemy ? 'E' : 'F')];
```

- [ ] **Run the new test to confirm it passes**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#unitMatsCoversAllRegisteredTypes -q
```

Expected: PASS.

- [ ] **Run the full Playwright suite to confirm no regressions**

```bash
mvn test -Pplaywright -q
```

Expected: all tests pass.

- [ ] **Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat(e15): team colour constants + makeDirTextures API + UNIT_MATS dispatch Refs #N Refs #83"
```

---

## Task 4: Retrofit Existing Draw Functions with `teamColor` Decals

Each function now receives `teamColor` and applies it to its SC2 decal zones. These tests pass both before and after (they're regression guards), and are written here before touching draw functions so any mistake shows up immediately.

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Write Playwright smoke tests for all four existing draw functions**

Add to `VisualizerRenderTest.java`:

```java
/**
 * Each existing draw function must produce non-transparent output for all
 * 4 directions and both team colours. Catches regressions when teamColor
 * parameter is added to existing draw function signatures.
 */
@Test
@Tag("browser")
void existingDrawFunctionsProduceNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    String[] fns     = {"drawProbe", "drawZealot", "drawStalker", "drawEnemy"};
    String[] colors  = {TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY};
    int[]    dirs    = {0, 1, 2, 3};

    for (String fn : fns) {
      for (String color : colors) {
        for (int dir : dirs) {
          Number alpha = (Number) page.evaluate(
              "() => window.__test.smokeTestDrawFn('" + fn + "', " + dir + ", '" + color + "')");
          assertThat(alpha.intValue())
              .as(fn + " dir=" + dir + " team=" + color + " centre alpha")
              .isGreaterThan(0);
        }
      }
    }
    page.close();
}

private static final String TEAM_COLOR_FRIENDLY = "#4488ff";
private static final String TEAM_COLOR_ENEMY    = "#ff4422";
```

- [ ] **Run the test to confirm it passes before retrofit** (it should — draw functions run fine with extra args ignored in JS)

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#existingDrawFunctionsProduceNonTransparentOutputForAllDirsAndTeams -q
```

Expected: PASS (JS silently ignores unused `teamColor` arg for now).

- [ ] **Retrofit `drawProbe`** — apply `teamColor` to outer shell highlights and body

`drawProbe` is at line ~471. Add `teamColor` as 4th parameter and replace hardcoded body gradient colours:

In all three dir cases (front, side, back), the body gradient (`const b = ...`) currently uses `'#88ccff'`, `'#4488dd'`, `'#224499'`. Replace the middle two stops with `teamColor` and `hexToRgba(teamColor, 0.6)`:

```js
function drawProbe(ctx, S, dir, teamColor) {
```

For the outer glow (first line in all cases):
```js
  grd.addColorStop(0, hexToRgba(teamColor, 0.3)); grd.addColorStop(1, 'rgba(0,0,0,0)');
```

For each body gradient `b`, change:
- stop 0: `hexToRgba(teamColor, 0.9)` (bright highlight)
- stop .6 or .7: `teamColor` (main body)
- stop 1: keep dark near-black (`'#112244'` → `hexToRgba(teamColor, 0.3)` tinted dark)

The psionic cannon energy (gold `#ffff88`/`#ffcc00`) and leg ellipses stay unchanged — those are canonical Probe geometry, not player-coloured in SC2.

Full updated `drawProbe` front section (the other two dirs follow the same pattern):

```js
function drawProbe(ctx, S, dir, teamColor) {
  const cx = S/2, cy = S/2 + 4;
  const grd = ctx.createRadialGradient(cx, cy, S*.05, cx, cy, S*.46);
  grd.addColorStop(0, hexToRgba(teamColor, 0.3)); grd.addColorStop(1, 'rgba(0,0,0,0)');
  ctx.fillStyle = grd; ctx.beginPath(); ctx.ellipse(cx,cy,S*.46,S*.46,0,0,Math.PI*2); ctx.fill();

  if (dir === 2) { // BACK
    const b = ctx.createRadialGradient(cx+S*.08,cy-S*.08,S*.02,cx,cy,S*.3);
    b.addColorStop(0, hexToRgba(teamColor, 0.9));
    b.addColorStop(.7, teamColor);
    b.addColorStop(1, hexToRgba(teamColor, 0.25));
    ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy,S*.3,S*.28,0,0,Math.PI*2); ctx.fill();
    ctx.strokeStyle=hexToRgba(teamColor, 0.7); ctx.lineWidth=3;
    ctx.beginPath(); ctx.moveTo(cx,cy-S*.28); ctx.lineTo(cx,cy-S*.42); ctx.stroke();
    ctx.fillStyle=hexToRgba(teamColor, 0.8); ctx.beginPath(); ctx.ellipse(cx,cy-S*.44,S*.04,S*.04,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle=teamColor;
    ctx.beginPath(); ctx.ellipse(cx-S*.1,cy+S*.26,S*.08,S*.05,0,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(cx+S*.1,cy+S*.26,S*.08,S*.05,0,0,Math.PI*2); ctx.fill();
    return;
  }
  if (dir === 1 || dir === 3) { // SIDE
    const flip = dir===3 ? -1 : 1;
    const b = ctx.createRadialGradient(cx+flip*S*.06,cy-S*.08,S*.02,cx,cy,S*.28);
    b.addColorStop(0, hexToRgba(teamColor, 0.9));
    b.addColorStop(.6, teamColor);
    b.addColorStop(1, hexToRgba(teamColor, 0.3));
    ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy,S*.28,S*.24,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='rgba(255,255,255,0.12)';
    ctx.beginPath(); ctx.ellipse(cx+flip*S*.08,cy-S*.1,S*.1,S*.07,-.3,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='white'; ctx.beginPath(); ctx.ellipse(cx+flip*S*.16,cy-S*.04,S*.08,S*.08,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#112244'; ctx.beginPath(); ctx.ellipse(cx+flip*S*.18,cy-S*.03,S*.046,S*.046,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='white'; ctx.beginPath(); ctx.ellipse(cx+flip*S*.15,cy-S*.07,S*.02,S*.02,0,0,Math.PI*2); ctx.fill();
    ctx.strokeStyle=hexToRgba(teamColor, 0.7); ctx.lineWidth=2;
    ctx.beginPath(); ctx.moveTo(cx+flip*S*.24,cy-S*.02); ctx.lineTo(cx+flip*S*.37,cy-S*.14); ctx.stroke();
    const eg = ctx.createRadialGradient(cx+flip*S*.4,cy-S*.17,0,cx+flip*S*.4,cy-S*.17,S*.09);
    eg.addColorStop(0,'#ffff88'); eg.addColorStop(.5,'#ffcc00'); eg.addColorStop(1,'rgba(255,180,0,0)');
    ctx.fillStyle=eg; ctx.beginPath(); ctx.ellipse(cx+flip*S*.4,cy-S*.17,S*.09,S*.09,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle=teamColor;
    ctx.beginPath(); ctx.ellipse(cx,cy+S*.26,S*.1,S*.055,0,0,Math.PI*2); ctx.fill();
    return;
  }
  // FRONT
  const b = ctx.createRadialGradient(cx-S*.08,cy-S*.1,S*.04,cx,cy,S*.3);
  b.addColorStop(0, hexToRgba(teamColor, 0.9));
  b.addColorStop(.6, teamColor);
  b.addColorStop(1, hexToRgba(teamColor, 0.3));
  ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy,S*.3,S*.28,0,0,Math.PI*2); ctx.fill();
  ctx.fillStyle='rgba(255,255,255,0.15)';
  ctx.beginPath(); ctx.ellipse(cx-S*.06,cy-S*.1,S*.12,S*.08,-.4,0,Math.PI*2); ctx.fill();
  [[-S*.1],[S*.1]].forEach(([ex]) => {
    const exx=cx+ex, eyy=cy-S*.06;
    ctx.fillStyle='white'; ctx.beginPath(); ctx.ellipse(exx,eyy,S*.07,S*.07,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#112244'; ctx.beginPath(); ctx.ellipse(exx+1,eyy+1,S*.04,S*.04,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='white'; ctx.beginPath(); ctx.ellipse(exx-1,eyy-2,S*.018,S*.018,0,0,Math.PI*2); ctx.fill();
  });
  ctx.strokeStyle=hexToRgba(teamColor, 0.7); ctx.lineWidth=2;
  ctx.beginPath(); ctx.moveTo(cx+S*.22,cy-S*.04); ctx.lineTo(cx+S*.35,cy-S*.16); ctx.stroke();
  const eg = ctx.createRadialGradient(cx+S*.38,cy-S*.19,0,cx+S*.38,cy-S*.19,S*.09);
  eg.addColorStop(0,'#ffff88'); eg.addColorStop(.5,'#ffcc00'); eg.addColorStop(1,'rgba(255,180,0,0)');
  ctx.fillStyle=eg; ctx.beginPath(); ctx.ellipse(cx+S*.38,cy-S*.19,S*.09,S*.09,0,0,Math.PI*2); ctx.fill();
  ctx.fillStyle=teamColor;
  ctx.beginPath(); ctx.ellipse(cx-S*.1,cy+S*.26,S*.08,S*.05,0,0,Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.ellipse(cx+S*.1,cy+S*.26,S*.08,S*.05,0,0,Math.PI*2); ctx.fill();
}
```

- [ ] **Retrofit `drawZealot`** — apply `teamColor` to psi-blade energy

`drawZealot` is at line ~530. Change signature and blade colours:

```js
function drawZealot(ctx, S, dir, teamColor) {
```

In `blade(bx)` inner function (front view):
```js
  function blade(bx) {
    const bg = ctx.createLinearGradient(bx-8,cy-S*.2,bx+8,cy+S*.2);
    bg.addColorStop(0, hexToRgba(teamColor, 0.9));
    bg.addColorStop(.5, teamColor);
    bg.addColorStop(1, hexToRgba(teamColor, 0.2));
    ctx.fillStyle=bg; ctx.beginPath(); ctx.ellipse(bx,cy,S*.07,S*.2,0,0,Math.PI*2); ctx.fill();
  }
```

For the side view blade:
```js
    const bg = ctx.createLinearGradient(cx+flip*S*.22,cy,cx+flip*S*.42,cy);
    bg.addColorStop(flip>0?0:1, hexToRgba(teamColor, 0.9));
    bg.addColorStop(flip>0?1:0, hexToRgba(teamColor, 0.0));
```

For the back view blades:
```js
    const bg1 = ctx.createLinearGradient(cx-S*.36,cy,cx-S*.28,cy);
    bg1.addColorStop(0,'rgba(0,0,0,0)'); bg1.addColorStop(1, teamColor);
    ...
    const bg2 = ctx.createLinearGradient(cx+S*.28,cy,cx+S*.36,cy);
    bg2.addColorStop(0, teamColor); bg2.addColorStop(1,'rgba(0,0,0,0)');
```

The Zealot body stays purple (canonical Protoss colour). Only blades carry team colour in SC2.

- [ ] **Retrofit `drawStalker`** — apply `teamColor` to eye glow

`drawStalker` is at line ~590. Change signature:

```js
function drawStalker(ctx, S, dir, teamColor) {
```

Update `drawEye(ex, ey, r)` inner function — middle glow uses teamColor:

```js
  function drawEye(ex, ey, r) {
    r = r ?? S*.09;
    ctx.fillStyle='#001122'; ctx.beginPath(); ctx.ellipse(ex,ey,r*1.3,r*1.3,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle=teamColor;  ctx.beginPath(); ctx.ellipse(ex,ey,r,r,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle=hexToRgba(teamColor, 0.7); ctx.beginPath(); ctx.ellipse(ex,ey,r*.6,r*.6,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='white';    ctx.beginPath(); ctx.ellipse(ex-r*.4,ey-r*.4,r*.22,r*.22,0,0,Math.PI*2); ctx.fill();
  }
```

- [ ] **Retrofit `drawEnemy`** — apply `teamColor` to spike tips and eye iris

`drawEnemy` is at line ~649. Change signature:

```js
function drawEnemy(ctx, S, dir, teamColor) {
```

In `spikes()`, change fill to `teamColor` instead of hardcoded `#dd2211`:

```js
  function spikes(num, startAngle) {
    ctx.fillStyle = teamColor;
    ...
  }
```

Eye iris colour (`#ffcc00` yellow) → `teamColor`:

```js
    ctx.fillStyle=teamColor; ctx.beginPath(); ctx.ellipse(exx,eyy,S*.075,S*.065,0,0,Math.PI*2); ctx.fill();
```

(Apply the same substitution in both the front and side-view eye drawing code.)

- [ ] **Run the smoke test again to confirm all pass after retrofit**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#existingDrawFunctionsProduceNonTransparentOutputForAllDirsAndTeams -q
```

Expected: PASS.

- [ ] **Run full Playwright suite**

```bash
mvn test -Pplaywright -q
```

Expected: all existing tests pass.

- [ ] **Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat(e15): retrofit Probe, Zealot, Stalker, Enemy draw fns with teamColor decals Refs #N Refs #83"
```

---

## Task 5: `drawMarine` — CMC Suit with Visor + Shoulder Pad Decals

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Write the failing smoke test**

Add to `VisualizerRenderTest.java`:

```java
/**
 * drawMarine must produce non-transparent output for all 4 directions
 * and both team colours. Returns -1 (not > 0) until drawMarine is defined.
 */
@Test
@Tag("browser")
void marineDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawMarine', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue())
            .as("drawMarine dir=" + dir + " team=" + color)
            .isGreaterThan(0);
      }
    }
    page.close();
}

/**
 * Happy path: Marine enemy unit spawns and renders as a sprite.
 * UNIT_MATS['MARINE_E'] must be registered and dispatch must resolve it.
 */
@Test
@Tag("browser")
void marineEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.MARINE, new Point2d(20, 20));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Marine enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Run both tests to confirm they fail**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#marineDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+marineEnemySpawnsAndRendersInVisualizer" -q
```

Expected: both FAIL — `drawMarine` returns -1; MARINE_E not in UNIT_MATS.

- [ ] **Add `drawMarine` function** to `visualizer.js` after `drawEnemy`:

```js
function drawMarine(ctx, S, dir, teamColor) {
  const cx = S/2, cy = S/2 + 4;
  // Outer glow
  const grd = ctx.createRadialGradient(cx, cy, S*.05, cx, cy, S*.44);
  grd.addColorStop(0, 'rgba(100,120,140,0.3)'); grd.addColorStop(1, 'rgba(0,0,0,0)');
  ctx.fillStyle = grd; ctx.beginPath(); ctx.ellipse(cx,cy,S*.44,S*.44,0,0,Math.PI*2); ctx.fill();

  if (dir === 2) { // BACK
    // CMC backpack + power coupling
    const b = ctx.createRadialGradient(cx+S*.06,cy-S*.06,S*.02,cx,cy,S*.3);
    b.addColorStop(0,'#8899aa'); b.addColorStop(.6,'#556677'); b.addColorStop(1,'#223344');
    ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy,S*.3,S*.28,0,0,Math.PI*2); ctx.fill();
    // Shoulder pads in teamColor
    ctx.fillStyle=teamColor;
    ctx.beginPath(); ctx.ellipse(cx-S*.24,cy-S*.08,S*.1,S*.07,0,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(cx+S*.24,cy-S*.08,S*.1,S*.07,0,0,Math.PI*2); ctx.fill();
    // Backpack power cell
    ctx.fillStyle='#334455';
    ctx.fillRect(cx-S*.06,cy-S*.28,S*.12,S*.18);
    ctx.fillStyle='#445566'; ctx.beginPath(); ctx.ellipse(cx,cy-S*.28,S*.06,S*.05,0,0,Math.PI*2); ctx.fill();
    // Boots
    ctx.fillStyle='#223344';
    ctx.beginPath(); ctx.ellipse(cx-S*.1,cy+S*.28,S*.1,S*.055,0,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(cx+S*.1,cy+S*.28,S*.1,S*.055,0,0,Math.PI*2); ctx.fill();
    return;
  }
  if (dir === 1 || dir === 3) { // SIDE
    const flip = dir===3 ? -1 : 1;
    // Body
    const b = ctx.createRadialGradient(cx+flip*S*.04,cy-S*.04,S*.02,cx,cy,S*.28);
    b.addColorStop(0,'#8899aa'); b.addColorStop(.5,'#556677'); b.addColorStop(1,'#223344');
    ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy,S*.28,S*.26,0,0,Math.PI*2); ctx.fill();
    // Helmet
    const h = ctx.createRadialGradient(cx+flip*S*.04,cy-S*.2,S*.02,cx,cy-S*.14,S*.18);
    h.addColorStop(0,'#6677aa'); h.addColorStop(.6,'#334466'); h.addColorStop(1,'#0d1822');
    ctx.fillStyle=h; ctx.beginPath(); ctx.ellipse(cx,cy-S*.14,S*.18,S*.18,0,0,Math.PI*2); ctx.fill();
    // Visor in teamColor (side — small slit)
    ctx.fillStyle=hexToRgba(teamColor, 0.75);
    ctx.beginPath(); ctx.ellipse(cx+flip*S*.1,cy-S*.16,S*.08,S*.04,0,0,Math.PI*2); ctx.fill();
    // Shoulder pad on visible side
    ctx.fillStyle=teamColor;
    ctx.beginPath(); ctx.ellipse(cx+flip*S*.22,cy-S*.08,S*.09,S*.065,0,0,Math.PI*2); ctx.fill();
    // Rifle barrel (pointing in facing direction)
    ctx.fillStyle='#222';
    ctx.fillRect(cx+flip*S*.2,cy-S*.02,flip*S*.26,S*.07);
    // Boot
    ctx.fillStyle='#223344';
    ctx.beginPath(); ctx.ellipse(cx,cy+S*.28,S*.12,S*.055,0,0,Math.PI*2); ctx.fill();
    return;
  }
  // FRONT
  // Body — barrel-chested CMC suit
  const b = ctx.createRadialGradient(cx-S*.06,cy-S*.04,S*.04,cx,cy,S*.32);
  b.addColorStop(0,'#8899aa'); b.addColorStop(.5,'#556677'); b.addColorStop(1,'#223344');
  ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy+S*.02,S*.3,S*.28,0,0,Math.PI*2); ctx.fill();
  // Shoulder pads in teamColor
  ctx.fillStyle=teamColor;
  ctx.beginPath(); ctx.ellipse(cx-S*.26,cy-S*.06,S*.11,S*.07,0,0,Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.ellipse(cx+S*.26,cy-S*.06,S*.11,S*.07,0,0,Math.PI*2); ctx.fill();
  // Helmet dome
  const h = ctx.createRadialGradient(cx-S*.06,cy-S*.2,S*.02,cx,cy-S*.16,S*.2);
  h.addColorStop(0,'#6677aa'); h.addColorStop(.5,'#334466'); h.addColorStop(1,'#0d1822');
  ctx.fillStyle=h; ctx.beginPath(); ctx.ellipse(cx,cy-S*.16,S*.2,S*.2,0,0,Math.PI*2); ctx.fill();
  // Visor in teamColor
  ctx.fillStyle=hexToRgba(teamColor, 0.75);
  ctx.beginPath(); ctx.ellipse(cx,cy-S*.17,S*.13,S*.065,0,0,Math.PI*2); ctx.fill();
  // Visor highlight
  ctx.fillStyle='rgba(255,255,255,0.35)';
  ctx.beginPath(); ctx.ellipse(cx-S*.04,cy-S*.19,S*.055,S*.025,-0.3,0,Math.PI*2); ctx.fill();
  // Right arm + rifle
  ctx.fillStyle='#445566';
  ctx.fillRect(cx+S*.2,cy-S*.03,S*.22,S*.08);
  ctx.fillStyle='#222';
  ctx.fillRect(cx+S*.3,cy-S*.01,S*.18,S*.05);
  // Boot
  ctx.fillStyle='#223344';
  ctx.beginPath(); ctx.ellipse(cx-S*.1,cy+S*.28,S*.1,S*.055,0,0,Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.ellipse(cx+S*.1,cy+S*.28,S*.1,S*.055,0,0,Math.PI*2); ctx.fill();
}
```

- [ ] **Register MARINE in `initSpriteMaterials`** — add after the STALKER lines:

```js
  UNIT_MATS['MARINE_F']   = makeDirTextures(drawMarine,   TEAM_COLOR_FRIENDLY);
  UNIT_MATS['MARINE_E']   = makeDirTextures(drawMarine,   TEAM_COLOR_ENEMY);
```

- [ ] **Run both tests to confirm they pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#marineDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+marineEnemySpawnsAndRendersInVisualizer" -q
```

Expected: both PASS.

- [ ] **Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat(e15): drawMarine — CMC suit with teamColor visor + shoulder pads Refs #N Refs #83"
```

---

## Task 6: `drawMarauder` — Heavy Bio-Suit with Visor Slit + Knee Plate Decals

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Write the failing smoke + E2E tests**

Add to `VisualizerRenderTest.java`:

```java
@Test
@Tag("browser")
void marauderDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawMarauder', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue())
            .as("drawMarauder dir=" + dir + " team=" + color)
            .isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void marauderEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.MARAUDER, new Point2d(22, 22));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Marauder enemy must render").isEqualTo(1);
    page.close();
}
```

- [ ] **Run both tests to confirm they fail**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#marauderDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+marauderEnemySpawnsAndRendersInVisualizer" -q
```

Expected: both FAIL.

- [ ] **Add `drawMarauder` function** after `drawMarine`:

```js
function drawMarauder(ctx, S, dir, teamColor) {
  const cx = S/2, cy = S/2 + 2;
  // Glow — heavier green-grey
  const grd = ctx.createRadialGradient(cx,cy,S*.06,cx,cy,S*.46);
  grd.addColorStop(0,'rgba(90,110,90,0.3)'); grd.addColorStop(1,'rgba(0,0,0,0)');
  ctx.fillStyle=grd; ctx.beginPath(); ctx.ellipse(cx,cy,S*.46,S*.46,0,0,Math.PI*2); ctx.fill();

  if (dir === 2) { // BACK
    const b = ctx.createRadialGradient(cx+S*.08,cy-S*.06,S*.04,cx,cy,S*.34);
    b.addColorStop(0,'#778866'); b.addColorStop(.6,'#445544'); b.addColorStop(1,'#1a221a');
    ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy+S*.04,S*.34,S*.3,0,0,Math.PI*2); ctx.fill();
    // Wide shoulder pads
    ctx.fillStyle='#556644';
    ctx.beginPath(); ctx.ellipse(cx-S*.3,cy-S*.06,S*.12,S*.09,0,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(cx+S*.3,cy-S*.06,S*.12,S*.09,0,0,Math.PI*2); ctx.fill();
    // Knee plates in teamColor
    ctx.fillStyle=teamColor;
    ctx.beginPath(); ctx.ellipse(cx-S*.12,cy+S*.22,S*.1,S*.06,0,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(cx+S*.12,cy+S*.22,S*.1,S*.06,0,0,Math.PI*2); ctx.fill();
    // Dual grenade launchers
    ctx.fillStyle='#222';
    ctx.fillRect(cx-S*.06,cy-S*.34,S*.12,S*.18);
    // Boots
    ctx.fillStyle='#223311';
    ctx.beginPath(); ctx.ellipse(cx-S*.12,cy+S*.3,S*.12,S*.06,0,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(cx+S*.12,cy+S*.3,S*.12,S*.06,0,0,Math.PI*2); ctx.fill();
    return;
  }
  if (dir === 1 || dir === 3) { // SIDE
    const flip = dir===3 ? -1 : 1;
    // Heavy body
    const b = ctx.createRadialGradient(cx+flip*S*.06,cy-S*.04,S*.04,cx,cy,S*.32);
    b.addColorStop(0,'#778866'); b.addColorStop(.5,'#445544'); b.addColorStop(1,'#1a221a');
    ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy,S*.32,S*.28,0,0,Math.PI*2); ctx.fill();
    // Angular helmet
    const h = ctx.createRadialGradient(cx+flip*S*.04,cy-S*.2,S*.02,cx,cy-S*.14,S*.2);
    h.addColorStop(0,'#778866'); h.addColorStop(.6,'#445544'); h.addColorStop(1,'#1a221a');
    ctx.fillStyle=h; ctx.beginPath(); ctx.ellipse(cx,cy-S*.14,S*.2,S*.18,0,0,Math.PI*2); ctx.fill();
    // Visor slit in teamColor
    ctx.fillStyle=hexToRgba(teamColor, 0.8);
    ctx.beginPath(); ctx.ellipse(cx+flip*S*.1,cy-S*.16,S*.09,S*.035,0,0,Math.PI*2); ctx.fill();
    // Visible shoulder pad
    ctx.fillStyle='#556644';
    ctx.beginPath(); ctx.ellipse(cx+flip*S*.28,cy-S*.06,S*.11,S*.08,0,0,Math.PI*2); ctx.fill();
    // Dual launchers
    ctx.fillStyle='#222';
    ctx.fillRect(cx+flip*S*.18,cy-S*.06,flip*S*.3,S*.07);
    ctx.fillRect(cx+flip*S*.18,cy+S*.02,flip*S*.3,S*.07);
    // Knee plate in teamColor
    ctx.fillStyle=teamColor;
    ctx.beginPath(); ctx.ellipse(cx,cy+S*.22,S*.1,S*.06,0,0,Math.PI*2); ctx.fill();
    // Boot
    ctx.fillStyle='#223311';
    ctx.beginPath(); ctx.ellipse(cx,cy+S*.3,S*.13,S*.06,0,0,Math.PI*2); ctx.fill();
    return;
  }
  // FRONT
  // Heavy armored body
  const b = ctx.createRadialGradient(cx-S*.08,cy-S*.06,S*.04,cx,cy,S*.36);
  b.addColorStop(0,'#889977'); b.addColorStop(.5,'#445544'); b.addColorStop(1,'#1a221a');
  ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy+S*.04,S*.36,S*.3,0,0,Math.PI*2); ctx.fill();
  // Wide shoulder pads
  ctx.fillStyle='#667755';
  ctx.beginPath(); ctx.ellipse(cx-S*.32,cy-S*.04,S*.13,S*.09,0,0,Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.ellipse(cx+S*.32,cy-S*.04,S*.13,S*.09,0,0,Math.PI*2); ctx.fill();
  // Helmet — more angular than Marine
  const h = ctx.createRadialGradient(cx-S*.04,cy-S*.2,S*.02,cx,cy-S*.16,S*.22);
  h.addColorStop(0,'#889977'); h.addColorStop(.6,'#445544'); h.addColorStop(1,'#0d1108');
  ctx.fillStyle=h; ctx.beginPath(); ctx.ellipse(cx,cy-S*.16,S*.22,S*.2,0,0,Math.PI*2); ctx.fill();
  // Visor slit in teamColor (narrower than Marine visor)
  ctx.fillStyle=hexToRgba(teamColor, 0.85);
  ctx.beginPath(); ctx.ellipse(cx,cy-S*.18,S*.14,S*.04,0,0,Math.PI*2); ctx.fill();
  // Chin guard
  ctx.fillStyle='#334433';
  ctx.beginPath(); ctx.ellipse(cx,cy-S*.08,S*.1,S*.06,0,0,Math.PI*2); ctx.fill();
  // Dual Punisher grenade launchers
  ctx.fillStyle='#222';
  ctx.fillRect(cx+S*.2,cy-S*.07,S*.24,S*.07);
  ctx.fillRect(cx+S*.2,cy+S*.01,S*.24,S*.07);
  ctx.fillStyle='#444';
  ctx.beginPath(); ctx.ellipse(cx+S*.46,cy-S*.035,S*.06,S*.05,0,0,Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.ellipse(cx+S*.46,cy+S*.045,S*.06,S*.05,0,0,Math.PI*2); ctx.fill();
  // Knee plates in teamColor
  ctx.fillStyle=teamColor;
  ctx.beginPath(); ctx.ellipse(cx-S*.14,cy+S*.22,S*.11,S*.062,0,0,Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.ellipse(cx+S*.14,cy+S*.22,S*.11,S*.062,0,0,Math.PI*2); ctx.fill();
  // Boots
  ctx.fillStyle='#223311';
  ctx.beginPath(); ctx.ellipse(cx-S*.12,cy+S*.3,S*.12,S*.06,0,0,Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.ellipse(cx+S*.12,cy+S*.3,S*.12,S*.06,0,0,Math.PI*2); ctx.fill();
}
```

- [ ] **Register MARAUDER in `initSpriteMaterials`** — add after MARINE lines:

```js
  UNIT_MATS['MARAUDER_F'] = makeDirTextures(drawMarauder, TEAM_COLOR_FRIENDLY);
  UNIT_MATS['MARAUDER_E'] = makeDirTextures(drawMarauder, TEAM_COLOR_ENEMY);
```

- [ ] **Run both tests to confirm they pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#marauderDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+marauderEnemySpawnsAndRendersInVisualizer" -q
```

Expected: both PASS.

- [ ] **Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat(e15): drawMarauder — heavy suit with teamColor visor slit + knee plates Refs #N Refs #83"
```

---

## Task 7: `drawMedivac` + `FLYING_UNITS` Y-offset

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Write the failing smoke + E2E + elevation tests**

Add to `VisualizerRenderTest.java`:

```java
@Test
@Tag("browser")
void medivacDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    for (String color : new String[]{TEAM_COLOR_FRIENDLY, TEAM_COLOR_ENEMY}) {
      for (int dir = 0; dir < 4; dir++) {
        Number alpha = (Number) page.evaluate(
            "() => window.__test.smokeTestDrawFn('drawMedivac', " + dir + ", '" + color + "')");
        assertThat(alpha.intValue())
            .as("drawMedivac dir=" + dir + " team=" + color)
            .isGreaterThan(0);
      }
    }
    page.close();
}

@Test
@Tag("browser")
void medivacEnemySpawnsAndRendersInVisualizer() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.MEDIVAC, new Point2d(24, 24));
    engine.observe();

    page.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate("() => window.__test.enemyCount()")).intValue();
    assertThat(count).as("one Medivac enemy must render").isEqualTo(1);
    page.close();
}

/**
 * Correctness: Medivac must spawn at a higher Three.js world Y than a ground unit.
 * TILE=0.7: ground Y = 0.7*0.65 = 0.455; flying Y = 0.7*1.5 = 1.05.
 */
@Test
@Tag("browser")
void medivacSpawnsHigherThanGroundUnit() throws Exception {
    // Test Marine Y first (ground unit)
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

    // Now test Medivac Y
    orchestrator.startGame(); // reset game state
    Page page2 = browser.newPage();
    page2.navigate(pageUrl.toString());
    page2.waitForFunction("() => window.__test?.wsConnected?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    simulatedGame.spawnEnemyUnit(UnitType.MEDIVAC, new Point2d(20, 20));
    engine.observe();
    page2.waitForFunction("() => window.__test.enemyCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    List<Double> medivacYs = ((List<?>) page2.evaluate("() => window.__test.allEnemyWorldY()"))
        .stream().map(v -> ((Number) v).doubleValue()).toList();
    double medivacY = medivacYs.get(0);
    page2.close();

    assertThat(medivacY)
        .as("Medivac world Y (%.3f) must be higher than Marine world Y (%.3f)".formatted(medivacY, marineY))
        .isGreaterThan(marineY + 0.3); // 1.05 vs 0.455 — margin of 0.3 is safe
}
```

- [ ] **Run all three Medivac tests to confirm they fail**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#medivacDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+medivacEnemySpawnsAndRendersInVisualizer+medivacSpawnsHigherThanGroundUnit" -q
```

Expected: all FAIL.

- [ ] **Add `FLYING_UNITS` constant** to `visualizer.js` after the `TEAM_COLOR_ENEMY` line:

```js
const FLYING_UNITS = new Set(['MEDIVAC']);
```

- [ ] **Update the sprite Y in `syncUnitLayer`** — replace the hardcoded Y on line ~413:

Replace:
```js
      sp.position.set(wp.x, TILE * 0.65, wp.z);
```

With:
```js
      const unitY = FLYING_UNITS.has(u.type) ? TILE * 1.5 : TILE * 0.65;
      sp.position.set(wp.x, unitY, wp.z);
```

Also update the 3D sphere model Y — replace `g.position.set(wp.x, 0, wp.z)` on line ~419:

```js
      g.position.set(wp.x, FLYING_UNITS.has(u.type) ? TILE * 0.85 : 0, wp.z);
```

- [ ] **Add `drawMedivac` function** after `drawMarauder`:

```js
function drawMedivac(ctx, S, dir, teamColor) {
  const cx = S/2, cy = S/2;
  // Hover glow
  const grd = ctx.createRadialGradient(cx,cy,S*.06,cx,cy,S*.46);
  grd.addColorStop(0,'rgba(180,200,220,0.28)'); grd.addColorStop(1,'rgba(0,0,0,0)');
  ctx.fillStyle=grd; ctx.beginPath(); ctx.ellipse(cx,cy,S*.46,S*.46,0,0,Math.PI*2); ctx.fill();

  if (dir === 2) { // BACK — engine pods visible
    // Hull
    ctx.fillStyle='#7788aa'; ctx.beginPath(); ctx.ellipse(cx,cy,S*.36,S*.2,0,0,Math.PI*2); ctx.fill();
    // Engine pods
    ctx.fillStyle='#556688';
    ctx.beginPath(); ctx.ellipse(cx-S*.3,cy+S*.04,S*.1,S*.08,0.3,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(cx+S*.3,cy+S*.04,S*.1,S*.08,-0.3,0,Math.PI*2); ctx.fill();
    // Engine glow in teamColor
    ctx.fillStyle=hexToRgba(teamColor, 0.8);
    ctx.beginPath(); ctx.ellipse(cx-S*.3,cy+S*.09,S*.07,S*.05,0.3,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(cx+S*.3,cy+S*.09,S*.07,S*.05,-0.3,0,Math.PI*2); ctx.fill();
    // Tail fin
    ctx.fillStyle='#445577';
    ctx.fillRect(cx-S*.04,cy-S*.24,S*.08,S*.16);
    // Running lights in teamColor
    ctx.fillStyle=teamColor;
    ctx.beginPath(); ctx.ellipse(cx-S*.36,cy-S*.04,S*.03,S*.03,0,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(cx+S*.36,cy-S*.04,S*.03,S*.03,0,0,Math.PI*2); ctx.fill();
    return;
  }
  if (dir === 1 || dir === 3) { // SIDE — profile view
    const flip = dir===3 ? -1 : 1;
    // Engine pod on far side
    ctx.fillStyle='#445577';
    ctx.beginPath(); ctx.ellipse(cx-flip*S*.28,cy+S*.04,S*.1,S*.08,flip*0.3,0,Math.PI*2); ctx.fill();
    ctx.fillStyle=hexToRgba(teamColor, 0.7);
    ctx.beginPath(); ctx.ellipse(cx-flip*S*.28,cy+S*.09,S*.06,S*.04,flip*0.3,0,Math.PI*2); ctx.fill();
    // Main hull — elongated
    ctx.fillStyle='#8899bb'; ctx.beginPath(); ctx.ellipse(cx,cy-S*.02,S*.4,S*.18,0,0,Math.PI*2); ctx.fill();
    // Red cross (side — one arm visible)
    ctx.fillStyle='rgba(255,60,60,0.9)';
    ctx.fillRect(cx-S*.04,cy-S*.12,S*.08,S*.24);
    ctx.fillRect(cx-S*.1,cy-S*.04,S*.2,S*.08);
    // Cockpit
    ctx.fillStyle='rgba(100,200,255,0.6)';
    ctx.beginPath(); ctx.ellipse(cx+flip*S*.18,cy-S*.04,S*.08,S*.07,0,0,Math.PI*2); ctx.fill();
    // Engine pod near side
    ctx.fillStyle='#556688';
    ctx.beginPath(); ctx.ellipse(cx+flip*S*.3,cy+S*.04,S*.1,S*.08,-flip*0.3,0,Math.PI*2); ctx.fill();
    ctx.fillStyle=hexToRgba(teamColor, 0.75);
    ctx.beginPath(); ctx.ellipse(cx+flip*S*.3,cy+S*.09,S*.07,S*.05,-flip*0.3,0,Math.PI*2); ctx.fill();
    // Running light
    ctx.fillStyle=teamColor;
    ctx.beginPath(); ctx.ellipse(cx+flip*S*.4,cy-S*.04,S*.03,S*.03,0,0,Math.PI*2); ctx.fill();
    return;
  }
  // FRONT — nose view
  // Engine pods
  ctx.fillStyle='#556688';
  ctx.beginPath(); ctx.ellipse(cx-S*.3,cy+S*.06,S*.1,S*.07,0.3,0,Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.ellipse(cx+S*.3,cy+S*.06,S*.1,S*.07,-0.3,0,Math.PI*2); ctx.fill();
  // Engine glow in teamColor
  ctx.fillStyle=hexToRgba(teamColor, 0.8);
  ctx.beginPath(); ctx.ellipse(cx-S*.3,cy+S*.1,S*.06,S*.04,0.3,0,Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.ellipse(cx+S*.3,cy+S*.1,S*.06,S*.04,-0.3,0,Math.PI*2); ctx.fill();
  // Main hull
  ctx.fillStyle='#8899bb'; ctx.beginPath(); ctx.ellipse(cx,cy,S*.36,S*.2,0,0,Math.PI*2); ctx.fill();
  // Red cross — canonical medical cross, never team-coloured
  ctx.fillStyle='rgba(255,60,60,0.9)';
  ctx.fillRect(cx-S*.04,cy-S*.14,S*.08,S*.28);
  ctx.fillRect(cx-S*.12,cy-S*.06,S*.24,S*.08);
  // Cockpit window
  ctx.fillStyle='rgba(100,200,255,0.6)';
  ctx.beginPath(); ctx.ellipse(cx-S*.14,cy-S*.04,S*.08,S*.06,0,0,Math.PI*2); ctx.fill();
  ctx.fillStyle='rgba(200,240,255,0.4)';
  ctx.beginPath(); ctx.ellipse(cx-S*.16,cy-S*.06,S*.03,S*.02,-0.3,0,Math.PI*2); ctx.fill();
  // Running lights in teamColor
  ctx.fillStyle=teamColor;
  ctx.beginPath(); ctx.ellipse(cx-S*.36,cy-S*.02,S*.03,S*.03,0,0,Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.ellipse(cx+S*.36,cy-S*.02,S*.03,S*.03,0,0,Math.PI*2); ctx.fill();
  // Hover ring
  ctx.strokeStyle=hexToRgba(teamColor, 0.35); ctx.lineWidth=2;
  ctx.beginPath(); ctx.ellipse(cx,cy,S*.42,S*.26,0,0,Math.PI*2); ctx.stroke();
}
```

- [ ] **Register MEDIVAC in `initSpriteMaterials`** — add after MARAUDER lines:

```js
  UNIT_MATS['MEDIVAC_F']  = makeDirTextures(drawMedivac,  TEAM_COLOR_FRIENDLY);
  UNIT_MATS['MEDIVAC_E']  = makeDirTextures(drawMedivac,  TEAM_COLOR_ENEMY);
```

- [ ] **Run all three Medivac tests to confirm they pass**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#medivacDrawFunctionProducesNonTransparentOutputForAllDirsAndTeams+medivacEnemySpawnsAndRendersInVisualizer+medivacSpawnsHigherThanGroundUnit" -q
```

Expected: all PASS.

- [ ] **Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat(e15): drawMedivac + FLYING_UNITS Y-offset — engine glow + running lights in teamColor Refs #N Refs #83"
```

---

## Task 8: Config Panel — Add MARAUDER and MEDIVAC to Wave Type Dropdown

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.html`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Write the failing test**

Add to `VisualizerRenderTest.java`:

```java
/**
 * Config panel dropdown must include all three Terran wave types.
 * This is a structural HTML test — not config panel visibility (which needs %emulated).
 * We assert the options exist in the DOM regardless of panel visibility.
 */
@Test
@Tag("browser")
void configPanelDropdownContainsTerranWaveTypes() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    int marineCount = page.locator("#cfg-unit-type option[value='MARINE']").count();
    int marauderCount = page.locator("#cfg-unit-type option[value='MARAUDER']").count();
    int medivacCount = page.locator("#cfg-unit-type option[value='MEDIVAC']").count();

    assertThat(marineCount).as("MARINE option must exist").isEqualTo(1);
    assertThat(marauderCount).as("MARAUDER option must exist").isEqualTo(1);
    assertThat(medivacCount).as("MEDIVAC option must exist").isEqualTo(1);
    page.close();
}
```

- [ ] **Run to confirm it fails**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#configPanelDropdownContainsTerranWaveTypes -q
```

Expected: FAIL — MARAUDER and MEDIVAC options not found.

- [ ] **Update `visualizer.html`** — add the two missing options to the `<select>`:

Replace:
```html
  <select id="cfg-unit-type">
    <option value="ZERGLING">ZERGLING</option>
    <option value="MARINE">MARINE</option>
  </select>
```

With:
```html
  <select id="cfg-unit-type">
    <option value="ZERGLING">ZERGLING</option>
    <option value="MARINE">MARINE</option>
    <option value="MARAUDER">MARAUDER</option>
    <option value="MEDIVAC">MEDIVAC</option>
  </select>
```

- [ ] **Run to confirm it passes**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#configPanelDropdownContainsTerranWaveTypes -q
```

Expected: PASS.

- [ ] **Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.html \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat(e15): add MARAUDER + MEDIVAC to config panel wave type dropdown Refs #N Refs #83"
```

---

## Task 9: Robustness — Unknown Unit Type Fallback

**Files:**
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Write robustness tests**

Add to `VisualizerRenderTest.java`:

```java
/**
 * UNIT_MATS must always include UNKNOWN_F and UNKNOWN_E fallback entries.
 * These are used when an unrecognised unit type is dispatched.
 */
@Test
@Tag("browser")
void unitMatsContainsUnknownFallbacks() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    @SuppressWarnings("unchecked")
    List<String> keys = (List<String>) page.evaluate("() => window.__test.unitMatsKeys()");
    assertThat(keys).contains("UNKNOWN_F", "UNKNOWN_E");
    page.close();
}

/**
 * Robustness: an enemy unit with an unrecognised type must not cause JS errors.
 * It should fall back to UNKNOWN_E and render as a sprite.
 * We inject a fake type via JS eval directly on the WebSocket handler.
 * The simpler proxy: spawn a known type, patch it via JS, verify no error occurs.
 *
 * Implementation: we exploit the fact that the visualizer falls back to UNKNOWN_E
 * for any key not in UNIT_MATS. We verify this by checking JS errors during
 * the full-loop smoke test (which runs 20 ticks), and also by asserting
 * UNIT_MATS.UNKNOWN_E is a 4-element array.
 */
@Test
@Tag("browser")
void unknownUnitTypeFallbackIsA4ElementArray() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    Number len = (Number) page.evaluate(
        "() => (UNIT_MATS['UNKNOWN_E'] ?? []).length");
    assertThat(len.intValue())
        .as("UNKNOWN_E must be a 4-element SpriteMaterial array")
        .isEqualTo(4);
    page.close();
}

/**
 * Coverage: all 7 unit types × 2 teams are registered after init.
 */
@Test
@Tag("browser")
void unitMatsCoversAllSevenTypesWithBothTeams() throws Exception {
    Page page = browser.newPage();
    page.navigate(pageUrl.toString());
    page.waitForFunction("() => window.__test?.threeReady?.() === true",
        null, new Page.WaitForFunctionOptions().setTimeout(8_000));

    @SuppressWarnings("unchecked")
    List<String> keys = (List<String>) page.evaluate("() => window.__test.unitMatsKeys()");
    assertThat(keys).contains(
        "PROBE_F",    "PROBE_E",
        "ZEALOT_F",   "ZEALOT_E",
        "STALKER_F",  "STALKER_E",
        "MARINE_F",   "MARINE_E",
        "MARAUDER_F", "MARAUDER_E",
        "MEDIVAC_F",  "MEDIVAC_E",
        "UNKNOWN_F",  "UNKNOWN_E"
    );
    page.close();
}
```

- [ ] **Run these tests**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#unitMatsContainsUnknownFallbacks+unknownUnitTypeFallbackIsA4ElementArray+unitMatsCoversAllSevenTypesWithBothTeams" -q
```

Expected: all PASS (no code changes needed — this exercises existing implementation from Tasks 3–7).

- [ ] **Commit**

```bash
git add src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "test(e15): robustness + coverage tests — UNKNOWN fallbacks, all 7 types × 2 teams Closes #N Refs #83"
```

---

## Task 10: Full Regression — All Tests Green

- [ ] **Run the complete Playwright suite**

```bash
mvn test -Pplaywright -q
```

Expected: all tests pass (17 existing + ~12 new = ~29 total).

- [ ] **Run the standard test suite** (non-Playwright)

```bash
mvn test -q
```

Expected: all 475+ tests pass.

- [ ] **If any test fails:** investigate before proceeding. Do not skip or `@Disabled` a test without understanding why it failed.

- [ ] **Final commit if any fixes were needed** (otherwise no commit needed here)

```bash
git commit -m "fix(e15): [describe fix] Refs #N Refs #83"
```

---

## Completion Checklist

- [ ] All 14 `UNIT_MATS` keys registered (7 types × 2 teams)
- [ ] `hexToRgba` helper is a pure function at module scope
- [ ] `FLYING_UNITS` set drives Y-offset — adding a new flying unit requires only one line change
- [ ] `drawEnemy` fallback receives teamColor — `UNKNOWN_F` looks friendly-coloured, `UNKNOWN_E` looks hostile
- [ ] Config panel has ZERGLING, MARINE, MARAUDER, MEDIVAC options
- [ ] All Playwright tests use `waitForFunction` — no `Thread.sleep` except the one pre-existing 50ms in `fullLoopRunsWithoutJsErrors`
- [ ] Every commit references `#N` (E15 issue) and `#83` (epic)
- [ ] `mvn test -q` green
- [ ] `mvn test -Pplaywright -q` green
