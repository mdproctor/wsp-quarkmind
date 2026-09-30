# HANDOFF — quarkmind

## Last Session

Completed #328 — synthesize morph-based unit events in `StrippedReplayFeatureExtractor`. Landed as `76297c8b` on main (4 commits after squash).

What was built:
- All morph types now emit timed UnitInit+UnitDone (was instant UnitBorn for standard unit morphs)
- `trackMorphSpending()` — economy tracking for unit and building morph costs
- Overseer supply fix in `applyEventToEconomy` UnitDone case (regression from UnitBorn→UnitInit reclassification)
- 5 unit morph abilLinks discovered from oracle replays and dispatched in `AbilityMapping`: Baneling=73, Ravager=309, BroodLord=194, Lurker=522, Overseer=221
- Selection-based morph multiplication via `morphMultiplier()` (capped at MAX_MULTIPLICATION=4)
- Calibrated morph times from Liquipedia: Baneling=448, Ravager=269, Lurker=403, BroodLord=538, Overseer=269

Key design decision: used `AbilityMapping.selectionSize()` for multiplication instead of `SelectionUnitLinkTracker.countMatching()` — simpler, avoids needing unitLink discovery for source units, accurate because morph selections are typically homogeneous.

## Follow-Up Issues Filed

- #329 — Archon morph source incorrectly hardcodes HighTemplar (DarkTemplar merges not distinguished). Bug, S/Low.
- #330 — Discover building morph abilLinks (OrbitalCommand, Lair, Hive, GreaterSpire, PlanetaryFortress). Bot replays don't emit building morph CmdEvents — needs human ladder replays or SC2 Galaxy Editor. Enhancement, S/Med.
- #331 — Re-calibrate morph times from replay data. Current values are Liquipedia-derived; protocol requires replay-based calibration. Enhancement, S/Low.

## Immediate Next Step

Continue epic #318 — remaining sub-issues: #325 (CreepTumor + special building types), #324 (auto-spawned unit events), #323 (expand UpgradeType enum).

## References

- Design spec: `specs/issue-328-morph-unit-events/2026-09-30-morph-unit-events-design.md`
- Implementation plan: `plans/2026-09-30-morph-unit-events.md`
- Decisions: `specs/issue-328-morph-unit-events/decisions.md`
- Design review: `/Users/mdproctor/reviews/casehub-quarkmind/issue-328-morph-unit-events-20260930-114935/`
