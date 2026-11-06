# HANDOFF — quarkmind

## Last Session

Closed #546 (tech spending overcount). Fixed ghost completion bug: buildings destroyed mid-construction left PendingCompletion callbacks alive, recording spending for buildings that never finished. Added `entityTag` to PendingCompletion so `handleBuildingDied` cancels associated completions. Improvement: mineralsUsedCurrentTechnology MAPE 30.7%→30.3% at 7-min. Diagnostic revealed the bulk of overcounting (30%) is structural — even GT reconstruction overcounts by ~15%, indicating a cost model semantics gap vs SC2's internal spending classification.

Total: 9 of 10 original issues closed, 6 of 7 remaining queue items still open.

## Immediate Next Step

Pick up any of the 6 remaining queue items. Best ROI candidates:
- #545 (OC/Lair morph timing, S/Med) — OC 85-98%, Lair 22-85% accuracy
- #547 (Factory/Starport undercount, S/Low) — 86-91% accuracy
- #548 (Creep Tumor, XS/Low) — 0-57% accuracy
- #549 (Unit mode tracking, S/Med) — SIEGE_TANK_SIEGED/RAVAGER/OVERSEER show 0

#538 and #539 (mineral/vespene balance) are M/High research items best deferred.

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — current accuracy
