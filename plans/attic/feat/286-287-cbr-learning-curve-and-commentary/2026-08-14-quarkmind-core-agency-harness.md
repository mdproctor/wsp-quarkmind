# quarkmind-core Agency Harness — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #278 — quarkmind-core — extract shared agency harness
**Issue group:** #278

**Goal:** Populate quarkmind-core with the shared agency framework — surgically extract generic patterns from quarkmind-sc2 and flesh out marker interfaces with real method signatures, keeping SC2 tests green at every step.

**Architecture:** Move generic types (TaskDefinition, MapCaseContext, MutableMapCaseContext, MilestoneTracker/Session) from quarkmind-sc2 to quarkmind-core via IntelliJ MCP refactoring. Flesh out empty marker interfaces (InteractionTrigger, InteractionPipeline, MomentDetector, VisibilitySPI, SpatialMemory, LlmRequestQueue) with method signatures extracted from SC2 patterns. Expand AgencyContext to carry tick state. Add AgencySession as a generic session concept.

**Tech Stack:** Java 21, Maven multi-module, casehub-engine-api, casehub-eidos-api, JUnit 5

## Global Constraints

- All file moves via IntelliJ MCP (`ide_move_file`) — never bash cp/mv for Java files
- `mvn test` green after every task — surgical, never break SC2
- SC2 depends on quarkmind-core, so moved classes are importable after relocation
- No new CaseHub foundation dependencies unless compilation requires it
- quarkmind-core must remain plain Java + CaseHub APIs — no Quarkus-specific imports
- Commit after each task with `Refs #278`

---

### Task 1: Move TaskDefinition to quarkmind-core

**Files:**
- Move: `../../quarkmind-core/src/main/java/io/quarkmind/agency/task/TaskDefinition.java` → `quarkmind-core/src/main/java/io/quarkmind/agency/task/`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/task/TaskDefinitionTest.java`

**Interfaces:**
- Consumes: `io.casehub.api.context.CaseContext` (from casehub-engine-api, already a core dependency)
- Produces: `io.quarkmind.agency.task.TaskDefinition` — interface with `getId()`, `getName()`, `requires()`, `activateIf()`, `execute(CaseContext)`, `produces()`, `testActivation(CaseContext)`

- [ ] **Step 1: Write test in quarkmind-core**

```java
package io.quarkmind.agency.task;

import io.casehub.api.context.CaseContext;
import org.junit.jupiter.api.Test;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;

class TaskDefinitionTest {

    @Test
    void defaultMethods_returnSensibleDefaults() {
        TaskDefinition task = new TaskDefinition() {
            @Override public String getId() { return "test-plugin"; }
            @Override public String getName() { return "Test Plugin"; }
            @Override public void execute(CaseContext ctx) {}
        };

        assertEquals("test-plugin", task.getId());
        assertEquals("Test Plugin", task.getName());
        assertEquals(Set.of(), task.requires());
        assertEquals(Set.of(), task.produces());
        assertNotNull(task.activateIf());
    }

