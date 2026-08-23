# Advisory Milestone Attestations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #224 — feat: advisory milestone attestations — per-advisor dominance assessment at game checkpoints
**Issue group:** #224

**Goal:** Extend milestone trust scoring from strategies to advisors, so each invoked advisor gets intermediate dominance attestations at game checkpoints.

**Architecture:** Reuse `MilestoneTrigger` and `DominanceAssessor` infrastructure. Extract a `MilestoneTracker` interface to enable trigger sharing between strategy and advisory sessions. Create `AdvisoryMilestoneOutcomeRecorder` (replacing `AdvisoryGameOutcomeRecorder`) with both milestone evaluation and game-end recording. Wire into `GameTickExecutor` alongside existing strategy milestone call.

**Tech Stack:** Java 21, Quarkus CDI, casehub-ledger-api (`OutcomeRecorder`, `AttestingOutcomeRecorder`), JUnit 5 + AssertJ + Mockito

## Global Constraints

- No `@QuarkusTest` — all tests are plain JUnit
- Domain model (`domain/`) remains framework-free
- SPI fallback: `evaluateMilestones()` is a no-op when `OutcomeRecorder` is not `instanceof AttestingOutcomeRecorder`
- Commit attribution: no `Co-Authored-By` trailers
- All commits reference `Refs #224`

---

### Task 1: Extract `MilestoneTracker` interface

`MilestoneTrigger.check()` currently takes the concrete `MilestoneSession` class. The advisory recorder needs its own session for independent fired tracking. Extract the `hasFired`/`markFired` contract into an interface so triggers accept either session type.

**Files:**
- Create: `src/main/java/io/quarkmind/agent/MilestoneTracker.java`
- Modify: `src/main/java/io/quarkmind/agent/MilestoneSession.java`
- Modify: `src/main/java/io/quarkmind/agent/MilestoneTrigger.java`
- Modify: `src/main/java/io/quarkmind/agent/FrameThresholdTrigger.java`
- Modify: `src/main/java/io/quarkmind/agent/GamePhaseTrigger.java`
- Modify: `src/main/java/io/quarkmind/agent/MilestoneOutcomeRecorder.java`

**Interfaces:**
- Produces: `MilestoneTracker` interface with `boolean hasFired(String)`, `void markFired(String)` — consumed by Task 3 (`AdvisoryMilestoneSession`) and Task 4 (`AdvisoryMilestoneOutcomeRecorder`)

- [ ] **Step 1: Write the failing test**

No new test — this is a structural refactor. Verify existing tests pass first:

```bash
mvn test -Dtest=MilestoneSessionTest,FrameThresholdTriggerTest,GamePhaseTriggerTest,MilestoneOutcomeRecorderTest -q
```

Expected: all PASS (baseline)

- [ ] **Step 2: Create `MilestoneTracker` interface**

```java
package io.quarkmind.agent;

public interface MilestoneTracker {
    boolean hasFired(String milestoneId);
    void markFired(String milestoneId);
}
```

- [ ] **Step 3: `MilestoneSession implements MilestoneTracker`**

Add `implements MilestoneTracker` to the class declaration. No method changes — `hasFired` and `markFired` already match the interface signatures.

```java
@ApplicationScoped
public class MilestoneSession implements MilestoneTracker {
```

- [ ] **Step 4: Update `MilestoneTrigger.check()` signature**

```java
public interface MilestoneTrigger {
    List<MilestoneEvent> check(long gameFrame, MilestoneTracker tracker);
}
```

- [ ] **Step 5: Update `FrameThresholdTrigger.check()`**

Change parameter type from `MilestoneSession` to `MilestoneTracker`:

```java
@Override
public List<MilestoneEvent> check(long gameFrame, MilestoneTracker tracker) {
    List<MilestoneEvent> events = new ArrayList<>();
    for (Threshold t : thresholds) {
        String milestoneId = "frame:" + t.frame();
        if (gameFrame >= t.frame() && !tracker.hasFired(milestoneId)) {
            tracker.markFired(milestoneId);
            events.add(new MilestoneEvent(milestoneId, t.weight()));
        }
    }
    return events;
}
```

- [ ] **Step 6: Update `GamePhaseTrigger.check()`**

Change parameter type from `MilestoneSession` to `MilestoneTracker`:

```java
@Override
public List<MilestoneEvent> check(long gameFrame, MilestoneTracker tracker) {
    ensureSubscribed();
    TacticalPosture phase = lastSeenPhase;
    if (phase == null) return List.of();

    String milestoneId = "phase:" + phase.posture();
    if (tracker.hasFired(milestoneId)) return List.of();

    tracker.markFired(milestoneId);
    double raw = (double) gameFrame / expectedGameLength;
    double weight = Math.max(minWeight, Math.min(maxWeight, raw));
    return List.of(new MilestoneEvent(milestoneId, weight));
}
```

