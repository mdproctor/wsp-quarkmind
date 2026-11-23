# HANDOFF — quarkmind

## Orientation for Next Session

**Economy accuracy improvement phase.** Five issues queued (#751-#755) targeting the mineral rate MAPE gap (38.2% vs 10-12% floor). The root cause is cycle cadence variance — EM worker cycles are too uniform (sigma 0.5 vs SC2 sigma 3+).

**Start with:** `work continue` — picks up #751 (skip initial worker mineral deduction, S/Low).

**Key reference:** `docs/sc2-physics-constants.md` — fully revised this session, all numbers current as of 2026-11-23.

---

## This Session (2026-11-23)

**Landed 8 issues:**
- #745, #746, #676, #747, #666 — economy physics and sliding window rate algorithm
- #748, #749, #750 — cleanup (buildingRadius removal, sweep test ranges, NYDUS_CANAL footprint)

**Created 5 improvement issues:** #751-#755 (starting worker deduction, cycle variance diagnostic, dynamic patch selection, walk collision, baseline regeneration)

**Revised:** `docs/sc2-physics-constants.md` — fixed 6 stale sections, added improvement delta table, updated accuracy plan

**Blog:** "Halving the Distance" — economy accuracy journey diary entry

**Filed:** Hortora/soredium#617 — `work.py start` pipeline bug (create_branches uses stale ctx.branch)

**Closed 12 stale issues:** #744, #745, #746, #676, #747, #739, #733, #734, #735, #736, #737, #666

---

## Queue

| # | Issue | Scale | What |
|---|-------|-------|------|
| 1 | **#751** | S/Low | Skip initial worker mineral deduction (-600 at tick 0) |
| 2 | #752 | M/Med | Per-worker cycle variance diagnostic |
| 3 | #753 | M/High | Dynamic worker patch selection on return |
| 4 | #754 | M/High | Walk-path collision between workers in transit |
| 5 | #755 | XS/Low | Regenerate standalone baseline |

---

## Notes

- **Pipeline bug (soredium#617):** `work.py start` create_branches step uses stale queue branch field. Workaround: call `branch_create.py create-branches` directly, then set queue branch via plan_manager.
- **Accuracy state:** minerals 84.9% balance / 38.2% rate, gas 61.1% balance / 18.7% rate. Targets: rate <=20%, balance <=60%.
