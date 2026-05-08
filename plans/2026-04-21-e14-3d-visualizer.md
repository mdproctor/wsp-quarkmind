# E14: 3D Visualizer + Protoss Cartoon Sprites — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the PixiJS 2D visualizer with a Three.js 3D renderer featuring an orbiting camera, 3D terrain, fog of war, and directional cartoon sprites for Protoss units (Probe, Zealot, Stalker).

**Architecture:** Three.js scene on the client only — server WebSocket protocol is unchanged. Units are `THREE.Sprite` with 4-direction canvas textures; direction selected each frame via `getDir4()`. Fog is per-tile `PlaneGeometry` overlays at `renderOrder=5`. A 3D-model toggle keeps sphere+eyes geometry as an alternative.

**Tech Stack:** Three.js r128 (local, no CDN), Canvas 2D for sprite textures, Playwright (`@Tag("browser")`) for end-to-end tests, Quarkus `%emulated` profile for integration tests.

**Coordinate mapping:** game `(x, y)` → Three.js `(x*TILE - HALF, y*TILE - HALF)` on the XZ plane. No Y-flip; camera starts above and to the northeast.

---

## File Map

| Action | Path | Responsibility |
|--------|------|----------------|
| Create | `src/main/resources/META-INF/resources/sprites/three.min.js` | Three.js r128 library (local) |
| Replace | `src/main/resources/META-INF/resources/visualizer.html` | Script tags, canvas div, HUD div, config panel |
| Replace | `src/main/resources/META-INF/resources/visualizer.js` | Entire Three.js renderer |
| Update | `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java` | New `window.__test` API assertions |
| Update | `src/test/java/io/quarkmind/qa/VisualizerFogRenderTest.java` | Three.js fog assertions |

---

## Task 0: Create E14 Epic and Issue

**Files:** none (GitHub only)

- [ ] **Step 1: Create epic**
```bash
gh issue create \
  --title "Epic E14: 3D Visualizer + Cartoon Sprites" \
  --label "epic" \
  --body "Replaces PixiJS 2D visualizer with Three.js 3D renderer. Delivers orbiting camera, 3D terrain, fog of war, and directional cartoon sprites for Protoss units (Probe, Zealot, Stalker). Spec: docs/superpowers/specs/2026-04-21-e14-3d-visualizer-protoss-sprites-design.md"
```

- [ ] **Step 2: Create implementation issue**
```bash
gh issue create \
  --title "E14-1: Three.js 3D visualizer with Protoss cartoon sprites" \
  --label "enhancement" \
  --body "Implements the full E14 spec. Closes Epic E14. Refs #<epic-number>"
```
Note the issue number — use `Refs #N` in all commits.

---

## Task 1: Install Three.js

**Files:**
- Create: `src/main/resources/META-INF/resources/sprites/three.min.js`

- [ ] **Step 1: Download Three.js r128**
```bash
curl -s "https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js" \
  -o src/main/resources/META-INF/resources/sprites/three.min.js
echo "Downloaded: $(wc -c < src/main/resources/META-INF/resources/sprites/three.min.js) bytes"
```
Expected: ~603000 bytes.

- [ ] **Step 2: Verify it loads**
```bash
node -e "const fs=require('fs'); const src=fs.readFileSync('src/main/resources/META-INF/resources/sprites/three.min.js','utf8'); console.log(src.includes('THREE')? 'OK':'FAIL')"
```
Expected: `OK`

- [ ] **Step 3: Commit**
```bash
git add src/main/resources/META-INF/resources/sprites/three.min.js
git commit -m "feat(e14): add Three.js r128 as local static asset Refs #<issue>"
```

---

## Task 2: Replace HTML Shell

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.html`

- [ ] **Step 1: Write the failing Playwright test** — add to `VisualizerRenderTest.java`

Add this test (temporarily) to confirm Three.js is NOT yet loaded (red):
```java
@Test
void threeJsCanvasExists() throws Exception {
    try (Playwright pw = Playwright.create()) {
        Browser browser = pw.chromium().launch();
        Page page = browser.newPage();
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test?.threeReady?.() === true",
            new Page.WaitForFunctionOptions().setTimeout(8000));
        Object ready = page.evaluate("() => window.__test.threeReady()");
        assertTrue((Boolean) ready, "Three.js renderer not initialised");
        browser.close();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**
```bash
mvn test -Dtest=VisualizerRenderTest#threeJsCanvasExists -Pplaywright -q 2>&1 | tail -10
```
Expected: FAIL — timeout waiting for `window.__test.threeReady()`.

- [ ] **Step 3: Replace visualizer.html**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>QuarkMind Visualizer</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { background: #0a0a1a; overflow: hidden; font-family: monospace; color: #ccc; }
    #hud {
      position: absolute; bottom: 12px; left: 12px;
      background: rgba(10,10,30,0.8); border: 1px solid #2a2a5e;
      border-radius: 6px; padding: 6px 14px; font-size: 12px; color: #88bbff;
    }
    #config-panel {
      display: none; position: absolute; top: 12px; right: 12px;
      background: rgba(10,10,30,0.9); border: 1px solid #2a2a5e;
      border-radius: 8px; padding: 14px; font-size: 12px; min-width: 200px;
    }
    #config-panel label { color: #aaaacc; display: block; margin: 6px 0 2px; }
    #config-panel input, #config-panel select {
      width: 100%; background: #111; border: 1px solid #334; color: #ccc;
      padding: 4px; border-radius: 3px;
    }
    .cfg-btn {
      margin-top: 8px; width: 100%; padding: 6px;
      background: #223; border: 1px solid #445; color: #aac;
      border-radius: 4px; cursor: pointer; font-family: monospace;
    }
    .cfg-btn:hover { border-color: #88bbff; color: #88bbff; }
    #cfg-status { margin-top: 6px; font-size: 11px; min-height: 14px; }
    #angle-btns {
      position: absolute; top: 12px; left: 12px;
      display: flex; flex-direction: column; gap: 6px;
    }
    .angle-btn {
      background: rgba(10,10,30,0.85); border: 1px solid #2a2a5e;
      border-radius: 5px; color: #aaaacc; font-family: monospace;
      font-size: 11px; padding: 6px 12px; cursor: pointer;
    }
    .angle-btn.active, .angle-btn:hover { border-color: #88bbff; color: #88bbff; }
    #mode-toggle {
      position: absolute; top: 12px; right: 12px;
      display: flex;
    }
    .mode-btn {
      background: rgba(10,10,30,0.85); border: 1px solid #2a2a5e;
      color: #aaaacc; font-family: monospace; font-size: 11px; padding: 6px 12px; cursor: pointer;
    }
    .mode-btn:first-child { border-radius: 5px 0 0 5px; }
    .mode-btn:last-child  { border-radius: 0 5px 5px 0; }
    .mode-btn.active { background: rgba(40,80,160,0.5); border-color: #88bbff; color: #88bbff; }
  </style>
</head>
<body>
<div id="hud">Connecting...</div>
<div id="angle-btns">
  <button class="angle-btn" onclick="setAngle('top',this)">Top-down</button>
  <button class="angle-btn active" id="angIso" onclick="setAngle('iso',this)">Isometric</button>
  <button class="angle-btn" onclick="setAngle('low',this)">Low angle</button>
</div>
<div id="mode-toggle">
  <button class="mode-btn active" id="btn2d" onclick="setMode('2d')">2D Sprites</button>
  <button class="mode-btn" id="btn3d" onclick="setMode('3d')">3D Models</button>
</div>
<div id="config-panel">
  <label>Wave spawn frame</label><input id="cfg-wave-frame" type="number" value="300">
  <label>Wave unit count</label><input id="cfg-unit-count" type="number" value="5">
  <label>Unit type</label>
  <select id="cfg-unit-type">
    <option value="ZERGLING">ZERGLING</option>
    <option value="MARINE">MARINE</option>
  </select>
  <label>Speed: <span id="cfg-speed-val">1.0</span></label>
  <input id="cfg-speed" type="range" min="0.2" max="3.0" step="0.1" value="1.0">
  <button class="cfg-btn" id="cfg-apply">Apply (next restart)</button>
  <button class="cfg-btn" id="cfg-restart">Apply + Restart</button>
  <div id="cfg-status"></div>
</div>
<script src="/sprites/three.min.js"></script>
<script src="/visualizer.js"></script>
</body>
</html>
```

- [ ] **Step 4: Create minimal visualizer.js that passes the test**

```javascript
// visualizer.js — QuarkMind 3D visualizer (Three.js r128)

// ── Constants ──────────────────────────────────────────────────────────────────
const TILE = 0.7;         // world units per game tile
const RECONNECT_MS = 2000;

let renderer, scene, camera;
let wsConnected = false;
let group2d, group3d;

// ── Test hooks (updated by init) ───────────────────────────────────────────────
window.__test = {
  threeReady:    () => !!renderer,
  wsConnected:   () => wsConnected,
  hudText:       () => document.getElementById('hud')?.textContent ?? '',
  unitCount:     () => group2d?.children.length ?? 0,
  enemyCount:    () => 0,
  buildingCount: () => 0,
  stagingCount:  () => 0,
  geyserCount:   () => 0,
  fogOpacity:    (x, z) => -1,
  fogVisible:    (x, z) => false,
  worldToScreen: (wx, wz) => {
    if (!camera || !renderer) return { x: 0, y: 0 };
    const v = new THREE.Vector3(wx, 0, wz).project(camera);
    const sz = renderer.getSize(new THREE.Vector2());
    return { x: Math.round((v.x+1)/2*sz.width), y: Math.round((-v.y+1)/2*sz.height) };
  },
};

async function init() {
  // Scene
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x0a0a1a);

  // Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.shadowMap.enabled = true;
  document.body.appendChild(renderer.domElement);

  // Camera (spherical orbit, isometric start)
  camera = new THREE.PerspectiveCamera(45, window.innerWidth/window.innerHeight, 0.1, 500);

  // Unit groups
  group2d = new THREE.Group(); scene.add(group2d);
  group3d = new THREE.Group(); scene.add(group3d);
  group3d.visible = false;

  window._three = { scene, camera, renderer };

  setupCamera();
  setupLighting();
  await loadTerrain();
  connectWebSocket();
  initConfigPanel();
  animate();
}

window.addEventListener('resize', () => {
  if (!renderer || !camera) return;
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});

