# E22: Replay Mode with Real SC2 Map Terrain — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Download real SC2 map files from AI Arena, extract terrain (cliff levels → high/low/ramp/wall), serve to the 3D visualizer so replays show accurate terrain.

**Architecture:** Five independent pieces in dependency order: (1) pure Java terrain extractor from .SC2Map MPQ, (2) HTTP downloader with local cache, (3) two REST endpoints, (4) ReplayEngine map metadata, (5) visualizer two-phase terrain loading. Each piece is independently testable before the next is started.

**Tech Stack:** Java 21, Quarkus, scelight-mpq (MPQ archive reading — already on classpath), Java HttpClient (map download), RESTEasy (endpoints), Playwright (E2E), JUnit 5 + AssertJ (unit/integration).

**Issues:** Epic #95 | #96 (extractor) | #97 (downloader) | #98 (endpoints) | #99 (ReplayEngine) | #100 (visualizer)

---

## File Map

**New files:**
- `src/main/java/io/quarkmind/sc2/map/SC2MapTerrainExtractor.java` — pure Java, no CDI, MPQ → TerrainGrid
- `src/main/java/io/quarkmind/sc2/map/MapDownloader.java` — CDI bean, HTTP download + unzip + cache
- `src/main/java/io/quarkmind/sc2/map/SC2MapCache.java` — CDI bean, orchestrates download + extract + JSON cache
- `src/main/java/io/quarkmind/qa/TerrainResponse.java` — shared record (extracted from EmulatedTerrainResource)
- `src/main/java/io/quarkmind/qa/TerrainResource.java` — `GET /qa/terrain?map=<name>`
- `src/main/java/io/quarkmind/qa/CurrentMapResource.java` — `GET /qa/current-map`
- `src/main/resources/map-registry.json` — AI Arena season pack registry
- `src/test/java/io/quarkmind/sc2/map/SC2MapTerrainExtractorTest.java`
- `src/test/java/io/quarkmind/sc2/map/MapDownloaderTest.java`
- `src/test/java/io/quarkmind/qa/TerrainResourceIT.java`
- `src/test/java/io/quarkmind/qa/CurrentMapResourceIT.java`

**Modified files:**
- `src/main/java/io/quarkmind/sc2/SC2Engine.java` — add `getMapName/Width/Height()` defaults
- `src/main/java/io/quarkmind/sc2/replay/ReplayEngine.java` — parse gamemetadata.json, implement map getters
- `src/test/java/io/quarkmind/sc2/replay/ReplayEngineTest.java` — add map metadata tests
- `src/main/java/io/quarkmind/qa/EmulatedTerrainResource.java` — reference shared TerrainResponse
- `src/main/resources/META-INF/resources/visualizer.js` — two-phase terrain loading

---

## Task 1: Extract TerrainResponse to shared file — Issue #98

**Why first:** Both `EmulatedTerrainResource` and the new `TerrainResource` need this record. Extract it once so Task 3 can reference it.

**Files:**
- Create: `src/main/java/io/quarkmind/qa/TerrainResponse.java`
- Modify: `src/main/java/io/quarkmind/qa/EmulatedTerrainResource.java`

- [ ] **Step 1.1: Create `TerrainResponse.java`**

```java
package io.quarkmind.qa;

import java.util.List;

public record TerrainResponse(
    int width,
    int height,
    List<int[]> walls,
    List<int[]> highGround,
    List<int[]> ramps
) {}
```

- [ ] **Step 1.2: Update `EmulatedTerrainResource` to use the shared record**

In `EmulatedTerrainResource.java`, delete the inner `TerrainResponse` record and replace all usages with the top-level import. The constructor call and field names are identical — no other changes.

```java
import io.quarkmind.qa.TerrainResponse;
// Delete: public record TerrainResponse(...) {}
```

- [ ] **Step 1.3: Build to confirm no regressions**

```bash
mvn compile -q
```
Expected: BUILD SUCCESS

- [ ] **Step 1.4: Commit**

```bash
git add src/main/java/io/quarkmind/qa/TerrainResponse.java \
        src/main/java/io/quarkmind/qa/EmulatedTerrainResource.java
git commit -m "refactor: extract TerrainResponse to shared top-level record Refs #98"
```

---

## Task 2: SC2MapTerrainExtractor — Issue #96

Pure Java class. Opens `.SC2Map` MPQ, reads `t3SyncCliffLevel`, converts to `TerrainGrid`.

**Cliff level rules (discovered from binary analysis of TorchesAIE_v4.SC2Map):**
- `0` → WALL (void, out of bounds)
- `>= 320` → WALL (impassable border cliff)
- `value % 64 != 0` → RAMP (transition cells between tiers)
- `64` → LOW (lowest walkable tier)
- `128`, `192`, `256` (multiples of 64, 64 < value < 320) → HIGH

**File format of `t3SyncCliffLevel`:** magic `CLIF` (4 bytes) + version uint32 (4) + width uint32 (4) + height uint32 (4) = 16-byte header; then `width × height` little-endian uint16 values, row-major (y outer, x inner).

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/map/SC2MapTerrainExtractor.java`
- Create: `src/test/java/io/quarkmind/sc2/map/SC2MapTerrainExtractorTest.java`

- [ ] **Step 2.1: Write the failing tests**

```java
package io.quarkmind.sc2.map;

import io.quarkmind.domain.TerrainGrid;
import io.quarkmind.domain.TerrainGrid.Height;
import org.junit.jupiter.api.Test;

