# Visualizer Deferred Work Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship four deferred visualizer improvements — mineral HUD formatting, probe spread, resource canvas sprites, and time-based UI tests — making the emulated engine showcase-ready.

**Architecture:** All changes are confined to `visualizer.js` (JS rendering), `visualizer.html` (CSS), and `VisualizerRenderTest.java` (Playwright tests). No Java domain model changes, no CDI beans, no server-side state. Implementation order: Task 1 (mineral HUD) → Task 2 (probe spread) → Task 3 (resource sprites) → Task 4 (time tests). Tasks 1 and 2 are prerequisites for Task 4 due to shared `parseMinerals()` usage.

**Tech Stack:** Three.js (visualizer rendering), Playwright + JUnit 5 + AssertJ (Playwright tests), `@QuarkusTest` mock profile, `SimulatedGame` (mock game state).

**Spec:** `docs/superpowers/specs/2026-05-31-visualizer-deferred-design.md`

**Run tests:** `mvn test -Pplaywright`

---

## Key Facts (read before implementing)

- `SimulatedGame.reset()` seeds: 12 probes (tags `probe-0`..`probe-11`), 1 nexus (`nexus-0`), 2 geysers (`geyser-0`, `geyser-1`). No mineral patches — add via `simulatedGame.spawnMineralPatchForTesting(pos, remaining)`.
- `simulatedGame.spawnFriendlyUnitForTesting(type, pos)` assigns tags starting at `"friendly-200"` (nextTag resets to 200 in `reset()`).
- `SimulatedGame.tick()` (called by `engine.tick()`) increments `gameFrame` by 1 and adds exactly 5 minerals per call. `INITIAL_MINERALS = 50`. `INITIAL_PROBES = 12`.
- `engine.observe()` pushes game state to the WebSocket. Always call it after ticks when you need the visualizer to update.
- `SC2Data.LOOPS_PER_TICK = 22`, `SC2Data.GAME_LOOPS_PER_SECOND = 22.4`. `gameFrame` in `SimulatedGame` is the outer tick count (not game loops). `gameTimeSeconds = gameFrame * 22 / 22.4`.
- `window.__test.hudText()` uses `textContent` — strips HTML tags. Returns the plain text of the HUD including all fields.
- `window.__test.sprite("unit:friendly-200")` returns `{x, y, visible, alpha}` where x/y are **rounded world coordinates** (Three.js x/z axes), not screen pixels. Use `window.__test.unitScreenPos(tag)` for screen pixels.
- `window.__test.focusOnFirstGeyser()` and `focusOnFirstMineral()` both exist. They aim the camera and return screen `{x, y}`. Use these before pixel sampling.
- `window.__test.samplePixel(x, y)` returns `{r, g, b, a}` from the WebGL canvas at screen coords.
- Playwright tests are `@Tag("browser")`, excluded from default surefire. Run with `-Pplaywright`.
- `parseMinerals()` is a `private static` method in `VisualizerRenderTest` — currently a char scanner, breaks on commas.

---

## Task 1: Mineral HUD Formatting

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js` — `updateHud()` (~line 968)
- Modify: `src/main/resources/META-INF/resources/visualizer.html` — `<style>` block (~line 6)
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java` — `parseMinerals()` (~line 150), `hudMineralCountIncreasesWithTicks()` (~line 434)

- [ ] **Step 1: Write failing test for comma formatting**

Add this test to `VisualizerRenderTest.java` immediately after `hudTextContainsMinerals()`:

```java
@Test
void mineralHudFormatsLargeValues() {
    Page page = openPage();
    engine.observe();
    page.waitForFunction(
        "() => window.__test.hudText().includes('Minerals:')",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    // Tick 190 times: minerals = 50 + 190*5 = 1000 → should render as "1,000"
    for (int i = 0; i < 190; i++) engine.tick();
    engine.observe();

    page.waitForFunction(
        "() => window.__test.hudText().includes('1,000') || " +
        "      window.__test.hudText().includes('1,0')",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    String hud = (String) page.evaluate("() => window.__test.hudText()");
    assertThat(hud).as("minerals 1000 must be formatted with comma").contains("1,000");
    page.close();
}
```

