# HANDOFF — quarkmind

## Last Session (2026-11-05)

Completed #530, #528 from epic #516 (Standalone EmulatedGame). Researched #529.

### What landed

- **#530** — Archon supply tracking. `supplyCost(ARCHON)` changed from 0 to 4. HT merge is now supply-neutral: 2 HT deaths free 4 supply, Archon birth adds 4.

- **#528** — Morph cost tracking. Three changes:
  1. `TrackerIntentExtractor` now emits MorphIntent on `ID_UNIT_TYPE_CHANGE` for resource-costing morphs (Baneling, Ravager, BroodLord, Lurker, Overseer)
  2. `EmulatedGame.handleMorph` records spending and updates unit type in-place for tracker-driven morphs (preserves tag for later UnitDied matching)
  3. `EconomyTracker.recordUnitDeath` subtracts total investment (morph cost + source unit cost) via `SC2Data.morphSourceType` lookup

- **#529** — SC2 spending classification research (partial). Confirmed our `isEconomyBuilding` classification matches SC2 exactly. Production buildings ARE technology (not army). Analytically tested reclassification: moving production buildings to army improves tech MAPE (~13%) but destroys army MAPE (~143%). The spending overcount is an accounting issue, not classification — likely initial units/buildings being counted, morph accounting (partially fixed by #528), and death subtraction gaps. Comment added to issue.

### Branch

`issue-530-archon-supply-tracking` — contains both #530 and #528 commits.

### Queue status

Epic #516 — Standalone EmulatedGame. Updated queue:

1. ~~#530 — Archon supply tracking~~ ✅
2. ~~#528 — Morph cost tracking~~ ✅
3. #529 — SC2 spending classification research — classification done, MAPE target (below 20%) needs baseline re-run after #528 to measure improvement
4. #519 — Gas income ramp-up and worker assignment (M/Med) — 25-50% MAPE
5. #518 — Distance-aware mineral mining model (L/High) — 22-54% MAPE
6. #520 — Standalone worker lifecycle tracking (M/Med) — blocked by #518, #519
7. #523 — Standalone building lifecycle (L/High) — structural, eliminates syncBuildings
8. #524 — Final gate: remove all GT syncs (S/Low) — blocked by all above

## Known Issues

- **Reconstituted data inconsistency**: blizzard_ladder_4.9.3 has mixed calibrated/uncalibrated files
- **blizzard_ladder_4.9.2 reconstituted data deleted** — needs re-export
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid
- **No oracle replay data on this machine** — StandaloneEmulatedGameBaselineTest cannot run; baseline needs regeneration after #528

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — standalone accuracy (pre-#528)
- `docs/benchmarks/tracker-driven-accuracy-baseline.md` — GT-synced accuracy
- Epic #516 — standalone EmulatedGame issue tracker
