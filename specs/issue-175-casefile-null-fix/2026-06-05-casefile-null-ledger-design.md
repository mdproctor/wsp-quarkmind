# Design: InMemoryCaseFile null fix (#175) + casehub-ledger integration (#156)

**Branch:** `issue-175-casefile-null-fix`
**Issues:** #175 (XS), #156 (L)
**Date:** 2026-06-05

---

## #175 — InMemoryCaseFile null correctness

### Root cause

`InMemoryCaseFile.put(key, null)` stores `ItemEntry(null, version)`. The `AtomicReference` is non-null, so `contains(key)` returns `true`, but `get(key, Type)` then calls `Optional.of(null)` which throws `NullPointerException`. The same `Optional.of` pattern appears in `fireEvent`.

### Fix — three sites in `InMemoryCaseFile.java` (casehub-poc)

| Site | Change | Rationale |
|------|--------|-----------|
| `put()` top | Add `if (value == null) return;` | Prevents null storage. Matches `MapCaseFile` contract. Makes `contains()` correct as side-effect — no null `ItemEntry` values enter the store. |
| `get()` line 80 | `Optional.of(...)` → `Optional.ofNullable(...)` | Defence in depth. |
| `fireEvent()` line 181 | `Optional.of(previous.value())` → `Optional.ofNullable(previous.value())` | Same bug, same fix. |

### Tests — plain JUnit (no Quarkus), `InMemoryCaseFileNullTest`

- `put(key, null)` → key remains absent: `contains()` false, `get()` empty
- `put(key, value)` then `put(key, null)` → no-op, key still holds original value
- `put(key, a)` then `put(key, b)` → normal update still works (regression guard)

### After fix

Rebuild and install casehub-poc: `mvn install -DskipTests` from the casehub-poc root.

**Quarkmind side:** leave existing `orElseThrow()` guards in place — still valid defensive coding. No quarkmind code changes for #175.

---

## #156 — casehub-ledger integration (Layer 4)

### Goal

Every agent plugin decision is captured as a `LedgerEntry` via `OutcomeRecorder`. This creates an audit trail and seeds the trust-score pipeline used by Layer 6 (#158). Captures are non-blocking (async CDI observer), so ledger I/O does not affect game loop latency.

### New components (quarkmind)

#### `GameSession @ApplicationScoped` — `io.quarkmind.agent`

Holds `volatile UUID id`, initialized at construction. `AgentOrchestrator` calls `gameSession.reset()` in `startGame()` and `stopGame()`. Provides stable per-game scope for `OutcomeRecord.subjectId`.

```java
@ApplicationScoped
public class GameSession {
    private volatile UUID id = UUID.randomUUID();
    public UUID id() { return id; }
    void reset() { id = UUID.randomUUID(); }
}
```

#### `PluginDecisionEvent` — `io.quarkmind.agent`

```java
public record PluginDecisionEvent(
    String actorId,             // plugin.getId()
    String capabilityTag,       // QuarkMindCapabilityTag constant
    String decision,            // human-readable, log/debug only
    AttestationVerdict verdict, // SUCCESS = real decision; UNCLEAR = fallback/default
    UUID gameSessionId,
    int gameFrame
) {}
```

`UNCLEAR` verdict signals a fallback/default — meaningful for trust scoring:
- Strategy = "MACRO" (Drools fired no rules) → `UNCLEAR`
- No nearest threat found → `UNCLEAR`
- No economics build queued (budget exhausted) → `UNCLEAR`

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

#### `PluginOutcomeAuditor @ApplicationScoped` — `io.quarkmind.plugin`

```java
@ApplicationScoped
public class PluginOutcomeAuditor {
    @Inject OutcomeRecorder outcomeRecorder;

    public void onDecision(@ObservesAsync PluginDecisionEvent e) {
        outcomeRecorder.record(OutcomeRecord.of(
            e.actorId(), e.gameSessionId(),
            e.capabilityTag(), e.verdict(), 0.1
        ));
    }
}
```

Confidence 0.1 = tick-level, per `OutcomeRecord` recommendation.

### Plugin wiring — `Event<PluginDecisionEvent>` injected into each plugin

| Plugin | Fire point | Verdict logic |
|--------|-----------|---------------|
| `DroolsStrategyTask` | after writing `STRATEGY` to CaseFile | `SUCCESS` if decision ≠ "MACRO"; `UNCLEAR` if Drools fired no strategy rule |
| `DroolsTacticsTask` | after nearest-threat resolution | `SUCCESS` if target found; `UNCLEAR` if no target |
| `DroolsScoutingTask` | after intel update | `SUCCESS` if enemy intel written; `UNCLEAR` if no update |
| `EconomicsDecisionService` | after build decision | `SUCCESS` if build intent queued; `UNCLEAR` if no budget |

`FlowEconomicsTask.execute()` only submits a tick to the flow — it does not fire the event. `EconomicsDecisionService` is the decision point.

All plugins inject `GameSession` to get `gameSessionId`. All inject `Event<PluginDecisionEvent>`.

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

`casehub-ledger-memory` provides `@Alternative @Priority(1)` implementations of all ledger persistence SPIs, displacing JPA repos by classpath presence for the mock/emulated/test profiles.

**CDI risk (GE-20260519-e13b01):** Adding `casehub-ledger` runtime can crash `@QuarkusTest` if `ReactiveLedgerEntryRepository` is unsatisfied. With `casehub-ledger-memory` on the classpath, in-memory repos should satisfy all ledger SPI requirements. TDD will confirm; a package-private `@ApplicationScoped` stub in `src/test/java/` resolves it if the memory module alone is insufficient.

### Integration test — `LedgerAuditIT @QuarkusTest` (`%test` profile)

- Calls `orchestrator.gameTick()` directly (scheduler disabled in `%test` profile)
- Waits briefly (`Thread.sleep(300)`) for async CDI observer to complete
- Asserts at least one `LedgerEntry` was persisted (inject in-memory `LedgerEntryRepository`)
- Asserts at least one entry has `actorId` matching a known plugin (e.g. `"strategy.drools"`)

### ARC42STORIES.MD update

Layer 4 row updated to ✅ complete. `#156` forward reference cleared.

### CLAUDE.md staleness

The CLAUDE.md note "Until migration is complete, new layer completions still require a LAYER-LOG entry" is stale — issue #166 (migration to ARC42STORIES.MD) is closed. Flag for `update-claude-md`.

---

## Sequence: #175 first, then #156

1. Fix and test `InMemoryCaseFile` in casehub-poc
2. Install casehub-poc snapshot to local Maven repo
3. Proceed with #156 TDD in quarkmind (ledger deps, new components, plugin wiring, integration test, ARC42STORIES.MD update)
