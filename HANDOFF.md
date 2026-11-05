# HANDOFF — quarkmind

## Last Session (2026-11-05)

Branch `issue-530-archon-supply-tracking` closed. Landed as `0a155e97` on main. Five issues completed across supply, morph costs, spending classification, and diagnostic investigation.

### What landed (0a155e97)

- **#530** — Archon supply tracking (supplyCost 0 → 4)
- **#528** — Morph cost tracking (MorphIntent from tracker, subtract investment on death)
- **#529** — SC2 spending classification (army MAPE 62% → 10%, tech MAPE 42.8% → 23.9%)
- **#531** — Map missing building types (UNKNOWN 218 → 0)
- **#533** — Technology spending diagnostic (24% MAPE is structural floor from PlayerStatsEvent update frequency)

Key findings from #533:
- Classification is correct — no reclassification improves MAPE
- ~25,925m in InProgress tech at 5-min (buildings under construction, not yet in "Current")
- SC2 score.proto has 5 categories (none/army/economy/technology/upgrade); tracker events fold to 3

Blog entries: "Measuring the Right Thing at the Wrong Time", "The Floor You Can't Fix"

### Accuracy

| Field | MAPE |
|-------|------|
| mineralsUsedCurrentArmy | 10.4% |
| mineralsUsedCurrentEconomy | 4.7% |
| mineralsUsedCurrentTechnology | **23.9%** (floor — structural) |

### Queue status

Epic #516 — Standalone EmulatedGame:

1. ~~#530 — Archon supply tracking~~ ✅
2. ~~#528 — Morph cost tracking~~ ✅
3. ~~#529 — SC2 spending classification~~ ✅
4. ~~#531 — Track missing building types~~ ✅
5. ~~#533 — Technology mineral remaining gap~~ ✅ (floor identified)
6. #532 — Vespene spending variance (S/Med)
7. #519 — Gas income ramp-up and worker assignment (M/Med) — 25% MAPE
8. #518 — Distance-aware mineral mining model (L/High) — 22-54% MAPE
9. #520 — Standalone worker lifecycle tracking (M/Med) — blocked by #518, #519
10. **#523 — Standalone building lifecycle (L/High) — NEXT**
11. #524 — Final gate: remove all GT syncs (S/Low) — blocked by all above

### Next session recommendation

Start with #523 (standalone building lifecycle). Key design inputs:
- TrackerIntentExtractor does NOT generate BuildIntents — buildings come entirely from GT sync
- Tracker events DO include UnitInit (placement), UnitDone (completion), UnitTypeChange (morphs), UnitDied (destruction) for buildings
- Focus on building COUNT accuracy (currently 100% from GT). Spending accuracy has the 24% structural floor regardless.
- score.proto says "technology" = "any structure that produces units or upgrades" — classification for spending is correct as-is

## Known Issues

- **work-end deleting plans** — user skipping work-end; closing issues via `gh issue close` directly
- **Reconstituted data inconsistency**: blizzard_ladder_4.9.3 has mixed calibrated/uncalibrated files
- **blizzard_ladder_4.9.2 reconstituted data deleted** — needs re-export
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — standalone accuracy
- `docs/blog/2026-11-05-mdp02-the-floor-you-cant-fix.md` — blog on eventual consistency
- `quarkmind-sc2/.../TechSpendingDiagnosticTest.java` — diagnostic test suite
- Epic #516 — standalone EmulatedGame issue tracker