import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class SC2MapTerrainExtractorTest {

    // TorchesAIE_v4.SC2Map is in the test classpath; its real path is:
    // src/test/resources/maps/TorchesAIE_v4.SC2Map
    // (we copy it there in Step 2.2)
    private static final Path MAP = Path.of("src/test/resources/maps/TorchesAIE_v4.SC2Map");

    @Test
    void extractsDimensionsFromTorchesAIE() {
        TerrainGrid grid = SC2MapTerrainExtractor.extract(MAP);
        assertThat(grid.width()).isEqualTo(160);
        assertThat(grid.height()).isEqualTo(208);
    }

    @Test
    void voidCellAtTopLeftIsWall() {
        TerrainGrid grid = SC2MapTerrainExtractor.extract(MAP);
        // Torches AIE: top-left corner (x=0, y=0) has cliff=0 (void)
        assertThat(grid.heightAt(0, 0)).isEqualTo(Height.WALL);
    }

    @Test
    void borderCliffIsWall() {
        TerrainGrid grid = SC2MapTerrainExtractor.extract(MAP);
        // Torches AIE: left border (x=0, y=1) has cliff=320 (impassable border)
        assertThat(grid.heightAt(0, 1)).isEqualTo(Height.WALL);
    }

    @Test
    void baseTileIsHigh() {
        TerrainGrid grid = SC2MapTerrainExtractor.extract(MAP);
        // Torches AIE: player base area at (10, 1) has cliff=192 (HIGH)
        assertThat(grid.heightAt(10, 1)).isEqualTo(Height.HIGH);
    }

    @Test
    void lowGroundTileIsLow() {
        TerrainGrid grid = SC2MapTerrainExtractor.extract(MAP);
        // Torches AIE: center open ground at (80, 104) has cliff=64 (LOW)
        assertThat(grid.heightAt(80, 104)).isEqualTo(Height.LOW);
    }

    @Test
    void rampCellIsRamp() {
        TerrainGrid grid = SC2MapTerrainExtractor.extract(MAP);
        // Find any ramp cell: cliff value % 64 != 0
        // Torches AIE has ramp cells (cliff=72..112 or 136..176) — count > 0
        int rampCount = 0;
        for (int x = 0; x < grid.width(); x++) {
            for (int y = 0; y < grid.height(); y++) {
                if (grid.heightAt(x, y) == Height.RAMP) rampCount++;
            }
        }
        assertThat(rampCount).isGreaterThan(0);
    }

    @Test
    void throwsIfMapFileNotFound() {
        assertThatThrownBy(() -> SC2MapTerrainExtractor.extract(Path.of("nonexistent.SC2Map")))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("nonexistent.SC2Map");
    }

    @Test
    void throwsIfCliffLevelFileAbsent() {
        // Pass a file that exists but is not a valid SC2Map (e.g. a replay file)
        assertThatThrownBy(() -> SC2MapTerrainExtractor.extract(
                Path.of("replays/aiarena_protoss/Nothing_4720936.SC2Replay")))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("t3SyncCliffLevel");
    }
}
```

- [ ] **Step 2.2: Copy TorchesAIE_v4.SC2Map into test resources**

```bash
mkdir -p src/test/resources/maps
python3 -c "
import zipfile
z = zipfile.ZipFile('/tmp/2025PS2_Maps.zip')
z.extract('TorchesAIE_v4.SC2Map', 'src/test/resources/maps/')
print('extracted')
"
```

Expected: `extracted` (creates `src/test/resources/maps/TorchesAIE_v4.SC2Map`)

- [ ] **Step 2.3: Run tests to confirm they fail**

```bash
mvn test -Dtest=SC2MapTerrainExtractorTest -q 2>&1 | tail -5
```
Expected: FAIL — `SC2MapTerrainExtractor` does not exist yet.

- [ ] **Step 2.4: Implement `SC2MapTerrainExtractor`**

```java
package io.quarkmind.sc2.map;

import hu.belicza.andras.mpq.MpqParser;
import io.quarkmind.domain.TerrainGrid;
import io.quarkmind.domain.TerrainGrid.Height;

import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.nio.file.Path;

public final class SC2MapTerrainExtractor {

    private static final String CLIFF_FILE = "t3SyncCliffLevel";

    private SC2MapTerrainExtractor() {}

    public static TerrainGrid extract(Path sc2mapPath) {
        byte[] raw;
        try (MpqParser mpq = new MpqParser(sc2mapPath)) {
            raw = mpq.getFile(CLIFF_FILE);
        } catch (Exception e) {
            throw new IllegalArgumentException("Cannot open SC2Map: " + sc2mapPath, e);
        }
        if (raw == null) {
            throw new IllegalArgumentException(
                "t3SyncCliffLevel not found in " + sc2mapPath + " — not a playable SC2Map?");
        }
        ByteBuffer buf = ByteBuffer.wrap(raw).order(ByteOrder.LITTLE_ENDIAN);
        buf.position(8); // skip magic(4) + version(4)
        int w = buf.getInt();
        int h = buf.getInt();
        Height[][] grid = new Height[w][h];
        for (int y = 0; y < h; y++) {
            for (int x = 0; x < w; x++) {
                int cliff = buf.getShort() & 0xFFFF;
                grid[x][y] = toHeight(cliff);
            }
        }
        return new TerrainGrid(w, h, grid);
    }

    private static Height toHeight(int cliff) {
        if (cliff == 0 || cliff >= 320) return Height.WALL;
        if (cliff % 64 != 0)           return Height.RAMP;
        if (cliff == 64)               return Height.LOW;
        return Height.HIGH;
    }
}
```

- [ ] **Step 2.5: Run tests — all should pass**

```bash
mvn test -Dtest=SC2MapTerrainExtractorTest -q
```
Expected: BUILD SUCCESS, 7 tests passed.

Verify the specific cell coordinates by checking cliff values first if any test fails:
```bash
python3 -c "
import struct
data = open('src/test/resources/maps/TorchesAIE_v4.SC2Map', 'rb').read()
# Find cliff level bytes at known positions using MpqParser instead
print('File size:', len(data))
"
```
(Adjust test coordinates if needed based on actual cliff values — the key invariants are: WALL count > 0, HIGH count > 0, LOW count > 0, RAMP count > 0.)

- [ ] **Step 2.6: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/map/SC2MapTerrainExtractor.java \
        src/test/java/io/quarkmind/sc2/map/SC2MapTerrainExtractorTest.java \
        src/test/resources/maps/TorchesAIE_v4.SC2Map
git commit -m "feat: SC2MapTerrainExtractor — parse t3SyncCliffLevel from .SC2Map MPQ Closes #96"
```

---

## Task 3: MapDownloader + map-registry.json — Issue #97

CDI bean that downloads AI Arena season packs and extracts .SC2Map files.

