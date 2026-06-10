# Layer 6: Teaching the Agent When to Blitz and When to Expand

**Date:** 2026-06-10

Layer 5 taught the agent *whether* to fire a plugin based on game conditions. Layer 6 teaches it *which* strategy to pursue — and whether that strategy is actually working.

The three strategies are simple in isolation. `DroolsStrategyTask` reasons per-tick from CEP intel. `EarlyPressureStrategyTask` commits to an aggressive build from tick one, no questions asked. `EconomicExpansionStrategyTask` expands the economy first and scales military later. What L6 adds is the ability to learn, across games, which of these pays off against a given type of opponent.

The mechanism: `StrategyTrustRouter` uses Bayesian Beta trust scores from `casehub-ledger`, keyed by opponent context (`vs.aggressive`, `vs.economic`, `vs.defensive`). Four phases — BOOTSTRAP when there's no data, QUALIFIED once the win rate clears the threshold, BORDERLINE and EXCLUDED for the edge cases. At game start, the router selects based on accumulated history; a mid-game checkpoint re-evaluates once `DroolsScoutingTask` has confidently classified the opponent's posture.

The spec went through four rounds of review before implementation. Each round found something real.

## What the spec missed

The most consequential finding came from bytecode inspection: `LedgerLifecycleAdapter`, a class that had lived quietly in the harness since Layer 4, was clearing the in-memory ledger at every game stop. The class existed for test isolation convenience — clean slate between games. The problem is it used a synchronous `@Observes GameStopped`, which runs inline during `Event.fire()`. The outcome recorder used `@ObservesAsync`, which runs on a worker thread after `fire()` returns. So every game: ledger cleared, then outcome written, then incremental trust update runs against a ledger containing exactly one entry. `decisionCount` permanently stuck at 1. The threshold of 10 observations was permanently unreachable.

No error. No warning. L6 would have appeared to work while producing no strategic learning at all.

The fix was to delete `LedgerLifecycleAdapter` entirely. Test isolation now lives where it belongs — in `@BeforeEach` for the tests that actually need it.

The `@ObservesAsync` question came up again on `GameOutcomeRecorder`. The obvious choice is async — it's recording to an in-memory store, but still, side effects should be async. The problem: `StrategySelector.reset()` fires synchronously on the next `GameStarted`. If the recording happens async, it may read the reset state from the new game instead of the completed game's strategy. Same class of bug, different direction. `GameOutcomeRecorder` uses `@Observes` — synchronous — for the same reason `GameSession` never resets in `stopGame()`.

## The spec was wrong about the bootstrap score

TDD caught a design error the spec missed. The algorithm used `phaseScore = 1.0` for BOOTSTRAP candidates — availability as a proxy. The tests for QUALIFIED candidates expected them to outrank BOOTSTRAP candidates. They failed. A QUALIFIED candidate at 0.82 trust produces a blend score of 0.892. BOOTSTRAP is 1.0. QUALIFIED always loses.

The invariant that makes trust routing useful — "proven track record beats no track record" — requires BOOTSTRAP below the minimum QUALIFIED score. With `blendFactor=0.6` and `threshold=0.65`, minimum QUALIFIED is about 0.838. Setting BOOTSTRAP to 0.5 preserves 0.5 < 0.838 under any reasonable configuration. The spec said 1.0; the implementation uses 0.5; the tests prove why.

## Three competing implementations broke existing tests

Adding three `@CaseType("starcraft-game") StrategyTask` implementations immediately caused `AmbiguousResolutionException` in every `@QuarkusTest` class that injected `StrategyTask` by interface. CDI sees three candidates; any single-bean injection point fails.

The fix is mechanical — inject the concrete type where a specific implementation is the subject under test — but it's a permanent change to how new competing seam implementations are handled. The protocol is in the docs now: when a seam has multiple active implementations, `@QuarkusTest` injects the concrete type.

## The state it leaves

L6 infrastructure is complete. The trust scores accumulate (once `casehub.ledger.trust-score.*` flags are set; without them it's a silent no-op). The routing mechanism is correct, verified by integration tests seeded with real trust scores.

What doesn't work yet: the win/loss signal. Every game records `SOUND` regardless of outcome, which means all strategies will eventually converge to near-identical trust scores. The routing selects among them but can't distinguish. The fix is a separate issue — reading `playerResult` from the SC2 API after game end — but the infrastructure waits for it.

The agent can now potentially learn which strategy to use. It just can't learn yet.
