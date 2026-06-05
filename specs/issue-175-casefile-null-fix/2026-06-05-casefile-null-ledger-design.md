# Design: InMemoryCaseFile null fix (#175) + casehub-ledger integration (#156)

**Branch:** `issue-175-casefile-null-fix`
**Issues:** #175 (XS), #156 (L)
**Date:** 2026-06-05

---

## #175 — InMemoryCaseFile null correctness

### Root cause

`InMemoryCaseFile.put(key, null)` stores `ItemEntry(null, version)`. The `AtomicReference` is
non-null, so `contains(key)` returns `true`, but `get(key, Type)` then calls `Optional.of(null)`
which throws `NullPointerException`. The same `Optional.of` pattern appears in `fireEvent`.

A secondary path: the constructor stores values from `initialState` without filtering nulls,
bypassing `put()`. A subsequent `put(key, newValue)` would then get an `ItemEntry(null, ver)`
as `previous`, causing `Optional.of(previous.value())` to NPE in `fireEvent`.

### Fix — four sites in `InMemoryCaseFile.java` (casehub-poc)

The goal is an airtight invariant: null values can never enter the store via any path.

| Site | Change | Rationale |
|------|--------|-----------|
| `put()` top | `if (value == null) return;` | Prevents null storage via put(). Matches `MapCaseFile` contract. Makes `contains()` correct as side-effect. |
| Constructor `initialState` forEach | Skip entries where `v == null` | Closes the constructor bypass path. Makes the null invariant fully airtight. |
| `get()` line 80 | Keep `Optional.of(...)` | With both null guards in place, null can never reach get(). `Optional.of` is the correct strong assertion. |
| `fireEvent()` line 181 | Keep `Optional.of(previous.value())` when `previous != null` | Same reasoning — null-value ItemEntries cannot exist after constructor filter + put() guard. |

No `Optional.ofNullable` anywhere — using it would silently mask an invariant violation.

### Tests — plain JUnit (no Quarkus), `InMemoryCaseFileNullTest`

- `put(key, null)` → key remains absent: `contains()` false, `get()` empty
- `put(key, value)` then `put(key, null)` → no-op, key still holds original value
- `put(key, a)` then `put(key, b)` → normal update still works (regression guard)
- Constructor with `Map.of("k", "v")` → key present and readable
- Constructor with `{k=null}` (passed via `new HashMap<>()`) → key filtered, absent after construction
- `fireEvent` first-write path: verify listener receives `Optional.empty()` as previous

### After fix

Rebuild and install: `mvn install -DskipTests` from
`/Users/mdproctor/claude/casehub-poc/casehub-persistence-memory/`.

**Quarkmind side:** leave existing `orElseThrow()` guards in place. No quarkmind code changes for #175.

---

## #156 — casehub-ledger integration (Layer 4)

### Goal

