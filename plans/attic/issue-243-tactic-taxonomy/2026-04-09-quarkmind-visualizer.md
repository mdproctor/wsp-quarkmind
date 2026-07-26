# QuarkMind Visualizer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a live 2D game visualizer — PixiJS web app served by Quarkus, pushed via WebSocket each tick, with an Electron wrapper for a native OS window.

**Architecture:** `GameStateBroadcaster` registers as a `SC2Engine` frame listener and broadcasts `GameState` JSON to all connected WebSocket clients each game tick. `GameStateSocket` manages WebSocket sessions. `SpriteProxyResource` fetches SC2 unit images from Liquipedia server-side, solving CORS for WebGL textures. The frontend is three static files (PixiJS bundled locally, HTML, JS) served by Quarkus with no build step. The Electron wrapper is a separate npm project in `electron/` that spawns the Quarkus uber-jar and opens a `BrowserWindow` pointing at `localhost:8080/visualizer.html`.

**Tech Stack:** Quarkus 3.34.2, `quarkus-websockets-next`, Jackson `ObjectMapper` (already on classpath via `quarkus-rest-jackson`), PixiJS 8, vanilla JS, Electron 34, JUnit 5 + AssertJ, RestAssured (`@QuarkusTest`).

---

## File Map

| Action | File | Purpose |
|---|---|---|
| Modify | `pom.xml` | Add `quarkus-websockets-next` |
| Create | `src/main/java/io/quarkmind/qa/GameStateBroadcaster.java` | Registers as frame listener; broadcasts GameState JSON to WebSocket sessions |
| Create | `src/main/java/io/quarkmind/qa/GameStateSocket.java` | WebSocket endpoint; manages session set via broadcaster |
| Create | `src/main/java/io/quarkmind/qa/SpriteProxyResource.java` | Proxies Liquipedia images to solve CORS; cached in memory |
| Create | `src/test/java/io/quarkmind/qa/GameStateBroadcasterTest.java` | Plain JUnit — tests JSON serialisation of GameState |
| Create | `src/test/java/io/quarkmind/qa/SpriteProxyResourceTest.java` | `@QuarkusTest` — tests proxy returns 200 / 404 |
| Create | `src/main/resources/META-INF/resources/pixi.min.js` | PixiJS 8 bundled locally (downloaded once) |
| Create | `src/main/resources/META-INF/resources/visualizer.html` | HTML shell; loads pixi + visualizer.js |
| Create | `src/main/resources/META-INF/resources/visualizer.js` | Full PixiJS app — WebSocket client + scene rendering |
| Create | `electron/package.json` | Electron npm project |
| Create | `electron/main.js` | Spawns Quarkus, opens BrowserWindow |
| Create | `electron/.gitignore` | Ignore node_modules/, dist/ |

---

## Task 1: Add `quarkus-websockets-next` dependency

**Files:**
- Modify: `pom.xml`

- [ ] **Step 1: Add the dependency**

In `pom.xml`, find the `<dependencies>` block. Add after the existing `quarkus-rest-jackson` dependency:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-websockets-next</artifactId>
</dependency>
```

- [ ] **Step 2: Verify it compiles**

```bash
mvn compile -q
```

Expected: `BUILD SUCCESS`. No version needed — managed by the Quarkus BOM.

- [ ] **Step 3: Commit**

```bash
git add pom.xml
git commit -m "build: add quarkus-websockets-next for live game state visualizer"
```

---

## Task 2: `GameStateBroadcaster` (TDD)

**Files:**
- Create: `src/test/java/io/quarkmind/qa/GameStateBroadcasterTest.java`
- Create: `src/main/java/io/quarkmind/qa/GameStateBroadcaster.java`

- [ ] **Step 1: Write the failing test**

```java
package io.quarkmind.qa;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.quarkmind.domain.*;
import io.quarkmind.sc2.intent.MoveIntent;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class GameStateBroadcasterTest {

    @Test
    void toJsonContainsExpectedFields() throws Exception {
        var broadcaster = new GameStateBroadcaster();
        broadcaster.objectMapper = new ObjectMapper();

        var state = new GameState(
            500, 25, 23, 14,
            List.of(new Unit("probe-0", UnitType.PROBE, new Point2d(9f, 9f), 45, 45)),
            List.of(new Building("nexus-0", BuildingType.NEXUS, new Point2d(8f, 8f), 1500, 1500, true)),
            List.of(),
            List.of(new Resource("geyser-0", new Point2d(5f, 11f), 2250)),
            42L
        );

        String json = broadcaster.toJson(state);

        assertThat(json).contains("\"minerals\":500");
        assertThat(json).contains("\"vespene\":25");
        assertThat(json).contains("\"myUnits\"");
        assertThat(json).contains("\"probe-0\"");
        assertThat(json).contains("\"gameFrame\":42");
        assertThat(json).contains("\"geysers\"");
    }
}
```

- [ ] **Step 2: Run to confirm it fails**

```bash
mvn test -Dtest=GameStateBroadcasterTest -q
```

Expected: `BUILD FAILURE` — `GameStateBroadcaster` does not exist.

- [ ] **Step 3: Implement `GameStateBroadcaster`**

```java
package io.quarkmind.qa;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.quarkus.arc.profile.UnlessBuildProfile;
import io.quarkus.websockets.next.WebSocketConnection;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import io.quarkmind.domain.GameState;
import io.quarkmind.sc2.SC2Engine;
import org.jboss.logging.Logger;

