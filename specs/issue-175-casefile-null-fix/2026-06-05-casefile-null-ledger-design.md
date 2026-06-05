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

### Fix — three sites in `InMemoryCaseFile.java` (casehub-poc)

| Site | Change | Rationale |
|------|--------|-----------|
| `put()` top | `if (value == null) return;` | Prevents null storage. Matches `MapCaseFile` contract. Makes `contains()` correct as side-effect — no null `ItemEntry` values enter the store. |
| `get()` line 80 | `Optional.of(...)` unchanged | After the null guard in `put()`, null cannot reach `get()`. Keep `Optional.of` — it is the correct strong assertion. `ofNullable` would make the impossible look possible and weaken the invariant. |
| `fireEvent()` line 181 | `Optional.of(previous.value())` → `Optional.ofNullable(previous.value())` | `previous` is from `ref.getAndSet()` in `put()` — it is the *previously stored* ItemEntry, which may have been stored before the null guard was added. `ofNullable` is correct here for the transition period. |

### Tests — plain JUnit (no Quarkus), `InMemoryCaseFileNullTest`

- `put(key, null)` → key remains absent: `contains()` false, `get()` empty
- `put(key, value)` then `put(key, null)` → no-op, key still holds original value
- `put(key, a)` then `put(key, b)` → normal update still works (regression guard)
- `fireEvent` null path: verify listener receives `Optional.empty()` as previous when first write

Thread safety: `put(null)` early-return is a read-only check on an immutable null reference;
no synchronization concern.

### After fix

Rebuild and install: `mvn install -DskipTests` from
`/Users/mdproctor/claude/casehub-poc/casehub-persistence-memory/`.

**Quarkmind side:** leave existing `orElseThrow()` guards in place. No quarkmind code changes for #175.

---

## #156 — casehub-ledger integration (Layer 4)

### Goal

Agent plugin decisions are captured as ledger entries via `OutcomeRecorder` on state
*transitions only* — not every tick. This creates a meaningful audit trail and seeds the
trust-score pipeline used at Layer 6 (#158). Transition-level recording gives Layer 6 a
clean signal (strategy changed, engagement started, intel arrived) rather than high-volume
tick-level noise (~88 entries/second at 22fps × 4 plugins).

### Verdict semantics

`AttestationVerdict` enum values: `SOUND, FLAGGED, ENDORSED, CHALLENGED`.

- `SOUND` — used for **all legitimate plugin executions**, including "nothing to do this tick"
  (no enemy in range, no budget, no new intel). Positive EigenTrust signal.
- `FLAGGED` — reserved for genuine plugin errors: uncaught exception, inconsistent output,
  unexpected null from a required CaseFile key. Negative EigenTrust signal.

Recording "no action" as `FLAGGED` would systematically penalize plugins for normal quiescent
game states and corrupt trust scores at Layer 6.

### Transition detection — when to fire

| Plugin | Fire when | `SOUND` condition | `FLAGGED` condition |
|--------|-----------|-------------------|---------------------|
| `DroolsStrategyTask` | `STRATEGY` value changes from previous | Strategy changed | Exception in rule evaluation |
| `DroolsTacticsTask` | new attack target selected (NEAREST_THREAT changes) | Target found | Exception resolving threat |
| `DroolsScoutingTask` | enemy intel first observed or enemy unit set changes | Intel updated | Exception in scouting rules |
| `EconomicsDecisionService` | new build intent queued | Intent queued | Exception in decision |

Each plugin tracks its previous output value in a `volatile` field. No external state store
required.

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
    int gameFrame               // for transition logging
) {}
```

No `decision` field — not persisted to `OutcomeRecord`, pure allocation pressure at game speed.
`gameFrame` is logged by the observer for debugging (which frame the transition occurred).

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

Cross-cutting infrastructure, not a game plugin. Belongs in `agent/`, not `plugin/`.

```java
@ApplicationScoped
public class PluginOutcomeAuditor {
    @Inject OutcomeRecorder outcomeRecorder;

    public void onDecision(@ObservesAsync PluginDecisionEvent e) {
        outcomeRecorder.record(OutcomeRecord.of(
            e.actorId(), e.gameSessionId(),
            e.capabilityTag(), e.verdict(), 0.1
        ));
        log.debugf("Ledger: %s %s frame=%d", e.actorId(), e.verdict(), e.gameFrame());
    }
}
```

Confidence 0.1 = tick-level (per `OutcomeRecord` recommendation). Transitions are infrequent
so 0.1 still applies — each captured event represents one observed game decision.

### Plugin wiring

Plugins already import from `io.quarkmind.agent` (`ResourceBudget`, `QuarkMindCaseFile`,
`StrategyTask`). Adding `Event<PluginDecisionEvent>` and `GameSession` injection continues the
same established pattern. No seam-rule violation.

| Plugin | Event source | Previous-state tracking |
|--------|-------------|------------------------|
| `DroolsStrategyTask` | end of `execute()` if strategy changed | `volatile String prevStrategy` |
| `DroolsTacticsTask` | end of `execute()` if NEAREST_THREAT changed | `volatile Unit prevTarget` |
| `DroolsScoutingTask` | end of `execute()` if enemy set changed | `volatile int prevEnemyCount` |
| `EconomicsDecisionService` | after `decide()` if a new intent was queued | stateless — fire on every queue |

`FlowEconomicsTask.execute()` only submits a tick to the flow; does not fire event. Economics
decision event fires from `EconomicsDecisionService`.

### Dependencies (pom.xml)

```xml
<!-- Ledger API — OutcomeRecorder SPI, OutcomeRecord, AttestationVerdict -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger-api</artifactId>
  <version>${casehub-ledger.version}</version>
