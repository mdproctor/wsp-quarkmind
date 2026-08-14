# QuarkMind Restructure Phase 1 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Spec:** `specs/quarkmind-restructure/2026-08-14-quarkmind-restructure-design.md`
**Decisions:** `specs/quarkmind-restructure/decisions.md` (D1–D17)

**Goal:** Convert QuarkMind from a single-module SC2 project into a multi-module mono-repo with a shared agency framework (quarkmind-core) and SC2-specific implementation (quarkmind-sc2), plus stub modules for future worlds.

**Architecture:** Phase 1 is purely structural. Move all existing code into quarkmind-sc2, create quarkmind-core with new SPI interfaces, wire SC2 to depend on core. No behavioural changes — all existing tests must pass throughout.

**Tech Stack:** Java 26, Maven, Quarkus, CaseHub foundations, IntelliJ MCP

## Global Constraints

- All refactoring via IntelliJ MCP — no bash file operations on source files
- No subagents — main session with human oversight
- SC2 test suite must stay green after every task
- Incremental commits at each stable point
- No behavioural changes in Phase 1 — structural only
- IntelliJ project must be open on `/Users/mdproctor/claude/casehub/quarkmind` before starting

## Pre-Implementation Checklist

- [ ] Ensure IntelliJ has quarkmind project open (`ide_index_status`)
- [ ] Verify current tests pass: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q`
- [ ] Create branch: `git -C /Users/mdproctor/claude/casehub/quarkmind checkout -b issue-XXX-quarkmind-restructure`
- [ ] Create matching GitHub issue for the restructure

---

### Task 1: Convert to Multi-Module Maven Project

Move all existing code into a `quarkmind-sc2` submodule. The current root `pom.xml` becomes a parent POM. This is the riskiest step — get it right and everything else is incremental.

**Files:**
- Modify: `pom.xml` (convert to parent POM with `<packaging>pom</packaging>`)
- Create: `quarkmind-sc2/pom.xml` (child POM inheriting from parent, gets all current dependencies)
- Move: `src/` → `quarkmind-sc2/src/` (use `ide_move_file` or bash for directory move since this is a bulk operation)
- Move: `data/` → `quarkmind-sc2/data/`
- Move: `replays/` → `quarkmind-sc2/replays/`

**Interfaces:**
- Produces: Multi-module Maven build where `quarkmind-sc2` contains all current code and passes all tests

- [ ] **Step 1: Create quarkmind-sc2 directory structure**

```bash
mkdir -p /Users/mdproctor/claude/casehub/quarkmind/quarkmind-sc2
```

- [ ] **Step 2: Move source tree into quarkmind-sc2**

```bash
# Move source, data, and replay directories
# This is a bulk directory move — not a code refactoring, so bash is acceptable
git -C /Users/mdproctor/claude/casehub/quarkmind mv src quarkmind-sc2/src
git -C /Users/mdproctor/claude/casehub/quarkmind mv data quarkmind-sc2/data
git -C /Users/mdproctor/claude/casehub/quarkmind mv replays quarkmind-sc2/replays
```

- [ ] **Step 3: Create quarkmind-sc2/pom.xml**

Copy current `pom.xml` content into `quarkmind-sc2/pom.xml`. Change the parent to point to the root quarkmind POM. Keep all dependencies and plugins. Change artifactId to `quarkmind-sc2`.

Key changes:
- `<parent>` points to `io.quarkmind:quarkmind` (the new root)
- `<artifactId>quarkmind-sc2</artifactId>`
- All existing dependencies stay
- All existing plugins stay
- All existing profiles stay

- [ ] **Step 4: Convert root pom.xml to parent POM**

Convert root `pom.xml`:
- Change `<artifactId>` from `quarkmind-agent` to `quarkmind`
- Add `<packaging>pom</packaging>`
- Add `<modules><module>quarkmind-sc2</module></modules>`
- Move shared properties to parent
- Move `dependencyManagement` to parent
- Remove dependencies (they belong in quarkmind-sc2)
- Remove build plugins (they belong in quarkmind-sc2)

The parent still inherits from `casehub-parent`.

- [ ] **Step 5: Update resource paths in quarkmind-sc2**

Check for any hardcoded paths in application.properties, test resources, or Drools DRL files that reference the old directory structure. Update as needed.

```bash
# Check for relative path references that might break
grep -r "src/main" /Users/mdproctor/claude/casehub/quarkmind/quarkmind-sc2/src/ --include="*.java" --include="*.properties" --include="*.yaml" -l
```

- [ ] **Step 6: Verify build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml clean compile
```

- [ ] **Step 7: Verify all tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml -q
```

- [ ] **Step 8: Reload IntelliJ project**

```
ide_reload_project(project_path="/Users/mdproctor/claude/casehub/quarkmind")
```

IntelliJ needs to re-index after the structural change.

- [ ] **Step 9: Verify IntelliJ sees the multi-module structure**

```
ide_project_status(project_path="/Users/mdproctor/claude/casehub/quarkmind")
```

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "refactor: convert to multi-module — move all code to quarkmind-sc2

Refs #XXX"
```

