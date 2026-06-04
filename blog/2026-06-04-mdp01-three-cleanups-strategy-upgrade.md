---
title: "Three cleanups and a strategy upgrade"
date: 2026-06-04
tags: [drools, rule-units, c2-intel, architecture]
entry_type: phase-update
---

The small stuff first: two cleanups closed quickly. The dead `MAP_CENTER` fallback in `DroolsTacticsTask` — two ternary guards that could never fire once the `NEAREST_THREAT` gate was in place — came out in two lines. The GitHub transfer from `mdproctor/quarkmind` to `casehubio/quarkmind` was straightforward once I found that `gh repo transfer` as a CLI subcommand doesn't exist in this version of gh (the REST API works fine).

The third issue was more interesting.

`DroolsStrategyTask` has been reading `ENEMY_UNITS` directly from the CaseFile and feeding the unit list into Drools. That's the observation layer — raw SC2 data. Scouting already classifies that data into `ENEMY_POSTURE` ("ALL_IN", "MACRO", "UNKNOWN") and `TIMING_ATTACK_INCOMING` before strategy ever runs. Strategy was bypassing the abstraction entirely and doing its own inference from raw units.

Writing the spec for that change exposed something I was wrong about. I claimed `ENEMY_POSTURE = "UNKNOWN"` unambiguously means the scout hasn't made contact yet — pre-contact only. That's not true. `ScoutingSessionManager` evicts unit events after three minutes, and `seenUnitTags` never resets within a game — so if the probe was killed early, posture can revert to UNKNOWN long after first contact. Three rounds of spec review caught this. The final design acknowledges it explicitly: ATTACK fires under UNKNOWN because it's equivalent to the old `not /enemies` signal (no current enemy contact), even though the semantics aren't identical. The edge case where a previously-scouted unit reappears but doesn't re-populate the event buffer — posture UNKNOWN, unit visible — is accepted as a known limitation. It's documented in a protocol rather than papered over.

The implementation hit one genuine surprise. I designed the DEFEND and ATTACK rules to use `eval()` on plain `StrategyRuleUnit` fields (`timingAttackIncoming`, `enemyPosture`). `eval()` alone compiles fine. `eval()` combined with `accumulate()` in the same rule doesn't — the generated lambda loses field scope and the build fails with a cryptic `cannot find symbol` in a generated class named something like `LambdaPredicateA685BF2DCD7428F516AE3214ED64CD15`. Nothing in the Drools or Quarkus documentation mentions this restriction. The fix was to feed posture and timing as `DataStore<String>` and `DataStore<Boolean>` facts and use pattern matching instead: `not /postureStore[this.equals("ALL_IN")]`. Cleaner than `eval()` anyway — PHREAK can alpha-index DataStore patterns; `eval()` cannot be indexed.

`BasicStrategyTask` was deleted in the same commit. It had no CDI annotations — not `@ApplicationScoped`, not `@Alternative`, nothing. It was never actually injected anywhere. The tests had a comment saying both implementations "share identical decisions," which was going to be false the moment the C2 rules landed. Keeping a class that was wrong and wasn't used seemed worse than the three-second delete.