Agent plugin decisions are captured as ledger entries via `OutcomeRecorder` on state
*transitions only* — not every tick. This creates a meaningful audit trail and seeds the
trust-score pipeline at Layer 6 (#158). Transition-level recording gives Layer 6 a clean
signal (strategy changed, new attack engagement, new intel observed, new build type queued).

### Verdict semantics

`AttestationVerdict` enum: `SOUND, FLAGGED, ENDORSED, CHALLENGED`.

- `SOUND` — all legitimate plugin executions, including "nothing to do this tick." Positive EigenTrust signal.
- `FLAGGED` — genuine plugin errors only: uncaught exception, inconsistent state. Negative signal.

"No action needed" is not `FLAGGED`. Recording quiescent ticks as `FLAGGED` would systematically
penalize plugins for normal game states and corrupt trust scores at Layer 6.

### Transition detection — when to fire

| Plugin | Fire when | Previous-state field | Verdict |
|--------|-----------|---------------------|---------|
| `DroolsStrategyTask` | `STRATEGY` value changes from previous tick | `volatile String prevStrategy` | `SOUND` on change; `FLAGGED` on exception |
| `DroolsTacticsTask` | NEAREST_THREAT tag changes (not position — Unit is a record; position changes every tick) | `volatile String prevTargetTag` | `SOUND` on new target; `FLAGGED` on exception |
| `DroolsScoutingTask` | enemy unit count or enemy set tag-hash changes | `volatile int prevEnemyHash` | `SOUND` on change; `FLAGGED` on exception |
| `EconomicsDecisionService` | queued build TYPE changes (Probe→Pylon is a transition; Probe→Probe is not) | `volatile String prevBuildType` | `SOUND` on new type; `FLAGGED` on exception |

On the **first call after game start** (`prevX == null`), any non-null output is a transition.
This means the first `execute()` call always fires an event — no complex state setup required.

**Unit tag note:** `Unit` is `record Unit(String tag, UnitType type, Point2d position, ...)`.
Record `equals()` compares all fields including `position`, which changes every tick. Never
compare `Unit` instances directly for transition detection — compare `unit.tag()` strings.

### New components (quarkmind)

#### `GameSession @ApplicationScoped` — `io.quarkmind.agent`

```java
@ApplicationScoped
public class GameSession {
    private volatile UUID id = UUID.randomUUID();
    public UUID id() { return id; }
    void reset() { id = UUID.randomUUID(); }
}
```

`AgentOrchestrator` calls `gameSession.reset()` in `startGame()` **only**. `stopGame()` does NOT
reset — async CDI observers dispatched during the last ticks may still be in the thread pool
queue and would be attributed to the wrong session if reset is called at game-end.

#### `PluginDecisionEvent` — `io.quarkmind.agent`

```java
public record PluginDecisionEvent(
    String actorId,             // plugin.getId()
    String capabilityTag,       // QuarkMindCapabilityTag constant
    AttestationVerdict verdict, // SOUND or FLAGGED
    UUID gameSessionId,
    int gameFrame               // logged by observer for which frame the transition occurred
) {}
```

No `decision` field — not persisted to `OutcomeRecord`.

#### `QuarkMindCapabilityTag` constants — `io.quarkmind.agent`

```java
public final class QuarkMindCapabilityTag {
    public static final String STRATEGY  = "starcraft.strategy";
    public static final String ECONOMICS = "starcraft.economics";
    public static final String TACTICS   = "starcraft.tactics";
    public static final String SCOUTING  = "starcraft.scouting";
    private QuarkMindCapabilityTag() {}
}
```

#### `PluginOutcomeAuditor @ApplicationScoped` — `io.quarkmind.agent`

Cross-cutting infrastructure. Belongs in `agent/`, not `plugin/`.

```java
@ApplicationScoped
public class PluginOutcomeAuditor {
    @Inject OutcomeRecorder outcomeRecorder;

    public void onDecision(@ObservesAsync PluginDecisionEvent e) {
        outcomeRecorder.record(OutcomeRecord.of(
            e.actorId(), e.gameSessionId(),
            e.capabilityTag(), e.verdict(), 0.7   // game-level decision scope
        ));
        log.debugf("Ledger: %s %s frame=%d", e.actorId(), e.verdict(), e.gameFrame());
    }
}
```

Confidence 0.7 = game-level decision scope (per `OutcomeRecord` recommendation). Transitions
are game-level signals regardless of how frequently they occur.

### Plugin wiring

Plugins already import from `io.quarkmind.agent` (`ResourceBudget`, `QuarkMindCaseFile`,
`StrategyTask`). Adding `Event<PluginDecisionEvent>` and `GameSession` injection follows the
same established pattern. No seam-rule violation.

`FlowEconomicsTask.execute()` only submits a tick to the flow; does not fire event.
`EconomicsDecisionService` is the decision point.

### Memory lifecycle

`InMemoryLedgerEntryRepository` should observe `@Observes GameStopped` and self-clear.
`AgentOrchestrator.stopGame()` already fires `gameStoppedEvent.fire(new GameStopped())`.
This keeps orchestrator clean (no injection of concrete memory type) and respects the CDI
event pattern already established in the orchestrator.

### Dependencies (pom.xml)

```xml
<!-- Ledger API — OutcomeRecorder SPI, OutcomeRecord, AttestationVerdict -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger-api</artifactId>
  <version>0.2-SNAPSHOT</version>
</dependency>

<!-- Ledger runtime — DefaultOutcomeRecorder @DefaultBean -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger</artifactId>
  <version>0.2-SNAPSHOT</version>
</dependency>

<!-- In-memory ledger SPIs — no JPA datasource needed for %mock/%emulated/%test -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger-memory</artifactId>
  <version>0.2-SNAPSHOT</version>
</dependency>
```

### Required application.properties additions

**Global (all profiles):**
```properties
# Required by DefaultOutcomeRecorder — throws ISE if absent and OutcomeRecord.attestorId is null
# LedgerConfig javadoc: "For QuarkMind: quarkmind:game-engine@v1"
casehub.ledger.outcome.default-attestor-id=quarkmind:game-engine@v1

# casehub.ledger.enabled defaults to true — no explicit set needed
# casehub.ledger.attestations.enabled defaults to true — no explicit set needed
# casehub.ledger.reactive.enabled has @WithDefault("false") — no explicit set needed
#   (it's a build-time property; setting at runtime does not gate CDI behavior)
```

**Per-profile config (must be spelled out for each profile — Quarkus has no wildcard):**

`%test` profile:
```properties
%test.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.memory.InMemoryLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryLedgerMerkleFrontierRepository,\
  io.casehub.ledger.memory.InMemoryActorTrustScoreRepository,\
  io.casehub.ledger.memory.InMemoryKeyRotationRepository,\
  io.casehub.ledger.memory.InMemoryAgentSigner,\
  io.casehub.ledger.memory.InMemoryActorIdentityBindingRepository,\
  io.casehub.ledger.memory.InMemoryReactiveLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryReactiveKeyRotationRepository

%test.quarkus.index-dependency.casehub-ledger-memory.group-id=io.casehub
%test.quarkus.index-dependency.casehub-ledger-memory.artifact-id=casehub-ledger-memory
%test.casehub.ledger.hash-chain.enabled=false
```

`%mock` profile — same 8 alternatives + index-dependency + hash-chain:
```properties
%mock.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.memory.InMemoryLedgerEntryRepository,\
  ... (same 8 as %test)
%mock.quarkus.index-dependency.casehub-ledger-memory.group-id=io.casehub
%mock.quarkus.index-dependency.casehub-ledger-memory.artifact-id=casehub-ledger-memory
%mock.casehub.ledger.hash-chain.enabled=false
```

`%emulated` profile — same blocks:
```properties
%emulated.quarkus.arc.selected-alternatives=\ ... (same 8)
%emulated.quarkus.index-dependency.casehub-ledger-memory.group-id=io.casehub
%emulated.quarkus.index-dependency.casehub-ledger-memory.artifact-id=casehub-ledger-memory
%emulated.casehub.ledger.hash-chain.enabled=false
```

**CDI risk (GE-20260519-e13b01):** With `casehub-ledger-memory` AND all 8 alternatives
explicitly selected, all ledger SPI requirements are satisfied without JPA. TDD will confirm.

### Integration test — `LedgerAuditIT @QuarkusTest` (`%test` profile)

Pattern follows `DroolsStrategyTaskTest`: inject the plugin and call `execute(cf)` directly
on a hand-constructed CaseFile. No `SimulatedGame` setters needed. On the first call,
`prevStrategy == null` and any non-null strategy output is a transition — the event fires
unconditionally regardless of game state.

```java
@QuarkusTest
class LedgerAuditIT {
    @Inject @CaseType("starcraft-game") StrategyTask strategyTask;
    @Inject LedgerEntryRepository ledgerRepo;
    @Inject GameSession gameSession;

    @BeforeEach void reset() { gameSession.reset(); }

    @Test
    void strategyTransitionWritesLedgerEntry() throws InterruptedException {
        CaseFile cf = caseFile(/* minimal valid state */);
        strategyTask.execute(cf);
        Thread.sleep(300); // CDI async observer
        List<LedgerEntry> entries = ledgerRepo.findBySubjectId(gameSession.id(), 0, 10);
        assertThat(entries).isNotEmpty();
        assertThat(entries).anyMatch(e -> "strategy.drools".equals(e.actorId()));
    }
}
```

Note: `LedgerEntryRepository.findBySubjectId()` — verify exact method signature against the
actual SPI before implementing.

### ARC42STORIES.MD update

Layer 4 row updated to ✅ complete. `#156` forward reference cleared.
Issue #166 (migration to ARC42STORIES.MD) is CLOSED. LAYER-LOG.md is retired. Do not write
a LAYER-LOG entry.

### CLAUDE.md update (numbered step)

Step 11 below: invoke `update-claude-md` to remove the stale "Until migration is complete,
new layer completions still require a LAYER-LOG entry" sentence.

---

## Sequence

1. Fix `InMemoryCaseFile` in casehub-poc (4 sites including constructor filter + test)
2. `mvn install -DskipTests` from casehub-poc/casehub-persistence-memory
3. Add pom.xml deps (casehub-ledger-api, casehub-ledger, casehub-ledger-memory) at version `0.2-SNAPSHOT`
4. Add `casehub.ledger.outcome.default-attestor-id` to global application.properties
5. Add per-profile `selected-alternatives` (8 beans), `index-dependency`, `hash-chain.enabled=false`
   for `%test`, `%mock`, `%emulated`
6. Add `GameSession`, `PluginDecisionEvent`, `QuarkMindCapabilityTag`, `PluginOutcomeAuditor`
   in `io.quarkmind.agent`
7. Wire `gameSession.reset()` in `AgentOrchestrator.startGame()`
8. Add `@Observes GameStopped` self-clear to `InMemoryLedgerEntryRepository` (in ledger-memory module)
9. Wire transition detection + `Event<PluginDecisionEvent>` firing in:
   - `DroolsStrategyTask` (prevStrategy string)
   - `DroolsTacticsTask` (prevTargetTag string — tag only, not Unit record)
   - `DroolsScoutingTask` (prevEnemyHash int)
   - `EconomicsDecisionService` (prevBuildType string)
10. Write `LedgerAuditIT` — verify method signatures on `LedgerEntryRepository` before
    writing the query call
11. Update ARC42STORIES.MD Layer 4 status to ✅
12. Invoke `update-claude-md` (remove stale LAYER-LOG reference)