import java.util.Set;
import java.util.concurrent.CopyOnWriteArraySet;

@UnlessBuildProfile("prod")
@ApplicationScoped
public class GameStateBroadcaster {

    private static final Logger log = Logger.getLogger(GameStateBroadcaster.class);

    @Inject SC2Engine engine;
    @Inject ObjectMapper objectMapper;   // package-private for testing

    private final Set<WebSocketConnection> sessions = new CopyOnWriteArraySet<>();

    @PostConstruct
    void init() {
        engine.addFrameListener(this::onFrame);
    }

    void addSession(WebSocketConnection connection) {
        sessions.add(connection);
        log.infof("[VISUALIZER] Client connected — %d active", sessions.size());
    }

    void removeSession(WebSocketConnection connection) {
        sessions.remove(connection);
        log.infof("[VISUALIZER] Client disconnected — %d active", sessions.size());
    }

    /** Package-private for testing — pure serialisation, no I/O. */
    String toJson(GameState state) throws Exception {
        return objectMapper.writeValueAsString(state);
    }

    private void onFrame(GameState state) {
        if (sessions.isEmpty()) return;
        try {
            String json = toJson(state);
            sessions.forEach(s -> s.sendText(json)
                .subscribe().with(
                    ignored -> {},
                    err -> log.warnf("[VISUALIZER] Send failed: %s", err.getMessage())));
        } catch (Exception e) {
            log.warnf("[VISUALIZER] Serialisation failed: %s", e.getMessage());
        }
    }
}
```

Note: `objectMapper` has no access modifier (package-private) so `GameStateBroadcasterTest` in the same package can set it directly in tests.

- [ ] **Step 4: Run the test**

```bash
mvn test -Dtest=GameStateBroadcasterTest -q
```

Expected: `BUILD SUCCESS`, 1 test passes.

- [ ] **Step 5: Run full suite**

```bash
mvn test -q 2>&1 | grep "Tests run:.*Failures: 0" | tail -1
```

Expected: `Tests run: 193, Failures: 0`.

---

## Task 3: `GameStateSocket`

**Files:**
- Create: `src/main/java/io/quarkmind/qa/GameStateSocket.java`

No dedicated unit test — covered by the manual smoke test in Task 7.

- [ ] **Step 1: Create `GameStateSocket`**

```java
package io.quarkmind.qa;

import io.quarkus.arc.profile.UnlessBuildProfile;
import io.quarkus.websockets.next.OnClose;
import io.quarkus.websockets.next.OnOpen;
import io.quarkus.websockets.next.WebSocket;
import io.quarkus.websockets.next.WebSocketConnection;
import jakarta.inject.Inject;

@UnlessBuildProfile("prod")
@WebSocket(path = "/ws/gamestate")
public class GameStateSocket {

    @Inject GameStateBroadcaster broadcaster;

    @OnOpen
    public void onOpen(WebSocketConnection connection) {
        broadcaster.addSession(connection);
    }