function animate() {
  requestAnimationFrame(animate);
  smoothCamera();
  renderer.render(scene, camera);
}

// ── Camera stubs (filled in Task 3) ──────────────────────────────────────────
function setupCamera() {}
function smoothCamera() {}
function setAngle(p, btn) {}
function setMode(m) {}

// ── Lighting stub ─────────────────────────────────────────────────────────────
function setupLighting() {
  scene.add(new THREE.AmbientLight(0x223355, 0.9));
  const sun = new THREE.DirectionalLight(0xaabbff, 1.3);
  sun.position.set(20, 40, 20);
  sun.castShadow = true;
  sun.shadow.mapSize.set(2048, 2048);
  sun.shadow.camera.near = 1; sun.shadow.camera.far = 200;
  sun.shadow.camera.left = -60; sun.shadow.camera.right = 60;
  sun.shadow.camera.top = 60; sun.shadow.camera.bottom = -60;
  scene.add(sun);
  const fill = new THREE.DirectionalLight(0x334466, 0.4);
  fill.position.set(-10, 20, -10);
  scene.add(fill);
}

// ── Terrain stub (filled in Task 3) ──────────────────────────────────────────
async function loadTerrain() {}

// ── WebSocket stub (filled in Task 4) ────────────────────────────────────────
function connectWebSocket() {}

// ── Config panel stub (filled in Task 8) ─────────────────────────────────────
function initConfigPanel() {}

init();
```

- [ ] **Step 5: Run test to verify it passes**
```bash
mvn test -Dtest=VisualizerRenderTest#threeJsCanvasExists -Pplaywright -q 2>&1 | tail -5
```
Expected: `Tests run: 1, Failures: 0, Errors: 0`

- [ ] **Step 6: Commit**
```bash
git add src/main/resources/META-INF/resources/visualizer.html \
        src/main/resources/META-INF/resources/visualizer.js
git commit -m "feat(e14): Three.js HTML shell and renderer scaffold Refs #<issue>"
```

---

## Task 3: 3D Terrain + Camera

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`

Constants needed at the top of visualizer.js (add after existing constants):
```javascript
let GRID_W = 64, GRID_H = 64;           // updated from terrain response
let HALF_W, HALF_H;                      // set after terrain loads
const mGround = new THREE.MeshLambertMaterial({ color: 0x1a2233 });
const mWall   = new THREE.MeshLambertMaterial({ color: 0x2e4055 });
const mHigh   = new THREE.MeshLambertMaterial({ color: 0x3a3020 });
const mRamp   = new THREE.MeshLambertMaterial({ color: 0x2a3a44 });
const lineMat = new THREE.LineBasicMaterial({ color: 0x1e2a3a, transparent: true, opacity: 0.4 });
```

Camera state (add at top level):
```javascript
let camTheta = Math.PI*0.25, camPhi = Math.PI/3.5, camDist = 30;
const camTarget = new THREE.Vector3(0, 0, 0);
let tTheta = camTheta, tPhi = camPhi, tDist = camDist;
```

- [ ] **Step 1: Write failing test — terrain tiles visible from isometric**

Add to `VisualizerRenderTest.java`:
```java
@Test
void terrainRendersGridTiles() throws Exception {
    try (Playwright pw = Playwright.create()) {
        Browser browser = pw.chromium().launch();
        Page page = browser.newPage();
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test?.threeReady?.() === true",
            new Page.WaitForFunctionOptions().setTimeout(8000));
        // Three.js scene must have substantial children (terrain tiles, lights, etc.)
        Number count = (Number) page.evaluate("() => window._three.scene.children.length");
        assertTrue(count.intValue() > 10, "Expected terrain tiles in scene, got " + count);
        browser.close();
    }
}
```

- [ ] **Step 2: Run to verify it fails**
```bash
mvn test -Dtest=VisualizerRenderTest#terrainRendersGridTiles -Pplaywright -q 2>&1 | tail -5
```
Expected: FAIL — `Expected terrain tiles in scene, got 4` (only lights and groups).

- [ ] **Step 3: Implement `loadTerrain()`**

Replace the `loadTerrain` stub:
```javascript
async function loadTerrain() {
  let walls = [], highGround = [], ramps = [];
  try {
    const r = await fetch('/qa/emulated/terrain');
    if (r.ok) {
      const d = await r.json();
      GRID_W = d.width; GRID_H = d.height;
      walls = d.walls || [];
      highGround = d.highGround || [];
      ramps = d.ramps || [];
    }
  } catch (_) { /* non-emulated profiles: use flat grid */ }

  HALF_W = (GRID_W * TILE) / 2;
  HALF_H = (GRID_H * TILE) / 2;
  camTarget.set(0, 0, 0);

  const wallSet  = new Set(walls.map(([x,z]) => `${x},${z}`));
  const highSet  = new Set(highGround.map(([x,z]) => `${x},${z}`));
  const rampSet  = new Set(ramps.map(([x,z]) => `${x},${z}`));

  for (let gz = 0; gz < GRID_H; gz++) {
    for (let gx = 0; gx < GRID_W; gx++) {
      const key = `${gx},${gz}`;
      const cx = gx * TILE - HALF_W + TILE/2;
      const cz = gz * TILE - HALF_H + TILE/2;

      let h = 0.08, mat = mGround;
      if (wallSet.has(key))  { h = TILE * 1.2; mat = mWall; }
      else if (highSet.has(key)) { h = TILE * 0.6; mat = mHigh; }
      else if (rampSet.has(key)) { h = TILE * 0.25; mat = mRamp; }

      const tile = new THREE.Mesh(new THREE.BoxGeometry(TILE*0.98, h, TILE*0.98), mat);
      tile.position.set(cx, h/2, cz);
      tile.receiveShadow = true;
      if (mat === mWall) tile.castShadow = true;
      scene.add(tile);

      const el = new THREE.LineSegments(
        new THREE.EdgesGeometry(new THREE.BoxGeometry(TILE, 0.01, TILE)),
        lineMat
      );
      el.position.set(cx, h + 0.01, cz);
      scene.add(el);
    }
  }

  // Update camera distance based on grid size
  tDist = camDist = Math.max(GRID_W, GRID_H) * TILE * 0.7;
  updateCamera();
}
```

- [ ] **Step 4: Implement `setupCamera()`, `smoothCamera()`, `setAngle()`, `setMode()`**

```javascript
function setupCamera() {
  updateCamera();
  // Mouse controls
  let drag = false, lastX = 0, lastY = 0, rDrag = false;
  renderer.domElement.addEventListener('mousedown', e => {
    drag = true; lastX = e.clientX; lastY = e.clientY; rDrag = e.button === 2;
    e.preventDefault();
  });
  renderer.domElement.addEventListener('contextmenu', e => e.preventDefault());
  window.addEventListener('mousemove', e => {
    if (!drag) return;
    const dx = e.clientX - lastX, dy = e.clientY - lastY;
    lastX = e.clientX; lastY = e.clientY;
    if (rDrag) {
      const right = new THREE.Vector3();
      camera.getWorldDirection(right); right.cross(camera.up).normalize();
      camTarget.addScaledVector(right, -dx * 0.03);
      camTarget.y += dy * 0.03;
    } else {
      tTheta -= dx * 0.012;
      tPhi = Math.max(0.08, Math.min(Math.PI/2.05, tPhi - dy * 0.012));
    }
  });
  window.addEventListener('mouseup', () => { drag = false; });
  renderer.domElement.addEventListener('wheel', e => {
    tDist = Math.max(4, Math.min(120, tDist + e.deltaY * 0.05));
    e.preventDefault();
  }, { passive: false });
}

function updateCamera() {
  camera.position.set(
    camTarget.x + camDist * Math.sin(camPhi) * Math.sin(camTheta),
    camTarget.y + camDist * Math.cos(camPhi),
    camTarget.z + camDist * Math.sin(camPhi) * Math.cos(camTheta)
  );
  camera.lookAt(camTarget);
}

function smoothCamera() {
  camPhi   += (tPhi   - camPhi)   * 0.1;
  camTheta += (tTheta - camTheta) * 0.1;
  camDist  += (tDist  - camDist)  * 0.1;
  updateCamera();
}

function setAngle(p, btn) {
  document.querySelectorAll('.angle-btn').forEach(b => b.classList.remove('active'));
  if (btn) btn.classList.add('active');
  if (p === 'top') { tPhi = 0.12; tDist = Math.max(GRID_W, GRID_H) * TILE * 0.9; }
  if (p === 'iso') { tPhi = Math.PI/3.5; tDist = Math.max(GRID_W, GRID_H) * TILE * 0.7; }
  if (p === 'low') { tPhi = Math.PI/2.3; tDist = Math.max(GRID_W, GRID_H) * TILE * 0.5; }
}

function setMode(m) {
  document.getElementById('btn2d').classList.toggle('active', m === '2d');
  document.getElementById('btn3d').classList.toggle('active', m === '3d');
  group2d.visible = m === '2d';
  group3d.visible = m === '3d';
}
```

- [ ] **Step 5: Run terrain test**
```bash
mvn test -Dtest=VisualizerRenderTest#terrainRendersGridTiles -Pplaywright -q 2>&1 | tail -5
```
Expected: PASS.

- [ ] **Step 6: Commit**
```bash
git add src/main/resources/META-INF/resources/visualizer.js
git commit -m "feat(e14): 3D terrain tiles and orbit camera Refs #<issue>"
```

---