- [ ] **Step 2: Write failing test for mineral tier CSS class**

Add this test after `mineralHudFormatsLargeValues()`:

```java
@Test
void mineralHudTierClass() {
    Page page = openPage();
    engine.observe();
    page.waitForFunction(
        "() => window.__test.hudText().includes('Minerals:')",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    // INITIAL_MINERALS = 50 → minerals-low tier (50-149)
    String tierAtStart = (String) page.evaluate(
        "() => document.getElementById('minerals-val')?.className ?? 'element-missing'");
    assertThat(tierAtStart).as("minerals=50 should show minerals-low class").isEqualTo("minerals-low");

    // Tick 20 times: minerals = 50 + 100 = 150 → no tier class
    for (int i = 0; i < 20; i++) engine.tick();
    engine.observe();
    page.waitForFunction(
        "() => { const el = document.getElementById('minerals-val'); " +
        "        return el && el.className === ''; }",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    String tierAt150 = (String) page.evaluate(
        "() => document.getElementById('minerals-val')?.className ?? 'element-missing'");
    assertThat(tierAt150).as("minerals=150 should have no tier class").isEmpty();
    page.close();
}
```

- [ ] **Step 3: Run new tests to confirm they fail**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#mineralHudFormatsLargeValues+mineralHudTierClass -q
```

Expected: both FAIL — `minerals-val` element not found, no comma formatting.

- [ ] **Step 4: Rewrite `parseMinerals()` in `VisualizerRenderTest.java`**

Replace lines 150–157 (the entire method body):

```java
/** Extract minerals integer from "Minerals: 1,234   Gas: ..." HUD text. */
private static int parseMinerals(String hud) {
    java.util.regex.Matcher m = java.util.regex.Pattern
        .compile("Minerals:\\s*([\\d,]+)").matcher(hud);
    if (!m.find()) throw new AssertionError("HUD text missing 'Minerals:': " + hud);
    return Integer.parseInt(m.group(1).replace(",", ""));
}
```

- [ ] **Step 5: Fix the inline JS regex in `hudMineralCountIncreasesWithTicks()`**

Find the `waitForFunction` call around line 434 that uses `\d+`. Replace with `[\d,]+` and add comma-stripping in the JS:

```java
page.waitForFunction(
    "() => { const m = window.__test.hudText().match(/Minerals:\\s*([\\d,]+)/); " +
    "        return m && parseInt(m[1].replace(/,/g,'')) > " + threshold + "; }",
    null, new Page.WaitForFunctionOptions().setTimeout(5_000));