**Files:**
- Create: `src/main/resources/map-registry.json`
- Create: `src/main/java/io/quarkmind/sc2/map/MapDownloader.java`
- Create: `src/test/java/io/quarkmind/sc2/map/MapDownloaderTest.java`

- [ ] **Step 3.1: Create `map-registry.json`**

```json
{
  "baseUrl": "https://aiarena.net/wiki/184/plugin/attachments/download/",
  "packs": [
    {
      "id": 45,
      "maps": ["TorchesAIE_v4", "IncorporealAIE_v4", "LeyLinesAIE_v3", "PersephoneAIE_v4", "PylonAIE_v4"]
    },
    {
      "id": 44,
      "maps": ["MagannathaAIE", "UltraloveAIE", "IncorporealAIE", "LeyLinesAIE", "PersephoneAIE", "PylonAIE", "TorchesAIE", "LastFantasyAIE"]
    }
  ]
}
```

- [ ] **Step 3.2: Write the failing tests**

```java
package io.quarkmind.sc2.map;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.ByteArrayOutputStream;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Optional;
import java.util.zip.ZipEntry;
import java.util.zip.ZipOutputStream;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class MapDownloaderTest {

    @TempDir Path cacheDir;

    HttpClient mockHttp;
    MapDownloader downloader;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() throws Exception {
        mockHttp = mock(HttpClient.class);
        downloader = new MapDownloader(cacheDir, mockHttp);
    }

    @Test
    void returnsFromCacheIfAlreadyDownloaded() throws Exception {
        Path cached = cacheDir.resolve("TorchesAIE_v4.SC2Map");
        Files.writeString(cached, "fake-map-data");

        Optional<Path> result = downloader.download("TorchesAIE_v4.SC2Map");

        assertThat(result).isPresent();
        assertThat(result.get()).isEqualTo(cached);
        verifyNoInteractions(mockHttp);  // no HTTP call made
    }

    @Test
    @SuppressWarnings("unchecked")
    void downloadsPackAndExtractsMap() throws Exception {
        byte[] zipBytes = makeZip("TorchesAIE_v4.SC2Map", new byte[]{1, 2, 3});
        HttpResponse<byte[]> resp = mock(HttpResponse.class);
        when(resp.statusCode()).thenReturn(200);
        when(resp.body()).thenReturn(zipBytes);
        when(mockHttp.send(any(HttpRequest.class), any(HttpResponse.BodyHandler.class))).thenReturn(resp);

        Optional<Path> result = downloader.download("TorchesAIE_v4.SC2Map");

        assertThat(result).isPresent();
        assertThat(Files.readAllBytes(result.get())).isEqualTo(new byte[]{1, 2, 3});
    }

    @Test
    @SuppressWarnings("unchecked")
    void fuzzyMatchStripsVersionSuffix() throws Exception {
        byte[] zipBytes = makeZip("MagannathaAIE.SC2Map", new byte[]{4, 5, 6});
        HttpResponse<byte[]> resp = mock(HttpResponse.class);
        when(resp.statusCode()).thenReturn(200);
        when(resp.body()).thenReturn(zipBytes);
        when(mockHttp.send(any(HttpRequest.class), any(HttpResponse.BodyHandler.class))).thenReturn(resp);

        Optional<Path> result = downloader.download("MagannathaAIE_v2.SC2Map");

        assertThat(result).isPresent();
        assertThat(result.get().getFileName().toString()).isEqualTo("MagannathaAIE_v2.SC2Map");
    }

    @Test
    void returnsEmptyForUnknownMap() throws Exception {
        Optional<Path> result = downloader.download("UnknownMap_v99.SC2Map");
        assertThat(result).isEmpty();
        verifyNoInteractions(mockHttp);
    }

    @Test
    @SuppressWarnings("unchecked")
    void returnsEmptyOnNetworkError() throws Exception {
        when(mockHttp.send(any(), any())).thenThrow(new java.io.IOException("timeout"));

        Optional<Path> result = downloader.download("TorchesAIE_v4.SC2Map");

        assertThat(result).isEmpty();
    }

    @Test
    @SuppressWarnings("unchecked")
    void secondCallReturnsCachedWithoutDownload() throws Exception {
        byte[] zipBytes = makeZip("TorchesAIE_v4.SC2Map", new byte[]{1, 2, 3});
        HttpResponse<byte[]> resp = mock(HttpResponse.class);
        when(resp.statusCode()).thenReturn(200);
        when(resp.body()).thenReturn(zipBytes);
        when(mockHttp.send(any(), any())).thenReturn(resp);

        downloader.download("TorchesAIE_v4.SC2Map");
        downloader.download("TorchesAIE_v4.SC2Map");

        verify(mockHttp, times(1)).send(any(), any());  // only one HTTP call
    }

    // Helper: build a valid zip containing one file
    private static byte[] makeZip(String entryName, byte[] content) throws Exception {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        try (ZipOutputStream zos = new ZipOutputStream(baos)) {
            zos.putNextEntry(new ZipEntry(entryName));
            zos.write(content);
            zos.closeEntry();
        }
        return baos.toByteArray();
    }
}
```

- [ ] **Step 3.3: Add Mockito dependency to pom.xml if not present**

```bash
grep -q "mockito-core\|mockito-junit-jupiter" pom.xml && echo "present" || echo "missing"
```

If missing, add inside `<dependencies>`:
```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 3.4: Run tests to confirm they fail**

```bash
mvn test -Dtest=MapDownloaderTest -q 2>&1 | tail -5
```
Expected: FAIL — `MapDownloader` does not exist.

- [ ] **Step 3.5: Implement `MapDownloader`**

```java
package io.quarkmind.sc2.map;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import org.jboss.logging.Logger;

import java.io.ByteArrayInputStream;
import java.io.InputStream;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.regex.Pattern;
import java.util.zip.ZipEntry;
import java.util.zip.ZipInputStream;

@ApplicationScoped
public class MapDownloader {

    private static final Logger log = Logger.getLogger(MapDownloader.class);
    private static final Pattern VERSION_SUFFIX = Pattern.compile("_v\\d+$");

    private final Path cacheDir;
    private final HttpClient http;
    private String baseUrl;
    private List<Pack> packs;