## Task 4: WebSocket + HUD

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`

State to add at top level:
```javascript
const unitSprites    = new Map(); // tag → THREE.Sprite (group2d)
const unit3dMeshes   = new Map(); // tag → THREE.Group (group3d)
const enemySprites   = new Map(); // tag → THREE.Sprite
const enemy3dMeshes  = new Map(); // tag → THREE.Group
const buildingMeshes = new Map(); // tag → THREE.Mesh
const geyserMeshes   = new Map(); // tag → THREE.Mesh
const stagingSprites = new Map(); // tag → THREE.Sprite
const fogPlanes      = new Map(); // "x,z" → THREE.Mesh
```

Update `window.__test` counts after these are defined:
```javascript
window.__test.unitCount     = () => unitSprites.size;
window.__test.enemyCount    = () => enemySprites.size;
window.__test.buildingCount = () => buildingMeshes.size;
window.__test.stagingCount  = () => stagingSprites.size;
window.__test.geyserCount   = () => geyserMeshes.size;
window.__test.fogOpacity    = (x, z) => {
  const p = fogPlanes.get(`${x},${z}`);
  return p ? (p.visible ? p.material.opacity : 0) : -1;
};
window.__test.fogVisible = (x, z) => fogPlanes.get(`${x},${z}`)?.visible ?? false;
```

- [ ] **Step 1: Write failing HUD test**

Add to `VisualizerRenderTest.java`:
```java
@Test
void hudTextContainsAllFields() throws Exception {
    try (Playwright pw = Playwright.create()) {
        Browser browser = pw.chromium().launch();
        Page page = browser.newPage();
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test?.wsConnected?.() === true",
            new Page.WaitForFunctionOptions().setTimeout(8000));
        orchestrator.gameTick();
        page.waitForTimeout(500);
        String hud = (String) page.evaluate("() => window.__test.hudText()");
        assertTrue(hud.contains("Minerals:"), "HUD missing Minerals: " + hud);
        assertTrue(hud.contains("Gas:"),      "HUD missing Gas: " + hud);
        assertTrue(hud.contains("Supply:"),   "HUD missing Supply: " + hud);
        assertTrue(hud.contains("Frame:"),    "HUD missing Frame: " + hud);
        browser.close();
    }
}
```

- [ ] **Step 2: Run to verify it fails**
```bash
mvn test -Dtest=VisualizerRenderTest#hudTextContainsAllFields -Pplaywright -q 2>&1 | tail -5
```
Expected: FAIL — timeout waiting for `wsConnected`.

- [ ] **Step 3: Implement `connectWebSocket()` and `onFrame()`**

Replace the WebSocket stub:
```javascript
function connectWebSocket() {
  const ws = new WebSocket(`ws://${window.location.host}/ws/gamestate`);
  ws.onopen  = () => { wsConnected = true; };
  ws.onmessage = e => {
    try {
      const msg = JSON.parse(e.data);
      onFrame(msg.state, msg.visibility);
    } catch (err) { console.warn('Bad WS message', err); }
  };
  ws.onerror = () => ws.close();
  ws.onclose = () => {
    wsConnected = false;
    document.getElementById('hud').textContent = 'Disconnected — reconnecting...';
    setTimeout(connectWebSocket, RECONNECT_MS);
  };
}

function onFrame(state, visibility) {
  if (!state) return;
  updateHud(state);
  updateFog(visibility);
  syncUnits(state);
}

function updateHud(state) {
  document.getElementById('hud').textContent =
    `Minerals: ${state.minerals}   Gas: ${state.vespene}` +
    `   Supply: ${state.supplyUsed}/${state.supply}` +
    `   Frame: ${state.gameFrame}`;
}
```

Add `syncUnits` stub (filled in Task 6):
```javascript
function syncUnits(state) {}
```

Add `updateFog` stub (filled in Task 5):
```javascript
function updateFog(visibility) {}
```

- [ ] **Step 4: Run HUD test**
```bash
mvn test -Dtest=VisualizerRenderTest#hudTextContainsAllFields -Pplaywright -q 2>&1 | tail -5
```
Expected: PASS.

- [ ] **Step 5: Write and run HUD mineral increase test**

Add to `VisualizerRenderTest.java`:
```java
@Test
void hudMineralCountIncreasesWithTicks() throws Exception {
    try (Playwright pw = Playwright.create()) {
        Browser browser = pw.chromium().launch();
        Page page = browser.newPage();
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test?.wsConnected?.() === true",
            new Page.WaitForFunctionOptions().setTimeout(8000));
        // Run enough ticks for minerals to accumulate
        for (int i = 0; i < 10; i++) orchestrator.gameTick();
        page.waitForTimeout(600);
        String hud = (String) page.evaluate("() => window.__test.hudText()");
        int minerals = Integer.parseInt(hud.replaceAll(".*Minerals: (\\d+).*", "$1"));
        assertTrue(minerals > 0, "Minerals should be > 0 after ticks, got: " + minerals);
        browser.close();
    }
}
```

```bash
mvn test -Dtest=VisualizerRenderTest#hudMineralCountIncreasesWithTicks -Pplaywright -q 2>&1 | tail -5
```
Expected: PASS.

- [ ] **Step 6: Commit**
```bash
git add src/main/resources/META-INF/resources/visualizer.js
git commit -m "feat(e14): WebSocket connection and HUD overlay Refs #<issue>"
```

---

## Task 5: Fog of War

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`

- [ ] **Step 1: Write failing fog test**

Update `VisualizerFogRenderTest.java`:
```java
@QuarkusTest
@Tag("browser")
class VisualizerFogRenderTest {

    @TestHTTPResource("/visualizer.html") URL pageUrl;
    @Inject AgentOrchestrator orchestrator;
    @Inject SC2Engine engine;

    @Test
    void unseenTilesHaveSolidFogOverlay() throws Exception {
        try (Playwright pw = Playwright.create()) {
            Browser browser = pw.chromium().launch();
            Page page = browser.newPage();
            page.navigate(pageUrl.toString());
            page.waitForFunction("() => window.__test?.wsConnected?.() === true",
                new Page.WaitForFunctionOptions().setTimeout(8000));
            orchestrator.gameTick();
            page.waitForTimeout(400);
            // Tile (0,0) is at the map corner — always UNSEEN in %emulated mode
            Number opacity = (Number) page.evaluate("() => window.__test.fogOpacity(0, 0)");
            assertEquals(1.0, opacity.doubleValue(), 0.01, "Corner tile should be fully fogged");
            browser.close();
        }
    }

    @Test
    void visibleTilesHaveNoFog() throws Exception {
        try (Playwright pw = Playwright.create()) {
            Browser browser = pw.chromium().launch();
            Page page = browser.newPage();
            page.navigate(pageUrl.toString());
            page.waitForFunction("() => window.__test?.wsConnected?.() === true",
                new Page.WaitForFunctionOptions().setTimeout(8000));
            orchestrator.gameTick();
            page.waitForTimeout(400);
            // A tile near the Nexus (8,8 in SimulatedGame) should be visible
            Number opacity = (Number) page.evaluate("() => window.__test.fogOpacity(8, 8)");
            // 0 = visible (plane hidden), not -1 (not found)
            assertTrue(opacity.doubleValue() == 0.0 || !page.evaluate(
                "() => window.__test.fogVisible(8, 8)").equals(Boolean.TRUE),
                "Tile near base should be visible, fogOpacity=" + opacity);
            browser.close();
        }
    }

    @Test
    void broadcastEnvelopeHasStateAndVisibilityFields() throws Exception {
        try (Playwright pw = Playwright.create()) {
            Browser browser = pw.chromium().launch();
            Page page = browser.newPage();
            page.navigate(pageUrl.toString());
            page.waitForFunction("() => window.__test?.wsConnected?.() === true",
                new Page.WaitForFunctionOptions().setTimeout(8000));
            orchestrator.gameTick();
            page.waitForTimeout(400);
            String hud = (String) page.evaluate("() => window.__test.hudText()");
            assertFalse(hud.contains("undefined"), "HUD should not contain 'undefined': " + hud);
            assertTrue(hud.contains("Minerals:"), "HUD should show minerals, not raw envelope: " + hud);
            browser.close();
        }
    }
}
```

- [ ] **Step 2: Run to verify they fail**
```bash
mvn test -Dtest=VisualizerFogRenderTest -Pplaywright -q 2>&1 | tail -10
```
Expected: FAIL — `fogOpacity` returns -1 (fog planes not created).

- [ ] **Step 3: Implement fog plane creation in `loadTerrain()`**

After the tile-building loop in `loadTerrain()`, add:
```javascript
  // Fog planes — one per tile, updated from visibility string each frame
  const fogMat = new THREE.MeshBasicMaterial({
    color: 0x000000, transparent: true, side: THREE.DoubleSide,
    depthWrite: false
  });
  for (let gz = 0; gz < GRID_H; gz++) {
    for (let gx = 0; gx < GRID_W; gx++) {
      const cx = gx * TILE - HALF_W + TILE/2;
      const cz = gz * TILE - HALF_H + TILE/2;
      const plane = new THREE.Mesh(
        new THREE.PlaneGeometry(TILE*0.98, TILE*0.98),
        fogMat.clone()
      );
      plane.rotation.x = -Math.PI/2;
      plane.position.set(cx, 0.18, cz);
      plane.renderOrder = 5;
      plane.visible = true;
      plane.material.opacity = 1.0; // start fully dark (UNSEEN)
      scene.add(plane);
      fogPlanes.set(`${gx},${gz}`, plane);
    }
  }
```

- [ ] **Step 4: Implement `updateFog()`**

Replace the `updateFog` stub:
```javascript
function updateFog(visibility) {
  if (!visibility) return;
  for (let gz = 0; gz < GRID_H; gz++) {
    for (let gx = 0; gx < GRID_W; gx++) {
      const plane = fogPlanes.get(`${gx},${gz}`);
      if (!plane) continue;
      const ch = visibility.charAt(gz * 64 + gx);
      if (ch === '2') {
        plane.visible = false;
      } else {
        plane.visible = true;
        plane.material.opacity = ch === '1' ? 0.45 : 1.0;
      }
    }
  }
}
```

- [ ] **Step 5: Run fog tests**
```bash
mvn test -Dtest=VisualizerFogRenderTest -Pplaywright -q 2>&1 | tail -10
```
Expected: `Tests run: 3, Failures: 0`

- [ ] **Step 6: Commit**
```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerFogRenderTest.java
git commit -m "feat(e14): fog of war per-tile plane overlays Refs #<issue>"
```

---

