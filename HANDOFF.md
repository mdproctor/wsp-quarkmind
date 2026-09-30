# HANDOFF — quarkmind

## Last Session

Closed #329, #330, #331 from epic #318. Landed as `72cb7399` on main (3 commits after squash).

What was built:
- #329 fix: Archon morph source now resolved from selection unitLink — DarkTemplar merges correctly identified (was hardcoded to HighTemplar). Added `tagToUnitLink` tracking in `AbilityMapping.onSelection()` with subgroup zipping.
- #330 partial: CC building morphs dispatched (abilLink=120, idx=1→OrbitalCommand, idx=0→PlanetaryFortress). Zerg building morphs undiscoverable from available data.
- #331 partial: Overseer (269→267, n=13) and Ravager (269→271, n=62) morph times calibrated via `MorphTimeCalibrationTest`. Baneling/Lurker/BroodLord blocked on ZvZ replay data.

Key finding: bot replays don't emit building morph CmdEvents (bots use API calls). Zerg morph calibration also blocked — AI Arena dataset is Protoss-focused.

## Follow-Up Issues Filed

- #332 — Discover Zerg building morph abilLinks (Lair, Hive, GreaterSpire). Needs ZvZ human replays.
- #333 — Calibrate Baneling, Lurker, BroodLord morph times. Same data dependency.

## Immediate Next Step

Continue epic #318 — #325 (CreepTumor + special building types) is the highest-impact remaining gap (959 missing events), independent of the ZvZ data blocker.

## References

- Diagnostic tests: `BuildingMorphDiscoveryTest.java`, `MorphTimeCalibrationTest.java`
- Previous session handover: `git show 4391501:HANDOFF.md`
