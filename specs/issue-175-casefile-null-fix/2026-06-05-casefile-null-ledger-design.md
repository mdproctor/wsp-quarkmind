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

Goal: airtight invariant — null values cannot enter the store via any path.

| Site | Change | Rationale |
|------|--------|-----------|
| `put()` top | `if (value == null) return;` | Prevents null storage via `put()`. Matches `MapCaseFile` contract. Makes `contains()` correct as side-effect. |
| Constructor `initialState` forEach | Skip entries where `v == null` | Closes the constructor bypass path. Makes the invariant fully airtight. |
| `get()` line 80 | Keep `Optional.of(...)` | With both null guards, null cannot reach `get()`. `Optional.of` is the correct strong assertion. |
| `fireEvent()` line 181 | Keep `Optional.of(previous.value())` when `previous != null` | Invariant is airtight. `Optional.of` is correct. |

No `Optional.ofNullable` anywhere — using it would silently mask an invariant violation.

### Tests — plain JUnit (no Quarkus), `InMemoryCaseFileNullTest`

- `put(key, null)` → key absent: `contains()` false, `get()` empty
- `put(key, value)` then `put(key, null)` → no-op, key still holds original value
- `put(key, a)` then `put(key, b)` → normal update works (regression guard)
- Constructor with `{k: "v"}` → key present and readable
- Constructor with `{k: null}` (via `HashMap`) → key filtered, absent after construction
- First-write `fireEvent` path: listener receives `Optional.empty()` as previous

### After fix

`mvn install -DskipTests` from `/Users/mdproctor/claude/casehub-poc/casehub-persistence-memory/`.

Quarkmind side: leave existing `orElseThrow()` guards. No quarkmind code changes for #175.

---

## #156 — casehub-ledger integration (Layer 4)

### Goal

Agent plugin decisions are captured as ledger entries via `OutcomeRecorder` on state
*transitions only*. This creates a meaningful audit trail and seeds the trust-score pipeline
at Layer 6 (#158). Transition-level recording gives Layer 6 a clean signal (strategy changed,
new attack engagement, new intel observed, new build type queued).

### Verdict semantics

`AttestationVerdict` enum: `SOUND, FLAGGED, ENDORSED, CHALLENGED`.

- `SOUND` — all legitimate plugin executions, including "nothing to do this tick." Positive EigenTrust signal.
- `FLAGGED` — genuine plugin errors only: uncaught exception, inconsistent state. Negative signal.

"No action needed" is not `FLAGGED`.

### Transition detection — when to fire

| Plugin | Fire when | Previous-state field | Hash / comparison |
|--------|-----------|---------------------|-------------------|
| `DroolsStrategyTask` | `STRATEGY` value changes | `volatile String prevStrategy` | String equality |
| `DroolsTacticsTask` | NEAREST_THREAT target changes | `volatile String prevTargetTag` | `unit.tag()` string — NOT `Unit` record (position changes every tick) |
| `DroolsScoutingTask` | enemy set changes | `volatile int prevEnemyHash` | `enemyUnits.stream().map(Unit::tag).sorted().collect(Collectors.joining()).hashCode()` — tag-only, sorted for determinism |
| `EconomicsDecisionService` | queued build TYPE changes | `volatile String prevBuildType` | String equality on build type name |

On the first call after game start (`prevX == null`), any non-null output is a transition —
event fires unconditionally. No complex test state setup required.

**Unit tag note:** `Unit` is `record Unit(String tag, UnitType type, Point2d position, ...)`.
Record `equals()` and `hashCode()` include `position`, which changes every tick. Never compare
`Unit` instances or use `Unit.hashCode()` for transition detection. Always extract `.tag()`.

### New components (quarkmind)

#### `GameSession @ApplicationScoped` — `io.quarkmind.agent`

```java
@ApplicationScoped
public class GameSession {
    private volatile UUID id = UUID.randomUUID();
    public UUID id() { return id; }
    public void reset() { id = UUID.randomUUID(); }  // public — required for CDI proxy access
}
```

`AgentOrchestrator` calls `gameSession.reset()` in `startGame()` **only**. `stopGame()` does
NOT reset — async CDI observers dispatched during final ticks must be attributed to the
correct session.

#### `PluginDecisionEvent` — `io.quarkmind.agent`