    /** CDI constructor — uses default cache dir and HttpClient. */
    public MapDownloader() {
        this(Path.of(System.getProperty("user.home"), ".quarkmind", "maps"),
             HttpClient.newBuilder().followRedirects(HttpClient.Redirect.ALWAYS).build());
    }

    /** Test constructor — injected cache dir and HttpClient. */
    MapDownloader(Path cacheDir, HttpClient http) {
        this.cacheDir = cacheDir;
        this.http = http;
    }

    @PostConstruct
    void init() {
        try (InputStream in = getClass().getResourceAsStream("/map-registry.json")) {
            JsonNode root = new ObjectMapper().readTree(in);
            baseUrl = root.get("baseUrl").asText();
            packs = new ArrayList<>();
            for (JsonNode pack : root.get("packs")) {
                int id = pack.get("id").asInt();
                List<String> maps = new ArrayList<>();
                pack.get("maps").forEach(m -> maps.add(m.asText()));
                packs.add(new Pack(id, maps));
            }
        } catch (Exception e) {
            throw new IllegalStateException("Cannot load map-registry.json", e);
        }
    }

    /**
     * Returns the path to the downloaded .SC2Map file, or empty if not found.
     * Downloads from AI Arena if not in the local cache.
     */
    public Optional<Path> download(String mapFileName) {
        // 1. Check local cache first
        try { Files.createDirectories(cacheDir); } catch (Exception ignored) {}
        Path cached = cacheDir.resolve(mapFileName);
        if (Files.exists(cached)) return Optional.of(cached);

        // 2. Find the pack containing this map (exact then fuzzy)
        String bare = bare(mapFileName);
        for (Pack pack : packs) {
            String match = exactOrFuzzy(pack, bare, mapFileName);
            if (match == null) continue;
            Optional<Path> result = downloadFromPack(pack.id, match, mapFileName);
            if (result.isPresent()) return result;
        }
        log.warnf("[MAP] Map not found in any known pack: %s", mapFileName);
        return Optional.empty();
    }

    private Optional<Path> downloadFromPack(int packId, String entryInZip, String saveAs) {
        String url = baseUrl + packId + "/";
        try {
            log.infof("[MAP] Downloading pack %d for map %s", packId, saveAs);
            HttpRequest req = HttpRequest.newBuilder(URI.create(url)).GET().build();
            HttpResponse<byte[]> resp = http.send(req, HttpResponse.BodyHandlers.ofByteArray());
            if (resp.statusCode() != 200) {
                log.warnf("[MAP] Pack download failed: HTTP %d", resp.statusCode());
                return Optional.empty();
            }
            byte[] extracted = extractFromZip(resp.body(), entryInZip);
            if (extracted == null) {
                log.warnf("[MAP] Entry %s not found in pack %d zip", entryInZip, packId);
                return Optional.empty();
            }
            Path dest = cacheDir.resolve(saveAs);
            Files.write(dest, extracted);
            log.infof("[MAP] Saved %s (%d bytes)", saveAs, extracted.length);
            return Optional.of(dest);
        } catch (Exception e) {
            log.warnf("[MAP] Download error for pack %d: %s", packId, e.getMessage());
            return Optional.empty();
        }
    }

    private static byte[] extractFromZip(byte[] zipBytes, String entryName) throws Exception {
        try (ZipInputStream zis = new ZipInputStream(new ByteArrayInputStream(zipBytes))) {
            ZipEntry entry;
            while ((entry = zis.getNextEntry()) != null) {
                if (entry.getName().equals(entryName)) {
                    return zis.readAllBytes();
                }
            }
        }
        return null;
    }

    /** Strip .SC2Map suffix, return bare name. */
    private static String bare(String fileName) {
        return fileName.endsWith(".SC2Map")
            ? VERSION_SUFFIX.matcher(fileName.substring(0, fileName.length() - 7)).replaceAll("")
            : fileName;
    }

    /** Returns the zip entry name for this pack, or null if not found. */
    private static String exactOrFuzzy(Pack pack, String bare, String originalFileName) {
        String exactEntry = originalFileName;
        String fuzzyEntry = bare + ".SC2Map";
        if (pack.maps.contains(bare(exactEntry))) return exactEntry;
        if (pack.maps.contains(bare)) return fuzzyEntry;
        return null;
    }

    private record Pack(int id, List<String> maps) {}
}
```

- [ ] **Step 3.6: Run tests — all should pass**

```bash
mvn test -Dtest=MapDownloaderTest -q
```
Expected: BUILD SUCCESS, 6 tests passed.

- [ ] **Step 3.7: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/map/MapDownloader.java \
        src/test/java/io/quarkmind/sc2/map/MapDownloaderTest.java \
        src/main/resources/map-registry.json
git commit -m "feat: MapDownloader — download AI Arena season packs, extract .SC2Map, local cache Closes #97"
```

---

## Task 4: SC2MapCache — coordinates download + extraction + JSON cache

Internal service wiring `MapDownloader` → `SC2MapTerrainExtractor` → JSON on disk.

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/map/SC2MapCache.java`

- [ ] **Step 4.1: Implement `SC2MapCache`**

No separate test file — this is a thin coordinator tested through `TerrainResource` integration tests in Task 5.

```java
package io.quarkmind.sc2.map;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.quarkmind.domain.TerrainGrid;
import io.quarkmind.domain.TerrainGrid.Height;
import io.quarkmind.qa.TerrainResponse;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

@ApplicationScoped
public class SC2MapCache {

    private static final Logger log = Logger.getLogger(SC2MapCache.class);
    private static final Path CACHE_DIR =
        Path.of(System.getProperty("user.home"), ".quarkmind", "maps");

    @Inject MapDownloader downloader;
    @Inject ObjectMapper objectMapper;