- [ ] **Step 7: Run tests to verify no regressions**

```bash
mvn test -Dtest=MilestoneSessionTest,FrameThresholdTriggerTest,GamePhaseTriggerTest,MilestoneOutcomeRecorderTest -q
```

Expected: all PASS — `MilestoneSession` still satisfies `MilestoneTracker`, triggers still work identically

- [ ] **Step 8: Commit**

```bash
git add src/main/java/io/quarkmind/agent/MilestoneTracker.java \
        src/main/java/io/quarkmind/agent/MilestoneSession.java \
        src/main/java/io/quarkmind/agent/MilestoneTrigger.java \
        src/main/java/io/quarkmind/agent/FrameThresholdTrigger.java \
        src/main/java/io/quarkmind/agent/GamePhaseTrigger.java
git commit -m "refactor(#224): extract MilestoneTracker interface for trigger reuse

Refs #224"
```

---

### Task 2: Extend `AdvisoryInvocationCounter` with frame tracking + update `AdvisoryCompletionObserver`

Change the counter's backing store from `Set<String>` to `ConcurrentHashMap<String, Long>` (advisorId → first invocation frame). Update the observer to pass the game frame.

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/AdvisoryInvocationCounter.java`
- Modify: `src/main/java/io/quarkmind/agent/AdvisoryCompletionObserver.java`
- Modify: `src/test/java/io/quarkmind/agent/AdvisoryCompletionObserverTest.java`
- Test: `src/test/java/io/quarkmind/agent/AdvisoryInvocationCounterTest.java` (new)

**Interfaces:**
- Produces: `AdvisoryInvocationCounter.record(String advisorId, long gameFrame)`, `OptionalLong firstFrame(String advisorId)`, `Set<String> snapshot()` — consumed by Task 4 (`AdvisoryMilestoneOutcomeRecorder`) and Task 4's test double

- [ ] **Step 1: Write `AdvisoryInvocationCounterTest`**

```java
package io.quarkmind.agent;

import io.quarkmind.sc2.GameStarted;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.OptionalLong;

import static org.assertj.core.api.Assertions.assertThat;

class AdvisoryInvocationCounterTest {

    private AdvisoryInvocationCounter counter;

    @BeforeEach
    void setUp() {
        counter = new AdvisoryInvocationCounter();
    }

    @Test
    void record_storesFirstFrameOnly() {
        counter.record("claude:crisis@v1", 1000L);
        counter.record("claude:crisis@v1", 5000L);

        assertThat(counter.firstFrame("claude:crisis@v1"))
            .isEqualTo(OptionalLong.of(1000L));
    }

    @Test
    void firstFrame_returnsEmpty_whenNotRecorded() {
        assertThat(counter.firstFrame("claude:unknown@v1"))
            .isEqualTo(OptionalLong.empty());
    }

    @Test
    void snapshot_returnsAdvisorIds() {
        counter.record("claude:crisis@v1", 1000L);
        counter.record("claude:economic@v1", 2000L);

        assertThat(counter.snapshot())
            .containsExactlyInAnyOrder("claude:crisis@v1", "claude:economic@v1");
    }