```java
public record PluginDecisionEvent(
    String actorId,             // plugin.getId()
    String capabilityTag,       // QuarkMindCapabilityTag constant
    AttestationVerdict verdict, // SOUND or FLAGGED
    UUID gameSessionId,
    int gameFrame               // logged by observer for tracing which frame the transition occurred
) {}
```

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
            e.capabilityTag(), e.verdict(), 0.7
        ));
        log.debugf("Ledger: %s %s frame=%d", e.actorId(), e.verdict(), e.gameFrame());
    }
}
```

Confidence 0.7 = game-level decision scope (transition events, not tick-level noise).

#### `LedgerLifecycleAdapter @ApplicationScoped` — `io.quarkmind.agent`

Follows `EconomicsLifecycle` pattern (see `plugin/flow/EconomicsLifecycle.java`): a
quarkmind-side `@ApplicationScoped` bean that bridges CDI lifecycle events to infrastructure.
`clear()` is not on the `LedgerEntryRepository` SPI — inject the concrete type via
`@Any Instance<>` so the call is safe in all profiles.

```java
@ApplicationScoped
public class LedgerLifecycleAdapter {

    @Inject @Any
    Instance<InMemoryLedgerEntryRepository> memoryLedger;

    void onGameStop(@Observes GameStopped event) {
        if (!memoryLedger.isUnsatisfied()) {
            memoryLedger.get().clear();
        }
    }
}
```

In `%sc2` (production): `InMemoryLedgerEntryRepository` is not in `selected-alternatives` →
`isUnsatisfied()` returns true → no-op. In `%test`/`%mock`/`%emulated`/`%replay`: bean is
selected → `clear()` called. `Instance<>` injection is always valid regardless of whether the bean exists
(resolution is lazy).

### Plugin wiring

Plugins already import from `io.quarkmind.agent`. Adding `Event<PluginDecisionEvent>` and
`GameSession` injection follows the same established pattern.

`FlowEconomicsTask.execute()` only submits a tick to the flow; does not fire event.
`EconomicsDecisionService` is the decision point.

### Dependencies (pom.xml)

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger-api</artifactId>
  <version>0.2-SNAPSHOT</version>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger</artifactId>
  <version>0.2-SNAPSHOT</version>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger-memory</artifactId>
  <version>0.2-SNAPSHOT</version>
</dependency>
```

### Required application.properties additions

**Global (all profiles):**
```properties
casehub.ledger.outcome.default-attestor-id=quarkmind:game-engine@v1
```

**`%test` profile:**
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

Note: the two reactive alternatives are listed defensively. With
`casehub.ledger.reactive.enabled=false` (the build-time default), these beans are excluded from
augmentation and are never actually resolved. They are harmless to list and protect against
a future profile that enables reactive.

**`%mock` profile:**
```properties
%mock.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.memory.InMemoryLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryLedgerMerkleFrontierRepository,\
  io.casehub.ledger.memory.InMemoryActorTrustScoreRepository,\
  io.casehub.ledger.memory.InMemoryKeyRotationRepository,\
  io.casehub.ledger.memory.InMemoryAgentSigner,\
  io.casehub.ledger.memory.InMemoryActorIdentityBindingRepository,\
  io.casehub.ledger.memory.InMemoryReactiveLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryReactiveKeyRotationRepository
%mock.quarkus.index-dependency.casehub-ledger-memory.group-id=io.casehub
%mock.quarkus.index-dependency.casehub-ledger-memory.artifact-id=casehub-ledger-memory
%mock.casehub.ledger.hash-chain.enabled=false
```

**`%emulated` profile:**
```properties
%emulated.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.memory.InMemoryLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryLedgerMerkleFrontierRepository,\
  io.casehub.ledger.memory.InMemoryActorTrustScoreRepository,\
  io.casehub.ledger.memory.InMemoryKeyRotationRepository,\
  io.casehub.ledger.memory.InMemoryAgentSigner,\
  io.casehub.ledger.memory.InMemoryActorIdentityBindingRepository,\
  io.casehub.ledger.memory.InMemoryReactiveLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryReactiveKeyRotationRepository
%emulated.quarkus.index-dependency.casehub-ledger-memory.group-id=io.casehub
%emulated.quarkus.index-dependency.casehub-ledger-memory.artifact-id=casehub-ledger-memory
%emulated.casehub.ledger.hash-chain.enabled=false
```