---

### Task 2: Create quarkmind-core Module with SPI Interfaces

Create the shared agency framework module with the key SPI interfaces. These are NEW interfaces — no existing code moves yet. SC2 will implement them in Task 3.

**Files:**
- Create: `quarkmind-core/pom.xml`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/AgencyLoop.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/AgencyContext.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/AgencyPhase.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/spi/WorldBridge.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/spi/WorldPerception.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/intent/Intent.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/intent/IntentQueue.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/needs/Need.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/needs/NeedState.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/needs/DispositionNeedModifier.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/spatial/NavigationSPI.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/spatial/VisibilitySPI.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/spatial/SpatialMemory.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/interaction/InteractionTrigger.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/interaction/InteractionPipeline.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/moment/MomentDetector.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/llm/LlmRequestQueue.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/needs/NeedStateTest.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/intent/IntentQueueTest.java`

**Interfaces:**
- Produces: `io.quarkmind.agency` package with all SPI interfaces and core types. These are the contracts each world implements.

**Design notes:**
- SPIs use Java generics to support both strings and rich models (D16)
- `WorldBridge<P extends WorldPerception, I extends Intent>` — generic perception and intent types
- `IntentQueue<I extends Intent>` — generic intent buffer
- `NeedState` is a concrete class (decaying floats with disposition modifiers) — not an SPI
- `AgencyLoop` wraps CaseEngine (D14) — thin mapping of agency phases to TaskDefinitions
- Keep interfaces minimal — add methods when the first world implementation needs them, not before

- [ ] **Step 1: Create quarkmind-core/pom.xml**

Minimal POM inheriting from quarkmind parent. Dependencies on casehub-engine-api and casehub-eidos-api only (the SPI interfaces reference these).

- [ ] **Step 2: Add quarkmind-core to parent POM modules**

Add `<module>quarkmind-core</module>` to root `pom.xml` modules list, BEFORE quarkmind-sc2 (build order matters).

- [ ] **Step 3: Write NeedState test**

```java
package io.quarkmind.agency.needs;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class NeedStateTest {
    @Test
    void needDecaysOverTime() {
        var state = new NeedState();
        state.set("hunger", 100.0);
        state.decay("hunger", 2.0);  // decay rate
        assertEquals(98.0, state.get("hunger"), 0.01);
    }

    @Test
    void needClampsBetweenZeroAndMax() {
        var state = new NeedState();
        state.set("hunger", 1.0);
        state.decay("hunger", 5.0);
        assertEquals(0.0, state.get("hunger"), 0.01);
    }

    @Test
    void satisfyIncreasesNeed() {
        var state = new NeedState();
        state.set("hunger", 50.0);
        state.satisfy("hunger", 30.0);
        assertEquals(80.0, state.get("hunger"), 0.01);
    }
}
```

- [ ] **Step 4: Run test — verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/quarkmind/quarkmind-core/pom.xml -Dtest=NeedStateTest -q
```

- [ ] **Step 5: Implement NeedState**

```java
package io.quarkmind.agency.needs;

import java.util.HashMap;
import java.util.Map;

public class NeedState {
    private final Map<String, Double> levels = new HashMap<>();
    private static final double MAX = 100.0;
    private static final double MIN = 0.0;

    public void set(String need, double value) {
        levels.put(need, clamp(value));
    }

    public double get(String need) {
        return levels.getOrDefault(need, 0.0);
    }

    public void decay(String need, double rate) {
        set(need, get(need) - rate);
    }

    public void satisfy(String need, double amount) {
        set(need, get(need) + amount);
    }

    private double clamp(double value) {
        return Math.max(MIN, Math.min(MAX, value));
    }
}
```

- [ ] **Step 6: Run test — verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/quarkmind/quarkmind-core/pom.xml -Dtest=NeedStateTest -q
```

- [ ] **Step 7: Create SPI interfaces**

Create each SPI interface as a minimal contract. These are marker interfaces and generic contracts — implementation comes in Task 3 (SC2) and future world modules.

Key interfaces (all in `io.quarkmind.agency`):

```java
// spi/WorldBridge.java
public interface WorldBridge<P extends WorldPerception, I extends Intent> {
    void connect();
    void disconnect();
    P perceive();
    void dispatch(IntentQueue<I> intents);
}

// spi/WorldPerception.java — marker interface
public interface WorldPerception {}

// intent/Intent.java — marker interface
public interface Intent {}

// intent/IntentQueue.java — generic buffer
public class IntentQueue<I extends Intent> { ... }

// spatial/NavigationSPI.java
public interface NavigationSPI { ... }

// spatial/VisibilitySPI.java
public interface VisibilitySPI { ... }

// interaction/InteractionTrigger.java
public interface InteractionTrigger { ... }