```

- [ ] **Step 6: Add CSS to `visualizer.html`**

Insert these three rules inside the existing `<style>` block, after `.mode-btn.active { ... }` (~line 48):

```css
    .minerals-critical { color: #ff4444; }
    .minerals-low      { color: #ffaa00; }
```

(Only two rules needed — the default tier has no class.)

- [ ] **Step 7: Update `updateHud()` in `visualizer.js`**

Replace the existing `updateHud()` function (~lines 967–972):

```js
function updateHud(state) {
  const m = state.minerals ?? 0;
  const tier = m < 50 ? 'minerals-critical' : m < 150 ? 'minerals-low' : '';
  // innerHTML is safe here — state.minerals is a server-computed integer through toLocaleString(),
  // never user-controlled. Do not extend this pattern to user-supplied strings.
  document.getElementById('hud').innerHTML =
    `Minerals: <span id="minerals-val" class="${tier}">${m.toLocaleString('en-US')}</span>` +
    `   Gas: ${state.vespene}` +
    `   Supply: ${state.supplyUsed}/${state.supply}` +
    `   Frame: ${state.gameFrame}`;
}
```

- [ ] **Step 8: Run all Playwright tests**

```bash
mvn test -Pplaywright -q
```

Expected: all tests pass, including the two new tests and the existing `hudMineralCountIncreasesWithTicks`.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/resources/META-INF/resources/visualizer.js \
  src/main/resources/META-INF/resources/visualizer.html \
  src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m \
  "feat(#131): mineral HUD comma formatting and colour tiers"
```

---

## Task 2: Probe Spread

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js` — add `applyUnitSpread()`, call from `syncUnits()`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java` — add spread test

- [ ] **Step 1: Write failing test**

Add this test to `VisualizerRenderTest.java`:

```java
@Test
void probesAtSamePositionAreSpreadApart() {
    Page page = openPage();

    // Spawn 3 probes at exactly the same tile — simulates probes harvesting the same patch.
    // nextTag resets to 200 in reset(), so tags are friendly-200, friendly-201, friendly-202.
    simulatedGame.spawnFriendlyUnitForTesting(UnitType.PROBE, new Point2d(12, 12));
    simulatedGame.spawnFriendlyUnitForTesting(UnitType.PROBE, new Point2d(12, 12));
    simulatedGame.spawnFriendlyUnitForTesting(UnitType.PROBE, new Point2d(12, 12));
    engine.observe();

    page.waitForFunction(
        "() => window.__test.unitCount() >= 15",  // 12 initial + 3 spawned
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    // sprite() returns rounded world x/z — after spread, positions must differ
    @SuppressWarnings("unchecked")
    Map<String, Object> pos0 = (Map<String, Object>) page.evaluate(
        "() => window.__test.sprite('unit:friendly-200')");
    @SuppressWarnings("unchecked")
    Map<String, Object> pos1 = (Map<String, Object>) page.evaluate(
        "() => window.__test.sprite('unit:friendly-201')");
    @SuppressWarnings("unchecked")
    Map<String, Object> pos2 = (Map<String, Object>) page.evaluate(
        "() => window.__test.sprite('unit:friendly-202')");

    assertThat(pos0).as("friendly-200 sprite must exist").isNotNull();
    assertThat(pos1).as("friendly-201 sprite must exist").isNotNull();
    assertThat(pos2).as("friendly-202 sprite must exist").isNotNull();

    // At least two of the three must have different x or y world coords
    boolean allSameX = pos0.get("x").equals(pos1.get("x")) && pos1.get("x").equals(pos2.get("x"));
    boolean allSameY = pos0.get("y").equals(pos1.get("y")) && pos1.get("y").equals(pos2.get("y"));
    assertThat(allSameX && allSameY)
        .as("probes at the same position must be spread — world coords must differ")
        .isFalse();

    page.close();
}
```

- [ ] **Step 2: Run test to confirm it fails**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#probesAtSamePositionAreSpreadApart -q
```

Expected: FAIL — all three sprites return the same x/y (no spread implemented yet).

- [ ] **Step 3: Add `applyUnitSpread()` to `visualizer.js`**

Add this function after `syncUnitLayer()` (~line 1220):

```js
// Spreads unit sprites that are co-located (within TILE*0.5 on both axes) into a
// uniform ring. Operates on rendered 2D sprite positions only — does not mutate
// game state or 3D meshes. Stable within a session (Map insertion order preserved).
// Note: raycaster clicks land on spread positions, not game-model positions — a
// minor footgun if future features need to correlate screen clicks to game coords.
function applyUnitSpread(spriteMap) {
  const THRESHOLD = TILE * 0.5;
  const SPREAD    = TILE * 0.32;

  // Collect all current sprite positions
  const entries = [];
  spriteMap.forEach((sp, tag) => {
    entries.push({ tag, sp, bx: sp.position.x, bz: sp.position.z });
  });

  // Group by proximity: O(n²) over sprites-per-patch — small n in practice
  const assigned = new Set();
  for (let i = 0; i < entries.length; i++) {
    if (assigned.has(entries[i].tag)) continue;
    const cluster = [entries[i]];
    for (let j = i + 1; j < entries.length; j++) {
      if (assigned.has(entries[j].tag)) continue;
      const dx = Math.abs(entries[i].bx - entries[j].bx);
      const dz = Math.abs(entries[i].bz - entries[j].bz);
      if (dx < THRESHOLD && dz < THRESHOLD) cluster.push(entries[j]);
    }
    if (cluster.length > 1) {
      const cx = cluster.reduce((s, e) => s + e.bx, 0) / cluster.length;
      const cz = cluster.reduce((s, e) => s + e.bz, 0) / cluster.length;
      cluster.forEach(({ sp }, idx) => {
        const angle = (idx / cluster.length) * Math.PI * 2;
        sp.position.x = cx + Math.cos(angle) * SPREAD;
        sp.position.z = cz + Math.sin(angle) * SPREAD;
      });
    }
    cluster.forEach(e => assigned.add(e.tag));
  }
}
```

- [ ] **Step 4: Call `applyUnitSpread()` from `syncUnits()`**

In `syncUnits()` (~line 1005), add the call as the last line of the function, after all `syncUnitLayer()` calls:

```js
function syncUnits(state) {
  syncBuildings(state.myBuildings       || []);
  syncEnemyBuildings(state.enemyBuildings || []);
  syncCreep(state.enemyBuildings          || []);
  syncGeysers(state.geysers               || []);
  syncMineralPatches(state.mineralPatches || []);
  syncUnitLayer(unitSprites,   unit3dMeshes,  state.myUnits          || [], false);
  syncUnitLayer(enemySprites,  enemy3dMeshes, state.enemyUnits        || [], true);
  syncUnitLayer(stagingSprites, stagingMeshes, state.enemyStagingArea  || [], true);
  applyUnitSpread(unitSprites);   // spread co-located friendly units (2D only)
}
```

- [ ] **Step 5: Run all Playwright tests**

```bash
mvn test -Pplaywright -q
```

Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/resources/META-INF/resources/visualizer.js \
  src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m \
  "feat(#131): spread co-located unit sprites around patch centroid"
```

---

## Task 3: Resource Canvas Sprites

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js` — add `makeResourceMaterial()`, `drawGeyser()`, `drawMineralPatch()`, update `syncGeysers()`, `syncMineralPatches()`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java` — add pixel assertions

- [ ] **Step 1: Write failing geyser pixel assertion**

Add this test to `VisualizerRenderTest.java`:

```java
@Test
void geyserSpriteRendersGreen() {
    Page page = openPage();
    engine.observe();

    // geysers are seeded by reset() — geyser-0 at tile (5,11), geyser-1 at (11,5)
    page.waitForFunction(
        "() => window.__test.geyserCount() >= 2",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));
    page.waitForFunction(
        "() => window.__test.threeReady()",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    // Aim camera at first geyser and get its screen position
    @SuppressWarnings("unchecked")
    Map<String, Object> pos = (Map<String, Object>) page.evaluate(
        "() => window.__test.focusOnFirstGeyser()");
    assertThat(pos).as("focusOnFirstGeyser must return a position").isNotNull();

    int sx = ((Number) pos.get("x")).intValue();
    int sy = ((Number) pos.get("y")).intValue();

    @SuppressWarnings("unchecked")
    Map<String, Object> pixel = (Map<String, Object>) page.evaluate(
        "() => window.__test.samplePixel(" + sx + ", " + sy + ")");
    assertThat(pixel).as("pixel sample must not be null").isNotNull();

    int r = ((Number) pixel.get("r")).intValue();
    int g = ((Number) pixel.get("g")).intValue();
    int b = ((Number) pixel.get("b")).intValue();

    // Geyser canvas: teal-green radial gradient. Green channel dominates.
    // (r > 10 || b > 10) confirms a gradient rendered — not a degenerate black pixel.
    assertThat(r).as("geyser pixel: R < G (green dominates)").isLessThan(g);
    assertThat(b).as("geyser pixel: B < G (green dominates)").isLessThan(g);
    assertThat(r > 10 || b > 10).as("geyser pixel: not solid-green — gradient has R or B > 10").isTrue();

    page.close();
}
```

- [ ] **Step 2: Write failing mineral patch pixel assertion**

Add this test after `geyserSpriteRendersGreen()`:

```java
@Test
void mineralPatchSpriteRendersBlue() {
    Page page = openPage();

    // reset() seeds no mineral patches — add one explicitly
    simulatedGame.spawnMineralPatchForTesting(new Point2d(10, 5), 1500);
    engine.observe();

    page.waitForFunction(
        "() => window.__test.mineralCount() >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));
    page.waitForFunction(
        "() => window.__test.threeReady()",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    @SuppressWarnings("unchecked")
    Map<String, Object> pos = (Map<String, Object>) page.evaluate(
        "() => window.__test.focusOnFirstMineral()");
    assertThat(pos).as("focusOnFirstMineral must return a position").isNotNull();

    int sx = ((Number) pos.get("x")).intValue();
    int sy = ((Number) pos.get("y")).intValue();

    @SuppressWarnings("unchecked")
    Map<String, Object> pixel = (Map<String, Object>) page.evaluate(
        "() => window.__test.samplePixel(" + sx + ", " + sy + ")");
    assertThat(pixel).as("pixel sample must not be null").isNotNull();

    int r = ((Number) pixel.get("r")).intValue();
    int g = ((Number) pixel.get("g")).intValue();
    int b = ((Number) pixel.get("b")).intValue();

    // Mineral patch canvas: blue radial gradient. Blue channel dominates.
    // (r > 10 || g > 10) confirms gradient rendered — not a degenerate black pixel.
    assertThat(b).as("mineral pixel: B > R (blue dominates)").isGreaterThan(r);
    assertThat(b).as("mineral pixel: B > G (blue dominates)").isGreaterThan(g);
    assertThat(r > 10 || g > 10).as("mineral pixel: not solid-blue — gradient has R or G > 10").isTrue();

    page.close();
}
```

- [ ] **Step 3: Run new tests to confirm they fail**

```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#geyserSpriteRendersGreen+mineralPatchSpriteRendersBlue" -q
```

Expected: both FAIL — the solid-color sprites likely fail the `(r > 10 || b > 10)` / `(r > 10 || g > 10)` gradient guards. If they happen to pass for the current solid-color materials, note this and proceed — the implementation still needs to happen for the canvas sprite.

- [ ] **Step 4: Add canvas draw functions and `makeResourceMaterial()` to `visualizer.js`**

Add these functions after the `RESOURCE_SPRITE_SIZE` constant (~line 1040):

```js
function drawGeyser(ctx, S) {
  const cx = S / 2, cy = S / 2;
  // Base: teal-green radial gradient
  const grd = ctx.createRadialGradient(cx, cy, S * 0.02, cx, cy, S * 0.4);
  grd.addColorStop(0, '#00cc88');
  grd.addColorStop(1, '#007755');
  ctx.fillStyle = grd;
  ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.4, S * 0.4, 0, 0, Math.PI * 2); ctx.fill();
  // Overlay: three concentric rings — gas-vent suggestion
  [0.30, 0.50, 0.70].forEach(frac => {
    ctx.strokeStyle = 'rgba(180,255,220,0.5)';
    ctx.lineWidth = 1;
    ctx.beginPath(); ctx.ellipse(cx, cy, S * frac * 0.4, S * frac * 0.4, 0, 0, Math.PI * 2); ctx.stroke();
  });
}

function drawMineralPatch(ctx, S) {
  const cx = S / 2, cy = S / 2;
  // Base: blue radial gradient (wider than tall — mineral deposit shape)
  const grd = ctx.createRadialGradient(cx, cy, S * 0.02, cx, cy, S * 0.45);
  grd.addColorStop(0, '#66ccff');
  grd.addColorStop(1, '#2277aa');
  ctx.fillStyle = grd;
  ctx.beginPath(); ctx.ellipse(cx, cy, S * 0.45, S * 0.28, 0, 0, Math.PI * 2); ctx.fill();
  // Overlay: two horizontal crystal-vein streaks
  ctx.strokeStyle = 'rgba(180,230,255,0.4)';
  ctx.lineWidth = 2;
  [-S * 0.06, S * 0.06].forEach(dy => {
    ctx.beginPath(); ctx.moveTo(cx - S * 0.3, cy + dy); ctx.lineTo(cx + S * 0.3, cy + dy); ctx.stroke();
  });
}

// Creates a canvas-texture SpriteMaterial for a resource type.
// Called once at startup; result stored as a module-level constant.
function makeResourceMaterial(type) {
  const c = document.createElement('canvas');
  c.width = c.height = 64;
  const ctx = c.getContext('2d');
  if (type === 'geyser')  drawGeyser(ctx, 64);
  else                    drawMineralPatch(ctx, 64);
  const tex = new THREE.CanvasTexture(c);
  return new THREE.SpriteMaterial({ map: tex, transparent: true, depthWrite: false, alphaTest: 0.05 });
}

const GEYSER_MAT  = makeResourceMaterial('geyser');
const MINERAL_MAT = makeResourceMaterial('mineral');
```

- [ ] **Step 5: Update `syncGeysers()` to use `GEYSER_MAT`**

In `syncGeysers()` (~line 1041), replace the `new THREE.Sprite(new THREE.SpriteMaterial(...))` block with:

```js
function syncGeysers(geysers) {
  const seen = new Set();
  geysers.forEach(g => {
    seen.add(g.tag);
    if (!geyserMeshes.has(g.tag)) {
      const s = RESOURCE_SPRITE_SIZE();
      const sp = new THREE.Sprite(GEYSER_MAT.clone());
      sp.scale.set(s * 0.9, s * 1.1, 1);   // slightly taller than square — column-like vent
      const wp = gw(g.position.x, g.position.y);
      sp.position.set(wp.x, TERRAIN_SURFACE_Y + s * 0.55, wp.z);
      scene.add(sp);
      geyserMeshes.set(g.tag, sp);
    }
  });
  geyserMeshes.forEach((m, tag) => {
    if (!seen.has(tag)) { scene.remove(m); geyserMeshes.delete(tag); }
  });
}
```

- [ ] **Step 6: Update `syncMineralPatches()` to use `MINERAL_MAT`**

In `syncMineralPatches()` (~line 1062), replace the inner block with:

```js
function syncMineralPatches(patches) {
  const seen = new Set();
  patches.forEach(p => {
    seen.add(p.tag);
    if (!mineralMeshes.has(p.tag)) {
      const s = RESOURCE_SPRITE_SIZE();
      const sp = new THREE.Sprite(MINERAL_MAT.clone());
      sp.scale.set(s * 1.4, s * 0.8, 1);   // wider than tall — mineral patch shape
      const wp = gw(p.position.x, p.position.y);
      sp.position.set(wp.x, TERRAIN_SURFACE_Y + s * 0.4, wp.z);
      scene.add(sp);
      mineralMeshes.set(p.tag, sp);
    }
  });
  mineralMeshes.forEach((m, tag) => {
    if (!seen.has(tag)) { scene.remove(m); mineralMeshes.delete(tag); }
  });
}
```

Note: `.clone()` on the material is used so each sprite gets its own material instance (needed if per-sprite opacity or colour changes are ever added later). The canvas texture is shared by reference.

- [ ] **Step 7: Run all Playwright tests**

```bash
mvn test -Pplaywright -q
```

Expected: all tests pass including both new pixel assertions.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/resources/META-INF/resources/visualizer.js \
  src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m \
  "feat(#131): canvas sprites for geysers and mineral patches via makeResourceMaterial()"
```

---

## Task 4: Time-Based UI Tests

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js` — add `gameTimeSeconds` to `window.__test`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java` — add `tickForSeconds()` helper and `mineralIncomeScalesWithGameTime` test

**Prerequisite:** Task 1 must be complete — `parseMinerals()` must handle commas before this test runs at scale.

- [ ] **Step 1: Write the failing test**

Add this test and helper to `VisualizerRenderTest.java`:

```java
/**
 * Advances the engine by enough outer ticks to cover the requested game-time seconds.
 * One outer tick = SC2Data.LOOPS_PER_TICK (22) game loops.
 * Calls engine.observe() after the loop so the visualizer receives updated state.
 */
private void tickForSeconds(double seconds) {
    int ticks = (int) Math.ceil(
        seconds * SC2Data.GAME_LOOPS_PER_SECOND / SC2Data.LOOPS_PER_TICK);
    for (int i = 0; i < ticks; i++) engine.tick();
    engine.observe();
}

/**
 * Verifies mineral income expressed in game-time seconds — not tick counts.
 * gameTimeSeconds = gameFrame * LOOPS_PER_TICK / GAME_LOOPS_PER_SECOND.
 * After 5 game-seconds (6 outer ticks), gameTimeSeconds ≈ 5.89.
 * SimulatedGame.tick() adds exactly 5 minerals per tick.
 */
@Test
void mineralIncomeScalesWithGameTime() {
    Page page = openPage();
    engine.observe();
    page.waitForFunction(
        "() => window.__test.hudText().includes('Minerals:')",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    tickForSeconds(5);  // ceil(5 * 22.4 / 22) = 6 ticks → gameFrame = 6 → ~5.89 game-seconds

    // Verify the JS time accessor reports ≥ 4.5 seconds (expected ~5.89)
    page.waitForFunction(
        "() => window.__test.gameTimeSeconds() >= 4.5",
        null, new Page.WaitForFunctionOptions().setTimeout(3_000));

    double reportedSeconds = ((Number) page.evaluate(
        "() => window.__test.gameTimeSeconds()")).doubleValue();
    assertThat(reportedSeconds)
        .as("gameTimeSeconds after 6 ticks must be ≥ 4.5")
        .isGreaterThanOrEqualTo(4.5);

    // Verify mineral income: 6 ticks × 5 minerals/tick + INITIAL_MINERALS = 80
    int minerals = parseMinerals((String) page.evaluate("() => window.__test.hudText()"));
    int ticks = (int) Math.ceil(5 * SC2Data.GAME_LOOPS_PER_SECOND / SC2Data.LOOPS_PER_TICK); // 6
    int expectedFloor = SC2Data.INITIAL_MINERALS + ticks * 5; // 50 + 30 = 80
    assertThat(minerals)
        .as("minerals after 5 game-seconds must be >= " + expectedFloor)
        .isGreaterThanOrEqualTo(expectedFloor);

    page.close();
}
```

- [ ] **Step 2: Run test to confirm it fails**

```bash
mvn test -Pplaywright -Dtest=VisualizerRenderTest#mineralIncomeScalesWithGameTime -q
```

Expected: FAIL — `window.__test.gameTimeSeconds` is not defined yet.

- [ ] **Step 3: Add `gameTimeSeconds` to `window.__test` in `visualizer.js`**

In the `window.__test` object (~line 65), add after `wsConnected`:

```js
  // gameFrame in SimulatedGame is the outer tick count (SC2Data.LOOPS_PER_TICK = 22 loops/tick).
  // Multiply by 22 before dividing by 22.4 (SC2Data.GAME_LOOPS_PER_SECOND — Faster speed).
  gameTimeSeconds: () => ((state?.gameFrame ?? 0) * 22) / 22.4,
```

- [ ] **Step 4: Run all Playwright tests**

```bash
mvn test -Pplaywright -q
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/resources/META-INF/resources/visualizer.js \
  src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m \
  "feat(#131): gameTimeSeconds API and time-parameterised mineral income test"
```

---

## Self-Review Checklist

- [x] Sub-task 2 (mineral HUD): `parseMinerals()` rewrite ✓, CSS ✓, `updateHud()` template ✓, existing test regex fix ✓, `minerals-val` span ✓
- [x] Sub-task 1 (probe spread): axis-aligned proximity clustering ✓, `TILE*0.32` spread ✓, 2D-only (3D intentional gap documented) ✓
- [x] Sub-task 3 (resource sprites): `makeResourceMaterial()` factory ✓, `drawGeyser()` ✓, `drawMineralPatch()` ✓, pixel assertions with gradient guard ✓
- [x] Sub-task 4 (time tests): correct formula `ceil(s * 22.4 / 22)` ✓, `gameTimeSeconds = gameFrame * 22 / 22.4` ✓, `observe()` called in helper ✓, income uses `ticks * 5` (SimulatedGame mock rate, not SC2Data formula) ✓
- [x] Implementation order: Task 1 before Task 4 ✓
- [x] No TBDs or placeholders ✓
- [x] Types/method names consistent across tasks ✓