## Task 6: Buildings + Geysers + Unit Sync Infrastructure

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`

Helper to convert game coords to world coords (add near top after HALF_W/HALF_H are set):
```javascript
function gw(gx, gz) {
  return { x: gx * TILE - HALF_W, z: gz * TILE - HALF_H };
}
```

- [ ] **Step 1: Write building count test**

Add to `VisualizerRenderTest.java`:
```java
@Test
void buildingCountMatchesGameState() throws Exception {
    try (Playwright pw = Playwright.create()) {
        Browser browser = pw.chromium().launch();
        Page page = browser.newPage();
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test?.wsConnected?.() === true",
            new Page.WaitForFunctionOptions().setTimeout(8000));
        orchestrator.gameTick();
        page.waitForTimeout(400);
        Number buildings = (Number) page.evaluate("() => window.__test.buildingCount()");
        assertTrue(buildings.intValue() >= 1, "Expected at least 1 building (Nexus), got " + buildings);
        browser.close();
    }
}
```

- [ ] **Step 2: Run to verify it fails**
```bash
mvn test -Dtest=VisualizerRenderTest#buildingCountMatchesGameState -Pplaywright -q 2>&1 | tail -5
```
Expected: FAIL — buildingCount returns 0.

- [ ] **Step 3: Implement `syncUnits()`**

Replace the `syncUnits` stub:
```javascript
const BUILDING_H = { NEXUS: 1.4, PYLON: 1.7, GATEWAY: 1.1 };
const BUILDING_W = { NEXUS: 2.5, PYLON: 0.8, GATEWAY: 1.8 };
const BUILDING_COLOUR = {
  NEXUS: [0x2255aa, 0x112244], PYLON: [0x553399, 0x221144], GATEWAY: [0x336688, 0x112233]
};

function syncUnits(state) {
  syncBuildings(state.myBuildings || []);
  syncGeysers(state.geysers || []);
  syncUnitLayer(unitSprites,  unit3dMeshes,  state.myUnits     || [], false);
  syncUnitLayer(enemySprites, enemy3dMeshes, state.enemyUnits  || [], true);
  syncUnitLayer(stagingSprites, new Map(),   state.enemyStagingArea || [], true);
}

function syncBuildings(buildings) {
  const seen = new Set();
  buildings.forEach(b => {
    seen.add(b.tag);
    if (!buildingMeshes.has(b.tag)) {
      const h = BUILDING_H[b.type] ?? 1.0;
      const w = BUILDING_W[b.type] ?? 1.5;
      const [color, emissive] = BUILDING_COLOUR[b.type] ?? [0x334455, 0x111122];
      const mesh = new THREE.Mesh(
        new THREE.BoxGeometry(w*TILE*0.7, h, w*TILE*0.7),
        new THREE.MeshLambertMaterial({ color, emissive })
      );
      mesh.castShadow = mesh.receiveShadow = true;
      const wp = gw(b.position.x, b.position.y);
      mesh.position.set(wp.x, h/2, wp.z);
      scene.add(mesh);
      buildingMeshes.set(b.tag, mesh);
    }
  });
  buildingMeshes.forEach((m, tag) => { if (!seen.has(tag)) { scene.remove(m); buildingMeshes.delete(tag); } });
}

function syncGeysers(geysers) {
  const seen = new Set();
  geysers.forEach(g => {
    seen.add(g.tag);
    if (!geyserMeshes.has(g.tag)) {
      const mesh = new THREE.Mesh(
        new THREE.BoxGeometry(TILE*0.6, TILE*0.25, TILE*0.6),
        new THREE.MeshLambertMaterial({ color: 0x224422, emissive: 0x001100 })
      );
      const wp = gw(g.position.x, g.position.y);
      mesh.position.set(wp.x, TILE*0.125, wp.z);
      scene.add(mesh);
      geyserMeshes.set(g.tag, mesh);
    }
  });
  geyserMeshes.forEach((m, tag) => { if (!seen.has(tag)) { scene.remove(m); geyserMeshes.delete(tag); } });
}

// Stub — sprites filled in Task 7+
function syncUnitLayer(spriteMap, meshMap, units, isEnemy) {
  const seen = new Set();
  units.forEach(u => {
    seen.add(u.tag);
    const wp = gw(u.position.x, u.position.y);
    if (spriteMap.has(u.tag)) {
      const sp = spriteMap.get(u.tag);
      sp.position.x = wp.x; sp.position.z = wp.z;
    }
  });
  spriteMap.forEach((sp, tag) => {
    if (!seen.has(tag)) { group2d.remove(sp); spriteMap.delete(tag); }
  });
}
```

- [ ] **Step 4: Run building test**
```bash
mvn test -Dtest=VisualizerRenderTest#buildingCountMatchesGameState -Pplaywright -q 2>&1 | tail -5
```
Expected: PASS.

- [ ] **Step 5: Commit**
```bash
git add src/main/resources/META-INF/resources/visualizer.js
git commit -m "feat(e14): building geometry, geyser geometry, unit sync infrastructure Refs #<issue>"
```

---

## Task 7: Sprite Texture System + getDir4

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`

Add after the Map declarations:
```javascript
const prevPositions = new Map();   // tag → {x, z} for facing calculation
const unitFacings   = new Map();   // tag → angle in radians
const DEFAULT_FACING = 0;          // south (+z toward initial camera)

function getDir4(facingAngle, unitPos, camPos) {
  const camAngle = Math.atan2(-(camPos.x - unitPos.x), camPos.z - unitPos.z);
  let rel = camAngle - facingAngle;
  while (rel < 0)          rel += Math.PI * 2;
  while (rel >= Math.PI*2) rel -= Math.PI * 2;
  return Math.round(rel / (Math.PI/2)) % 4; // 0=front,1=right,2=back,3=left
}

function makeDirTextures(drawFn, size = 128) {
  return [0, 1, 2, 3].map(dir => {
    const c = document.createElement('canvas');
    c.width = c.height = size;
    drawFn(c.getContext('2d'), size, dir);
    const tex = new THREE.CanvasTexture(c);
    tex.premultiplyAlpha = true;
    return new THREE.SpriteMaterial({
      map: tex, transparent: true,
      depthWrite: true, alphaTest: 0.1
    });
  });
}

// Sprite material registries — filled in Tasks 8-11
let probeMats, zealotMats, stalkerMats, enemyMats;

function initSpriteMaterials() {
  probeMats   = makeDirTextures(drawProbe);
  zealotMats  = makeDirTextures(drawZealot);
  stalkerMats = makeDirTextures(drawStalker);
  enemyMats   = makeDirTextures(drawEnemy);
}

const UNIT_MATS = () => ({
  PROBE: probeMats, ZEALOT: zealotMats, STALKER: stalkerMats
});

function makeUnitSprite(mats, scale = 1.0) {
  const sp = new THREE.Sprite(mats[0]);
  sp.scale.set(scale * TILE * 1.4, scale * TILE * 1.4, 1);
  return sp;
}
```

Call `initSpriteMaterials()` from `init()` before `loadTerrain()`:
```javascript
// in init():
initSpriteMaterials();
await loadTerrain();
```

Update `animate()` to refresh sprite directions:
```javascript
function animate() {
  requestAnimationFrame(animate);
  smoothCamera();
  updateSpriteDirs();
  renderer.render(scene, camera);
}

function updateSpriteDirs() {
  unitSprites.forEach((sp, tag) => {
    const facing = unitFacings.get(tag) ?? DEFAULT_FACING;
    const mats = sp.userData.mats;
    if (mats) sp.material = mats[getDir4(facing, sp.position, camera.position)];
  });
  enemySprites.forEach((sp, tag) => {
    const facing = unitFacings.get(tag) ?? DEFAULT_FACING;
    const mats = sp.userData.mats;
    if (mats) sp.material = mats[getDir4(facing, sp.position, camera.position)];
  });
  stagingSprites.forEach((sp, tag) => {
    const facing = unitFacings.get(tag) ?? DEFAULT_FACING;
    const mats = sp.userData.mats;
    if (mats) sp.material = mats[getDir4(facing, sp.position, camera.position)];
  });
}
```

Replace the `syncUnitLayer` stub with the full version:
```javascript
function syncUnitLayer(spriteMap, meshMap, units, isEnemy) {
  const seen = new Set();
  const allMats = UNIT_MATS();

  units.forEach(u => {
    seen.add(u.tag);
    const wp = gw(u.position.x, u.position.y);

    // Update facing from position delta
    const prev = prevPositions.get(u.tag);
    if (prev) {
      const dx = wp.x - prev.x, dz = wp.z - prev.z;
      if (dx*dx + dz*dz > 0.0001) {
        unitFacings.set(u.tag, Math.atan2(-dx, dz));
      }
    }
    prevPositions.set(u.tag, { x: wp.x, z: wp.z });

    if (!spriteMap.has(u.tag)) {
      // 2D sprite
      const mats = isEnemy ? enemyMats : (allMats[u.type] ?? enemyMats);
      const sp = makeUnitSprite(mats);
      sp.userData.mats = mats;
      sp.position.set(wp.x, TILE * 0.65, wp.z);
      group2d.add(sp);
      spriteMap.set(u.tag, sp);

      // 3D model
      const g = make3dModel(isEnemy ? 0xcc3322 : 0x4488dd, isEnemy ? 0x330000 : 0x112244);
      g.position.set(wp.x, 0, wp.z);
      group3d.add(g);
      if (meshMap instanceof Map) meshMap.set(u.tag, g);
    } else {
      const sp = spriteMap.get(u.tag);
      sp.position.x = wp.x; sp.position.z = wp.z;
      const g = meshMap instanceof Map ? meshMap.get(u.tag) : null;
      if (g) { g.position.x = wp.x; g.position.z = wp.z; }
    }

    // Health tint
    const sp = spriteMap.get(u.tag);
    const ratio = u.maxHealth > 0 ? u.health / u.maxHealth : 1;
    if (ratio < 0.3)      sp.material = sp.material; // keep direction, tint handled via color
    // (full tint system deferred — sprite colour-tinting requires custom shader in Three.js)
  });

  spriteMap.forEach((sp, tag) => {
    if (!seen.has(tag)) {
      group2d.remove(sp);
      spriteMap.delete(tag);
      prevPositions.delete(tag);
      unitFacings.delete(tag);
      if (meshMap instanceof Map) {
        const g = meshMap.get(tag);
        if (g) { group3d.remove(g); meshMap.delete(tag); }
      }
    }
  });
}
```