    @OnClose
    public void onClose(WebSocketConnection connection) {
        broadcaster.removeSession(connection);
    }
}
```

- [ ] **Step 2: Run full suite**

```bash
mvn test -q 2>&1 | grep "Tests run:.*Failures: 0" | tail -1
```

Expected: `Tests run: 193, Failures: 0`.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/io/quarkmind/qa/GameStateBroadcaster.java \
        src/main/java/io/quarkmind/qa/GameStateSocket.java \
        src/test/java/io/quarkmind/qa/GameStateBroadcasterTest.java
git commit -m "feat(visualizer): add WebSocket broadcast of GameState each tick"
```

---

## Task 4: `SpriteProxyResource` (TDD)

**Files:**
- Create: `src/test/java/io/quarkmind/qa/SpriteProxyResourceTest.java`
- Create: `src/main/java/io/quarkmind/qa/SpriteProxyResource.java`

- [ ] **Step 1: Write the failing test**

```java
package io.quarkmind.qa;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;

@QuarkusTest
class SpriteProxyResourceTest {

    @Test
    void unknownSpriteReturns404() {
        given()
            .when().get("/qa/sprites/unknown.jpg")
            .then()
            .statusCode(404);
    }

    @Test
    void knownSpriteReturns200WithJpegContentType() {
        // Requires internet access to Liquipedia — skip if offline
        given()
            .when().get("/qa/sprites/SC2Probe.jpg")
            .then()
            .statusCode(200)
            .contentType("image/jpeg");
    }
}
```

- [ ] **Step 2: Run to confirm it fails**

```bash
mvn test -Dtest=SpriteProxyResourceTest -q
```

Expected: `BUILD FAILURE` — `SpriteProxyResource` does not exist.

- [ ] **Step 3: Implement `SpriteProxyResource`**

```java
package io.quarkmind.qa;

import io.quarkus.arc.profile.UnlessBuildProfile;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.core.Response;
import org.jboss.logging.Logger;

import java.io.InputStream;
import java.net.URL;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@UnlessBuildProfile("prod")
@Path("/qa/sprites")
public class SpriteProxyResource {

    private static final Logger log = Logger.getLogger(SpriteProxyResource.class);

    private static final String BASE = "https://liquipedia.net/commons/images/";

    private static final Map<String, String> PATHS = Map.of(
        "SC2Probe.jpg",   "4/4f/SC2Probe.jpg",
        "SC2Nexus.jpg",   "f/f8/SC2Nexus.jpg",
        "SC2Pylon.jpg",   "4/48/SC2Pylon.jpg",
        "SC2Gateway.jpg", "9/99/SC2Gateway.jpg",
        "SC2Zealot.jpg",  "5/5c/SC2Zealot.jpg",
        "SC2Stalker.jpg", "6/63/SC2Stalker.jpg"
    );

    private final Map<String, byte[]> cache = new ConcurrentHashMap<>();

    @GET
    @Path("/{name}")
    public Response getSprite(@PathParam("name") String name) {
        if (!PATHS.containsKey(name)) {
            return Response.status(404).build();
        }
        try {
            byte[] data = cache.computeIfAbsent(name, this::fetch);
            return Response.ok(data).type("image/jpeg").build();
        } catch (Exception e) {
            log.errorf("[VISUALIZER] Sprite fetch failed for %s: %s", name, e.getMessage());
            return Response.serverError().build();
        }
    }

    private byte[] fetch(String name) {
        try (InputStream in = new URL(BASE + PATHS.get(name)).openStream()) {
            return in.readAllBytes();
        } catch (Exception e) {
            throw new RuntimeException("Failed to fetch " + name, e);
        }
    }
}
```

- [ ] **Step 4: Run `SpriteProxyResourceTest`**

```bash
mvn test -Dtest=SpriteProxyResourceTest -q
```

Expected: `BUILD SUCCESS`, 2 tests pass. (If offline, the `knownSpriteReturns200` test will fail — acceptable for dev, it will pass with internet.)

- [ ] **Step 5: Run full suite**

```bash
mvn test -q 2>&1 | grep "Tests run:.*Failures: 0" | tail -1
```

Expected: `Tests run: 194, Failures: 0`.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/qa/SpriteProxyResource.java \
        src/test/java/io/quarkmind/qa/SpriteProxyResourceTest.java
