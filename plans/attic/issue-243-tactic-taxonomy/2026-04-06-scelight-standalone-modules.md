# Scelight Standalone Modules Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Split Scelight's MPQ parser and s2protocol decoder into two standalone Maven modules (`scelight-mpq` and `scelight-s2protocol`) that publish to local Maven and can be consumed by starcraft-agent without dragging in Scelight's Swing UI.

**Architecture:** Create a `feature/standalone-modules` branch in the Scelight repo. Copy the relevant source files into new module `src/main/java/` directories — original `src-app-libs/` and `src-app/` are untouched so the existing Scelight build continues to work. Four small code changes remove the Scelight infrastructure dependencies (Env, Utils, Settings) from the copies. A quick-build script rebuilds only the two modules. The starcraft-agent project adds them as Maven dependencies.

**Tech Stack:** Java 11 (module poms override Scelight's Java 8 default), Maven 3.x, JBoss Logging (already in starcraft-agent stack).

**Work directory:** `/Users/mdproctor/claude/scelight`

---

## File Map

```
scelight/ (branch: feature/standalone-modules)
  scelight-mpq/
    pom.xml                                          ← new
    src/main/java/
      hu/belicza/andras/mpq/                         ← copied from src-app-libs/
        AlgorithmUtil.java
        IMpqContent.java
        InvalidMpqArchiveException.java
        MpqContent.java
        MpqDataInput.java
        MpqParser.java
        model/
          BlockTable.java  HashTable.java  Header.java
          MpqArchive.java  UserData.java
      org/apache/tools/bzip2/                        ← copied from src-app-libs/

  scelight-s2protocol/
    pom.xml                                          ← new
    src/main/java/
      hu/scelight/sc2/rep/                           ← copied from src-app/hu/scelight/sc2/rep/
        s2prot/
          Protocol.java                              ← 4 changes (Env/Utils removed)
          VersionedDecoder.java  BitPackedDecoder.java
          BitPackedBuffer.java  Event.java  EventFactory.java
          type/  build/*.dat
        model/                                       ← copied unchanged
        factory/
          RepContent.java  RepParserEngine.java      ← 2 changes (Env removed)
      hu/sllauncher/util/
        Pair.java                                    ← new minimal copy (no IPair dependency)

  scripts/
    publish-replay-libs.sh                           ← new quick-build script

  pom.xml                                            ← add 2 new <module> entries
```

---

## Task 1: Create Branch + scelight-mpq Module

**Files:**
- Create: `scelight-mpq/pom.xml`
- Copy: `src-app-libs/hu/belicza/andras/mpq/` → `scelight-mpq/src/main/java/hu/belicza/andras/mpq/`
- Copy: `src-app-libs/org/apache/tools/bzip2/` → `scelight-mpq/src/main/java/org/apache/tools/bzip2/`

- [ ] **Step 1: Create branch**

```bash
cd /Users/mdproctor/claude/scelight
git checkout -b feature/standalone-modules
```

- [ ] **Step 2: Create module directory and copy sources**

```bash
mkdir -p scelight-mpq/src/main/java
cp -r src-app-libs/hu scelight-mpq/src/main/java/
cp -r src-app-libs/org scelight-mpq/src/main/java/
# Verify the copy
find scelight-mpq/src/main/java -name "*.java" | wc -l
```

Expected: ~12 Java files copied.

- [ ] **Step 3: Create scelight-mpq/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>hu.scelight</groupId>
    <artifactId>scelight-mpq</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>Scelight MPQ Parser</name>
    <description>Standalone MPQ archive parser extracted from Scelight. Supports bzip2, zlib, and PKWARE compression.</description>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.release>11</maven.compiler.release>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 4: Compile scelight-mpq**

```bash
cd /Users/mdproctor/claude/scelight
mvn compile -pl scelight-mpq -q
```

Expected: BUILD SUCCESS. The MPQ parser has zero external dependencies — it should compile cleanly.

- [ ] **Step 5: Commit**

```bash
git add scelight-mpq/
git commit -m "feat: add scelight-mpq standalone module — MPQ parser + BZip2 support"
```

---

## Task 2: Create scelight-s2protocol Module (Source Copy)

**Files:**
- Copy: `src-app/hu/scelight/sc2/rep/` → `scelight-s2protocol/src/main/java/hu/scelight/sc2/rep/`
- Create: `scelight-s2protocol/src/main/java/hu/sllauncher/util/Pair.java`
- Create: `scelight-s2protocol/pom.xml`

- [ ] **Step 1: Copy sources (excluding cache and repproc)**

We exclude `cache/` and `repproc/` packages — they depend on `PersistentMap`, `VersionBean`, and heavy Scelight infrastructure. We only need `parseReplay()` from `RepParserEngine`, not `getRepProc()` (which requires the caching layer).

```bash
mkdir -p scelight-s2protocol/src/main/java/hu/scelight/sc2/rep
# Copy everything except cache/ and repproc/
for dir in s2prot model factory; do
  cp -r src-app/hu/scelight/sc2/rep/$dir \
       scelight-s2protocol/src/main/java/hu/scelight/sc2/rep/$dir
done
# Verify
find scelight-s2protocol/src/main/java -name "*.java" | wc -l
```

Expected: ~35 Java files (s2prot, model, factory only).

- [ ] **Step 2: Create minimal Pair.java**

```java
// scelight-s2protocol/src/main/java/hu/sllauncher/util/Pair.java
package hu.sllauncher.util;

import java.util.Objects;

/**
 * A pair of generic-type objects. Minimal standalone copy extracted from Scelight.
 */
public class Pair<T1, T2> {

    public final T1 value1;
    public final T2 value2;

    public Pair(final T1 value1, final T2 value2) {
        this.value1 = value1;
        this.value2 = value2;
    }

    @Override
    public boolean equals(final Object o) {
        if (o == this) return true;
        if (!(o instanceof Pair)) return false;
        final Pair<?, ?> p = (Pair<?, ?>) o;
        return Objects.equals(value1, p.value1) && Objects.equals(value2, p.value2);
    }

    @Override
    public int hashCode() {
        return Objects.hash(value1, value2);
    }
}
```

- [ ] **Step 3: Create scelight-s2protocol/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>hu.scelight</groupId>
    <artifactId>scelight-s2protocol</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>Scelight s2protocol Decoder</name>
    <description>Standalone SC2 replay decoder extracted from Scelight. Parses .SC2Replay files into structured event streams.</description>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.release>11</maven.compiler.release>
    </properties>

    <dependencies>
        <dependency>
            <groupId>hu.scelight</groupId>
            <artifactId>scelight-mpq</artifactId>
            <version>1.0.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>org.jboss.logging</groupId>
            <artifactId>jboss-logging</artifactId>
            <version>3.5.3.Final</version>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 4: Attempt compile — expect failures**

```bash
cd /Users/mdproctor/claude/scelight
mvn compile -pl scelight-s2protocol -am -q 2>&1 | grep "ERROR\|cannot find" | head -20
```

Expected: Errors referencing `hu.scelight.service.env.Env`, `hu.scelight.service.settings.Settings`, `hu.scelight.util.Utils`. These are the exactly four replacements in the next task.

- [ ] **Step 5: Commit the source copy (pre-fix)**

```bash
git add scelight-s2protocol/
git commit -m "feat: add scelight-s2protocol standalone module — source copy, pre-dependency-removal"
```

---

## Task 3: Remove Scelight Infrastructure Dependencies

**Files to modify (in the scelight-s2protocol copy only):**
- `scelight-s2protocol/src/main/java/hu/scelight/sc2/rep/s2prot/Protocol.java`
- `scelight-s2protocol/src/main/java/hu/scelight/sc2/rep/factory/RepParserEngine.java`

### Fix Protocol.java

- [ ] **Step 1: Replace imports in Protocol.java**

Remove these imports:
```java
import hu.scelight.service.env.Env;
import hu.scelight.service.settings.Settings;
import hu.scelight.util.Utils;
```

Add these imports:
```java
import java.nio.charset.StandardCharsets;
import java.util.Base64;
import java.util.HashSet;
import org.jboss.logging.Logger;
```

- [ ] **Step 2: Replace Env.LOGGER with JBoss Logger in Protocol.java**

Add field after the class declaration opening:
```java
private static final Logger LOGGER = Logger.getLogger(Protocol.class);
```

Replace `Env.LOGGER.error(...)` with `LOGGER.error(...)` (one occurrence in the `get()` method).

- [ ] **Step 3: Replace Env.APP_SETTINGS usage in Protocol.java**

The `get()` method currently has:
```java
if (baseBuild > LATEST_BASE_BUILD && Env.APP_SETTINGS.get(Settings.USE_LATEST_S2PROTOCOL)) {
```

Replace with:
```java
if (baseBuild > LATEST_BASE_BUILD && USE_LATEST_FOR_UNKNOWN_BUILDS) {
```

Add this constant near the top of the class (after `LATEST_BASE_BUILD`):
```java
/**
 * If true, replay files from unknown/newer base builds will be decoded using
 * the latest bundled protocol. Set to true for permissive parsing.
 */
public static boolean USE_LATEST_FOR_UNKNOWN_BUILDS = true;
```

- [ ] **Step 4: Replace Utils.asNewSet in Protocol.java**

The `BETA_BASE_BUILD_SET` field currently uses:
```java
public static final Set<Integer> BETA_BASE_BUILD_SET = Utils.asNewSet(17266, 18468, ...);
```

Replace with:
```java
public static final Set<Integer> BETA_BASE_BUILD_SET = new HashSet<>(Arrays.asList(
    17266, 18468, 19458, 19595, 21955, 24764, 27950, 28272, 34784, 34835,
    36442, 38535, 38624, 39117, 39948, 40384, 40977, 41128, 41219, 41973,
    44169, 44293, 44743, 44765, 45186, 45364, 45386, 45542, 45556, 45593,
    45737, 45944, 47932, 49957));
```

Add `import java.util.Arrays;` to the imports.

- [ ] **Step 5: Replace Env.UTF8 in Protocol.java**

In `decodeAttributesEvents()`, find:
```java
attr.value = lastZeroIdx < 0 ? new String(valueBuff, 0, 4, Env.UTF8) : new String(valueBuff, lastZeroIdx + 1, 3 - lastZeroIdx, Env.UTF8);
```

Replace with:
```java
attr.value = lastZeroIdx < 0 ? new String(valueBuff, 0, 4, StandardCharsets.UTF_8) : new String(valueBuff, lastZeroIdx + 1, 3 - lastZeroIdx, StandardCharsets.UTF_8);
```

### Fix RepParserEngine.java

- [ ] **Step 6: Replace imports in RepParserEngine.java**

Remove these imports:
```java
import hu.scelight.service.env.Env;
import hu.scelight.util.Utils;
import hu.sllauncher.bean.VersionBean;
```

Add:
```java
import java.util.Base64;
import org.jboss.logging.Logger;
```

- [ ] **Step 7: Replace Env.LOGGER, remove VersionBean, and remove getRepProc() in RepParserEngine.java**

Add field after the class declaration opening:
```java
private static final Logger LOGGER = Logger.getLogger(RepParserEngine.class);
```

Remove the VERSION constant entirely:
```java
// DELETE THIS LINE:
public static final VersionBean VERSION = new VersionBean(1, 3, 0);
```

Replace all `Env.LOGGER.debug(...)` → `LOGGER.debug(...)` and `Env.LOGGER.info(...)` → `LOGGER.info(...)`.

**Also delete the entire `getRepProc()` method** — it depends on `RepProcCache` (excluded package). The method is ~30 lines starting with `public static RepProcessor getRepProc(...)`. Remove it completely. `parseReplay()` is the only method we need.

- [ ] **Step 8: Replace Utils.toBase64String in RepParserEngine.java**

Find:
```java
final String replayKey = Utils.toBase64String(md.digest(mpqAttributes)).substring(0, 27);
```

Replace with:
```java
final String replayKey = Base64.getEncoder().encodeToString(md.digest(mpqAttributes)).substring(0, 27);
```

- [ ] **Step 9: Compile — expect success**

```bash
cd /Users/mdproctor/claude/scelight
mvn compile -pl scelight-s2protocol -am -q
```

Expected: BUILD SUCCESS. If any remaining `Env`/`Utils`/`Settings` references remain, the error output will show the exact file and line — fix each one following the patterns above.

- [ ] **Step 10: Commit fixes**

```bash
git add scelight-s2protocol/src/
git commit -m "fix: remove Scelight infrastructure dependencies from standalone s2protocol module (Env, Utils, Settings, VersionBean)"
```

---

## Task 4: Register Modules in Root pom.xml + Add Quick-Build Script

**Files:**
- Modify: `pom.xml` (root)
- Create: `scripts/publish-replay-libs.sh`

- [ ] **Step 1: Add modules to root pom.xml**

In the root `pom.xml`, add a `<modules>` section if it doesn't exist (the current root pom is a single-module build). Add after `</properties>`:

```xml
<modules>
    <module>scelight-mpq</module>
    <module>scelight-s2protocol</module>
</modules>
```

**Note:** This adds the new modules to the root build but does NOT affect the existing Scelight main app compilation — the existing source roots (`src-app`, `src-app-libs`, etc.) are still compiled by the root pom's own `<build><sourceDirectory>` configuration. The new modules are additive.

- [ ] **Step 2: Verify root build still works**

```bash
cd /Users/mdproctor/claude/scelight
mvn compile -pl scelight-mpq,scelight-s2protocol -am -q
```

Expected: BUILD SUCCESS for both modules. Do NOT run `mvn compile` on the full root at this point — the existing Scelight build has many dependencies (Swing, mail, etc.) that are not the focus here.

- [ ] **Step 3: Create quick-build script**

```bash
mkdir -p scripts
```

Create `scripts/publish-replay-libs.sh`:

```bash
#!/usr/bin/env bash
# Builds and installs scelight-mpq and scelight-s2protocol to local Maven (~/.m2).
# Fast build — only compiles ~50 Java files, no tests, no full Scelight build.
# Run this after any change to the standalone modules.
set -e
cd "$(dirname "$0")/.."
echo "Building scelight-mpq and scelight-s2protocol..."
mvn install -pl scelight-mpq,scelight-s2protocol -am -DskipTests -q
echo "Done. Artifacts installed to ~/.m2/repository/hu/scelight/"
echo "  scelight-mpq:1.0.0-SNAPSHOT"
echo "  scelight-s2protocol:1.0.0-SNAPSHOT"
```

```bash
chmod +x scripts/publish-replay-libs.sh
```

- [ ] **Step 4: Run the quick-build to publish locally**

```bash
cd /Users/mdproctor/claude/scelight
./scripts/publish-replay-libs.sh
```

Expected output:
```
Building scelight-mpq and scelight-s2protocol...
Done. Artifacts installed to ~/.m2/repository/hu/scelight/
  scelight-mpq:1.0.0-SNAPSHOT
  scelight-s2protocol:1.0.0-SNAPSHOT
```

Verify:
```bash
ls ~/.m2/repository/hu/scelight/scelight-mpq/1.0.0-SNAPSHOT/
ls ~/.m2/repository/hu/scelight/scelight-s2protocol/1.0.0-SNAPSHOT/
```

Expected: `scelight-mpq-1.0.0-SNAPSHOT.jar` and `scelight-s2protocol-1.0.0-SNAPSHOT.jar` present.

- [ ] **Step 5: Commit**

```bash
git add pom.xml scripts/
git commit -m "feat: register standalone modules in root pom; add publish-replay-libs.sh quick-build script"
```

---

## Task 5: Wire into starcraft-agent

**Files:**
- Modify: `/Users/mdproctor/claude/starcraft/pom.xml`
- Modify: `/Users/mdproctor/claude/starcraft/CLAUDE.md`

- [ ] **Step 1: Add dependencies to starcraft-agent pom.xml**

In `/Users/mdproctor/claude/starcraft/pom.xml`, add after the CaseHub dependency:

```xml
<!-- SC2 Replay parsing — extracted from Scelight (Apache 2.0) -->
<!-- Source: /Users/mdproctor/claude/scelight, branch: feature/standalone-modules -->
<!-- Build: cd /Users/mdproctor/claude/scelight && ./scripts/publish-replay-libs.sh -->
<dependency>
    <groupId>hu.scelight</groupId>
    <artifactId>scelight-mpq</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
<dependency>
    <groupId>hu.scelight</groupId>
    <artifactId>scelight-s2protocol</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

- [ ] **Step 2: Update CLAUDE.md prerequisites**

In `/Users/mdproctor/claude/starcraft/CLAUDE.md`, under the CaseHub dependency section, add:

```markdown
## Replay Library Dependency

The SC2 replay parser (`scelight-mpq` + `scelight-s2protocol`) is a local build from the Scelight fork:

```bash
cd /Users/mdproctor/claude/scelight && ./scripts/publish-replay-libs.sh
```

Run this after any change to the Scelight `feature/standalone-modules` branch, or when setting up a new environment. The script takes ~10 seconds.
```

- [ ] **Step 3: Verify starcraft-agent compiles with new dependencies**

```bash
cd /Users/mdproctor/claude/starcraft
mvn compile -q
```

Expected: BUILD SUCCESS. The new JARs are on the classpath but nothing in the agent uses them yet — they compile in silently.

- [ ] **Step 4: Run full test suite**

```bash
cd /Users/mdproctor/claude/starcraft
mvn test -q
```

Expected: BUILD SUCCESS, all existing tests pass.

- [ ] **Step 5: Commit**

```bash
cd /Users/mdproctor/claude/starcraft
git add pom.xml CLAUDE.md
git commit -m "feat: add scelight-mpq and scelight-s2protocol dependencies for SC2 replay parsing"
```

---

## Done When

- [ ] `./scripts/publish-replay-libs.sh` completes in under 15 seconds
- [ ] Both JARs present in `~/.m2/repository/hu/scelight/`
- [ ] `mvn compile -q` in `starcraft-agent` succeeds with the new dependencies
- [ ] `mvn test -q` in `starcraft-agent` still passes all tests
- [ ] `NATIVE.md` updated: add entries for `scelight-mpq` (✅ zero reflection — native safe) and `scelight-s2protocol` (⚠️ resource inclusion needed for `build/*.dat` files)