    @Test
    void requires_canBeOverridden() {
        TaskDefinition task = new TaskDefinition() {
            @Override public String getId() { return "gated"; }
            @Override public String getName() { return "Gated"; }
            @Override public Set<String> requires() { return Set.of("READY", "STRATEGY"); }
            @Override public void execute(CaseContext ctx) {}
        };

        assertEquals(Set.of("READY", "STRATEGY"), task.requires());
    }
}
```

- [ ] **Step 2: Run test — verify it fails (TaskDefinition not in core yet)**

Run: `mvn test -pl quarkmind-core -Dtest=TaskDefinitionTest -q`
Expected: compilation failure — `TaskDefinition` does not exist in quarkmind-core

- [ ] **Step 3: Move TaskDefinition via IntelliJ MCP**

Use `ide_move_file`:
- file: `../../quarkmind-core/src/main/java/io/quarkmind/agency/task/TaskDefinition.java`
- destination: `quarkmind-core/src/main/java/io/quarkmind/agency/task/`

IntelliJ updates all 30+ references in quarkmind-sc2 (StrategyTask, EconomicsTask, TacticsTask, ScoutingTask, MomentDetectionSeam, SC2StrategyRouterTask, PluginDispatchBroker, TickOrchestratorWorker, QuarkMindCaseHub, all test stubs).

- [ ] **Step 4: Verify compilation across all modules**

Run: `mvn compile`
Expected: BUILD SUCCESS — all imports updated by IntelliJ

- [ ] **Step 5: Run all tests**

Run: `mvn test -q`
Expected: BUILD SUCCESS — all SC2 tests pass with the new import path

- [ ] **Step 6: Commit**

```
feat(#278): move TaskDefinition to quarkmind-core

Refs #278
```

---

### Task 2: Move MapCaseContext and MutableMapCaseContext to quarkmind-core

**Files:**
- Move: `../../quarkmind-core/src/main/java/io/quarkmind/agency/context/MapCaseContext.java` → `quarkmind-core/src/main/java/io/quarkmind/agency/context/`
- Move: `../../quarkmind-core/src/main/java/io/quarkmind/agency/context/MutableMapCaseContext.java` → `quarkmind-core/src/main/java/io/quarkmind/agency/context/`
- Move: `../../quarkmind-core/src/test/java/io/quarkmind/agency/context/MapCaseContextTest.java` → `quarkmind-core/src/test/java/io/quarkmind/agency/context/`

**Interfaces:**
- Consumes: `io.casehub.api.context.CaseContext` (from casehub-engine-api)
- Produces: `io.quarkmind.agency.context.MapCaseContext` — read-only CaseContext backed by Map; `io.quarkmind.agency.context.MutableMapCaseContext` — writable CaseContext with mutation tracking via `mutations()`

- [ ] **Step 1: Move MapCaseContext via IntelliJ MCP**

Use `ide_move_file`:
- file: `../../quarkmind-core/src/main/java/io/quarkmind/agency/context/MapCaseContext.java`
- destination: `quarkmind-core/src/main/java/io/quarkmind/agency/context/`

IntelliJ updates 100+ references across quarkmind-sc2 production and test code.

- [ ] **Step 2: Move MutableMapCaseContext via IntelliJ MCP**

Use `ide_move_file`:
- file: `../../quarkmind-core/src/main/java/io/quarkmind/agency/context/MutableMapCaseContext.java`
- destination: `quarkmind-core/src/main/java/io/quarkmind/agency/context/`

- [ ] **Step 3: Move MapCaseContextTest via IntelliJ MCP**

Use `ide_move_file`:
- file: `../../quarkmind-core/src/test/java/io/quarkmind/agency/context/MapCaseContextTest.java`
- destination: `quarkmind-core/src/test/java/io/quarkmind/agency/context/`

This test has no SC2-specific imports — it only tests MapCaseContext directly.

- [ ] **Step 4: Verify compilation across all modules**

Run: `mvn compile`

If compilation fails with missing `jackson-databind` in quarkmind-core, add to `quarkmind-core/pom.xml`:
```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```
(Version managed by parent POM. Only add if casehub-engine-api doesn't pull it transitively.)

- [ ] **Step 5: Run all tests**

Run: `mvn test -q`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```
feat(#278): move MapCaseContext and MutableMapCaseContext to quarkmind-core

Refs #278
```

---

### Task 3: Move MilestoneTracker and MilestoneSession to quarkmind-core

**Files:**
- Move: `../../quarkmind-core/src/main/java/io/quarkmind/agency/milestone/MilestoneTracker.java` → `quarkmind-core/src/main/java/io/quarkmind/agency/milestone/`
- Move: `../../quarkmind-core/src/main/java/io/quarkmind/agency/milestone/MilestoneSession.java` → `quarkmind-core/src/main/java/io/quarkmind/agency/milestone/`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/milestone/MilestoneSessionTest.java`

**Interfaces:**
- Consumes: nothing from core (standalone)
- Produces: `io.quarkmind.agency.milestone.MilestoneTracker` — interface with `hasFired(String)`, `markFired(String)`; `io.quarkmind.agency.milestone.MilestoneSession` — `@ApplicationScoped` implementation with concurrent collections and `reset()`, `entryId(String)`, `setEntryId(String, UUID)`

- [ ] **Step 1: Write test in quarkmind-core**

```java
package io.quarkmind.agency.milestone;

import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class MilestoneSessionTest {

    @Test
    void hasFired_returnsFalse_beforeMarking() {
        var session = new MilestoneSession();
        assertFalse(session.hasFired("milestone-1"));
    }

    @Test
    void markFired_thenHasFired_returnsTrue() {
        var session = new MilestoneSession();
        session.markFired("milestone-1");
        assertTrue(session.hasFired("milestone-1"));
    }

    @Test
    void entryId_returnsEmpty_beforeSet() {
        var session = new MilestoneSession();
        assertTrue(session.entryId("strategy-1").isEmpty());
    }

    @Test
    void setEntryId_thenEntryId_returnsValue() {
        var session = new MilestoneSession();
        UUID id = UUID.randomUUID();
        session.setEntryId("strategy-1", id);
        assertEquals(id, session.entryId("strategy-1").orElseThrow());
    }

    @Test
    void reset_clearsAllState() {
        var session = new MilestoneSession();
        session.markFired("m1");
        session.setEntryId("s1", UUID.randomUUID());
        session.reset();
        assertFalse(session.hasFired("m1"));
        assertTrue(session.entryId("s1").isEmpty());
    }
}
```

- [ ] **Step 2: Move MilestoneTracker via IntelliJ MCP**

Use `ide_move_file`:
- file: `../../quarkmind-core/src/main/java/io/quarkmind/agency/milestone/MilestoneTracker.java`
- destination: `quarkmind-core/src/main/java/io/quarkmind/agency/milestone/`

- [ ] **Step 3: Move MilestoneSession via IntelliJ MCP**

Use `ide_move_file`:
- file: `../../quarkmind-core/src/main/java/io/quarkmind/agency/milestone/MilestoneSession.java`
- destination: `quarkmind-core/src/main/java/io/quarkmind/agency/milestone/`

- [ ] **Step 4: Check for CDI API dependency**

If compilation fails on `jakarta.enterprise.context.ApplicationScoped`, add to `quarkmind-core/pom.xml`:
```xml
<dependency>
    <groupId>jakarta.enterprise</groupId>
    <artifactId>jakarta.enterprise.cdi-api</artifactId>
    <scope>provided</scope>
</dependency>
```

Run: `mvn compile`
Expected: BUILD SUCCESS

- [ ] **Step 5: Run all tests**

Run: `mvn test -q`
Expected: BUILD SUCCESS (core tests pass, SC2 tests pass with updated imports)

- [ ] **Step 6: Commit**

```
feat(#278): move MilestoneTracker and MilestoneSession to quarkmind-core

Refs #278
```

---

### Task 4: Expand AgencyContext with tick and state bag

**Files:**
- Modify: `quarkmind-core/src/main/java/io/quarkmind/agency/AgencyContext.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/AgencyContextTest.java`

**Interfaces:**
- Consumes: `NeedState` (existing)
- Produces: `AgencyContext` — expanded with `tick()`, `setTick(long)`, `put(String, Object)`, `get(String)`, `getAs(String, Class<T>)`, `contains(String)`

- [ ] **Step 1: Write failing test**

```java
package io.quarkmind.agency;

import io.quarkmind.agency.needs.NeedState;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class AgencyContextTest {

    @Test
    void tickDefaultsToZero() {
        var ctx = new AgencyContext(new NeedState());
        assertEquals(0L, ctx.tick());
    }

    @Test
    void setTickUpdatesValue() {
        var ctx = new AgencyContext(new NeedState());
        ctx.setTick(42L);
        assertEquals(42L, ctx.tick());
    }

    @Test
    void putAndGet() {
        var ctx = new AgencyContext(new NeedState());
        ctx.put("minerals", 150);
        assertEquals(150, ctx.get("minerals"));
    }

    @Test
    void getAsReturnsTypedValue() {
        var ctx = new AgencyContext(new NeedState());
        ctx.put("minerals", 150);
        assertEquals(150, ctx.getAs("minerals", Integer.class));
    }

    @Test
    void getAsReturnsNullForWrongType() {
        var ctx = new AgencyContext(new NeedState());
        ctx.put("minerals", 150);
        assertNull(ctx.getAs("minerals", String.class));
    }

    @Test
    void containsReturnsTrueForPresentKey() {
        var ctx = new AgencyContext(new NeedState());
        ctx.put("ready", true);
        assertTrue(ctx.contains("ready"));
    }

    @Test
    void containsReturnsFalseForAbsentKey() {
        var ctx = new AgencyContext(new NeedState());
        assertFalse(ctx.contains("missing"));
    }

    @Test
    void needStateIsAccessible() {
        var needs = new NeedState();
        needs.set("hunger", 80.0);
        var ctx = new AgencyContext(needs);
        assertEquals(80.0, ctx.needState().get("hunger"), 0.01);
    }
}
```

- [ ] **Step 2: Run test — verify failure**

Run: `mvn test -pl quarkmind-core -Dtest=AgencyContextTest -q`
Expected: FAIL — `tick()`, `setTick()`, `put()`, `get()`, `getAs()`, `contains()` not defined

- [ ] **Step 3: Implement expanded AgencyContext**

Replace `AgencyContext.java` content with:

```java
package io.quarkmind.agency;

import io.quarkmind.agency.needs.NeedState;
import java.util.HashMap;
import java.util.Map;

public class AgencyContext {

    private final NeedState needState;
    private long tick;
    private final Map<String, Object> state = new HashMap<>();

    public AgencyContext(NeedState needState) {
        this.needState = needState;
    }

    public NeedState needState() { return needState; }

    public long tick() { return tick; }

    public void setTick(long tick) { this.tick = tick; }

    public void put(String key, Object value) { state.put(key, value); }

    public Object get(String key) { return state.get(key); }

    @SuppressWarnings("unchecked")
    public <T> T getAs(String key, Class<T> type) {
        Object v = state.get(key);
        return type.isInstance(v) ? (T) v : null;
    }

    public boolean contains(String key) { return state.containsKey(key); }
}
```

- [ ] **Step 4: Run tests — verify pass**

Run: `mvn test -pl quarkmind-core -q`
Expected: BUILD SUCCESS — all core tests pass

- [ ] **Step 5: Run SC2 tests — verify no breakage**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: BUILD SUCCESS — AgencyContext is not yet used by SC2

- [ ] **Step 6: Commit**

```
feat(#278): expand AgencyContext with tick and state bag

Refs #278
```

---

### Task 5: Flesh out InteractionTrigger and InteractionPipeline

**Files:**
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/interaction/TriggerEvent.java`
- Modify: `quarkmind-core/src/main/java/io/quarkmind/agency/interaction/InteractionTrigger.java`
- Modify: `quarkmind-core/src/main/java/io/quarkmind/agency/interaction/InteractionPipeline.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/interaction/InteractionTriggerTest.java`

**Interfaces:**
- Consumes: `AgencyContext` (from Task 4)
- Produces: `TriggerEvent(String type, Map<String, Object> data)` — immutable event record; `InteractionTrigger.evaluate(AgencyContext) → Optional<TriggerEvent>`; `InteractionPipeline.evaluate(AgencyContext)`

- [ ] **Step 1: Write failing test**

```java
package io.quarkmind.agency.interaction;

import io.quarkmind.agency.AgencyContext;
import io.quarkmind.agency.needs.NeedState;
import org.junit.jupiter.api.Test;
import java.util.Map;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;

class InteractionTriggerTest {

    @Test
    void triggerEvent_isImmutable() {
        var event = new TriggerEvent("crisis", Map.of("severity", "high"));
        assertEquals("crisis", event.type());
        assertEquals("high", event.data().get("severity"));
        assertThrows(UnsupportedOperationException.class, () -> event.data().put("x", "y"));
    }

    @Test
    void trigger_evaluatesContext_andFiresWhenConditionMet() {
        InteractionTrigger trigger = ctx -> {
            if (ctx.contains("crisis")) {
                return Optional.of(new TriggerEvent("crisis", Map.of("frame", ctx.tick())));
            }
            return Optional.empty();
        };

        var ctx = new AgencyContext(new NeedState());
        ctx.setTick(100L);

        assertTrue(trigger.evaluate(ctx).isEmpty());

        ctx.put("crisis", true);
        var event = trigger.evaluate(ctx);
        assertTrue(event.isPresent());
        assertEquals("crisis", event.get().type());
        assertEquals(100L, event.get().data().get("frame"));
    }

    @Test
    void pipeline_evaluatesContext() {
        var fired = new boolean[]{false};
        InteractionPipeline pipeline = ctx -> fired[0] = true;

        var ctx = new AgencyContext(new NeedState());
        pipeline.evaluate(ctx);
        assertTrue(fired[0]);
    }
}
```

- [ ] **Step 2: Run test — verify failure**

Run: `mvn test -pl quarkmind-core -Dtest=InteractionTriggerTest -q`
Expected: FAIL — `TriggerEvent` not found, `evaluate` method not on interfaces

- [ ] **Step 3: Create TriggerEvent record**

Create `quarkmind-core/src/main/java/io/quarkmind/agency/interaction/TriggerEvent.java`:

```java
package io.quarkmind.agency.interaction;

import java.util.Map;

public record TriggerEvent(String type, Map<String, Object> data) {
    public TriggerEvent {
        data = data != null ? Map.copyOf(data) : Map.of();
    }
}
```

- [ ] **Step 4: Update InteractionTrigger**

Replace `InteractionTrigger.java` content with:

```java
package io.quarkmind.agency.interaction;

import io.quarkmind.agency.AgencyContext;
import java.util.Optional;

public interface InteractionTrigger {
    Optional<TriggerEvent> evaluate(AgencyContext context);
}
```

- [ ] **Step 5: Update InteractionPipeline**

Replace `InteractionPipeline.java` content with:

```java
package io.quarkmind.agency.interaction;

import io.quarkmind.agency.AgencyContext;

public interface InteractionPipeline {
    void evaluate(AgencyContext context);
}
```

- [ ] **Step 6: Run tests — verify pass**

Run: `mvn test -pl quarkmind-core -q`
Expected: BUILD SUCCESS

- [ ] **Step 7: Run SC2 tests — verify no breakage**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: BUILD SUCCESS — SC2 doesn't implement these interfaces yet

- [ ] **Step 8: Commit**

```
feat(#278): flesh out InteractionTrigger and InteractionPipeline SPIs

Refs #278
```

---

### Task 6: Flesh out MomentDetector with MomentEvent

**Files:**
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/moment/MomentEvent.java`
- Modify: `quarkmind-core/src/main/java/io/quarkmind/agency/moment/MomentDetector.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/moment/MomentDetectorTest.java`

**Interfaces:**
- Consumes: `AgencyContext` (from Task 4)
- Produces: `MomentEvent(String type, long tick, Map<String, Object> data)` — immutable event; `MomentDetector.detect(AgencyContext) → List<MomentEvent>`

- [ ] **Step 1: Write failing test**

```java
package io.quarkmind.agency.moment;

import io.quarkmind.agency.AgencyContext;
import io.quarkmind.agency.needs.NeedState;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class MomentDetectorTest {

    @Test
    void momentEvent_isImmutable() {
        var event = new MomentEvent("battle_start", 500L, Map.of("units", 12));
        assertEquals("battle_start", event.type());
        assertEquals(500L, event.tick());
        assertThrows(UnsupportedOperationException.class, () -> event.data().put("x", "y"));
    }

    @Test
    void momentEvent_handlesNullData() {
        var event = new MomentEvent("first_contact", 100L, null);
        assertNotNull(event.data());
        assertTrue(event.data().isEmpty());
    }

    @Test
    void detector_returnsEmptyWhenNoMoments() {
        MomentDetector detector = ctx -> List.of();
        var ctx = new AgencyContext(new NeedState());
        assertTrue(detector.detect(ctx).isEmpty());
    }

    @Test
    void detector_returnsEventsWhenConditionsMet() {
        MomentDetector detector = ctx -> {
            if (ctx.contains("supply_blocked")) {
                return List.of(new MomentEvent("supply_block", ctx.tick(), Map.of()));
            }
            return List.of();
        };

        var ctx = new AgencyContext(new NeedState());
        ctx.setTick(200L);
        ctx.put("supply_blocked", true);

        var events = detector.detect(ctx);
        assertEquals(1, events.size());
        assertEquals("supply_block", events.get(0).type());
        assertEquals(200L, events.get(0).tick());
    }
}
```

- [ ] **Step 2: Run test — verify failure**

Run: `mvn test -pl quarkmind-core -Dtest=MomentDetectorTest -q`
Expected: FAIL — `MomentEvent` not found, `detect` method not on interface

- [ ] **Step 3: Create MomentEvent record**

Create `quarkmind-core/src/main/java/io/quarkmind/agency/moment/MomentEvent.java`:

```java
package io.quarkmind.agency.moment;

import java.util.Map;

public record MomentEvent(String type, long tick, Map<String, Object> data) {
    public MomentEvent {
        data = data != null ? Map.copyOf(data) : Map.of();
    }
}
```

- [ ] **Step 4: Update MomentDetector**

Replace `MomentDetector.java` content with:

```java
package io.quarkmind.agency.moment;

import io.quarkmind.agency.AgencyContext;
import java.util.List;

public interface MomentDetector {
    List<MomentEvent> detect(AgencyContext context);
}
```

- [ ] **Step 5: Run tests — verify pass**

Run: `mvn test -pl quarkmind-core -q`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```
feat(#278): flesh out MomentDetector with MomentEvent

Refs #278
```

---

### Task 7: Flesh out spatial SPIs

**Files:**
- Modify: `quarkmind-core/src/main/java/io/quarkmind/agency/spatial/VisibilitySPI.java`
- Modify: `quarkmind-core/src/main/java/io/quarkmind/agency/spatial/SpatialMemory.java`
- Modify: `quarkmind-core/src/main/java/io/quarkmind/agency/spatial/NavigationSPI.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/spatial/SpatialSPITest.java`

**Interfaces:**
- Consumes: nothing (standalone SPIs)
- Produces: `VisibilitySPI<E>` with `visible()`, `remembered()`; `SpatialMemory` with `remember(String, Map)`, `recall(String)`, `knownLocations()`; `NavigationSPI` with `isReachable(double, double)` (unchanged), `pathTo(double, double)`

- [ ] **Step 1: Write failing test**

```java
package io.quarkmind.agency.spatial;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.HashMap;

import static org.junit.jupiter.api.Assertions.*;

class SpatialSPITest {

    @Test
    void visibilitySPI_returnsVisibleAndRemembered() {
        VisibilitySPI<String> vis = new VisibilitySPI<>() {
            @Override public Set<String> visible() { return Set.of("unit-1", "unit-2"); }
            @Override public Set<String> remembered() { return Set.of("unit-3"); }
        };

        assertEquals(2, vis.visible().size());
        assertEquals(1, vis.remembered().size());
        assertTrue(vis.visible().contains("unit-1"));
        assertTrue(vis.remembered().contains("unit-3"));
    }

    @Test
    void spatialMemory_rememberAndRecall() {
        var memory = new SpatialMemory() {
            private final Map<String, Map<String, Object>> store = new HashMap<>();

            @Override public void remember(String locationId, Map<String, Object> observation) {
                store.put(locationId, Map.copyOf(observation));
            }
            @Override public Map<String, Object> recall(String locationId) {
                return store.getOrDefault(locationId, Map.of());
            }
            @Override public Set<String> knownLocations() {
                return Set.copyOf(store.keySet());
            }
        };

        memory.remember("base-1", Map.of("minerals", 1500));
        assertEquals(1500, memory.recall("base-1").get("minerals"));
        assertTrue(memory.knownLocations().contains("base-1"));
        assertTrue(memory.recall("unknown").isEmpty());
    }

    @Test
    void navigationSPI_pathToDefaultReturnsEmptyList() {
        NavigationSPI nav = (x, y) -> true;
        assertTrue(nav.pathTo(10.0, 20.0).isEmpty());
    }

    @Test
    void navigationSPI_pathToCanBeOverridden() {
        NavigationSPI nav = new NavigationSPI() {
            @Override public boolean isReachable(double x, double y) { return true; }
            @Override public List<double[]> pathTo(double x, double y) {
                return List.of(new double[]{0, 0}, new double[]{x, y});
            }
        };

        var path = nav.pathTo(10.0, 20.0);
        assertEquals(2, path.size());
        assertArrayEquals(new double[]{10.0, 20.0}, path.get(1));
    }
}
```

- [ ] **Step 2: Run test — verify failure**

Run: `mvn test -pl quarkmind-core -Dtest=SpatialSPITest -q`
Expected: FAIL — methods not defined on interfaces

- [ ] **Step 3: Update VisibilitySPI**

Replace `VisibilitySPI.java` content with:

```java
package io.quarkmind.agency.spatial;

import java.util.Set;

public interface VisibilitySPI<E> {
    Set<E> visible();
    Set<E> remembered();
}
```

- [ ] **Step 4: Update SpatialMemory**

Replace `SpatialMemory.java` content with:

```java
package io.quarkmind.agency.spatial;

import java.util.Map;
import java.util.Set;

public interface SpatialMemory {
    void remember(String locationId, Map<String, Object> observation);
    Map<String, Object> recall(String locationId);
    Set<String> knownLocations();
}
```

- [ ] **Step 5: Update NavigationSPI**

Replace `NavigationSPI.java` content with:

```java
package io.quarkmind.agency.spatial;

import java.util.List;

public interface NavigationSPI {
    boolean isReachable(double x, double y);

    default List<double[]> pathTo(double x, double y) {
        return List.of();
    }
}
```

- [ ] **Step 6: Run tests — verify pass**

Run: `mvn test -pl quarkmind-core -q`
Expected: BUILD SUCCESS

- [ ] **Step 7: Run SC2 tests — verify no breakage**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: BUILD SUCCESS — SC2 doesn't implement these interfaces yet

- [ ] **Step 8: Commit**

```
feat(#278): flesh out spatial SPIs — VisibilitySPI, SpatialMemory, NavigationSPI

Refs #278
```

---

### Task 8: Flesh out LlmRequestQueue

**Files:**
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/llm/LlmRequest.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/llm/LlmPriority.java`
- Modify: `quarkmind-core/src/main/java/io/quarkmind/agency/llm/LlmRequestQueue.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/llm/LlmRequestQueueTest.java`

**Interfaces:**
- Consumes: nothing (standalone)
- Produces: `LlmPriority` enum (LOW, NORMAL, HIGH, URGENT); `LlmRequest(String prompt, LlmPriority priority, Map<String, Object> metadata)` record; `LlmRequestQueue` with `submit(LlmRequest)`, `pendingCount()`, `hasCapacity()`

- [ ] **Step 1: Write failing test**

```java
package io.quarkmind.agency.llm;

import org.junit.jupiter.api.Test;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class LlmRequestQueueTest {

    @Test
    void llmPriority_ordering() {
        assertTrue(LlmPriority.URGENT.ordinal() > LlmPriority.HIGH.ordinal());
        assertTrue(LlmPriority.HIGH.ordinal() > LlmPriority.NORMAL.ordinal());
        assertTrue(LlmPriority.NORMAL.ordinal() > LlmPriority.LOW.ordinal());
    }

    @Test
    void llmRequest_isImmutable() {
        var request = new LlmRequest("analyse position", LlmPriority.HIGH, Map.of("role", "advisor"));
        assertEquals("analyse position", request.prompt());
        assertEquals(LlmPriority.HIGH, request.priority());
        assertThrows(UnsupportedOperationException.class, () -> request.metadata().put("x", "y"));
    }

    @Test
    void llmRequest_handlesNullMetadata() {
        var request = new LlmRequest("prompt", LlmPriority.NORMAL, null);
        assertNotNull(request.metadata());
        assertTrue(request.metadata().isEmpty());
    }

    @Test
    void queue_interface_isImplementable() {
        LlmRequestQueue queue = new LlmRequestQueue() {
            private int count = 0;
            @Override public void submit(LlmRequest request) { count++; }
            @Override public int pendingCount() { return count; }
            @Override public boolean hasCapacity() { return count < 10; }
        };

        assertTrue(queue.hasCapacity());
        assertEquals(0, queue.pendingCount());

        queue.submit(new LlmRequest("test", LlmPriority.NORMAL, Map.of()));
        assertEquals(1, queue.pendingCount());
        assertTrue(queue.hasCapacity());
    }
}
```

- [ ] **Step 2: Run test — verify failure**

Run: `mvn test -pl quarkmind-core -Dtest=LlmRequestQueueTest -q`
Expected: FAIL — `LlmPriority`, `LlmRequest` not found, methods not on `LlmRequestQueue`

- [ ] **Step 3: Create LlmPriority enum**

Create `quarkmind-core/src/main/java/io/quarkmind/agency/llm/LlmPriority.java`:

```java
package io.quarkmind.agency.llm;

public enum LlmPriority {
    LOW,
    NORMAL,
    HIGH,
    URGENT
}
```

- [ ] **Step 4: Create LlmRequest record**

Create `quarkmind-core/src/main/java/io/quarkmind/agency/llm/LlmRequest.java`:

```java
package io.quarkmind.agency.llm;

import java.util.Map;

public record LlmRequest(String prompt, LlmPriority priority, Map<String, Object> metadata) {
    public LlmRequest {
        metadata = metadata != null ? Map.copyOf(metadata) : Map.of();
    }
}
```

- [ ] **Step 5: Update LlmRequestQueue**

Replace `LlmRequestQueue.java` content with:

```java
package io.quarkmind.agency.llm;

public interface LlmRequestQueue {
    void submit(LlmRequest request);
    int pendingCount();
    boolean hasCapacity();
}
```

- [ ] **Step 6: Run tests — verify pass**

Run: `mvn test -pl quarkmind-core -q`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```
feat(#278): flesh out LlmRequestQueue with LlmRequest and LlmPriority

Refs #278
```

---

### Task 9: Add AgencySession (generalized from GameSession)

**Files:**
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/session/AgencySession.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/session/AgencySessionTest.java`

**Interfaces:**
- Consumes: nothing (standalone)
- Produces: `AgencySession` — session identity with `id()`, `reset()`, `setId(UUID)`. SC2's `GameSession` can extend this in follow-up work.

- [ ] **Step 1: Write failing test**

```java
package io.quarkmind.agency.session;

import org.junit.jupiter.api.Test;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class AgencySessionTest {

    @Test
    void newSession_hasRandomId() {
        var session = new AgencySession();
        assertNotNull(session.id());
    }

    @Test
    void reset_generatesNewId() {
        var session = new AgencySession();
        UUID first = session.id();
        session.reset();
        assertNotEquals(first, session.id());
    }

    @Test
    void setId_overridesCurrentId() {
        var session = new AgencySession();
        UUID custom = UUID.randomUUID();
        session.setId(custom);
        assertEquals(custom, session.id());
    }

    @Test
    void twoSessions_haveDifferentIds() {
        var s1 = new AgencySession();
        var s2 = new AgencySession();
        assertNotEquals(s1.id(), s2.id());
    }
}
```

- [ ] **Step 2: Run test — verify failure**

Run: `mvn test -pl quarkmind-core -Dtest=AgencySessionTest -q`
Expected: FAIL — `AgencySession` not found

- [ ] **Step 3: Implement AgencySession**

Create `quarkmind-core/src/main/java/io/quarkmind/agency/session/AgencySession.java`:

```java
package io.quarkmind.agency.session;

import java.util.UUID;

public class AgencySession {

    private volatile UUID id = UUID.randomUUID();

    public UUID id() { return id; }

    public void reset() { id = UUID.randomUUID(); }

    public void setId(UUID id) { this.id = id; }
}
```

- [ ] **Step 4: Run tests — verify pass**

Run: `mvn test -pl quarkmind-core -q`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```
feat(#278): add AgencySession — generic session identity

Refs #278
```

---

### Task 10: Full cross-module verification

**Files:**
- No changes — verification only

**Interfaces:**
- Consumes: all tasks above
- Produces: confidence that all modules compile and test green

- [ ] **Step 1: Clean build all modules**

Run: `mvn clean compile`
Expected: BUILD SUCCESS across quarkmind-core, quarkmind-sc2, and all stub modules

- [ ] **Step 2: Run all tests**

Run: `mvn test`
Expected: BUILD SUCCESS — all quarkmind-core tests pass, all quarkmind-sc2 tests pass

- [ ] **Step 3: Verify quarkmind-core test count**

Run: `mvn test -pl quarkmind-core -q`

Expected tests:
- `NeedStateTest` (5 tests — existing)
- `IntentQueueTest` (3 tests — existing)
- `AgencyContextTest` (8 tests — Task 4)
- `MilestoneSessionTest` (5 tests — Task 3)
- `MapCaseContextTest` (10+ tests — moved in Task 2)
- `TaskDefinitionTest` (2 tests — Task 1)
- `InteractionTriggerTest` (3 tests — Task 5)
- `MomentDetectorTest` (4 tests — Task 6)
- `SpatialSPITest` (4 tests — Task 7)
- `LlmRequestQueueTest` (4 tests — Task 8)
- `AgencySessionTest` (4 tests — Task 9)

Total: ~50+ tests in quarkmind-core

- [ ] **Step 4: Verify quarkmind-sc2 test count unchanged**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: same test count as before #278 — no tests lost during extraction