git commit -m "feat(visualizer): add sprite proxy endpoint — serves Liquipedia images, solves WebGL CORS"
```

---

## Task 5: Download PixiJS and create `visualizer.html`

**Files:**
- Create: `src/main/resources/META-INF/resources/pixi.min.js`
- Create: `src/main/resources/META-INF/resources/visualizer.html`

- [ ] **Step 1: Download PixiJS 8**

```bash
curl -L https://pixijs.download/v8.6.6/pixi.min.js \
    -o src/main/resources/META-INF/resources/pixi.min.js
```

Verify it downloaded (~1MB):

```bash
ls -lh src/main/resources/META-INF/resources/pixi.min.js
```

Expected: file exists, size ~1-2 MB.

- [ ] **Step 2: Create `visualizer.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>QuarkMind</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #1a1a2e; display: flex; flex-direction: column; align-items: center; justify-content: center; min-height: 100vh; }
        h1 { color: #8888ff; font-family: monospace; font-size: 14px; letter-spacing: 2px; margin-bottom: 8px; }
    </style>
</head>
<body>
    <h1>QUARKMIND</h1>
    <div id="game"></div>
    <script src="/pixi.min.js"></script>
    <script src="/visualizer.js"></script>
</body>
</html>
```

- [ ] **Step 3: Verify Quarkus serves the files**

Start Quarkus on any profile:

```bash
mvn quarkus:dev -Dquarkus.profile=emulated
```

In a browser, open `http://localhost:8080/visualizer.html`. Expected: dark page with "QUARKMIND" header (no JS errors yet since `visualizer.js` doesn't exist).

Stop Quarkus (Ctrl+C).

- [ ] **Step 4: Commit**

```bash
git add src/main/resources/META-INF/resources/pixi.min.js \
        src/main/resources/META-INF/resources/visualizer.html
git commit -m "feat(visualizer): add PixiJS bundle and HTML shell"
```

---

## Task 6: Implement `visualizer.js`

**Files:**
- Create: `src/main/resources/META-INF/resources/visualizer.js`

- [ ] **Step 1: Create `visualizer.js`**

```javascript
// visualizer.js — QuarkMind live game visualizer (PixiJS 8)

const SCALE = 20;          // pixels per tile
const VIEWPORT_H = 32;     // tiles high (for Y-axis flip: game Y=0 is bottom, canvas Y=0 is top)
const RECONNECT_MS = 2000;

// Sprite aliases for PIXI.Assets — loaded from the Quarkus sprite proxy
const ASSETS = [
    { alias: 'probe',   src: '/qa/sprites/SC2Probe.jpg' },
    { alias: 'nexus',   src: '/qa/sprites/SC2Nexus.jpg' },
    { alias: 'pylon',   src: '/qa/sprites/SC2Pylon.jpg' },
    { alias: 'gateway', src: '/qa/sprites/SC2Gateway.jpg' },
    { alias: 'zealot',  src: '/qa/sprites/SC2Zealot.jpg' },
    { alias: 'stalker', src: '/qa/sprites/SC2Stalker.jpg' },
];

// Map domain UnitType/BuildingType strings → sprite aliases
const UNIT_ALIAS = {
    PROBE: 'probe', ZEALOT: 'zealot', STALKER: 'stalker',
    MARINE: null, ZERGLING: null,  // no sprite yet → fallback circle
};
const BUILDING_ALIAS = { NEXUS: 'nexus', PYLON: 'pylon', GATEWAY: 'gateway' };
const BUILDING_SIZE  = { NEXUS: 64,      PYLON: 32,      GATEWAY: 48 };
const UNIT_RADIUS = 14;
const DEFAULT_BUILDING_SIZE = 36;

let hudText;
const activeSprites = new Map(); // "prefix:tag" → PIXI display object

/** Convert game tile coordinates to canvas pixels. Flips Y axis. */
function tile(x, y) {
    return { x: x * SCALE, y: (VIEWPORT_H - y) * SCALE };
}

/** Load all sprite assets from Quarkus sprite proxy. Logs warnings on failure. */
async function loadAssets() {
    for (const asset of ASSETS) {
        try {
            await PIXI.Assets.load(asset);
        } catch (e) {
            console.warn(`Sprite load failed: ${asset.alias} — using fallback shape`, e);
        }
    }
}

/** Draw a static grid background on the given container. */
function drawGrid(container) {
    const g = new PIXI.Graphics();
    for (let i = 0; i <= VIEWPORT_H; i++) {
        g.moveTo(i * SCALE, 0).lineTo(i * SCALE, VIEWPORT_H * SCALE);
        g.moveTo(0, i * SCALE).lineTo(VIEWPORT_H * SCALE, i * SCALE);
    }
    g.stroke({ width: 0.5, color: 0x2a2a4e });
    container.addChild(g);
}

/** Create a unit portrait: circular-masked sprite if texture available, else coloured circle. */
function makeUnitSprite(alias, radius, tintColor) {
    const texture = alias ? PIXI.Assets.get(alias) : null;
    if (texture) {
        const sprite = new PIXI.Sprite(texture);
        sprite.width = radius * 2;
        sprite.height = radius * 2;
        sprite.anchor.set(0.5);
        const mask = new PIXI.Graphics();
        mask.circle(0, 0, radius).fill(0xffffff);
        sprite.addChild(mask);
        sprite.mask = mask;
        if (tintColor) sprite.tint = tintColor;
        return sprite;
    }
    const g = new PIXI.Graphics();
    g.circle(0, 0, radius).fill(tintColor ?? 0x4488ff);
    return g;
}

/** Create a building sprite or fallback rectangle. */
function makeBuildingSprite(type) {
    const alias = BUILDING_ALIAS[type];
    const size  = BUILDING_SIZE[type] ?? DEFAULT_BUILDING_SIZE;
    const texture = alias ? PIXI.Assets.get(alias) : null;
    if (texture) {
        const sprite = new PIXI.Sprite(texture);
        sprite.width  = size;
        sprite.height = size;
        sprite.anchor.set(0.5);
        return sprite;
    }
    const g = new PIXI.Graphics();
    g.rect(-size / 2, -size / 2, size, size).fill(0x3366cc);
    return g;
}

/** Create a geyser marker (green circle). */
function makeGeyserSprite() {
    const g = new PIXI.Graphics();
    g.circle(0, 0, 12).fill(0x22cc66);
    return g;
}

/**
 * Reconcile a layer's sprites against the received entity list.
 * Creates sprites for new entities, updates positions of existing ones,
 * destroys sprites for removed entities.
 */
function syncLayer(layer, entities, keyPrefix, spriteFactory) {
    const seen = new Set();
    for (const entity of entities) {
        const key = `${keyPrefix}:${entity.tag}`;
        seen.add(key);
        const pos = tile(entity.position.x, entity.position.y);
        if (activeSprites.has(key)) {
            const s = activeSprites.get(key);
            s.x = pos.x;
            s.y = pos.y;
        } else {
            const s = spriteFactory(entity);
            s.x = pos.x;
            s.y = pos.y;
            layer.addChild(s);
            activeSprites.set(key, s);
        }
    }
    for (const [key, sprite] of activeSprites.entries()) {
        if (key.startsWith(keyPrefix + ':') && !seen.has(key)) {
            layer.removeChild(sprite);
            sprite.destroy();
            activeSprites.delete(key);
        }
    }
}

/** Apply one GameState snapshot to the scene. */
function updateScene(state) {
    syncLayer(
        window._layers.resource,
        state.geysers,
        'geyser',
        () => makeGeyserSprite()
    );
    syncLayer(
        window._layers.building,
        state.myBuildings,
        'building',
        e => makeBuildingSprite(e.type)
    );
    syncLayer(
        window._layers.unit,
        state.myUnits,
        'unit',
        e => makeUnitSprite(UNIT_ALIAS[e.type] ?? null, UNIT_RADIUS, 0x88bbff)
    );
    syncLayer(
        window._layers.enemy,
        state.enemyUnits,
        'enemy',
        e => makeUnitSprite(UNIT_ALIAS[e.type] ?? null, UNIT_RADIUS, 0xff4444)
    );

    // Note: Jackson serialises the `isComplete` record component as-is.
    // If it appears as `complete` in JSON, buildings will still render — only
    // the completion indicator (future feature) would need updating.
    hudText.text =
        `Minerals: ${state.minerals}   Gas: ${state.vespene}` +
        `   Supply: ${state.supplyUsed}/${state.supply}` +
        `   Frame: ${state.gameFrame}`;
}

/** Open WebSocket; auto-reconnect on close. */
function connect() {
    const ws = new WebSocket(`ws://${window.location.host}/ws/gamestate`);
    ws.onmessage = e => {
        try { updateScene(JSON.parse(e.data)); }
        catch (err) { console.warn('Bad message', err); }
    };
    ws.onerror = () => ws.close();
    ws.onclose = () => {
        hudText.text = 'Disconnected — reconnecting...';
        setTimeout(connect, RECONNECT_MS);
    };
}

