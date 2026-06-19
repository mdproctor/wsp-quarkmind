# QuarkMind Handover — 2026-06-19

## Last Session

Branch `issue-201-fix-ci` closed. Fixed Build and Test CI (red since 2026-06-12): inlined scelight-mpq/s2protocol (150 Java + 122 .dat files, Apache 2.0) and casehub-core/persistence-memory (55 Java files, POC) directly into src/main/java/; added casehubio/* GitHub Packages repository for the 0.2-SNAPSHOT deps. Single squashed commit on main. 1006 tests pass; CI green.

## Immediate Next Step

Pick new work from What's Next. Run `/work` to start a branch.

## What's Left

- quarkmind#74 — genericise unit/building definitions — trademark removal · L · Med
- quarkmind#192 — CBR reference implementation · L · High (needs engine Phase 2 first)
- casehubio/parent#280 — sync quarkmind.md: L7 Comparison Baseline now complete

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #74 | Genericise unit/building definitions | L | Med | Independent |
| #192 | CBR reference implementation | L | High | Needs engine Phase 2 first |
| engine#483+484 | Phase 2: signalAndAwaitSync + SequenceWorker | L | Med | Engine team work — unblocks #192 |

## Build Note

Build on main is clean. `mvn install` passes (1006 tests). CI is now green on main.
