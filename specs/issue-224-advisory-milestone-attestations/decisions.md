## D1: Trigger sharing model

**Choice:** Shared triggers — advisory milestones fire at the same game checkpoints as strategy milestones (same MilestoneTrigger instances). Eligibility differs: advisors must have been invoked before the milestone frame.
**Alternatives:**
- Independent trigger evaluation — separate trigger instances for advisors, decoupled from strategy milestones
**Rationale:** Milestones represent game checkpoints; evaluating both strategies and advisors at the same point is semantically correct. The only difference is the eligibility filter.
**Trade-offs:** Advisors can't have custom milestone schedules independent of strategies.
**Exploration:** quick
**Status:** captured

## D2: Entry lifecycle

**Choice:** Single ledger entry per advisor per game. First milestone creates via `recordAndReturnId()`, subsequent milestones and game-end use `addAttestation()`.
**Alternatives:**
- Separate entries per milestone — each milestone creates its own entry
- Separate entries for milestones vs game-end — milestone entries distinct from game-end entry
**Rationale:** Preserves `decisionCount = games played` (not inflated by milestone count), critical for `minimumObservations` threshold in trust routing. Matches strategy-side pattern.
**Trade-offs:** Requires coordination between milestone and game-end recording within the same class.
**Exploration:** quick
**Status:** captured

## D3: Class ownership

**Choice:** Extend `AdvisoryGameOutcomeRecorder` into `AdvisoryMilestoneOutcomeRecorder` — single class owns both milestone evaluation and game-end recording.
**Alternatives:**
- Keep `AdvisoryGameOutcomeRecorder` for game-end only, create separate `AdvisoryMilestoneRecorder` — two classes share entry lifecycle
**Rationale:** Mirrors strategy-side pattern (MilestoneOutcomeRecorder replaced GameOutcomeRecorder). Single class avoids entry creation coordination, race conditions, and split ownership of the same ledger entry.
**Trade-offs:** Larger class with two responsibilities (milestone + game-end). Acceptable given they share entry lifecycle.
**Exploration:** quick
**Status:** captured

## D4: First-invocation frame tracking

**Choice:** Extend `AdvisoryInvocationCounter` to store `Map<String, Long>` (advisorId → first invocation frame) instead of `Set<String>`. Add `firstFrame(advisorId)` method; `record()` gains a `long gameFrame` parameter.
**Alternatives:**
- Separate tracker in the recorder — parallel `Map<String, Long>` duplicating "was this advisor invoked?"
**Rationale:** `AdvisoryInvocationCounter` is already the single source of truth for advisory invocations. Adding frame data is a natural extension, not a new concern. Avoids duplicate tracking.
**Trade-offs:** `record()` signature changes — callers must pass game frame. Only one caller (`AdvisoryCompletionObserver`) exists.
**Exploration:** quick
**Status:** captured

## D5: Milestone confidence formula

**Choice:** Same formula as strategies: `temporalWeight × |dominanceScore|`. Advisory's own LLM confidence is not mixed in.
**Alternatives:**
- Weighted with advisory confidence — `temporalWeight × |dominanceScore| × advisoryConfidence`
- Fixed confidence per milestone — e.g., 0.5 regardless of dominance
**Rationale:** Milestone attestation measures "was the game going well at this checkpoint?" — same question for strategies and advisors. Advisory LLM confidence is already captured in `DeferredAdvisoryEvaluator`'s recommendation-quality dimension. Mixing signals conflates dominance trajectory with recommendation quality.
**Trade-offs:** No differentiation between high-confidence and low-confidence advisors at milestone time.
**Exploration:** quick
**Status:** captured

## D6: GameTickExecutor wiring

**Choice:** Parallel call — add `advisoryMilestoneOutcomeRecorder.evaluateMilestones(gameState)` alongside existing `milestoneOutcomeRecorder.evaluateMilestones(gameState)` in GameTickExecutor. Both share CDI-injected triggers and assessor.
**Alternatives:**
- Unified MilestoneDispatcher — abstraction iterating all MilestoneEvaluable implementations
- CDI event from strategy recorder — advisory recorder observes strategy milestone events
**Rationale:** Simplest approach. Two parallel calls, no new abstractions for two consumers. Avoids coupling advisory to strategy evaluation ordering.
**Trade-offs:** Two call sites in GameTickExecutor instead of one. Acceptable — both are side-effect calls in the same block.
**Depends on:** D1 (shared triggers)
**Exploration:** quick
**Status:** captured
