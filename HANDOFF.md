# Handover — 2026-05-17

**Head commit (project):** `d78217d` — docs: add docs/protocols/ to project artifacts table
**Head commit (workspace):** `d5220ec` — docs: update handover with epic close cleanup items

## What Changed This Session

All cleanup items from the previous handover are complete:

- **#137 closed** — `gh issue close 137`
- **DESIGN.md synced** — Phase 6 pipeline components documented (`sc2/replay/` table, `EmulatedGame` injection methods, test classes, report profile, deferred issues #141/#140/#138)
- **`design/.meta` removed** from workspace main
- **Plan archived** to `plans/attic/epic-phase-6/`
- **`HarnessDebugTest.java` deleted** (untracked debug file, never committed)
- **`docs/protocols/` created** in project — first protocol: `PP-20260517-72f3c9` (§Section anchors required in JOURNAL.md for epic-close merge)
- **Garden:** 2 new entries — epic-close-orphaned-meta gotcha, journal-§section-anchor gotcha

**Key finding captured as protocol:** The epic close journal merge silently skips plain prose — only `§Section`-anchored entries (`### YYYY-MM-DD · §SectionName`) are merged into DESIGN.md. `java-update-design` (via `java-git-commit`) writes these automatically; standalone commits don't.

## Immediate Next Step

**Start Phase 7 or the next issue.** All Phase 6 housekeeping is complete. See `docs/roadmap-sc2-engine.md` for options:
- `#141` Saturation mining model — needed for exact unit count in `ReplayValidationTest`
- `#138` Terran/Zerg `EmulatedGame` mechanics
- `#140` Terran replay files for `AbilityMapping` discovery

Run `/epic` from the workspace to open the next epic.

## Open Issues

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## References

| Context | Where |
|---------|-------|
| Protocol index | `docs/protocols/INDEX.md` |
| DESIGN.md | `docs/DESIGN.md` |
| Roadmap | `docs/roadmap-sc2-engine.md` |