**`%replay` profile:**
```properties
%replay.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.memory.InMemoryLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryLedgerMerkleFrontierRepository,\
  io.casehub.ledger.memory.InMemoryActorTrustScoreRepository,\
  io.casehub.ledger.memory.InMemoryKeyRotationRepository,\
  io.casehub.ledger.memory.InMemoryAgentSigner,\
  io.casehub.ledger.memory.InMemoryActorIdentityBindingRepository,\
  io.casehub.ledger.memory.InMemoryReactiveLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryReactiveKeyRotationRepository
%replay.quarkus.index-dependency.casehub-ledger-memory.group-id=io.casehub
%replay.quarkus.index-dependency.casehub-ledger-memory.artifact-id=casehub-ledger-memory
%replay.casehub.ledger.hash-chain.enabled=false
```

### Integration test — `LedgerAuditIT @QuarkusTest` (`%test` profile)

Follows `DroolsStrategyTaskTest` pattern: inject the plugin, call `execute(cf)` directly on
a hand-constructed CaseFile. On first call, `prevStrategy == null` and any non-null output is
a transition — event fires unconditionally. No `SimulatedGame` setters needed.

`GameSession.reset()` is `public` — required for CDI proxy access in `@QuarkusTest`.
Package-private methods are not accessible through `@ApplicationScoped` CDI proxies.

```java
@QuarkusTest
class LedgerAuditIT {
    @Inject @CaseType("starcraft-game") StrategyTask strategyTask;
    @Inject LedgerEntryRepository ledgerRepo;
    @Inject GameSession gameSession;

    @BeforeEach void reset() { gameSession.reset(); }

    // Single test method by design: DroolsStrategyTask.prevStrategy is a volatile field
    // on an @ApplicationScoped bean and persists across @Test methods in the same run.
    // The first call (prevStrategy == null) always fires a transition. A second method
    // using the same CaseFile state will produce the same strategy output — no change,
    // no event, assertion fails. Future additions must either produce a different strategy
    // value or add a package-private clearPrevState() on DroolsStrategyTask.
    @Test
    void strategyTransitionWritesLedgerEntry() throws InterruptedException {
        CaseFile cf = caseFile(/* minimal valid state per DroolsStrategyTaskTest helper */);
        strategyTask.execute(cf);
        Thread.sleep(300); // CDI @ObservesAsync dispatches asynchronously
        List<LedgerEntry> entries = ledgerRepo.findBySubjectId(gameSession.id());
        assertThat(entries).isNotEmpty();
        assertThat(entries).anyMatch(e -> "strategy.drools".equals(e.actorId()));
    }
}
```

### ARC42STORIES.MD update

Layer 4 row updated to ✅. `#156` forward reference cleared.
Issue #166 (migration to ARC42STORIES.MD) is CLOSED. LAYER-LOG.md is retired.

---

## Sequence

1. Fix `InMemoryCaseFile` (4 sites: `put()` null guard, constructor filter, `get()` `Optional.of` kept, `fireEvent()` `Optional.of` kept) + test
2. `mvn install -DskipTests` from `casehub-poc/casehub-persistence-memory/`
3. Add pom.xml deps (`casehub-ledger-api`, `casehub-ledger`, `casehub-ledger-memory`) at `0.2-SNAPSHOT` (artifacts already installed in local repo; reinstall via `mvn install -DskipTests` from `casehub/ledger/` if stale)
4. Add `casehub.ledger.outcome.default-attestor-id=quarkmind:game-engine@v1` globally
5. Add `%test`, `%mock`, `%emulated` property blocks (8 alternatives each, index-dependency, hash-chain=false)
6. Add `GameSession` (public `reset()`), `PluginDecisionEvent`, `QuarkMindCapabilityTag`,
   `PluginOutcomeAuditor` in `io.quarkmind.agent`
7. Add `LedgerLifecycleAdapter` in `io.quarkmind.agent` (`@Any Instance<>` + `@Observes GameStopped`)
8. Wire `gameSession.reset()` in `AgentOrchestrator.startGame()`
9. Wire transition detection + `Event<PluginDecisionEvent>` in plugins:
   - `DroolsStrategyTask` — `volatile String prevStrategy`
   - `DroolsTacticsTask` — `volatile String prevTargetTag` (tag only, not `Unit`)
   - `DroolsScoutingTask` — `volatile int prevEnemyHash` (tag-sorted string hash)
   - `EconomicsDecisionService` — `volatile String prevBuildType`
10. Write `LedgerAuditIT` using `ledgerRepo.findBySubjectId(gameSession.id())`
11. Update ARC42STORIES.MD Layer 4 status to ✅
12. Invoke `update-claude-md` (remove stale LAYER-LOG reference)
