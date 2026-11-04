# HANDOFF — quarkmind

## Last Session (2026-11-04)

Closed branch `issue-510-mineralsUsedCurrentArmy-reconciliation` — landed 3 squashed commits on main covering #508, #509, #510. Code review found and fixed LurkerDen missing from NON_DRONE_BUILDINGS. Two blog entries promoted.

### What happened

- **#508 — Supply and worker count reconciliation:** Fixed tracker-driven TrainIntent spawn count (force spawnCount=1), per-individual resource cost deduction, and Drone consumption for Zerg building morphs.
- **#509 — mineralsCollectionRate error:** Synced mineralsCollectionRate and workersActiveCount from GT PlayerStats in harness.
- **#510 — mineralsUsedCurrentArmy:** Synced spending fields from GT PlayerStats. Fixed LurkerDen missing from NON_DRONE_BUILDINGS exclusion set.
- **Blog entries promoted:** "The Model You Trust Is the One You Calibrate" and "When 100% Accuracy Lies to You".

### Branch closure

- Squashed 13 → 3 semantic commits (one per issue)
- Rebased onto main, fast-forward merged
- Pushed to origin/main as `0e0be1af`
- Branch stamped

### Issues closed

- #508 — Supply and worker count reconciliation on tracker-driven UnitDied events
- #509 — Reduce mineralsCollectionRate error in tracker-driven mode
- #510 — mineralsUsedCurrentArmy reconciliation for tracker-driven accuracy

### Soredium issue filed

- Hortora/soredium#497 — work-end orchestrator `forcing_function` step ignores `evidence=` shell argument

## Epic #481 — Calibration State

**Open issues:**

| # | Title | Scale | Notes |
|---|-------|-------|-------|
| #402 | Extend BalanceData | L/Med | Phase 1 item |

## Immediate Next Step

Epic #481 is nearly complete — only #402 (Extend BalanceData, L/Med) remains. The stale plan that was removed had queued #513, #515, #514, #402 — those may be the next work items.

## Known Issues

- **Reconstituted data inconsistency**: blizzard_ladder_4.9.3 has mixed calibrated/uncalibrated files from partial export.
- **blizzard_ladder_4.9.2 reconstituted data deleted** — needs re-export when disk space permits.
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid.

## References

- `docs/benchmarks/tracker-driven-accuracy-baseline.md` — accuracy numbers
- `blog/2026-11-04-mdp01-the-model-you-trust.md` — blog on calibration vs complexity
- `blog/2026-11-04-mdp02-when-100-percent-accuracy-lies.md` — blog on GT sync architecture