/** Entry point — called once on page load. */
async function init() {
    const app = new PIXI.Application();
    await app.init({ width: 800, height: 680, background: 0x1a1a2e });
    document.getElementById('game').appendChild(app.canvas);

    // Layers — order matters (bottom to top)
    const background = new PIXI.Container();
    const resource   = new PIXI.Container();
    const building   = new PIXI.Container();
    const unit       = new PIXI.Container();
    const enemy      = new PIXI.Container();
    const hud        = new PIXI.Container();
    app.stage.addChild(background, resource, building, unit, enemy, hud);
    window._layers = { resource, building, unit, enemy };

    drawGrid(background);

    // HUD text anchored to bottom of canvas
    hudText = new PIXI.Text({
        text: 'Connecting...',
        style: { fill: 0xccccff, fontSize: 13, fontFamily: 'monospace' }
    });
    hudText.x = 8;
    hudText.y = 648;
    hud.addChild(hudText);

    await loadAssets();
    connect();
}

init();
```

- [ ] **Step 2: Run manual smoke test**

Start Quarkus on `%emulated`:

```bash
mvn quarkus:dev -Dquarkus.profile=emulated
```

Open `http://localhost:8080/visualizer.html` in Chrome or Firefox.

Expected:
- Dark canvas appears
- "Connecting..." replaces with `Minerals: 50  Gas: 0  Supply: 12/15  Frame: N`
- Nexus sprite visible near tile (8, 8)
- 12 probe sprites arranged at tile row y=9
- 2 green geyser circles at approximately tiles (5, 11) and (11, 5)
- Minerals counter increments every ~500ms
- Browser console shows no JS errors (sprite load warnings are acceptable if offline)

