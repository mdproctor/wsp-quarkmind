# Handover — 2026-05-08

**Head commit:** `b617eee` — blog entry "Clean Desk, Bigger Picture"

## What Changed This Session

**Panel inspect fixed:**
- `showBuildingPanelAsync` never cleared/drew the portrait canvas — buildings showed stale unit portrait or blank. Fixed.
- Global `canvas { width: 100vw !important }` CSS rule stretched the 64×64 portrait to full viewport width. Fixed with `#up-portrait { width: 64px !important; height: 64px !important }`.
- Panel was fetching `/qa/unit/{tag}` on every click (~1.5s in `quarkus:dev` due to live-reload scanner). Now reads from `lastState` (WebSocket cache) — instant.
- 4 new Playwright panel E2E tests. `scene.updateMatrixWorld(true)` required in `clickUnit`/`clickBuilding` before raycasting (matrixWorld not updated until first render frame).

**Building collision (#125, closed):**
- `SC2Data.buildingRadius(BuildingType)` — circular footprint radii (2.5 for Nexus/CC/Hatchery, 1.5 for 3×3, 1.0 for 2×2).
- `enforceWall()` blocks unit entry into completed building footprints. Entry-only semantics (workers inside Nexus radius at spawn move freely). Always active (no terrain grid required).

**Full tidy-up sprint:**
- 10 stale issues closed (#116–125 — enemy AI + pathfinder epics and children, panel/collision).
- DESIGN.md: PixiJS→Three.js throughout, test count, Current State, Combat Model, Open Questions, E6 added.
- 5 new ADRs (0002–0006) + `docs/adr/INDEX.md` created.
- `Observer.supplyCost` fixed 2→1 with test.
- `docs/roadmap-sc2-engine.md` rewritten — Phases 0–3 marked complete, Phase 5 progress table (E1–E6), Phase 6 planned.
- `pixi.min.js` deleted (dead since E14 Three.js migration).
- `MODULES.md` updated — split deferred indefinitely.
- Scouting calibration: PvT TERRAN_3RAX threshold raised 5→7 (normal play peaks at 6 Marines; false-positive rate was ~50%). #115 closed.
- 3 forage entries submitted to garden (Three.js matrixWorld race, Quarkus dev-mode HTTP latency, global canvas CSS rule).

**Phase 6 planning (brainstorm, no epic opened yet):**
- Goal confirmed: replay-accurate forward simulation — feed real player commands into `EmulatedGame`, compare state against `ReplaySimulatedGame` tick by tick.
- Approach 1 (incremental): economic layer first (exact match), then movement (within 0.5 tiles), then combat.
- Infrastructure already exists: `GameEventStream`, `UnitOrderTracker`. Missing: ability command extraction (build/train from CmdEvent ability IDs) and `ReplayValidationHarness`.
- Recorded in `docs/roadmap-sc2-engine.md` Phase 6.

## Immediate Next Step

**Start Phase 6 epic.** Open a GitHub issue/epic for "Replay-Accurate Forward Simulation", then begin:
1. Extract build/train ability commands from replay `CmdEvent`s (ability ID → Intent translation)
2. Write `ReplayValidationHarness` — runs `ReplaySimulatedGame` + `EmulatedGame` in parallel, records divergence per tick
3. First validation target: economic layer (build times, train times, resource rates) — assert exact match

## Open Issues

| # | What | Status |
|---|------|--------|
| #13 | Live SC2 smoke test | Blocked on SC2 |
| #14 | GraalVM native image | Blocked on #13 |
| #74 | YAML unit definitions | Parked |

## New Workflow Conventions

See `memory/feedback_workflow.md` — IntelliJ MCPs required, TDD in planning, code review before completion, prefer IntelliJ over Bash for semantic ops.

## References

| Context | Where |
|---------|-------|
| Blog entry (this session) | `docs/_posts/2026-05-08-mdp01-clean-desk-bigger-picture.md` |
| Phase 6 roadmap | `docs/roadmap-sc2-engine.md` |
| ADR index | `docs/adr/INDEX.md` |
| Prior handover | `git show HEAD~1:HANDOFF.md` |