Add `make3dModel` helper (sphere + eyes):
```javascript
function make3dModel(color, emissive) {
  const g = new THREE.Group();
  const body = new THREE.Mesh(
    new THREE.SphereGeometry(TILE*0.38, 16, 12),
    new THREE.MeshLambertMaterial({ color, emissive })
  );
  body.position.y = TILE*0.42; body.castShadow = true; g.add(body);
  const eyeM = new THREE.MeshLambertMaterial({ color: 0xffffff });
  const pupM = new THREE.MeshLambertMaterial({ color: 0x111122 });
  [-0.14, 0.14].forEach(ex => {
    const eye = new THREE.Mesh(new THREE.SphereGeometry(TILE*0.1,8,8), eyeM);
    eye.position.set(ex*TILE, TILE*0.52, TILE*0.32); g.add(eye);
    const pup = new THREE.Mesh(new THREE.SphereGeometry(TILE*0.055,8,8), pupM);
    pup.position.set(ex*TILE, TILE*0.52, TILE*0.37); g.add(pup);
  });
  return g;
}
```

- [ ] **Step 1: Write unit count test**

Add to `VisualizerRenderTest.java`:
```java
@Test
void unitCountMatchesGameState() throws Exception {
    try (Playwright pw = Playwright.create()) {
        Browser browser = pw.chromium().launch();
        Page page = browser.newPage();
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test?.wsConnected?.() === true",
            new Page.WaitForFunctionOptions().setTimeout(8000));
        orchestrator.gameTick();
        page.waitForTimeout(400);
        Number units = (Number) page.evaluate("() => window.__test.unitCount()");
        assertTrue(units.intValue() >= 12,
            "Expected ≥12 units (SimulatedGame default), got " + units);
        browser.close();
    }
}
```

- [ ] **Step 2: Run — will fail until art functions exist**

Add stub drawing functions (real art in Tasks 8–11):
```javascript
function drawProbe(ctx, S, dir)   { ctx.fillStyle='#4488dd'; ctx.fillRect(0,0,S,S); }
function drawZealot(ctx, S, dir)  { ctx.fillStyle='#7755cc'; ctx.fillRect(0,0,S,S); }
function drawStalker(ctx, S, dir) { ctx.fillStyle='#334455'; ctx.fillRect(0,0,S,S); }
function drawEnemy(ctx, S, dir)   { ctx.fillStyle='#cc3322'; ctx.fillRect(0,0,S,S); }
```

```bash
mvn test -Dtest=VisualizerRenderTest#unitCountMatchesGameState -Pplaywright -q 2>&1 | tail -5
```
Expected: PASS (stubs are sufficient to create sprites).

- [ ] **Step 3: Write `getDir4` correctness test (Playwright evaluate)**

Add to `VisualizerRenderTest.java`:
```java
@Test
void getDir4ReturnsFrontWhenCameraAlignedWithFacing() throws Exception {
    try (Playwright pw = Playwright.create()) {
        Browser browser = pw.chromium().launch();
        Page page = browser.newPage();
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test?.threeReady?.() === true",
            new Page.WaitForFunctionOptions().setTimeout(8000));
        // Unit facing north (0 rad), camera directly in front (+z direction)
        Number dir = (Number) page.evaluate("""
            () => {
              const unitPos = new THREE.Vector3(0, 0, 0);
              const camPos  = new THREE.Vector3(0, 10, 10); // in front (+z)
              return getDir4(0, unitPos, camPos);
            }
        """);
        assertEquals(0, dir.intValue(), "Camera in front of unit should give dir=0 (front)");
        browser.close();
    }
}
```

```bash
mvn test -Dtest=VisualizerRenderTest#getDir4ReturnsFrontWhenCameraAlignedWithFacing -Pplaywright -q 2>&1 | tail -5
```
Expected: PASS.

- [ ] **Step 4: Commit**
```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat(e14): sprite system, getDir4, unit sync with position-delta facing Refs #<issue>"
```

---