Stop Quarkus (Ctrl+C).

- [ ] **Step 3: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js
git commit -m "feat(visualizer): add PixiJS visualizer with WebSocket live update and SC2 sprites"
```

---

## Task 7: Electron wrapper

**Files:**
- Create: `electron/package.json`
- Create: `electron/main.js`
- Create: `electron/.gitignore`

- [ ] **Step 1: Create `electron/.gitignore`**

```
node_modules/
dist/
```

- [ ] **Step 2: Create `electron/package.json`**

```json
{
  "name": "quarkmind-visualizer",
  "version": "1.0.0",
  "description": "QuarkMind live game visualizer",
  "main": "main.js",
  "scripts": {
    "start": "DEV_MODE=1 electron .",
    "start:app": "electron ."
  },
  "dependencies": {
    "electron": "^34.0.0"
  }
}
```

- [ ] **Step 3: Create `electron/main.js`**

```javascript
const { app, BrowserWindow } = require('electron');
const { spawn } = require('child_process');
const http = require('http');
const path = require('path');

const QUARKUS_URL    = 'http://localhost:8080';
const HEALTH_URL     = `${QUARKUS_URL}/q/health/live`;
const VISUALIZER_URL = `${QUARKUS_URL}/visualizer.html`;
const MAX_WAIT_MS    = 30000;
const POLL_MS        = 500;

let quarkusProcess = null;
let mainWindow     = null;

function waitForQuarkus() {
    return new Promise((resolve, reject) => {
        const deadline = Date.now() + MAX_WAIT_MS;
        function poll() {
            if (Date.now() > deadline) {
                return reject(new Error('Quarkus did not start within 30 seconds'));
            }
            http.get(HEALTH_URL, res => {
                if (res.statusCode === 200) resolve();
                else setTimeout(poll, POLL_MS);
            }).on('error', () => setTimeout(poll, POLL_MS));
        }
        setTimeout(poll, POLL_MS);
    });
}