// moment/MomentDetector.java
public interface MomentDetector { ... }
```

- [ ] **Step 8: Verify full build passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml -q
```

Both quarkmind-core tests and quarkmind-sc2 tests must pass.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat: quarkmind-core module with agency SPIs and NeedState

Introduces the shared agency framework: WorldBridge SPI, Intent/IntentQueue,
NeedState with decay/satisfy, spatial and interaction SPIs.

Refs #XXX"
```

---

### Task 3: Wire quarkmind-sc2 to Depend on quarkmind-core

Add quarkmind-core as a dependency of quarkmind-sc2. This is a wiring step — SC2 code doesn't implement the SPIs yet (that's future work when worlds need the abstraction). The goal is just establishing the dependency direction.

**Files:**
- Modify: `quarkmind-sc2/pom.xml` (add quarkmind-core dependency)

**Interfaces:**
- Consumes: quarkmind-core module from Task 2
- Produces: quarkmind-sc2 depends on quarkmind-core (dependency direction established)

- [ ] **Step 1: Add quarkmind-core dependency to quarkmind-sc2/pom.xml**

```xml
<dependency>
    <groupId>io.quarkmind</groupId>
    <artifactId>quarkmind-core</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 2: Verify build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml -q
```

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/pom.xml
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "chore: quarkmind-sc2 depends on quarkmind-core

Establishes the dependency direction: worlds depend on core, not the reverse.

Refs #XXX"
```

---

### Task 4: Create Stub Modules

Create empty Maven modules for each future world. Each has a minimal POM, depends on quarkmind-core, and contains a single placeholder class to verify the build.

**Files:**
- Create: `quarkmind-town/pom.xml`
- Create: `quarkmind-town/src/main/java/io/quarkmind/town/package-info.java`
- Create: `quarkmind-minecraft/pom.xml`
- Create: `quarkmind-minecraft/src/main/java/io/quarkmind/minecraft/package-info.java`
- Create: `quarkmind-evennia/pom.xml`
- Create: `quarkmind-evennia/src/main/java/io/quarkmind/evennia/package-info.java`
- Create: `quarkmind-sonaria/pom.xml`
- Create: `quarkmind-sonaria/src/main/java/io/quarkmind/sonaria/package-info.java`
- Create: `quarkmind-godot-mcp/pom.xml`
- Create: `quarkmind-godot-mcp/src/main/java/io/quarkmind/godot/package-info.java`
- Modify: Root `pom.xml` (add all modules)

**Interfaces:**
- Consumes: quarkmind-core from Task 2
- Produces: All modules declared and building

- [ ] **Step 1: Create each stub module**

For each module: create directory, create minimal `pom.xml` inheriting from quarkmind parent with quarkmind-core dependency, create `package-info.java`.

quarkmind-town POM adds Quarkus dependencies (it's the first world to build — needs websockets, rest, scheduler). Others are truly minimal.

- [ ] **Step 2: Add all modules to root pom.xml**

```xml
<modules>
    <module>quarkmind-core</module>
    <module>quarkmind-sc2</module>
    <module>quarkmind-town</module>
    <module>quarkmind-minecraft</module>
    <module>quarkmind-evennia</module>
    <module>quarkmind-sonaria</module>
    <module>quarkmind-godot-mcp</module>
</modules>
```

- [ ] **Step 3: Verify full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/quarkmind/pom.xml -q
```

All modules compile. SC2 tests still pass.

- [ ] **Step 4: Reload IntelliJ**

```
ide_reload_project(project_path="/Users/mdproctor/claude/casehub/quarkmind")
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat: stub modules for town, minecraft, evennia, sonaria, godot-mcp

Each module depends on quarkmind-core. Ready for parallel Phase 2 development.

Refs #XXX"
```

---

### Task 5: Update Documentation

Update project documentation to reflect the new multi-module structure.

**Files:**
- Modify: `CLAUDE.md` (update code organisation section, build commands)
- Modify: `MODULES.md` (replace "deferred indefinitely" with actual module structure)
- Modify: `ARC42STORIES.MD` (add note about multi-module restructure)

- [ ] **Step 1: Update CLAUDE.md**

Update the "Code Organisation" section to reflect modules. Update build commands to use `-pl quarkmind-sc2` for SC2-specific work. Add quarkmind-core to the description.

- [ ] **Step 2: Update MODULES.md**

Replace the "potential future split" table with the actual module structure. Document what each module contains and its status.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add CLAUDE.md MODULES.md
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "docs: update project docs for multi-module structure

Refs #XXX"
```

---

## Post-Implementation Verification

- [ ] `mvn clean test` passes from root (all modules)
- [ ] `mvn test -pl quarkmind-sc2` passes (SC2 isolation)
- [ ] `mvn test -pl quarkmind-core` passes (core isolation)
- [ ] IntelliJ shows all modules with correct dependencies
- [ ] No circular dependencies between modules
- [ ] `git log --oneline` shows clean incremental commits