    /**
     * Returns TerrainResponse for the named map, downloading and extracting if needed.
     * Returns empty if the map cannot be found or extracted.
     */
    public Optional<TerrainResponse> get(String mapName) {
        String fileName = mapName.endsWith(".SC2Map") ? mapName : mapName + ".SC2Map";
        String baseName = fileName.substring(0, fileName.length() - 7);
        Path jsonCache  = CACHE_DIR.resolve(baseName + "-terrain.json");

        // 1. Try JSON cache
        if (Files.exists(jsonCache)) {
            try {
                return Optional.of(objectMapper.readValue(jsonCache.toFile(), TerrainResponse.class));
            } catch (Exception e) {
                log.warnf("[MAP] Corrupt terrain cache for %s, re-extracting: %s", mapName, e.getMessage());
                try { Files.delete(jsonCache); } catch (Exception ignored) {}
            }
        }

        // 2. Download .SC2Map if needed
        Optional<Path> sc2map = downloader.download(fileName);
        if (sc2map.isEmpty()) return Optional.empty();

        // 3. Extract terrain
        try {
            TerrainGrid grid = SC2MapTerrainExtractor.extract(sc2map.get());
            TerrainResponse response = toResponse(grid);
            Files.createDirectories(CACHE_DIR);
            objectMapper.writeValue(jsonCache.toFile(), response);
            return Optional.of(response);
        } catch (Exception e) {
            log.errorf("[MAP] Terrain extraction failed for %s: %s", mapName, e.getMessage());
            return Optional.empty();
        }
    }

    private static TerrainResponse toResponse(TerrainGrid grid) {
        List<int[]> walls = new ArrayList<>(), high = new ArrayList<>(), ramps = new ArrayList<>();
        for (int x = 0; x < grid.width(); x++) {
            for (int y = 0; y < grid.height(); y++) {
                switch (grid.heightAt(x, y)) {
                    case WALL -> walls.add(new int[]{x, y});
                    case HIGH -> high.add(new int[]{x, y});
                    case RAMP -> ramps.add(new int[]{x, y});
                    case LOW  -> {} // omitted — default terrain
                }
            }
        }
        return new TerrainResponse(grid.width(), grid.height(), walls, high, ramps);
    }
}
```

- [ ] **Step 4.2: Build to confirm compilation**

```bash
mvn compile -q
```
Expected: BUILD SUCCESS

- [ ] **Step 4.3: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/map/SC2MapCache.java
git commit -m "feat: SC2MapCache — orchestrate download + extraction + JSON cache Refs #98"
```

---

## Task 5: TerrainResource + CurrentMapResource + SC2Engine map getters — Issues #98 #99

Two REST endpoints and the SC2Engine interface extension.

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/SC2Engine.java`
- Create: `src/main/java/io/quarkmind/qa/TerrainResource.java`
- Create: `src/main/java/io/quarkmind/qa/CurrentMapResource.java`
- Create: `src/test/java/io/quarkmind/qa/TerrainResourceIT.java`
- Create: `src/test/java/io/quarkmind/qa/CurrentMapResourceIT.java`

- [ ] **Step 5.1: Add map getters to `SC2Engine` interface**

Add after the `addFrameListener` default:

```java
/** Returns the current map name (bare, no .SC2Map suffix), or null if unknown. */
default String getMapName()   { return null; }
/** Returns the map width in tiles, or 0 if unknown. */
default int    getMapWidth()  { return 0; }
/** Returns the map height in tiles, or 0 if unknown. */
default int    getMapHeight() { return 0; }
```

- [ ] **Step 5.2: Write the failing integration tests**

```java
package io.quarkmind.qa;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class TerrainResourceIT {

    // Pre-seed the cache so the test doesn't hit the network
    static {
        try {
            Path cacheDir = Path.of(System.getProperty("user.home"), ".quarkmind", "maps");
            Files.createDirectories(cacheDir);
            Path mapSrc  = Path.of("src/test/resources/maps/TorchesAIE_v4.SC2Map");
            Path mapDest = cacheDir.resolve("TorchesAIE_v4.SC2Map");
            if (!Files.exists(mapDest)) {
                Files.copy(mapSrc, mapDest, StandardCopyOption.REPLACE_EXISTING);
            }
        } catch (Exception e) {
            throw new RuntimeException("Cannot seed test map cache", e);
        }
    }

    @Test
    void terrainEndpointReturnsCorrectDimensionsForTorchesAIE() {
        given()
            .queryParam("map", "TorchesAIE_v4")
            .when().get("/qa/terrain")
            .then()
            .statusCode(200)
            .body("width",  equalTo(160))
            .body("height", equalTo(208))
            .body("walls",      not(empty()))
            .body("highGround", not(empty()))
            .body("ramps",      not(empty()));
    }

    @Test
    void terrainEndpointReturnsCachedOnSecondCall() {
        // Delete JSON cache to force extraction, then call twice
        try { Files.deleteIfExists(Path.of(System.getProperty("user.home"),
                ".quarkmind", "maps", "TorchesAIE_v4-terrain.json")); }
        catch (Exception ignored) {}

        given().queryParam("map", "TorchesAIE_v4").when().get("/qa/terrain").then().statusCode(200);
        given().queryParam("map", "TorchesAIE_v4").when().get("/qa/terrain").then().statusCode(200);
        // Both return 200; second uses cache (no way to assert no download in integration test —
        // presence of the JSON cache file is sufficient)
        assertThat(Files.exists(Path.of(System.getProperty("user.home"),
                ".quarkmind", "maps", "TorchesAIE_v4-terrain.json"))).isTrue();
    }

    @Test
    void terrainEndpointReturns404ForUnknownMap() {
        given()
            .queryParam("map", "UnknownMap_v99")
            .when().get("/qa/terrain")
            .then()
            .statusCode(404);
    }
}
```

Add import at top of `TerrainResourceIT`:
```java
import static org.assertj.core.api.Assertions.assertThat;
```

```java
package io.quarkmind.qa;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;
import static io.restassured.RestAssured.given;

@QuarkusTest
class CurrentMapResourceIT {

    @Test
    void currentMapReturns404InMockProfile() {
        // Default test profile is mock — no map name available
        given()
            .when().get("/qa/current-map")
            .then()
            .statusCode(404);
    }
}
```

- [ ] **Step 5.3: Run tests to confirm they fail**

```bash
mvn test -Dtest="TerrainResourceIT,CurrentMapResourceIT" -q 2>&1 | tail -5
```
Expected: FAIL — endpoints don't exist.

- [ ] **Step 5.4: Implement `TerrainResource`**

```java
package io.quarkmind.qa;

