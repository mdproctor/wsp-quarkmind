# Advisory Milestone Attestations — Design Spec

**Issue:** quarkmind#224
**Date:** 2026-08-14
**Depends on:** #191 (milestone infrastructure — closed), engine#648 (`addAttestation` SPI)

---

## Problem

`AdvisoryGameOutcomeRecorder` records a single game-end attestation per advisor. An advisor that gives good advice early but the game is lost late gets the same penalty as one that gave bad advice throughout. The milestone infrastructure (#191) solves this for strategies — #224 extends it to advisors.

The twist: advisors are invoked at different points during a game (unlike strategies, which are selected at game start). An advisor first invoked at frame 5000 shouldn't receive milestone attestations for checkpoints at frame 4032.

## Solution

Extend the milestone trust scoring pattern from strategies to advisors. Reuse `MilestoneTrigger` and `DominanceAssessor` infrastructure. Each invoked advisor gets milestone attestations at shared game checkpoints, filtered by first-invocation frame. A single ledger entry per advisor per game preserves `decisionCount = games played`.

### Confidence model

Same formula as strategy milestones:

```
confidence = temporalWeight × |dominanceScore|
```

- **temporalWeight** — from the `MilestoneTrigger` (frame-based or phase-based)
- **dominanceScore** — from `DominanceAssessor.assess(GameState)`, in [-1.0, +1.0]
- **Verdict** — `dominanceScore > 0` → ENDORSED, `< 0` → CHALLENGED
- **Dead zone** — `|dominanceScore| < deadZoneThreshold` → skip

Advisory LLM confidence is not mixed in — it is already captured in `DeferredAdvisoryEvaluator`'s `recommendation-quality` dimension. Milestone confidence measures dominance trajectory, not recommendation quality.

---

## Component Architecture

### Changed components

**`AdvisoryInvocationCounter`** — extend backing store from `Set<String>` to `ConcurrentHashMap<String, Long>` (advisorId → first invocation frame).

```java
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
```

- `record()` stores frame only on first invocation (`putIfAbsent`)
- `firstFrame()` returns the frame when the advisor was first invoked
- `snapshot()` returns `Set<String>` — backward compatible
- `onGameStarted()` clears the map (same lifecycle as today)

**`AdvisoryCompletionObserver`** — update call to `invocationCounter.record(event.advisorId(), event.gameFrame())`.

**`AdvisoryMilestoneOutcomeRecorder`** (replaces `AdvisoryGameOutcomeRecorder`) — single class owning the full advisory attestation lifecycle.

Two responsibilities:

1. **`evaluateMilestones(GameState)`** — called each tick from `GameTickExecutor`. If milestones disabled or SPI not available (`OutcomeRecorder` not `instanceof AttestingOutcomeRecorder`), returns immediately. Iterates shared `MilestoneTrigger` instances. For each fired trigger, iterates each invoked advisor from `AdvisoryInvocationCounter.snapshot()`:
   - **Eligibility:** naturally handled by temporal ordering — `snapshot()` only contains advisors already invoked at the current game frame, so an advisor invoked at frame 5000 is absent from the set when a frame-4032 milestone fires. No explicit frame comparison needed
   - **Fired check:** `advisoryMilestoneSession.hasFired(advisorId + ":" + milestoneId)` — skip if already fired for this advisor
   - Run `dominanceAssessor.assess(state)` → apply dead zone → compute confidence
   - First attestation per advisor: `recordAndReturnId()` → store UUID in `AdvisoryMilestoneSession`
   - Subsequent: `addAttestation(entryId, verdict, confidence, "game-outcome")`
   - Mark `advisoryMilestoneSession.markFired(advisorId + ":" + milestoneId)`

2. **`onGameStopped(@Observes GameStopped)`** — for each invoked advisor:
   - If entry exists in `AdvisoryMilestoneSession` (milestones fired): `addAttestation(entryId, verdict, 1.0, "game-outcome")`
   - If no entry (no milestones fired, or advisor invoked after last milestone): `record(OutcomeRecord.of(...))` — identical to today's `AdvisoryGameOutcomeRecorder` behavior

**SPI fallback:** When `OutcomeRecorder` is not `instanceof AttestingOutcomeRecorder`, `evaluateMilestones()` is a no-op. `onGameStopped()` records via `record()` only (current behavior). Milestones activate automatically when the SPI lands.

**`GameTickExecutor`** — inject `AdvisoryMilestoneOutcomeRecorder`. Add call to `advisoryMilestoneOutcomeRecorder.evaluateMilestones(gameState)` alongside the existing `milestoneOutcomeRecorder.evaluateMilestones(gameState)` call, gated by `!isCoachMode`.

### New components

**`AdvisoryMilestoneSession`** — `@ApplicationScoped` per-game state, same pattern as `MilestoneSession`.

```java
@ApplicationScoped
public class AdvisoryMilestoneSession {
    private final Map<String, UUID> entryIds = new ConcurrentHashMap<>();
    private final Set<String> firedMilestones = ConcurrentHashMap.newKeySet();

    public Optional<UUID> entryId(String advisorId) { ... }
    public void setEntryId(String advisorId, UUID id) { ... }
    public boolean hasFired(String compoundKey) { ... }  // "advisorId:milestoneId"
    public void markFired(String compoundKey) { ... }
    public void reset() { entryIds.clear(); firedMilestones.clear(); }
}
```

Compound keys (`"advisorId:milestoneId"`) because milestone eligibility is per-advisor — milestone "frame:4032" might be eligible for advisor A (invoked at frame 1000) but not advisor B (invoked at frame 5000). `MilestoneSession` uses plain `milestoneId` because strategies don't have per-entity eligibility.

Reset on `@Observes GameStarted`. Uses `ConcurrentHashMap` for thread-safe visibility between game-start thread and scheduler thread.

### Unchanged components

- `MilestoneOutcomeRecorder` — strategy milestones (orthogonal)
- `MilestoneSession` — strategy-specific session state
- `MilestoneTrigger`, `FrameThresholdTrigger`, `GamePhaseTrigger` — shared, injected via CDI
- `DominanceAssessor`, `MultiFactorDominanceAssessor` — shared
- `MilestoneConfig` — shared configuration (no separate advisory toggle)
- `DeferredAdvisoryEvaluator` — recommendation-quality dimension (orthogonal)
- `GameSession` — provides subjectId

### Deleted components

- `AdvisoryGameOutcomeRecorder` — fully replaced by `AdvisoryMilestoneOutcomeRecorder`

---

## Configuration

No new configuration. Advisory milestones are controlled by the existing `quarkmind.milestones.enabled` flag and share the same dead-zone threshold and trigger configuration as strategy milestones.

**Profile behavior:** Same as strategy milestones:
- `%sc2`: enabled (meaningful WIN/LOSS/TIE outcomes)
- `%test`: enabled (ephemeral trust storage)
- `%mock`, `%emulated`, `%emulated-sc2`: disabled (UNKNOWN results create one-sided signal)
- `%replay`, `%coach`: disabled

---

## Testing

### Unit tests (plain JUnit)

| Test | Covers |
|------|--------|
| `AdvisoryMilestoneOutcomeRecorderTest` | Milestone → correct verdict/confidence for eligible advisors, ineligible advisors skipped (invoked after milestone frame), dead zone → skip, game-end appends to existing entry, no milestones → creates entry at game-end, SPI fallback (not `AttestingOutcomeRecorder` → `evaluateMilestones()` no-op, game-end via `record()`) |
| `AdvisoryInvocationCounterTest` (extended) | `record(id, frame)` stores first frame only (putIfAbsent), `firstFrame()` returns correct value, `snapshot()` still returns `Set<String>`, `onGameStarted()` clears map |
| `AdvisoryMilestoneSessionTest` | Entry ID per advisor, compound fired tracking ("advisorId:milestoneId"), reset, concurrent access |

### Existing tests updated

- `AdvisoryGameOutcomeRecorderTest` → renamed to `AdvisoryMilestoneOutcomeRecorderTest`, existing game-end tests preserved, milestone tests added
- `AdvisoryCompletionObserverTest` — updated for new `record(id, frame)` signature
- `GameTickExecutorMigrationTest` — verify new call site

### No new integration tests

Same SPI gap as strategy milestones — integration tests wait for `AttestingOutcomeRecorder` from the engine.

---

## Out of scope

- Independent advisory trigger schedules (D1 trade-off — advisors share strategy milestone checkpoints)
- Advisory confidence weighting in milestone formula (D5 trade-off — LLM confidence is a separate dimension)
- Proportional game-end attribution for advisors invoked at different times (analogous to #225 for strategies)