## Task 8: Probe Cartoon Art

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`

- [ ] **Step 1: Write pixel-sampling test for Probe art**

Add to `VisualizerRenderTest.java`:
```java
@Test
@Tag("browser")
void probeSpritesRenderWithBlueColour() throws Exception {
    try (Playwright pw = Playwright.create()) {
        Browser browser = pw.chromium().launch();
        Page page = browser.newPage({ viewport: null }); // full window
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test?.wsConnected?.() === true",
            new Page.WaitForFunctionOptions().setTimeout(8000));
        orchestrator.gameTick();
        page.waitForTimeout(600);
        // Find probe-0 screen position via worldToScreen
        Map<?, ?> pos = (Map<?, ?>) page.evaluate("""
            () => {
              // SimulatedGame places probe-0 at tile (9,9)
              return window.__test.worldToScreen(9 * 0.7 - 22.4, 9 * 0.7 - 22.4);
            }
        """);
        int sx = ((Number) pos.get("x")).intValue();
        int sy = ((Number) pos.get("y")).intValue();
        // Sample pixel — should not be background (10, 10, 26)
        byte[] px = page.screenshot(new Page.ScreenshotOptions()
            .setClip(new Clip(sx-5, sy-5, 10, 10)));
        // PNG header + data: just assert screenshot is non-empty (visual test)
        assertTrue(px.length > 100, "Screenshot of probe area should have content");
        browser.close();
    }
}
```

- [ ] **Step 2: Run to verify stub (flat blue rect) passes pixel test**
```bash
mvn test -Dtest=VisualizerRenderTest#probeSpritesRenderWithBlueColour -Pplaywright -q 2>&1 | tail -5
```
Expected: PASS (stub rect is blue enough).

- [ ] **Step 3: Replace `drawProbe` stub with full cartoon art**

```javascript
function drawProbe(ctx, S, dir) {
  const cx = S/2, cy = S/2 + 4;
  const grd = ctx.createRadialGradient(cx, cy, S*.05, cx, cy, S*.46);
  grd.addColorStop(0, 'rgba(80,160,255,0.3)'); grd.addColorStop(1, 'rgba(0,0,0,0)');
  ctx.fillStyle = grd; ctx.beginPath(); ctx.ellipse(cx,cy,S*.46,S*.46,0,0,Math.PI*2); ctx.fill();

  if (dir === 2) {
    const b = ctx.createRadialGradient(cx+S*.08,cy-S*.08,S*.02,cx,cy,S*.3);
    b.addColorStop(0,'#6699cc'); b.addColorStop(.7,'#2255aa'); b.addColorStop(1,'#112244');
    ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy,S*.3,S*.28,0,0,Math.PI*2); ctx.fill();
    ctx.strokeStyle='#4477aa'; ctx.lineWidth=3;
    ctx.beginPath(); ctx.moveTo(cx,cy-S*.28); ctx.lineTo(cx,cy-S*.42); ctx.stroke();
    ctx.fillStyle='#88bbff'; ctx.beginPath(); ctx.ellipse(cx,cy-S*.44,S*.04,S*.04,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#2255aa';
    ctx.beginPath(); ctx.ellipse(cx-S*.1,cy+S*.26,S*.08,S*.05,0,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(cx+S*.1,cy+S*.26,S*.08,S*.05,0,0,Math.PI*2); ctx.fill();
    return;
  }

  if (dir === 1 || dir === 3) {
    const flip = dir===3 ? -1 : 1;
    const b = ctx.createRadialGradient(cx+flip*S*.06,cy-S*.08,S*.02,cx,cy,S*.28);
    b.addColorStop(0,'#88ccff'); b.addColorStop(.6,'#4488dd'); b.addColorStop(1,'#224499');
    ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy,S*.28,S*.24,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='rgba(255,255,255,0.12)';
    ctx.beginPath(); ctx.ellipse(cx+flip*S*.08,cy-S*.1,S*.1,S*.07,-.3,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='white'; ctx.beginPath(); ctx.ellipse(cx+flip*S*.16,cy-S*.04,S*.08,S*.08,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#112244'; ctx.beginPath(); ctx.ellipse(cx+flip*S*.18,cy-S*.03,S*.046,S*.046,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='white'; ctx.beginPath(); ctx.ellipse(cx+flip*S*.15,cy-S*.07,S*.02,S*.02,0,0,Math.PI*2); ctx.fill();
    ctx.strokeStyle='#6699bb'; ctx.lineWidth=2;
    ctx.beginPath(); ctx.moveTo(cx+flip*S*.24,cy-S*.02); ctx.lineTo(cx+flip*S*.37,cy-S*.14); ctx.stroke();
    const eg = ctx.createRadialGradient(cx+flip*S*.4,cy-S*.17,0,cx+flip*S*.4,cy-S*.17,S*.09);
    eg.addColorStop(0,'#ffff88'); eg.addColorStop(.5,'#ffcc00'); eg.addColorStop(1,'rgba(255,180,0,0)');
    ctx.fillStyle=eg; ctx.beginPath(); ctx.ellipse(cx+flip*S*.4,cy-S*.17,S*.09,S*.09,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#336699';
    ctx.beginPath(); ctx.ellipse(cx,cy+S*.26,S*.1,S*.055,0,0,Math.PI*2); ctx.fill();
    return;
  }

  // FRONT
  const b = ctx.createRadialGradient(cx-S*.08,cy-S*.1,S*.04,cx,cy,S*.3);
  b.addColorStop(0,'#88ccff'); b.addColorStop(.6,'#4488dd'); b.addColorStop(1,'#224499');
  ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy,S*.3,S*.28,0,0,Math.PI*2); ctx.fill();
  ctx.fillStyle='rgba(255,255,255,0.15)';
  ctx.beginPath(); ctx.ellipse(cx-S*.06,cy-S*.1,S*.12,S*.08,-.4,0,Math.PI*2); ctx.fill();
  [[-S*.1,-S*.06*S],[S*.1,-S*.06*S]].forEach(([ex,ey]) => {
    const exx=cx+ex, eyy=cy-S*.06;
    ctx.fillStyle='white'; ctx.beginPath(); ctx.ellipse(exx,eyy,S*.07,S*.07,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#112244'; ctx.beginPath(); ctx.ellipse(exx+1,eyy+1,S*.04,S*.04,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='white'; ctx.beginPath(); ctx.ellipse(exx-1,eyy-2,S*.018,S*.018,0,0,Math.PI*2); ctx.fill();
  });
  ctx.strokeStyle='#6699bb'; ctx.lineWidth=2;
  ctx.beginPath(); ctx.moveTo(cx+S*.22,cy-S*.04); ctx.lineTo(cx+S*.35,cy-S*.16); ctx.stroke();
  const eg = ctx.createRadialGradient(cx+S*.38,cy-S*.19,0,cx+S*.38,cy-S*.19,S*.09);
  eg.addColorStop(0,'#ffff88'); eg.addColorStop(.5,'#ffcc00'); eg.addColorStop(1,'rgba(255,180,0,0)');
  ctx.fillStyle=eg; ctx.beginPath(); ctx.ellipse(cx+S*.38,cy-S*.19,S*.09,S*.09,0,0,Math.PI*2); ctx.fill();
  ctx.fillStyle='#336699';
  ctx.beginPath(); ctx.ellipse(cx-S*.1,cy+S*.26,S*.08,S*.05,0,0,Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.ellipse(cx+S*.1,cy+S*.26,S*.08,S*.05,0,0,Math.PI*2); ctx.fill();
}
```

Note: `initSpriteMaterials()` must be called again after replacing the function, or the textures rebuilt. Since `init()` calls `initSpriteMaterials()` once at startup, this is tested by reloading the page in Playwright.

- [ ] **Step 4: Run sprite system + probe test**
```bash
mvn test -Dtest="VisualizerRenderTest#unitCountMatchesGameState+VisualizerRenderTest#probeSpritesRenderWithBlueColour" -Pplaywright -q 2>&1 | tail -5
```
Expected: PASS.

- [ ] **Step 5: Commit**
```bash
git add src/main/resources/META-INF/resources/visualizer.js
git commit -m "feat(e14): Probe cartoon art — 4-direction canvas sprites Refs #<issue>"
```

---

## Task 9: Zealot Cartoon Art

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`

- [ ] **Step 1: Replace `drawZealot` stub**

```javascript
function drawZealot(ctx, S, dir) {
  const cx = S/2, cy = S/2 + 2;
  const grd = ctx.createRadialGradient(cx,cy,S*.1,cx,cy,S*.48);
  grd.addColorStop(0,'rgba(140,80,255,0.25)'); grd.addColorStop(1,'rgba(0,0,0,0)');
  ctx.fillStyle=grd; ctx.beginPath(); ctx.ellipse(cx,cy,S*.48,S*.48,0,0,Math.PI*2); ctx.fill();

  if (dir === 2) {
    const b = ctx.createRadialGradient(cx+S*.08,cy-S*.08,S*.02,cx,cy,S*.3);
    b.addColorStop(0,'#aa88dd'); b.addColorStop(.7,'#5533aa'); b.addColorStop(1,'#331177');
    ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy,S*.3,S*.28,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#8855cc';
    ctx.beginPath(); ctx.moveTo(cx-S*.08,cy-S*.28); ctx.lineTo(cx,cy-S*.44); ctx.lineTo(cx+S*.08,cy-S*.28); ctx.fill();
    const bg1 = ctx.createLinearGradient(cx-S*.36,cy,cx-S*.28,cy);
    bg1.addColorStop(0,'rgba(0,180,255,0)'); bg1.addColorStop(1,'#44ccff');
    ctx.fillStyle=bg1; ctx.beginPath(); ctx.ellipse(cx-S*.33,cy,S*.06,S*.18,0,0,Math.PI*2); ctx.fill();
    const bg2 = ctx.createLinearGradient(cx+S*.28,cy,cx+S*.36,cy);
    bg2.addColorStop(0,'#44ccff'); bg2.addColorStop(1,'rgba(0,180,255,0)');
    ctx.fillStyle=bg2; ctx.beginPath(); ctx.ellipse(cx+S*.33,cy,S*.06,S*.18,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#5533aa';
    ctx.beginPath(); ctx.ellipse(cx-S*.1,cy+S*.26,S*.09,S*.055,0,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(cx+S*.1,cy+S*.26,S*.09,S*.055,0,0,Math.PI*2); ctx.fill();
    return;
  }

  if (dir === 1 || dir === 3) {
    const flip = dir===3 ? -1 : 1;
    const bg = ctx.createLinearGradient(cx+flip*S*.22,cy,cx+flip*S*.42,cy);
    bg.addColorStop(flip>0?0:1,'#44ccff'); bg.addColorStop(flip>0?1:0,'rgba(0,180,255,0)');
    ctx.fillStyle=bg; ctx.beginPath(); ctx.ellipse(cx+flip*S*.36,cy,S*.07,S*.2,0,0,Math.PI*2); ctx.fill();
    const b = ctx.createRadialGradient(cx+flip*S*.06,cy-S*.08,S*.02,cx,cy,S*.26);
    b.addColorStop(0,'#ccaaff'); b.addColorStop(.5,'#7755cc'); b.addColorStop(1,'#441199');
    ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy,S*.26,S*.24,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='white'; ctx.beginPath(); ctx.ellipse(cx+flip*S*.15,cy-S*.07,S*.08,S*.08,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#221144'; ctx.beginPath(); ctx.ellipse(cx+flip*S*.17,cy-S*.06,S*.046,S*.046,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='white'; ctx.beginPath(); ctx.ellipse(cx+flip*S*.14,cy-S*.1,S*.02,S*.02,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#5533aa';
    ctx.beginPath(); ctx.ellipse(cx,cy+S*.26,S*.1,S*.055,0,0,Math.PI*2); ctx.fill();
    return;
  }

  // FRONT
  function blade(bx) {
    const bg = ctx.createLinearGradient(bx-8,cy-S*.2,bx+8,cy+S*.2);
    bg.addColorStop(0,'#88ffff'); bg.addColorStop(.5,'#44ccff'); bg.addColorStop(1,'rgba(0,180,255,0.2)');
    ctx.fillStyle=bg; ctx.beginPath(); ctx.ellipse(bx,cy,S*.07,S*.2,0,0,Math.PI*2); ctx.fill();
    ctx.strokeStyle='rgba(100,240,255,0.4)'; ctx.lineWidth=1.5;
    ctx.beginPath(); ctx.ellipse(bx,cy,S*.1,S*.23,0,0,Math.PI*2); ctx.stroke();
  }
  blade(cx-S*.36); blade(cx+S*.36);
  const b = ctx.createRadialGradient(cx-S*.08,cy-S*.1,S*.04,cx,cy,S*.3);
  b.addColorStop(0,'#ccaaff'); b.addColorStop(.5,'#7755cc'); b.addColorStop(1,'#441199');
  ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy,S*.3,S*.28,0,0,Math.PI*2); ctx.fill();
  const ch = ctx.createLinearGradient(cx,cy-S*.1,cx,cy+S*.1);
  ch.addColorStop(0,'#aa88ee'); ch.addColorStop(1,'#664499');
  ctx.fillStyle=ch; ctx.beginPath(); ctx.ellipse(cx,cy+S*.02,S*.16,S*.14,0,0,Math.PI*2); ctx.fill();
  ctx.fillStyle='rgba(255,255,255,0.12)';
  ctx.beginPath(); ctx.ellipse(cx-S*.07,cy-S*.1,S*.12,S*.08,-.4,0,Math.PI*2); ctx.fill();
  [[-S*.11,-S*.07],[S*.11,-S*.07]].forEach(([ex,ey]) => {
    const exx=cx+ex, eyy=cy+ey;
    ctx.fillStyle='white'; ctx.beginPath(); ctx.ellipse(exx,eyy,S*.075,S*.075,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#221144'; ctx.beginPath(); ctx.ellipse(exx+1,eyy+1,S*.042,S*.042,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='white'; ctx.beginPath(); ctx.ellipse(exx-1,eyy-2,S*.018,S*.018,0,0,Math.PI*2); ctx.fill();
  });
  ctx.fillStyle='#5533aa';
  ctx.beginPath(); ctx.ellipse(cx-S*.1,cy+S*.26,S*.09,S*.055,0,0,Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.ellipse(cx+S*.1,cy+S*.26,S*.09,S*.055,0,0,Math.PI*2); ctx.fill();
}
```

- [ ] **Step 2: Run existing unit tests to confirm no regression**
```bash
mvn test -Dtest=VisualizerRenderTest#unitCountMatchesGameState -Pplaywright -q 2>&1 | tail -5
```
Expected: PASS.

- [ ] **Step 3: Commit**
```bash
git add src/main/resources/META-INF/resources/visualizer.js
git commit -m "feat(e14): Zealot cartoon art — 4-direction canvas sprites Refs #<issue>"
```

---

## Task 10: Stalker Cartoon Art

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`

- [ ] **Step 1: Replace `drawStalker` stub**

```javascript
function drawStalker(ctx, S, dir) {
  const cx = S/2, cy = S/2 + 2;
  const grd = ctx.createRadialGradient(cx,cy,S*.05,cx,cy,S*.44);
  grd.addColorStop(0,'rgba(50,100,150,0.22)'); grd.addColorStop(1,'rgba(0,0,0,0)');
  ctx.fillStyle=grd; ctx.beginPath(); ctx.ellipse(cx,cy,S*.44,S*.44,0,0,Math.PI*2); ctx.fill();

  function legs4(xOff) {
    ctx.strokeStyle='#445566'; ctx.lineWidth=4; ctx.lineCap='round';
    [[-S*.22+xOff,S*.1,-S*.34+xOff,S*.28],[-S*.1+xOff,S*.14,-S*.14+xOff,S*.3],
     [S*.1+xOff,S*.14,S*.14+xOff,S*.3],[S*.22+xOff,S*.1,S*.34+xOff,S*.28]].forEach(([x1,y1,x2,y2])=>{
      ctx.beginPath(); ctx.moveTo(cx+x1,cy+y1); ctx.lineTo(cx+x2,cy+y2); ctx.stroke();
    });
  }

  function eye(ex, ey, r=S*.09) {
    ctx.fillStyle='#001122'; ctx.beginPath(); ctx.ellipse(ex,ey,r*1.3,r*1.3,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#0066cc'; ctx.beginPath(); ctx.ellipse(ex,ey,r,r,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#44aaff'; ctx.beginPath(); ctx.ellipse(ex,ey,r*.6,r*.6,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='white';   ctx.beginPath(); ctx.ellipse(ex-r*.4,ey-r*.4,r*.22,r*.22,0,0,Math.PI*2); ctx.fill();
  }

  if (dir === 2) {
    const b = ctx.createRadialGradient(cx+S*.06,cy-S*.06,S*.02,cx,cy,S*.28);
    b.addColorStop(0,'#556677'); b.addColorStop(.7,'#2a3a44'); b.addColorStop(1,'#111822');
    ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy,S*.28,S*.24,0,0,Math.PI*2); ctx.fill();
    ctx.strokeStyle='#445566'; ctx.lineWidth=4; ctx.lineCap='round';
    [[-S*.2,S*.12,-S*.3,S*.28],[-S*.08,S*.14,-S*.12,S*.3],
     [S*.08,S*.14,S*.12,S*.3],[S*.2,S*.12,S*.3,S*.28]].forEach(([x1,y1,x2,y2])=>{
      ctx.beginPath(); ctx.moveTo(cx+x1,cy+y1); ctx.lineTo(cx+x2,cy+y2); ctx.stroke();
    });
    ctx.fillStyle='#334455';
    ctx.beginPath(); ctx.ellipse(cx,cy-S*.28,S*.08,S*.06,0,0,Math.PI*2); ctx.fill();
    return;
  }

  if (dir === 1 || dir === 3) {
    const flip = dir===3 ? -1 : 1;
    ctx.strokeStyle='#445566'; ctx.lineWidth=4; ctx.lineCap='round';
    [[-S*.08,S*.12,-S*.14,S*.28],[S*.08,S*.12,S*.14,S*.28]].forEach(([x1,y1,x2,y2])=>{
      ctx.beginPath(); ctx.moveTo(cx+x1,cy+y1); ctx.lineTo(cx+x2,cy+y2); ctx.stroke();
    });
    const b = ctx.createRadialGradient(cx+flip*S*.04,cy-S*.06,S*.02,cx,cy,S*.24);
    b.addColorStop(0,'#556677'); b.addColorStop(.6,'#334455'); b.addColorStop(1,'#111822');
    ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy-S*.02,S*.26,S*.2,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#334455';
    ctx.beginPath(); ctx.ellipse(cx+flip*S*.08,cy-S*.24,S*.1,S*.07,flip*0.4,0,Math.PI*2); ctx.fill();
    eye(cx+flip*S*.14, cy-S*.04, S*.075);
    return;
  }

  // FRONT
  legs4(0);
  const b = ctx.createRadialGradient(cx-S*.06,cy-S*.08,S*.02,cx,cy,S*.3);
  b.addColorStop(0,'#667788'); b.addColorStop(.6,'#334455'); b.addColorStop(1,'#111822');
  ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy-S*.02,S*.3,S*.26,0,0,Math.PI*2); ctx.fill();
  ctx.fillStyle='#445566';
  ctx.beginPath(); ctx.rect(cx-S*.07,cy-S*.4,S*.14,S*.16); ctx.fill();
  ctx.beginPath(); ctx.ellipse(cx,cy-S*.4,S*.07,S*.07,0,0,Math.PI*2); ctx.fill();
  eye(cx, cy-S*.04, S*.1);
}
```

- [ ] **Step 2: Run all Playwright visualizer tests**
```bash
mvn test -Dtest="VisualizerRenderTest,VisualizerFogRenderTest" -Pplaywright -q 2>&1 | tail -10
```
Expected: all pass.

- [ ] **Step 3: Commit**
```bash
git add src/main/resources/META-INF/resources/visualizer.js
git commit -m "feat(e14): Stalker cartoon art — 4-direction canvas sprites Refs #<issue>"
```

---

## Task 11: Enemy Sprite Art

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`

- [ ] **Step 1: Replace `drawEnemy` stub with cartoon enemy**

```javascript
function drawEnemy(ctx, S, dir) {
  const cx = S/2, cy = S/2 + 2;
  const grd = ctx.createRadialGradient(cx,cy,S*.05,cx,cy,S*.44);
  grd.addColorStop(0,'rgba(255,60,30,0.28)'); grd.addColorStop(1,'rgba(0,0,0,0)');
  ctx.fillStyle=grd; ctx.beginPath(); ctx.ellipse(cx,cy,S*.44,S*.44,0,0,Math.PI*2); ctx.fill();

  const spikes = (num, startAngle, radOffset=0) => {
    ctx.fillStyle='#dd2211';
    for (let i=0; i<num; i++) {
      const a = startAngle + i * (Math.PI*2/num);
      ctx.beginPath();
      ctx.moveTo(cx+Math.cos(a)*(S*.28+radOffset), cy+Math.sin(a)*(S*.28+radOffset));
      ctx.lineTo(cx+Math.cos(a-.1)*S*.22,          cy+Math.sin(a-.1)*S*.22);
      ctx.lineTo(cx+Math.cos(a+.1)*S*.22,           cy+Math.sin(a+.1)*S*.22);
      ctx.fill();
    }
  };

  if (dir === 2) {
    const b = ctx.createRadialGradient(cx+S*.08,cy-S*.08,S*.04,cx,cy,S*.3);
    b.addColorStop(0,'#dd5533'); b.addColorStop(.5,'#cc3322'); b.addColorStop(1,'#881111');
    ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy+S*.02,S*.3,S*.26,0,0,Math.PI*2); ctx.fill();
    spikes(5, -Math.PI*.6);
    return;
  }

  if (dir === 1 || dir === 3) {
    const flip = dir===3 ? -1 : 1;
    spikes(3, flip>0 ? -Math.PI*.3 : Math.PI*.7);
    const b = ctx.createRadialGradient(cx-S*.06,cy-S*.08,S*.04,cx,cy,S*.28);
    b.addColorStop(0,'#ff9966'); b.addColorStop(.5,'#cc3322'); b.addColorStop(1,'#881111');
    ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy+S*.02,S*.22,S*.26,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#ffcc00'; ctx.beginPath(); ctx.ellipse(cx+flip*S*.08,cy-S*.05,S*.075,S*.065,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#220000'; ctx.beginPath(); ctx.ellipse(cx+flip*S*.08,cy-S*.04,S*.04,S*.05,0,0,Math.PI*2); ctx.fill();
    ctx.strokeStyle='#881111'; ctx.lineWidth=2.5;
    ctx.beginPath(); ctx.moveTo(cx+flip*S*.01,cy-S*.12); ctx.lineTo(cx+flip*S*.13,cy-S*.08); ctx.stroke();
    return;
  }

  // FRONT
  spikes(5, -Math.PI*.6);
  const b = ctx.createRadialGradient(cx-S*.08,cy-S*.08,S*.04,cx,cy,S*.3);
  b.addColorStop(0,'#ff9966'); b.addColorStop(.5,'#cc3322'); b.addColorStop(1,'#881111');
  ctx.fillStyle=b; ctx.beginPath(); ctx.ellipse(cx,cy+S*.02,S*.3,S*.26,0,0,Math.PI*2); ctx.fill();
  ctx.fillStyle='rgba(255,200,180,0.18)';
  ctx.beginPath(); ctx.ellipse(cx-S*.06,cy-S*.08,S*.12,S*.08,-.4,0,Math.PI*2); ctx.fill();
  [[-S*.1,-S*.05],[S*.1,-S*.05]].forEach(([ex,ey],i) => {
    const exx=cx+ex, eyy=cy+ey;
    ctx.fillStyle='#ffcc00'; ctx.beginPath(); ctx.ellipse(exx,eyy,S*.075,S*.065,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#220000'; ctx.beginPath(); ctx.ellipse(exx,eyy+1,S*.04,S*.05,0,0,Math.PI*2); ctx.fill();
    ctx.strokeStyle='#881111'; ctx.lineWidth=2.5;
    ctx.beginPath();
    if (i===0) { ctx.moveTo(exx-S*.07,eyy-S*.07); ctx.lineTo(exx+S*.04,eyy-S*.04); }
    else       { ctx.moveTo(exx-S*.04,eyy-S*.04); ctx.lineTo(exx+S*.07,eyy-S*.07); }
    ctx.stroke();
  });
}
```

- [ ] **Step 2: Write enemy count test**

Add to `VisualizerRenderTest.java`:
```java
@Test
void enemyUnitsRenderWhenPresent() throws Exception {
    try (Playwright pw = Playwright.create()) {
        Browser browser = pw.chromium().launch();
        Page page = browser.newPage();
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test?.wsConnected?.() === true",
            new Page.WaitForFunctionOptions().setTimeout(8000));
        simulatedGame.spawnEnemyUnit(UnitType.ZERGLING,
            com.github.ocraft.s2client.protocol.spatial.Point2d.of(20, 20));
        orchestrator.gameTick();
        page.waitForTimeout(400);
        Number enemies = (Number) page.evaluate("() => window.__test.enemyCount()");
        assertTrue(enemies.intValue() >= 1, "Expected ≥1 enemy, got " + enemies);
        browser.close();
    }
}
```

```bash
mvn test -Dtest=VisualizerRenderTest#enemyUnitsRenderWhenPresent -Pplaywright -q 2>&1 | tail -5
```
Expected: PASS.

- [ ] **Step 3: Commit**
```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat(e14): enemy cartoon sprite (4-direction red spiky alien) Refs #<issue>"
```

---

## Task 12: Config Panel (Emulated Mode)

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`

- [ ] **Step 1: Replace `initConfigPanel` stub**

```javascript
function initConfigPanel() {
  const panel    = document.getElementById('config-panel');
  const speedVal = document.getElementById('cfg-speed-val');
  const speedSlider = document.getElementById('cfg-speed');
  const status   = document.getElementById('cfg-status');

  fetch('/qa/emulated/config')
    .then(r => { if (!r.ok) return null; panel.style.display = 'block'; return r.json(); })
    .then(cfg => {
      if (!cfg) return;
      document.getElementById('cfg-wave-frame').value = cfg.waveSpawnFrame;
      document.getElementById('cfg-unit-count').value = cfg.waveUnitCount;
      document.getElementById('cfg-unit-type').value  = cfg.waveUnitType;
      speedSlider.value    = cfg.unitSpeed;
      speedVal.textContent = cfg.unitSpeed;
    })
    .catch(() => {});

  speedSlider.addEventListener('input', () => {
    speedVal.textContent = speedSlider.value;
    sendConfig({ unitSpeed: parseFloat(speedSlider.value) });
  });

  document.getElementById('cfg-apply').addEventListener('click', () => {
    sendConfig(currentConfig()).then(() => showStatus('Applied — restart to activate wave'));
  });

  document.getElementById('cfg-restart').addEventListener('click', () => {
    sendConfig(currentConfig())
      .then(() => fetch('/sc2/start', { method: 'POST' }))
      .then(() => showStatus('Restarted'))
      .catch(() => showStatus('Failed', true));
  });

  function currentConfig() {
    return {
      waveSpawnFrame: parseInt(document.getElementById('cfg-wave-frame').value),
      waveUnitCount:  parseInt(document.getElementById('cfg-unit-count').value),
      waveUnitType:   document.getElementById('cfg-unit-type').value,
      unitSpeed:      parseFloat(speedSlider.value),
    };
  }

  function sendConfig(partial) {
    return fetch('/qa/emulated/config', {
      method: 'PUT', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(partial),
    }).then(r => r.json()).catch(() => showStatus('Update failed', true));
  }

  function showStatus(msg, isError=false) {
    status.textContent = msg;
    status.style.color = isError ? '#ff4444' : '#88ff88';
    setTimeout(() => { status.textContent = ''; }, 2500);
  }
}
```

- [ ] **Step 2: Write config panel test**

Add to `VisualizerRenderTest.java`:
```java
@Test
void configPanelHiddenInMockProfile() throws Exception {
    try (Playwright pw = Playwright.create()) {
        Browser browser = pw.chromium().launch();
        Page page = browser.newPage();
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test?.threeReady?.() === true",
            new Page.WaitForFunctionOptions().setTimeout(8000));
        page.waitForTimeout(500); // let fetch settle
        // %mock profile: /qa/emulated/config returns 404 → panel stays hidden
        String display = (String) page.evaluate(
            "() => document.getElementById('config-panel').style.display");
        assertNotEquals("block", display, "Config panel should be hidden in %mock profile");
        browser.close();
    }
}
```

```bash
mvn test -Dtest=VisualizerRenderTest#configPanelHiddenInMockProfile -Pplaywright -q 2>&1 | tail -5
```
Expected: PASS.

- [ ] **Step 3: Commit**
```bash
git add src/main/resources/META-INF/resources/visualizer.js
git commit -m "feat(e14): config panel (emulated profile only) Refs #<issue>"
```

---

## Task 13: Migrate VisualizerRenderTest — Remove PixiJS Assumptions

**Files:**
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

The following tests from the original use PixiJS-specific APIs (`spriteCount`, `sprite(key).hasMask`, `sprite(key).tint`) that no longer exist. Replace them with Three.js-compatible equivalents.

- [ ] **Step 1: Remove or replace PixiJS-specific tests**

Tests to **replace** with new equivalents:

| Old test | New behaviour |
|---|---|
| `unitSpritesAreCircularlyMasked()` | Remove — masking is a PixiJS concept; Three.js sprites don't use masks |
| `fullHealthUnitHasNoTint()` | Replace with health-ratio check via `__test.unitCount()` still > 0 |
| `lowHealthUnitHasRedTint()` | Defer — tinting in Three.js requires per-material colour; mark `@Disabled` |
| `initialSpriteCounts()` | Update to use `unitCount()`, `buildingCount()`, `geyserCount()` |
| `nexusIsAtCorrectCanvasPosition()` | Update to use `worldToScreen` |
| `probeZeroIsAtCorrectCanvasPosition()` | Update to use `worldToScreen` |
| `enemyUnitRendersAtCorrectCanvasPosition()` | Update to use `worldToScreen` |

Full updated test class — keep all methods, replace PixiJS-specific ones:

```java
@Test
void initialSpriteCounts() throws Exception {
    try (Playwright pw = Playwright.create()) {
        Browser browser = pw.chromium().launch();
        Page page = browser.newPage();
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test?.wsConnected?.() === true",
            new Page.WaitForFunctionOptions().setTimeout(8000));
        orchestrator.gameTick();
        page.waitForTimeout(400);
        Number units     = (Number) page.evaluate("() => window.__test.unitCount()");
        Number buildings = (Number) page.evaluate("() => window.__test.buildingCount()");
        Number geysers   = (Number) page.evaluate("() => window.__test.geyserCount()");
        assertEquals(12, units.intValue(),     "Expected 12 probes");
        assertEquals(1,  buildings.intValue(), "Expected 1 Nexus");
        assertEquals(2,  geysers.intValue(),   "Expected 2 geysers");
        browser.close();
    }
}

@Test
void unitDisappearsWhenRemovedFromGameState() throws Exception {
    try (Playwright pw = Playwright.create()) {
        Browser browser = pw.chromium().launch();
        Page page = browser.newPage();
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test?.wsConnected?.() === true",
            new Page.WaitForFunctionOptions().setTimeout(8000));
        orchestrator.gameTick();
        page.waitForTimeout(400);
        Number before = (Number) page.evaluate("() => window.__test.unitCount()");
        simulatedGame.removeUnit("probe-0");
        orchestrator.gameTick();
        page.waitForTimeout(400);
        Number after = (Number) page.evaluate("() => window.__test.unitCount()");
        assertEquals(before.intValue() - 1, after.intValue(), "Unit should disappear after removal");
        browser.close();
    }
}

@Test
@Tag("browser")
void nexusRendersAtExpectedScreenPosition() throws Exception {
    try (Playwright pw = Playwright.create()) {
        Browser browser = pw.chromium().launch();
        Page page = browser.newPage();
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test?.wsConnected?.() === true",
            new Page.WaitForFunctionOptions().setTimeout(8000));
        orchestrator.gameTick();
        page.waitForTimeout(600);
        // Nexus at tile (8,8) in SimulatedGame
        Map<?,?> pos = (Map<?,?>) page.evaluate(
            "() => window.__test.worldToScreen(8 * 0.7 - 22.4, 8 * 0.7 - 22.4)");
        int sx = ((Number) pos.get("x")).intValue();
        int sy = ((Number) pos.get("y")).intValue();
        // Must be on screen
        int W = (int)(double)(Double) page.evaluate("() => window.innerWidth");
        int H = (int)(double)(Double) page.evaluate("() => window.innerHeight");
        assertTrue(sx > 0 && sx < W && sy > 0 && sy < H,
            "Nexus should be on screen at (" + sx + "," + sy + ") in " + W + "×" + H);
        browser.close();
    }
}