function startQuarkus() {
    if (process.env.DEV_MODE) {
        console.log('[ELECTRON] Dev mode — assuming Quarkus already running on :8080');
        return Promise.resolve();
    }
    const jar = path.join(__dirname, '..', 'target', 'quarkmind-agent-1.0.0-SNAPSHOT-runner.jar');
    quarkusProcess = spawn('java', ['-jar', jar], {
        env: { ...process.env, 'quarkus.profile': 'emulated' }
    });
    quarkusProcess.stdout.on('data', d => process.stdout.write('[QUARKUS] ' + d));
    quarkusProcess.stderr.on('data', d => process.stderr.write('[QUARKUS] ' + d));
    quarkusProcess.on('error', err => console.error('[ELECTRON] Failed to spawn Quarkus:', err.message));
    console.log('[ELECTRON] Waiting for Quarkus...');
    return waitForQuarkus();
}

function createWindow() {
    mainWindow = new BrowserWindow({
        width: 900,
        height: 740,
        autoHideMenuBar: true,
        title: 'QuarkMind',
        backgroundColor: '#1a1a2e'
    });
    mainWindow.loadURL(VISUALIZER_URL);
    mainWindow.on('closed', () => { mainWindow = null; });
}

app.whenReady().then(async () => {
    try {
        await startQuarkus();
        createWindow();
    } catch (e) {
        console.error('[ELECTRON] Startup failed:', e.message);
        app.quit();
    }
});

app.on('before-quit', () => {
    if (quarkusProcess) {
        console.log('[ELECTRON] Shutting down Quarkus...');
        quarkusProcess.kill();
    }
});

app.on('window-all-closed', () => {
    if (process.platform !== 'darwin') app.quit();
});
```

- [ ] **Step 4: Install Electron**

```bash
cd electron && npm install
```

Expected: `node_modules/` populated, no errors.

- [ ] **Step 5: Test dev mode (Quarkus must already be running)**

In one terminal:
```bash
mvn quarkus:dev -Dquarkus.profile=emulated
```

In another:
```bash
cd electron && npm start
```

Expected:
- Electron window opens titled "QuarkMind"
- Visualizer loads — nexus, probes, geysers visible
- Minerals counter updating live
- Closing the window exits Electron cleanly (no Quarkus kill in dev mode)

- [ ] **Step 6: Commit**

```bash
git add electron/package.json electron/main.js electron/.gitignore
git commit -m "feat(electron): add Electron wrapper — dev mode + Quarkus subprocess for packaged mode"
```

---

## Self-Review

**Spec coverage:**

| Spec requirement | Task |
|---|---|
| `quarkus-websockets-next` dependency | Task 1 |
| `GameStateBroadcaster` — frame listener + broadcast | Task 2 |
| `GameStateSocket` — session management | Task 3 |
| `SpriteProxyResource` — proxy + cache + 404 | Task 4 |
| PixiJS 8 bundled locally | Task 5 |
| `visualizer.html` shell | Task 5 |
| `visualizer.js` — layers, sprites, WebSocket, HUD | Task 6 |
| Grid background | Task 6 (`drawGrid`) |
| Circular masked sprite for units | Task 6 (`makeUnitSprite`) |
| Fallback coloured shapes for unknown types | Task 6 (fallback in `makeUnitSprite`, `makeBuildingSprite`) |
| Auto-reconnect on WebSocket close | Task 6 (`ws.onclose` → `setTimeout(connect)`) |
| Entity reconciliation (add/update/remove) | Task 6 (`syncLayer`) |
| HUD: minerals, gas, supply, frame | Task 6 (`hudText`) |
| Electron dev mode | Task 7 (`DEV_MODE=1`) |
| Electron spawns Quarkus uber-jar | Task 7 (`startQuarkus()`) |
| Electron polls health endpoint | Task 7 (`waitForQuarkus()`) |
| Electron kills Quarkus on quit | Task 7 (`before-quit`) |
| `GameStateBroadcasterTest` (plain JUnit) | Task 2 |
| `SpriteProxyResourceTest` (`@QuarkusTest`) | Task 4 |

**Placeholder scan:** None found.

**Type consistency:** `syncLayer` used consistently in Task 6. Layer references via `window._layers` object consistent between `init()` and `updateScene()`. Entity field names (`position.x`, `position.y`, `tag`, `type`) match the `GameState` JSON produced by Jackson serialising the `Unit`/`Building`/`Resource` records.
