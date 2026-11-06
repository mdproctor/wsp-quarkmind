# HANDOFF — quarkmind

## Last Session

Closed #546 (tech spending ghost completion) and #547 (Factory/Starport add-on swap undercount). Ghost completion fix: buildings destroyed mid-construction left PendingCompletion callbacks alive — added entityTag to cancel them. Add-on swap fix: MORPH_BUILDING_TARGETS didn't include add-on variant names (FactoryReactor, StarportTechLab, etc.) — added ADDON_SWAP_TARGETS as instant type changes. Factory/Starport/Barracks all now at 100%. Building totals improved 94.7→95.5% at 7-min.

Total: 10 of 10 original issues closed, 5 of 7 remaining queue items still open.

## Immediate Next Step

Pick up any of the 5 remaining queue items:
- #545 (OC/Lair morph timing, S/Med) — OC 95-98%, Lair 22-85% accuracy
- #548 (Creep Tumor, XS/Low) — 0-57% accuracy
- #549 (Unit mode tracking, S/Med) — SIEGE_TANK_SIEGED/RAVAGER/OVERSEER show 0
- #538 (Mineral balance, M/High) — research
- #539 (Vespene balance, M/Med) — research

#545 has the highest building accuracy impact — Lair at 22% accuracy is the worst non-CreepTumor building gap.

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — current accuracy