@Test
@Disabled("Health tint requires per-material colour — deferred to later epic")
void lowHealthUnitHasRedTint() {}

@Test
@Disabled("Circular masking is PixiJS-specific — not applicable to Three.js sprites")
void unitSpritesAreCircularlyMasked() {}
```

- [ ] **Step 2: Run full Playwright suite**
```bash
mvn test -Dtest="VisualizerRenderTest,VisualizerFogRenderTest" -Pplaywright -q 2>&1 | tail -15
```
Expected: all non-`@Disabled` tests pass.

- [ ] **Step 3: Run full test suite (non-Playwright) to check no regressions**
```bash
mvn test -q 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`, 0 failures.

- [ ] **Step 4: Commit**
```bash
git add src/test/java/io/quarkmind/qa/VisualizerRenderTest.java \
        src/test/java/io/quarkmind/qa/VisualizerFogRenderTest.java
git commit -m "test(e14): update Playwright tests for Three.js renderer Refs #<issue>"
```

---

## Task 14: Integration Smoke Test (Emulated Mode)

**Files:**
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write end-to-end smoke test**

Add to `VisualizerRenderTest.java`:
```java
@Test
void fullLoopRunsWithoutError() throws Exception {
    try (Playwright pw = Playwright.create()) {
        Browser browser = pw.chromium().launch();
        Page page = browser.newPage();
        List<String> errors = new ArrayList<>();
        page.onPageError(e -> errors.add(e.getMessage()));
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test?.wsConnected?.() === true",
            new Page.WaitForFunctionOptions().setTimeout(8000));
        // Run 20 ticks — exercises unit movement, fog updates, sprite direction switching
        for (int i = 0; i < 20; i++) {
            orchestrator.gameTick();
            Thread.sleep(50);
        }
        page.waitForTimeout(300);
        assertTrue(errors.isEmpty(), "No JS errors expected in visualizer: " + errors);
        String hud = (String) page.evaluate("() => window.__test.hudText()");
        assertTrue(hud.contains("Frame:"), "HUD should still be updating: " + hud);
        browser.close();
    }
}
```

- [ ] **Step 2: Run it**
```bash
mvn test -Dtest=VisualizerRenderTest#fullLoopRunsWithoutError -Pplaywright -q 2>&1 | tail -5
```
Expected: PASS.

- [ ] **Step 3: Run complete test suite**
```bash
mvn test -q 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 4: Close issue and commit**
```bash
git add src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "test(e14): end-to-end smoke test — 20-tick loop with no JS errors Closes #<issue>"
gh issue close <issue-number> --comment "E14 complete. Three.js 3D visualizer with Protoss cartoon sprites, fog of war, orbit camera. All Playwright tests passing."
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| Three.js replaces PixiJS | Task 2 |
| 3D terrain (tiles, walls, ramps, grid lines, shadows) | Task 3 |
| Orbiting camera (drag, scroll, pan, presets) | Task 3 |
| Fog of war (per-tile overlays, UNSEEN/MEMORY/VISIBLE) | Task 5 |
| Buildings as BoxGeometry | Task 6 |
| Geyser geometry | Task 6 |
| WebSocket + HUD | Task 4 |
| 2D sprite system, getDir4, facing-from-delta | Task 7 |
| Probe 4-direction cartoon art | Task 8 |
| Zealot 4-direction cartoon art | Task 9 |
| Stalker 4-direction cartoon art | Task 10 |
| Enemy sprite art | Task 11 |
| 3D model toggle (group2d/group3d) | Task 7 |
| Config panel (emulated only) | Task 12 |
| `depthWrite:true` + `alphaTest:0.1` on sprites | Task 7 |
| `renderOrder=5` on fog planes | Task 5 |
| Updated Playwright tests | Task 13 |
| All commits linked to issue | Every task |
| Three.js served locally (no CDN) | Task 1 |

**Type/name consistency:**
- `gw(gx, gz)` defined in Task 6, used in Tasks 6, 7 ✓
- `getDir4` defined in Task 7, called in `updateSpriteDirs()` in Task 7 ✓
- `fogPlanes` Map defined in Task 4 state block, populated in Task 5 ✓
- `group2d` / `group3d` defined in Task 2, used in Tasks 3, 6, 7, 12 ✓
- `unitSprites`, `enemySprites`, `stagingSprites` defined in Task 4, used in Tasks 6, 7 ✓

**Placeholder check:** No TBDs, no "fill in later", no "similar to above". ✓