    @Test
    void onGameStarted_clearsAll() {
        counter.record("claude:crisis@v1", 1000L);
        counter.onGameStarted(new GameStarted());

        assertThat(counter.snapshot()).isEmpty();
        assertThat(counter.firstFrame("claude:crisis@v1"))
            .isEqualTo(OptionalLong.empty());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -Dtest=AdvisoryInvocationCounterTest -q
```

Expected: FAIL — `record()` signature mismatch (currently takes only `String`)

- [ ] **Step 3: Update `AdvisoryInvocationCounter`**

```java
package io.quarkmind.agent;

import io.quarkmind.sc2.GameStarted;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;

import java.util.OptionalLong;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class AdvisoryInvocationCounter {

    private final ConcurrentHashMap<String, Long> invokedAdvisors = new ConcurrentHashMap<>();

    public void record(String advisorId, long gameFrame) {
        invokedAdvisors.putIfAbsent(advisorId, gameFrame);
    }

    public OptionalLong firstFrame(String advisorId) {
        Long frame = invokedAdvisors.get(advisorId);
        return frame != null ? OptionalLong.of(frame) : OptionalLong.empty();
    }

    public Set<String> snapshot() {
        return Set.copyOf(invokedAdvisors.keySet());
    }

    void onGameStarted(@Observes GameStarted event) {
        invokedAdvisors.clear();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -Dtest=AdvisoryInvocationCounterTest -q
```

Expected: PASS

- [ ] **Step 5: Update `AdvisoryCompletionObserver`**

Change the `onAdvisoryCompleted` method to pass the game frame:

```java
void onAdvisoryCompleted(@Observes AdvisoryCompleted event) {
    invocationCounter.record(event.advisorId(), event.gameFrame());
}
```

- [ ] **Step 6: Update `AdvisoryCompletionObserverTest`**

The existing tests should still pass — `record()` now takes two args. Update the test to also verify frame recording:

Add a new test method:

```java
@Test
void onAdvisoryCompleted_recordsFirstInvocationFrame() {
    AdvisoryInvocationCounter counter = new AdvisoryInvocationCounter();
    AdvisoryCompletionObserver observer = new AdvisoryCompletionObserver(counter);

    AdvisoryCompleted first = new AdvisoryCompleted(
        "claude:crisis-aggressive@v1", "advisory-crisis", 1000L,
        "First rec", 0.85, 120L, Map.of()
    );
    AdvisoryCompleted second = new AdvisoryCompleted(
        "claude:crisis-aggressive@v1", "advisory-crisis", 5000L,
        "Second rec", 0.90, 100L, Map.of()
    );

    observer.onAdvisoryCompleted(first);
    observer.onAdvisoryCompleted(second);

    assertThat(counter.firstFrame("claude:crisis-aggressive@v1"))
        .isEqualTo(java.util.OptionalLong.of(1000L));
}
```

- [ ] **Step 7: Run all affected tests**

```bash
mvn test -Dtest=AdvisoryInvocationCounterTest,AdvisoryCompletionObserverTest -q
```

Expected: all PASS

- [ ] **Step 8: Check for compilation errors from signature change**

The old `record(String)` signature is gone. Check if any other callers exist:

Use `ide_find_references` on `AdvisoryInvocationCounter.record` to find all call sites. The only callers should be `AdvisoryCompletionObserver` and tests. Fix the test doubles in `AdvisoryGameOutcomeRecorderTest` — the `TestAdvisoryInvocationCounter.unwrap()` creates an anonymous subclass; since `record()` now takes two params, the override is still valid (it overrides `snapshot()` only). Verify compilation:

```bash
mvn compile -q
```

Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add src/main/java/io/quarkmind/agent/AdvisoryInvocationCounter.java \
        src/main/java/io/quarkmind/agent/AdvisoryCompletionObserver.java \
        src/test/java/io/quarkmind/agent/AdvisoryInvocationCounterTest.java \
        src/test/java/io/quarkmind/agent/AdvisoryCompletionObserverTest.java
git commit -m "feat(#224): extend AdvisoryInvocationCounter with first-invocation frame tracking

Refs #224"
```

---

### Task 3: Create `AdvisoryMilestoneSession`

Per-advisor session state for milestone entry IDs and fired tracking. Implements `MilestoneTracker` so triggers accept it directly.

**Files:**
- Create: `src/main/java/io/quarkmind/agent/AdvisoryMilestoneSession.java`
- Create: `src/test/java/io/quarkmind/agent/AdvisoryMilestoneSessionTest.java`

**Interfaces:**
- Consumes: `MilestoneTracker` interface (from Task 1)
- Produces: `AdvisoryMilestoneSession` — `entryId(String)`, `setEntryId(String, UUID)`, `hasFired(String)`, `markFired(String)`, `reset()` — consumed by Task 4

- [ ] **Step 1: Write `AdvisoryMilestoneSessionTest`**

```java
package io.quarkmind.agent;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class AdvisoryMilestoneSessionTest {

    private AdvisoryMilestoneSession session;

    @BeforeEach
    void setUp() {
        session = new AdvisoryMilestoneSession();
    }

    @Test
    void entryId_returnsEmpty_whenNotSet() {
        assertThat(session.entryId("claude:crisis@v1")).isEmpty();
    }

    @Test
    void entryId_returnsValue_afterSet() {
        UUID id = UUID.randomUUID();
        session.setEntryId("claude:crisis@v1", id);
        assertThat(session.entryId("claude:crisis@v1")).contains(id);
    }

    @Test
    void entryId_separatePerAdvisor() {
        UUID id1 = UUID.randomUUID();
        UUID id2 = UUID.randomUUID();
        session.setEntryId("claude:crisis@v1", id1);
        session.setEntryId("claude:economic@v1", id2);
        assertThat(session.entryId("claude:crisis@v1")).contains(id1);
        assertThat(session.entryId("claude:economic@v1")).contains(id2);
    }

    @Test
    void hasFired_returnsFalse_initially() {
        assertThat(session.hasFired("frame:4032")).isFalse();
    }

    @Test
    void hasFired_returnsTrue_afterMark() {
        session.markFired("frame:4032");
        assertThat(session.hasFired("frame:4032")).isTrue();
    }

    @Test
    void reset_clearsAllState() {
        UUID id = UUID.randomUUID();
        session.setEntryId("claude:crisis@v1", id);
        session.markFired("frame:4032");

        session.reset();

        assertThat(session.entryId("claude:crisis@v1")).isEmpty();
        assertThat(session.hasFired("frame:4032")).isFalse();
    }

    @Test
    void implementsMilestoneTracker() {
        assertThat(session).isInstanceOf(MilestoneTracker.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -Dtest=AdvisoryMilestoneSessionTest -q
```

Expected: FAIL — class does not exist

- [ ] **Step 3: Implement `AdvisoryMilestoneSession`**

```java
package io.quarkmind.agent;

import jakarta.enterprise.context.ApplicationScoped;
import io.quarkmind.sc2.GameStarted;
import jakarta.enterprise.event.Observes;

import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class AdvisoryMilestoneSession implements MilestoneTracker {

    private final Map<String, UUID> entryIds = new ConcurrentHashMap<>();
    private final Set<String> firedMilestones = ConcurrentHashMap.newKeySet();

    public Optional<UUID> entryId(String advisorId) {
        return Optional.ofNullable(entryIds.get(advisorId));
    }

    public void setEntryId(String advisorId, UUID id) {
        entryIds.put(advisorId, id);
    }

    @Override
    public boolean hasFired(String milestoneId) {
        return firedMilestones.contains(milestoneId);
    }

    @Override
    public void markFired(String milestoneId) {
        firedMilestones.add(milestoneId);
    }

    public void reset() {
        entryIds.clear();
        firedMilestones.clear();
    }

    void onGameStarted(@Observes GameStarted event) {
        reset();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -Dtest=AdvisoryMilestoneSessionTest -q
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/quarkmind/agent/AdvisoryMilestoneSession.java \
        src/test/java/io/quarkmind/agent/AdvisoryMilestoneSessionTest.java
git commit -m "feat(#224): add AdvisoryMilestoneSession for per-advisor milestone tracking

Refs #224"
```

---

### Task 4: Create `AdvisoryMilestoneOutcomeRecorder` (replaces `AdvisoryGameOutcomeRecorder`)

Single class owning both milestone evaluation and game-end recording for advisors. Uses shared triggers (via `MilestoneTracker`), shared `DominanceAssessor`, and its own `AdvisoryMilestoneSession`.

**Files:**
- Create: `src/main/java/io/quarkmind/agent/AdvisoryMilestoneOutcomeRecorder.java`
- Create: `src/test/java/io/quarkmind/agent/AdvisoryMilestoneOutcomeRecorderTest.java`
- Delete: `src/main/java/io/quarkmind/agent/AdvisoryGameOutcomeRecorder.java` (use `ide_refactor_safe_delete`)
- Delete: `src/test/java/io/quarkmind/agent/AdvisoryGameOutcomeRecorderTest.java`

**Interfaces:**
- Consumes: `MilestoneTracker` (Task 1), `AdvisoryInvocationCounter.snapshot()` / `firstFrame()` (Task 2), `AdvisoryMilestoneSession` (Task 3), `MilestoneTrigger.check()`, `DominanceAssessor.assess()`, `MilestoneConfig`, `OutcomeRecorder`, `GameSession`
- Produces: `evaluateMilestones(GameState)` — consumed by Task 5 (`GameTickExecutor`)

- [ ] **Step 1: Write the test — game-end only (no milestones), preserving existing behavior**

Create `AdvisoryMilestoneOutcomeRecorderTest.java`. Start with the game-end tests migrated from `AdvisoryGameOutcomeRecorderTest`:

```java
package io.quarkmind.agent;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.OutcomeRecorder;
import io.quarkmind.domain.DominanceScore;
import io.quarkmind.domain.GameState;
import io.quarkmind.sc2.GameResult;
import io.quarkmind.sc2.GameStarted;
import io.quarkmind.sc2.GameStopped;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.OptionalLong;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

import static org.assertj.core.api.Assertions.assertThat;

class AdvisoryMilestoneOutcomeRecorderTest {

    private RecordingOutcomeRecorder outcomeRecorder;
    private StubAdvisoryInvocationCounter invocationCounter;
    private GameSession gameSession;
    private AdvisoryMilestoneSession advisorySession;
    private AdvisoryMilestoneOutcomeRecorder recorder;

    @BeforeEach
    void setUp() {
        outcomeRecorder = new RecordingOutcomeRecorder();
        invocationCounter = new StubAdvisoryInvocationCounter();
        gameSession = new GameSession();
        gameSession.setCaseId(UUID.randomUUID());
        advisorySession = new AdvisoryMilestoneSession();

        recorder = new AdvisoryMilestoneOutcomeRecorder(
            outcomeRecorder,
            invocationCounter.unwrap(),
            gameSession,
            advisorySession,
            state -> new DominanceScore(0.5, Map.of()),
            List.of(new FrameThresholdTrigger(List.of(
                new FrameThresholdTrigger.Threshold(4032, 0.3)))),
            true, 0.15);
    }

    // --- game-end only (no milestones fired) ---

    @Test
    void gameEnd_victory_recordsEndorsedForEachAdvisor() {
        invocationCounter.setInvoked(Map.of(
            "claude:crisis@v1", 1000L,
            "claude:economic@v1", 2000L));
        recorder.onGameStarted(new GameStarted());
        recorder.onGameStopped(new GameStopped(GameResult.WIN));

        assertThat(outcomeRecorder.records).hasSize(2);
        assertThat(outcomeRecorder.records)
            .allMatch(r -> r.verdict() == AttestationVerdict.ENDORSED)
            .allMatch(r -> r.confidence() == 1.0)
            .allMatch(r -> r.capabilityTag().equals("game-outcome"))
            .extracting(OutcomeRecord::actorId)
            .containsExactlyInAnyOrder("claude:crisis@v1", "claude:economic@v1");
    }

    @Test
    void gameEnd_defeat_recordsChallenged() {
        invocationCounter.setInvoked(Map.of("claude:strategic@v1", 1000L));
        recorder.onGameStarted(new GameStarted());
        recorder.onGameStopped(new GameStopped(GameResult.LOSS));

        assertThat(outcomeRecorder.records).hasSize(1);
        assertThat(outcomeRecorder.records.get(0).verdict())
            .isEqualTo(AttestationVerdict.CHALLENGED);
    }

    @Test
    void gameEnd_tie_recordsSound() {
        invocationCounter.setInvoked(Map.of("claude:crisis@v1", 1000L));
        recorder.onGameStarted(new GameStarted());
        recorder.onGameStopped(new GameStopped(GameResult.TIE));

        assertThat(outcomeRecorder.records.get(0).verdict())
            .isEqualTo(AttestationVerdict.SOUND);
    }

    @Test
    void gameEnd_unknown_skips() {
        invocationCounter.setInvoked(Map.of("claude:crisis@v1", 1000L));
        recorder.onGameStarted(new GameStarted());
        recorder.onGameStopped(new GameStopped(GameResult.UNKNOWN));

        assertThat(outcomeRecorder.records).isEmpty();
    }

    @Test
    void gameEnd_emptyInvocationSet_skips() {
        invocationCounter.setInvoked(Map.of());
        recorder.onGameStarted(new GameStarted());
        recorder.onGameStopped(new GameStopped(GameResult.WIN));

        assertThat(outcomeRecorder.records).isEmpty();
    }

    // --- SPI fallback ---

    @Test
    void evaluateMilestones_withoutSpi_isNoOp() {
        invocationCounter.setInvoked(Map.of("claude:crisis@v1", 1000L));
        recorder.onGameStarted(new GameStarted());

        recorder.evaluateMilestones(gameStateAtFrame(5000));

        assertThat(outcomeRecorder.records).isEmpty();
        assertThat(outcomeRecorder.attestations).isEmpty();
    }

    // --- milestones disabled ---

    @Test
    void evaluateMilestones_whenDisabled_isNoOp() {
        recorder = new AdvisoryMilestoneOutcomeRecorder(
            outcomeRecorder, invocationCounter.unwrap(), gameSession,
            advisorySession,
            state -> new DominanceScore(0.5, Map.of()),
            List.of(), false, 0.15);

        invocationCounter.setInvoked(Map.of("claude:crisis@v1", 1000L));
        recorder.onGameStarted(new GameStarted());

        recorder.evaluateMilestones(gameStateAtFrame(5000));

        assertThat(outcomeRecorder.attestations).isEmpty();
    }

    @Test
    void gameEnd_recordsEvenWhenMilestonesDisabled() {
        recorder = new AdvisoryMilestoneOutcomeRecorder(
            outcomeRecorder, invocationCounter.unwrap(), gameSession,
            advisorySession,
            state -> new DominanceScore(0.5, Map.of()),
            List.of(), false, 0.15);

        invocationCounter.setInvoked(Map.of("claude:crisis@v1", 1000L));
        recorder.onGameStarted(new GameStarted());
        recorder.onGameStopped(new GameStopped(GameResult.WIN));

        assertThat(outcomeRecorder.records).hasSize(1);
    }

    // --- session reset ---

    @Test
    void gameStarted_resetsAdvisoryMilestoneSession() {
        advisorySession.markFired("frame:4032");
        advisorySession.setEntryId("claude:crisis@v1", UUID.randomUUID());

        recorder.onGameStarted(new GameStarted());

        assertThat(advisorySession.hasFired("frame:4032")).isFalse();
        assertThat(advisorySession.entryId("claude:crisis@v1")).isEmpty();
    }

    // --- helpers ---

    private static GameState gameStateAtFrame(long frame) {
        return new GameState(200, 100, 15, 6, List.of(), List.of(), List.of(), List.of(), List.of(), List.of(), List.of(), frame, null, PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY, Set.of(), Set.of());
    }

    static class RecordingOutcomeRecorder implements OutcomeRecorder {
        final List<OutcomeRecord> records = new ArrayList<>();
        final List<Attestation> attestations = new ArrayList<>();

        @Override
        public UUID record(OutcomeRecord record) {
            records.add(record);
            return UUID.randomUUID();
        }

        @Override
        public void addAttestation(UUID id, AttestationVerdict verdict,
                                   double confidence, String dimension) {
            attestations.add(new Attestation(id, verdict, confidence, dimension));
        }

        record Attestation(UUID entryId, AttestationVerdict verdict,
                           double confidence, String dimension) {}
    }

    static class StubAdvisoryInvocationCounter {
        private final ConcurrentHashMap<String, Long> invoked = new ConcurrentHashMap<>();

        void setInvoked(Map<String, Long> advisors) {
            invoked.clear();
            invoked.putAll(advisors);
        }

        AdvisoryInvocationCounter unwrap() {
            return new AdvisoryInvocationCounter() {
                @Override
                public Set<String> snapshot() {
                    return Set.copyOf(invoked.keySet());
                }

                @Override
                public OptionalLong firstFrame(String advisorId) {
                    Long frame = invoked.get(advisorId);
                    return frame != null ? OptionalLong.of(frame) : OptionalLong.empty();
                }
            };
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -Dtest=AdvisoryMilestoneOutcomeRecorderTest -q
```

Expected: FAIL — `AdvisoryMilestoneOutcomeRecorder` class does not exist

- [ ] **Step 3: Implement `AdvisoryMilestoneOutcomeRecorder`**

```java
package io.quarkmind.agent;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.OutcomeRecorder;
import io.quarkmind.domain.DominanceScore;
import io.quarkmind.domain.GameState;
import io.quarkmind.sc2.GameResult;
import io.quarkmind.sc2.GameStarted;
import io.quarkmind.sc2.GameStopped;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Set;
import java.util.UUID;

@ApplicationScoped
public class AdvisoryMilestoneOutcomeRecorder {

    private static final Logger log = Logger.getLogger(AdvisoryMilestoneOutcomeRecorder.class);

    private final OutcomeRecorder outcomeRecorder;
    private final AdvisoryInvocationCounter invocationCounter;
    private final GameSession gameSession;
    private final AdvisoryMilestoneSession advisorySession;
    private final DominanceAssessor dominanceAssessor;
    private final List<MilestoneTrigger> triggers;
    private final boolean milestonesEnabled;
    private final double deadZoneThreshold;

    @Inject
    AdvisoryMilestoneOutcomeRecorder(
            OutcomeRecorder outcomeRecorder,
            AdvisoryInvocationCounter invocationCounter,
            GameSession gameSession,
            AdvisoryMilestoneSession advisorySession,
            DominanceAssessor dominanceAssessor,
            @Any Instance<MilestoneTrigger> triggerInstances,
            MilestoneConfig config) {
        this.outcomeRecorder = outcomeRecorder;
        this.invocationCounter = invocationCounter;
        this.gameSession = gameSession;
        this.advisorySession = advisorySession;
        this.dominanceAssessor = dominanceAssessor;
        this.triggers = triggerInstances.stream().toList();
        this.milestonesEnabled = config.enabled();
        this.deadZoneThreshold = config.deadZoneThreshold();
    }

    AdvisoryMilestoneOutcomeRecorder(
            OutcomeRecorder outcomeRecorder,
            AdvisoryInvocationCounter invocationCounter,
            GameSession gameSession,
            AdvisoryMilestoneSession advisorySession,
            DominanceAssessor dominanceAssessor,
            List<MilestoneTrigger> triggers,
            boolean milestonesEnabled,
            double deadZoneThreshold) {
        this.outcomeRecorder = outcomeRecorder;
        this.invocationCounter = invocationCounter;
        this.gameSession = gameSession;
        this.advisorySession = advisorySession;
        this.dominanceAssessor = dominanceAssessor;
        this.triggers = triggers;
        this.milestonesEnabled = milestonesEnabled;
        this.deadZoneThreshold = deadZoneThreshold;
    }

    void onGameStarted(@Observes GameStarted event) {
        advisorySession.reset();
    }

    public void evaluateMilestones(GameState state) {
        if (!milestonesEnabled) { return; }

        Set<String> invokedAdvisors = invocationCounter.snapshot();
        if (invokedAdvisors.isEmpty()) { return; }

        for (MilestoneTrigger trigger : triggers) {
            List<MilestoneEvent> events = trigger.check(state.gameFrame(), advisorySession);
            for (MilestoneEvent event : events) {
                DominanceScore score = dominanceAssessor.assess(state);
                if (Math.abs(score.overall()) < deadZoneThreshold) {
                    log.debugf("[ADVISORY-MILESTONE] Dead zone — skipped %s (score=%.3f)",
                        event.milestoneId(), score.overall());
                    continue;
                }

                AttestationVerdict verdict = score.overall() > 0
                    ? AttestationVerdict.ENDORSED
                    : AttestationVerdict.CHALLENGED;
                double confidence = event.temporalWeight() * Math.abs(score.overall());

                for (String advisorId : invokedAdvisors) {
                    recordAdvisoryMilestone(advisorId, verdict, confidence, event.milestoneId());
                }
            }
        }
    }

    private void recordAdvisoryMilestone(String advisorId, AttestationVerdict verdict,
                                         double confidence, String milestoneId) {
        var existingEntry = advisorySession.entryId(advisorId);
        if (existingEntry.isPresent()) {
            outcomeRecorder.addAttestation(existingEntry.get(), verdict, confidence, "game-outcome");
        } else {
            UUID entryId = outcomeRecorder.record(OutcomeRecord.of(
                advisorId, gameSession.id(), "game-outcome", verdict, confidence));
            advisorySession.setEntryId(advisorId, entryId);
        }
        log.debugf("[ADVISORY-MILESTONE] Recorded: advisor=%s milestone=%s verdict=%s confidence=%.3f",
            advisorId, milestoneId, verdict, confidence);
    }

    void onGameStopped(@Observes GameStopped event) {
        if (event.result() == GameResult.UNKNOWN) {
            log.infof("[ADVISORY-MILESTONE] Game ended with unknown result — skipped");
            return;
        }

        Set<String> invokedAdvisors = invocationCounter.snapshot();
        if (invokedAdvisors.isEmpty()) {
            log.infof("[ADVISORY-MILESTONE] No advisors invoked this game — skipped");
            return;
        }

        AttestationVerdict verdict = switch (event.result()) {
            case WIN -> AttestationVerdict.ENDORSED;
            case LOSS -> AttestationVerdict.CHALLENGED;
            case TIE -> AttestationVerdict.SOUND;
            case UNKNOWN -> throw new AssertionError("unreachable — guarded above");
        };

        for (String advisorId : invokedAdvisors) {
            var existingEntry = advisorySession.entryId(advisorId);
            if (existingEntry.isPresent()) {
                outcomeRecorder.addAttestation(existingEntry.get(), verdict, 1.0, "game-outcome");
            } else {
                outcomeRecorder.record(OutcomeRecord.of(
                    advisorId, gameSession.id(), "game-outcome", verdict, 1.0));
            }
        }

        log.infof("[ADVISORY-MILESTONE] Game-end recorded: advisors=%s result=%s verdict=%s",
            invokedAdvisors, event.result(), verdict);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
mvn test -Dtest=AdvisoryMilestoneOutcomeRecorderTest -q
```

Expected: PASS

- [ ] **Step 5: Delete `AdvisoryGameOutcomeRecorder` and its test**

Use `ide_refactor_safe_delete` on `AdvisoryGameOutcomeRecorder`. Then delete the test file.

Verify no remaining references:

Use `ide_find_references` on `AdvisoryGameOutcomeRecorder` — should return only import/Javadoc references in other files that need updating.

Delete the files:
- `src/main/java/io/quarkmind/agent/AdvisoryGameOutcomeRecorder.java`
- `src/test/java/io/quarkmind/agent/AdvisoryGameOutcomeRecorderTest.java`

- [ ] **Step 6: Verify full compilation**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS. If any files referenced `AdvisoryGameOutcomeRecorder`, fix them (update Javadoc references to `AdvisoryMilestoneOutcomeRecorder`).

- [ ] **Step 7: Run full test suite**

```bash
mvn test -q
```

Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git add src/main/java/io/quarkmind/agent/AdvisoryMilestoneOutcomeRecorder.java \
        src/test/java/io/quarkmind/agent/AdvisoryMilestoneOutcomeRecorderTest.java
git rm src/main/java/io/quarkmind/agent/AdvisoryGameOutcomeRecorder.java \
       src/test/java/io/quarkmind/agent/AdvisoryGameOutcomeRecorderTest.java
git add -u  # catch any updated references
git commit -m "feat(#224): add AdvisoryMilestoneOutcomeRecorder replacing AdvisoryGameOutcomeRecorder

Refs #224"
```

---

### Task 5: Wire `AdvisoryMilestoneOutcomeRecorder` into `GameTickExecutor`

Add the advisory milestone call alongside the existing strategy milestone call.

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/GameTickExecutor.java`
- Modify: `src/test/java/io/quarkmind/agent/GameTickExecutorMigrationTest.java`

**Interfaces:**
- Consumes: `AdvisoryMilestoneOutcomeRecorder.evaluateMilestones(GameState)` (from Task 4)

- [ ] **Step 1: Write the failing test**

Add to `GameTickExecutorMigrationTest`:

```java
@Test
void execute_callsAdvisoryMilestoneEvaluationInAiMode() {
    GameState state = stubGameState(5000L, 200, 100);
    when(engine.observe()).thenReturn(state);
    when(caseHub.signalAndAwaitSync(any(), any(), any())).thenReturn(mock(CaseContext.class));

    executor.execute();

    verify(advisoryMilestoneOutcomeRecorder).evaluateMilestones(state);
}

@Test
void execute_coachMode_skipsAdvisoryMilestones() {
    executor.gameMode = "coach";
    GameState state = stubGameState(100L, 400, 200);
    when(engine.observe()).thenReturn(state);
    CaseContext ctx = mock(CaseContext.class);
    when(caseHub.signalAndAwaitSync(eq(sessionId), any(), any())).thenReturn(ctx);
    when(coachingTriggerBuilder.build(any(CaseContext.class), anyLong())).thenReturn(Collections.emptyMap());

    executor.execute();

    verify(advisoryMilestoneOutcomeRecorder, never()).evaluateMilestones(any());
}
```

Also add the mock field and setUp line:

```java
private AdvisoryMilestoneOutcomeRecorder advisoryMilestoneOutcomeRecorder;

// In setUp():
advisoryMilestoneOutcomeRecorder = mock(AdvisoryMilestoneOutcomeRecorder.class);

// In executor field assignment block:
executor.advisoryMilestoneOutcomeRecorder = advisoryMilestoneOutcomeRecorder;
```

Update the existing `execute_coachMode_skipsMilestonesAndAdvisory` test to also verify `advisoryMilestoneOutcomeRecorder`:

```java
verify(advisoryMilestoneOutcomeRecorder, never()).evaluateMilestones(any());
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -Dtest=GameTickExecutorMigrationTest -q
```

Expected: FAIL — `advisoryMilestoneOutcomeRecorder` field does not exist on `GameTickExecutor`

- [ ] **Step 3: Add injection and call to `GameTickExecutor`**

Add the field:

```java
@Inject
AdvisoryMilestoneOutcomeRecorder advisoryMilestoneOutcomeRecorder;
```

Add the call in `execute()`, immediately after the existing strategy milestone call, inside the same `!isCoachMode` guard:

```java
// Milestone evaluation: only in AI mode
if (!isCoachMode) {
    milestoneOutcomeRecorder.evaluateMilestones(gameState);
    advisoryMilestoneOutcomeRecorder.evaluateMilestones(gameState);
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
mvn test -Dtest=GameTickExecutorMigrationTest -q
```

Expected: PASS

- [ ] **Step 5: Run full test suite**

```bash
mvn test -q
```

Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/agent/GameTickExecutor.java \
        src/test/java/io/quarkmind/agent/GameTickExecutorMigrationTest.java
git commit -m "feat(#224): wire AdvisoryMilestoneOutcomeRecorder into GameTickExecutor

Refs #224"
```

---

### Task 6: Update CLAUDE.md test listings

Add the new test classes to the CLAUDE.md test listings.

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Add new tests to unit test listing**

Add to the unit tests list in the appropriate alphabetical position:
- `AdvisoryInvocationCounterTest`
- `AdvisoryMilestoneOutcomeRecorderTest`
- `AdvisoryMilestoneSessionTest`

Remove from the listing:
- `AdvisoryGameOutcomeRecorderTest` (deleted)

- [ ] **Step 2: Verify no stale references**

Search CLAUDE.md for `AdvisoryGameOutcomeRecorder` — update any remaining references to `AdvisoryMilestoneOutcomeRecorder`.

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#224): update CLAUDE.md test listings for advisory milestone attestations

Refs #224"
```