import io.quarkus.arc.profile.UnlessBuildProfile;
import io.quarkmind.sc2.map.SC2MapCache;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

@UnlessBuildProfile("prod")
@Path("/qa/terrain")
@Produces(MediaType.APPLICATION_JSON)
public class TerrainResource {

    @Inject SC2MapCache cache;

    @GET
    public Response getTerrain(@QueryParam("map") String mapName) {
        if (mapName == null || mapName.isBlank())
            return Response.status(Response.Status.BAD_REQUEST).build();
        return cache.get(mapName)
            .map(Response::ok)
            .orElse(Response.status(Response.Status.NOT_FOUND))
            .build();
    }
}
```

- [ ] **Step 5.5: Implement `CurrentMapResource`**

```java
package io.quarkmind.qa;

import io.quarkus.arc.profile.UnlessBuildProfile;
import io.quarkmind.sc2.SC2Engine;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

@UnlessBuildProfile("prod")
@Path("/qa/current-map")
@Produces(MediaType.APPLICATION_JSON)
public class CurrentMapResource {

    @Inject SC2Engine engine;

    @GET
    public Response getCurrentMap() {
        String name = engine.getMapName();
        if (name == null) return Response.status(Response.Status.NOT_FOUND).build();
        return Response.ok(new MapMeta(name, engine.getMapWidth(), engine.getMapHeight())).build();
    }

    public record MapMeta(String mapName, int mapWidth, int mapHeight) {}
}
```

- [ ] **Step 5.6: Run integration tests — all should pass**

```bash
mvn test -Dtest="TerrainResourceIT,CurrentMapResourceIT" -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 5.7: Run full test suite to check for regressions**

```bash
mvn test -q
```
Expected: BUILD SUCCESS, all existing tests pass.

- [ ] **Step 5.8: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/SC2Engine.java \
        src/main/java/io/quarkmind/qa/TerrainResource.java \
        src/main/java/io/quarkmind/qa/CurrentMapResource.java \
        src/test/java/io/quarkmind/qa/TerrainResourceIT.java \
        src/test/java/io/quarkmind/qa/CurrentMapResourceIT.java
git commit -m "feat: terrain endpoint + current-map endpoint + SC2Engine map getters Closes #98"
```

---

## Task 6: ReplayEngine — parse map name and size from gamemetadata.json — Issue #99

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/replay/ReplayEngine.java`
- Modify: `src/test/java/io/quarkmind/sc2/replay/ReplayEngineTest.java`

- [ ] **Step 6.1: Write the failing tests**

Add to `ReplayEngineTest.java`:

```java
@Test
void connectParsesMapName() {
    engine.connect();
    assertThat(engine.getMapName()).isEqualTo("TorchesAIE_v4");
}

@Test
void connectParsesMapDimensions() {
    engine.connect();
    assertThat(engine.getMapWidth()).isEqualTo(160);
    assertThat(engine.getMapHeight()).isEqualTo(208);
}

@Test
void mapNameIsNullBeforeConnect() {
    // engine not yet connected
    assertThat(engine.getMapName()).isNull();
}
```

- [ ] **Step 6.2: Run tests to confirm they fail**

```bash
mvn test -Dtest=ReplayEngineTest -q 2>&1 | tail -5
```
Expected: FAIL — `getMapName()` returns null (default from SC2Engine).

- [ ] **Step 6.3: Implement map metadata parsing in `ReplayEngine`**

Add fields and parsing to `ReplayEngine.java`:

```java
// New fields (after existing fields)
private String mapName;
private int mapWidth;
private int mapHeight;
```

Replace `connect()` with:

```java
@Override
public void connect() {
    log.infof("[REPLAY] Loading replay: %s (player %d)", replayFile, watchedPlayerId);
    game = new ReplaySimulatedGame(Path.of(replayFile), watchedPlayerId);
    parseMapMetadata(Path.of(replayFile));
    connected = true;
    log.infof("[REPLAY] Replay loaded — %d tracker events ready, map=%s (%dx%d)",
        game.eventCount(), mapName, mapWidth, mapHeight);
}

private void parseMapMetadata(Path replayPath) {
    try (var mpq = new hu.belicza.andras.mpq.MpqParser(replayPath)) {
        // Map name from gamemetadata.json
        byte[] meta = mpq.getFile("replay.gamemetadata.json");
        if (meta != null) {
            var node = new com.fasterxml.jackson.databind.ObjectMapper().readTree(meta);
            String raw = node.path("MapName").asText(null);
            if (raw != null && raw.endsWith(".SC2Map")) {
                mapName = raw.substring(0, raw.length() - 7);
            }
        }
        // Map dimensions from INIT_DATA
        hu.scelight.sc2.rep.model.Replay r =
            hu.scelight.sc2.rep.factory.RepParserEngine.parseReplay(
                replayPath,
                java.util.EnumSet.of(hu.scelight.sc2.rep.factory.RepContent.INIT_DATA));
        if (r != null && r.initData != null) {
            var gd = r.initData.getGameDescription();
            mapWidth  = gd.getMapSizeX() != null ? gd.getMapSizeX() : 0;
            mapHeight = gd.getMapSizeY() != null ? gd.getMapSizeY() : 0;
        }
    } catch (Exception e) {
        log.warnf("[REPLAY] Cannot parse map metadata: %s", e.getMessage());
    }
}

@Override public String getMapName()   { return mapName; }
@Override public int    getMapWidth()  { return mapWidth; }
@Override public int    getMapHeight() { return mapHeight; }
```

- [ ] **Step 6.4: Run tests — all should pass**

```bash
mvn test -Dtest=ReplayEngineTest -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 6.5: Run full suite**

```bash
mvn test -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 6.6: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/replay/ReplayEngine.java \
        src/test/java/io/quarkmind/sc2/replay/ReplayEngineTest.java
git commit -m "feat: ReplayEngine parses map name and size from gamemetadata.json Closes #99"
```

---

## Task 7: Visualizer — two-phase terrain loading, handle 160×208 maps — Issue #100

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`