</dependency>

<!-- Ledger runtime — DefaultOutcomeRecorder @DefaultBean -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger</artifactId>
  <version>${casehub-ledger.version}</version>
</dependency>

<!-- In-memory ledger SPIs — no JPA datasource needed for %mock/%emulated/%test -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger-memory</artifactId>
  <version>${casehub-ledger.version}</version>
</dependency>
```

### Required application.properties additions

**Global (all profiles):**
```properties
# Required by DefaultOutcomeRecorder — throws ISE if absent and OutcomeRecord.attestorId is null
casehub.ledger.outcome.default-attestor-id=quarkmind:game-engine@v1
```

**`%test` profile:**
```properties
# Activate in-memory ledger alternatives (required — @Priority(1) alone is not enough)
%test.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.memory.InMemoryLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryLedgerMerkleFrontierRepository,\
  io.casehub.ledger.memory.InMemoryActorTrustScoreRepository,\
  io.casehub.ledger.memory.InMemoryKeyRotationRepository,\
  io.casehub.ledger.memory.InMemoryAgentSigner

# Quarkus cannot scan Jandex-less jars — required for in-memory alternatives
%test.quarkus.index-dependency.casehub-ledger-memory.group-id=io.casehub
%test.quarkus.index-dependency.casehub-ledger-memory.artifact-id=casehub-ledger-memory

# Disable hash chain (not needed for in-memory tutorial)
%test.casehub.ledger.hash-chain.enabled=false
%test.casehub.ledger.reactive.enabled=false
```

**`%mock` and `%emulated` profiles:** same selected-alternatives + index-dependency blocks as
`%test`. `casehub.ledger.enabled` defaults to `true` — no explicit set needed.

**CDI risk (GE-20260519-e13b01):** Adding `casehub-ledger` runtime can crash `@QuarkusTest` if
`ReactiveLedgerEntryRepository` is unsatisfied. With `casehub-ledger-memory` on the classpath
AND the alternatives explicitly selected, all ledger SPIs are satisfied. TDD will confirm; a
package-private `@ApplicationScoped` test stub resolves it if the memory module alone is
insufficient (see GE).

### Memory lifecycle

`InMemoryLedgerEntryRepository` accumulates entries indefinitely. Wire `ledgerRepo.clear()` in
`AgentOrchestrator.stopGame()` to prevent unbounded growth across games. Transition-level
recording reduces volume significantly (a 30-min game produces O(hundreds) of events, not
O(tens of thousands)).

### Integration test — `LedgerAuditIT @QuarkusTest` (`%test` profile)

Deliberate state setup required before `gameTick()` — strategy plugin only fires a transition
event when READY, ENEMY_POSTURE, and TIMING_ATTACK_INCOMING are present in the CaseFile:

```java
simulatedGame.setReady(true);
simulatedGame.setEnemyPosture("AGGRESSIVE");
simulatedGame.setTimingAttack(false);
// ... minimal game state so strategy rules fire and change from default
```

Assertions:
- After `gameTick()` + `Thread.sleep(300)` (async observer): at least one `LedgerEntry` persisted
- At least one entry has `actorId = "strategy.drools"` and `capabilityTag = "starcraft.strategy"`
- All entries have `SOUND` verdict (no errors injected)

### ARC42STORIES.MD update

Layer 4 row updated to ✅ complete. `#156` forward reference cleared. (Issue #166 — migration
to ARC42STORIES.MD — is CLOSED. LAYER-LOG.md is retired. Do not write a LAYER-LOG entry.)

### CLAUDE.md update (concrete step)

Remove the stale "Until migration is complete, new layer completions still require a LAYER-LOG
entry" language. Update to reference ARC42STORIES.MD as the sole architecture record.
Invoke `update-claude-md` as a numbered step after committing #156.

---

## Sequence

1. Fix `InMemoryCaseFile` in casehub-poc (3 sites + test)
2. `mvn install -DskipTests` in casehub-poc to publish fixed snapshot
3. Add pom.xml deps (casehub-ledger-api, casehub-ledger, casehub-ledger-memory)
4. Add application.properties config (default-attestor-id, selected-alternatives, index-dependency)
5. Add `GameSession`, `PluginDecisionEvent`, `QuarkMindCapabilityTag` in `agent/`
6. Add `PluginOutcomeAuditor` in `agent/` (not `plugin/`)
7. Wire transition detection + event firing in all four plugins / EconomicsDecisionService
8. Wire `gameSession.reset()` in `startGame()`; wire `ledgerRepo.clear()` in `stopGame()`
9. Write `LedgerAuditIT` with deliberate state setup
10. Update ARC42STORIES.MD Layer 4 status
11. Invoke `update-claude-md` (remove stale LAYER-LOG reference)
