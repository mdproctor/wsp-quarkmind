# Handover — 2026-05-16

**Head commit (project):** `50c5728` — docs: update CLAUDE.md with Phase 6 test classes  
**Head commit (workspace):** `63ff55b` — docs: add blog entry 2026-05-16

## What Changed This Session

**Phase 6 — Replay-Accurate Forward Simulation: complete and merged to main.**

Key components landed:
- `GameEventStream` refactored to thin MPQ reader (`events(Path) → List<Event>`)
- `AbilityMapping` — stateful CmdEvent → TrainIntent converter; ability IDs discovered empirically via `AbilityDiscoveryTest` from PvZ replays
- `ReplayCommand`, `TimedIntent`, `ReplayCommandStream`, `ReplayCommandExtractor` pipeline
- `EmulatedGame` extended: `injectReplayBuilding(Building)`, `markReplayBuildingComplete(String)`, `setMiningProbes(int)`
- `DivergenceReport` + `ReplayValidationHarness` + `ReplayValidationTest` (passes: maxUnitDelta ≤ 2)
- `ReplayValidationReportTest` (`mvn test -Preport`) for full divergence dump

**Critical finding:** bot SC2 replays use `abilLink=42` (Smart) for building placement — identical to movement, indistinguishable from GAME_EVENTS. Solution: inject buildings from tracker events into EmulatedGame each tick. Garden entry `GE-20260516-42a11c` captures this as undocumented behaviour.

**Issues filed this session:**
- #138 — Terran/Zerg EmulatedGame mechanics  
- #140 — Terran `.SC2Replay` files for AbilityMapping discovery  
- #141 — Saturation-aware mining model (needed for exact unit count match)

**Skill fix:** protocol sweep no longer skips when `docs/protocols/` is absent — the skill now creates it. Both `handover` and `protocol` skills updated in cc-praxis (commit `f5397b4`).

## Immediate Cleanup Required (do at session start)

1. **Sync DESIGN.md** — Phase 6 components are NOT in `docs/DESIGN.md`. Run `java-update-design` to add `AbilityMapping`, `ReplayCommandExtractor`, `ReplayValidationHarness`, `DivergenceReport` to the module table and test list. The `sc2/replay/` row currently only lists `ReplayEngine` and `ReplaySimulatedGame`.

2. **Close issue #137** — epic close workflow was never run:
   ```bash
   gh issue close 137 --repo mdproctor/quarkmind
   ```

3. **Remove `design/.meta` from workspace** — epic-phase-6 closed but `.meta` was merged to workspace main:
   ```bash
   git -C ~/claude/public/quarkmind rm design/.meta
   git -C ~/claude/public/quarkmind commit -m "chore: remove epic-phase-6 metadata after close"
   ```
   Keep `design/JOURNAL.md` for reference.

4. **Delete debug file** — `src/test/java/io/quarkmind/sc2/replay/HarnessDebugTest.java` is untracked (created during harness debugging). Review and delete if not needed.

## Immediate Next Step

**Start the next epic.** Phase 6 is complete on main. Options per `docs/roadmap-sc2-engine.md`:
- Saturation mining model (#141) — needed for exact unit count validation
- Terran/Zerg EmulatedGame (#138) — blockers for multi-race validation
- Phase 7 scope TBD

Run `/epic` from the workspace to open the next epic.

## Open Issues

| # | What | Status |
|---|------|--------|
| #13 | Live SC2 smoke test | Blocked on SC2 |
| #14 | GraalVM native image | Blocked on #13 |
| #74 | YAML unit definitions | Parked |
| #138 | Terran/Zerg EmulatedGame | Open |
| #140 | Terran replay files for AbilityMapping | Open |
| #141 | Saturation mining model | Open |

## References

| Context | Where |
|---------|-------|
| Phase 6 spec | `docs/superpowers/specs/2026-05-14-phase6-replay-validation-design.md` |
| Phase 6 plan | `~/claude/public/quarkmind/plans/2026-05-15-phase6-replay-validation.md` |
| Roadmap | `docs/roadmap-sc2-engine.md` |
| Design journal | `~/claude/public/quarkmind/design/JOURNAL.md` |
| Blog entry | `~/claude/public/quarkmind/blog/2026-05-16-mdp01-smart-command-hides.md` |