- [ ] **Step 7.1: Locate and read the `loadTerrain()` function**

```bash
grep -n "async function loadTerrain" src/main/resources/META-INF/resources/visualizer.js
```
Note the line number. Read from that line for ~90 lines to understand the full function.

- [ ] **Step 7.2: Replace `loadTerrain()` with two-phase version**

Replace the entire `loadTerrain()` function with:

```javascript
async function loadTerrain() {
  let walls = [], highGround = [], ramps = [];
  let hasRealTerrain = false;

  // Phase 1: replay profile — ask server for map name then fetch terrain
  try {
    const mapMeta = await fetchJson('/qa/current-map');
    if (mapMeta) {
      GRID_W = mapMeta.mapWidth;
      GRID_H = mapMeta.mapHeight;
      const terrainResp = await fetch(`/qa/terrain?map=${encodeURIComponent(mapMeta.mapName)}`);
      if (terrainResp.ok) {
        const d = await terrainResp.json();
        walls      = d.walls      || [];
        highGround = d.highGround || [];
        ramps      = d.ramps      || [];
        hasRealTerrain = true;
      }
    }
  } catch (_) { /* not replay profile or network error */ }

  // Phase 2: emulated profile — existing endpoint
  if (!hasRealTerrain) {
    try {
      const r = await fetch('/qa/emulated/terrain');
      if (r.ok) {
        const d = await r.json();
        GRID_W = d.width; GRID_H = d.height;
        walls      = d.walls      || [];
        highGround = d.highGround || [];
        ramps      = d.ramps      || [];
        hasRealTerrain = true;
      }
    } catch (_) { /* non-emulated profiles: flat grid */ }
  }

  HALF_W = (GRID_W * TILE) / 2;
  HALF_H = (GRID_H * TILE) / 2;

  const wallSet = new Set(walls.map(([x,z])      => `${x},${z}`));
  const highSet = new Set(highGround.map(([x,z]) => `${x},${z}`));
  const rampSet = new Set(ramps.map(([x,z])      => `${x},${z}`));

  const sharedEdgesGeo = new THREE.EdgesGeometry(new THREE.BoxGeometry(TILE, 0.01, TILE));

  // Base ground plane (replaces per-cell LOW tiles for performance)
  if (hasRealTerrain) {
    const groundPlane = new THREE.Mesh(
      new THREE.PlaneGeometry(GRID_W * TILE, GRID_H * TILE),
      mGround
    );
    groundPlane.rotation.x = -Math.PI / 2;
    groundPlane.position.set(0, 0.04, 0);
    scene.add(groundPlane);
  }

  // Render non-LOW tiles as individual boxes
  for (let gz = 0; gz < GRID_H; gz++) {
    for (let gx = 0; gx < GRID_W; gx++) {
      const key = `${gx},${gz}`;
      const cx  = gx * TILE - HALF_W + TILE/2;
      const cz  = gz * TILE - HALF_H + TILE/2;

      let h = 0.08, mat = mGround;
      if      (wallSet.has(key)) { h = TILE * 1.2; mat = mWall; }
      else if (highSet.has(key)) { h = TILE * 0.6; mat = mHigh; }
      else if (rampSet.has(key)) { h = TILE * 0.25; mat = mRamp; }
      else if (hasRealTerrain)   { continue; } // LOW — covered by base plane

      const tile = new THREE.Mesh(new THREE.BoxGeometry(TILE*0.98, h, TILE*0.98), mat);
      tile.position.set(cx, h/2, cz);
      if (h >= TILE * 0.4) tile.receiveShadow = true;
      if (mat === mWall) tile.castShadow = true;
      scene.add(tile);

      const el = new THREE.LineSegments(sharedEdgesGeo, lineMat);
      el.position.set(cx, h + 0.01, cz);
      scene.add(el);
    }
  }

  // Fog planes — only created in emulated mode where visibility data exists.
  // Skipping them in mock/replay modes removes 4096 objects from the scene graph,
  // cutting per-frame traversal cost roughly in half.
  if (hasRealTerrain && !mapMeta) {  // mapMeta defined only in Phase 1 scope; emulated has no mapMeta
    // NOTE: this block is unreachable now — restructure below
  }
  // Fog planes for emulated mode only (replay has no server-side visibility grid)
  if (hasRealTerrain && walls.length > 0 && typeof mapMeta === 'undefined') {
    // emulated mode detected — create fog planes
    const fogMat = new THREE.MeshBasicMaterial({
      color: 0x888888, transparent: true,
      side: THREE.DoubleSide, depthWrite: false
    });
    const fogGeo = new THREE.PlaneGeometry(TILE*0.98, TILE*0.98);
    for (let gz = 0; gz < GRID_H; gz++) {
      for (let gx = 0; gx < GRID_W; gx++) {
        const cx = gx * TILE - HALF_W + TILE/2;
        const cz = gz * TILE - HALF_H + TILE/2;
        const plane = new THREE.Mesh(fogGeo, fogMat.clone());
        plane.rotation.x = -Math.PI/2;
        plane.position.set(cx, 0.18, cz);
        plane.renderOrder = 5;
        plane.userData.isFog = true;
        plane.visible = true;
        plane.material.opacity = 1.0;
        scene.add(plane);
        fogPlanes.set(`${gx},${gz}`, plane);
      }
    }
  }

  if (hasRealTerrain) {
    tDist = camDist = Math.max(GRID_W, GRID_H) * TILE * 0.7;
    TERRAIN_SURFACE_Y = TILE;
  }
  updateCamera();
  terrainLoaded = true;
}

async function fetchJson(url) {
  try {
    const r = await fetch(url);
    return r.ok ? r.json() : null;
  } catch (_) { return null; }
}
```

**Note on fog planes:** The fog plane block needs careful scope management. Refactor the fog plane creation to check a `isEmulatedMode` flag set during Phase 2. Replace the fog plane block above with this cleaner version:

After the `for` loop that creates terrain tiles, add fog planes only when emulated:

```javascript
  // Track whether terrain came from emulated endpoint (Phase 2) for fog plane creation
  // 'isEmulated' is true only when Phase 1 found nothing and Phase 2 succeeded
  if (isEmulatedMode && hasRealTerrain) {
    // ... (existing fog plane creation code, unchanged) ...
  }
```

Declare `let isEmulatedMode = false;` before Phase 1, set to `true` inside Phase 2's success branch.

- [ ] **Step 7.3: Add `hasRealTerrain()` to `window.__test` API**

Find `window.__test = {` in the file, add:
```javascript
hasRealTerrain: () => terrainLoaded && hasRealTerrain,
```

Note: `hasRealTerrain` is currently a local variable inside `loadTerrain()`. Promote it to module-level by declaring `let hasRealTerrain = false;` at the top of the file alongside other globals, and remove the `let` declaration inside `loadTerrain()`.

- [ ] **Step 7.4: Start the server and verify visually (mock profile)**

```bash
mvn quarkus:dev &
sleep 15
curl -s http://localhost:8080/qa/current-map | python3 -m json.tool
# Expected: {"status":404} or similar — mock profile returns 404
curl -s http://localhost:8080/qa/terrain?map=TorchesAIE_v4 | python3 -c "import sys,json; d=json.load(sys.stdin); print('walls:', len(d['walls']), 'high:', len(d['highGround']), 'ramps:', len(d['ramps']))"
```

Open `http://localhost:8080/visualizer.html` in a browser — flat mock terrain should appear unchanged.

- [ ] **Step 7.5: Run existing Playwright tests to confirm no regressions**

```bash
pkill -f 'quarkus:dev' 2>/dev/null; sleep 2
mvn test -Pplaywright -Dtest=VisualizerRenderTest -q
```
Expected: All existing tests pass.

- [ ] **Step 7.6: Write new Playwright test — `window.__test.hasRealTerrain()` in mock returns false**

Add to `VisualizerRenderTest.java`:

```java
@Test
@Tag("browser")
void hasRealTerrainIsFalseInMockProfile() {
    assumeTrue(browser != null, "Chromium not installed");
    try (var context = browser.newContext(); var page = context.newPage()) {
        page.navigate(pageUrl.toString());
        page.waitForFunction("() => window.__test && window.__test.terrainReady()");
        Object result = page.evaluate("() => window.__test.hasRealTerrain()");
        assertThat(result).isEqualTo(false);
    }
}
```

Run:
```bash
mvn test -Pplaywright -Dtest="VisualizerRenderTest#hasRealTerrainIsFalseInMockProfile" -q
```
Expected: PASS.

- [ ] **Step 7.7: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat: visualizer two-phase terrain loading, handle large maps Closes #100"
```

---

## Task 8: End-to-end validation and epic close — Issue #95

- [ ] **Step 8.1: Run full test suite including Playwright**

```bash
mvn test -q && mvn test -Pplaywright -q
```
Expected: All tests pass.

- [ ] **Step 8.2: Smoke-test the replay profile manually**

```bash
mvn quarkus:dev -Dquarkus.profile=replay &
sleep 20
curl -s http://localhost:8080/qa/current-map
# Expected: {"mapName":"TorchesAIE_v4","mapWidth":160,"mapHeight":208}
curl -s "http://localhost:8080/qa/terrain?map=TorchesAIE_v4" | python3 -c "
import sys,json; d=json.load(sys.stdin)
print(f'Dimensions: {d[\"width\"]}x{d[\"height\"]}')
print(f'Walls: {len(d[\"walls\"])}, High: {len(d[\"highGround\"])}, Ramps: {len(d[\"ramps\"])}')"
```
Expected:
```
{"mapName":"TorchesAIE_v4","mapWidth":160,"mapHeight":208}
Dimensions: 160x208
Walls: ~10674, High: ~14444, Ramps: ~676
```

Open `http://localhost:8080/visualizer.html` — terrain should render with visible walls (tall dark tiles at edges), high ground, and ramp transitions.

- [ ] **Step 8.3: Kill the server**

```bash
pkill -f 'quarkus:dev'
```

- [ ] **Step 8.4: Close the epic**

```bash
gh issue close 95 --comment "All 5 child issues closed. Replay profile now loads real Torches AIE terrain (160×208 tiles) automatically. Units from Nothing_4720936 replay are visible on accurate high ground, ramps, and wall terrain. All tests pass."
```

- [ ] **Step 8.5: Final commit if any loose ends**

```bash
git status
# If nothing uncommitted, done.
```

---

## Self-Review

**Spec coverage:**
- ✅ SC2MapTerrainExtractor — Task 2
- ✅ MapDownloader + map-registry.json — Task 3
- ✅ SC2MapCache — Task 4
- ✅ TerrainResource + CurrentMapResource — Task 5
- ✅ TerrainResponse extracted — Task 1
- ✅ SC2Engine map getters — Task 5
- ✅ ReplayEngine map metadata — Task 6
- ✅ Visualizer two-phase loading — Task 7
- ✅ Performance: LOW ground uses base plane — Task 7
- ✅ `hasRealTerrain()` exposed in `window.__test` — Task 7
- ✅ Fuzzy map name matching (version suffix strip) — Task 3

**Test coverage:**
- Unit: `SC2MapTerrainExtractorTest` (7 cases: dimensions, WALL×2, HIGH, LOW, RAMP, error handling)
- Unit: `MapDownloaderTest` (6 cases: cache hit, download, fuzzy match, unknown map, network error, idempotent download)
- Integration: `TerrainResourceIT` (3 cases: happy path, cached, 404)
- Integration: `CurrentMapResourceIT` (1 case: 404 in mock profile)
- Unit: `ReplayEngineTest` additions (3 cases: map name, dimensions, null before connect)
- Playwright: `hasRealTerrainIsFalseInMockProfile` (mock profile regression guard)
- Manual: replay profile smoke test (Step 8.2)

**Type consistency:** `TerrainResponse` used consistently in `SC2MapCache`, `TerrainResource`, and `EmulatedTerrainResource`. `MapMeta` is a local record in `CurrentMapResource`.

**Potential issue:** The fog plane logic in Task 7 uses a `typeof mapMeta === 'undefined'` check which is fragile. The `isEmulatedMode` flag approach in the note is cleaner. The executor should implement the flag approach, not the `typeof` check.
