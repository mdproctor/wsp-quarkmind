# QuarkMind Handover — 2026-06-28

## Last Session

Two branches closed. #171 (EmulatedSC2Server) — code review caught tag collisions and null safety gaps, 14 commits squashed to 3. #206 (POM parent inheritance) — quarkmind now inherits from casehub-parent:0.2-SNAPSHOT, groupId changed to io.casehub, version aligned to 0.2-SNAPSHOT. Quarkus downgraded from 3.34.2 to 3.32.2 to match parent — all 1022 tests pass. Garden: 3 entries submitted, 1 protocol captured.

## Immediate Next Step

Pick new work from What's Next. Run `/work` to start a branch.

## What's Left

- quarkmind#74 — genericise unit/building definitions — trademark removal · L · Med
- quarkmind#192 — CBR reference implementation · L · High (needs engine Phase 2 first)
- quarkmind#203 — SC2DebugScenarioRunner support for %emulated-sc2 · S · Low
- quarkmind#204 — RealSC2Engine.addFrameListener() no-op — visualizer silent in %sc2 and %emulated-sc2 · S · Low
- quarkmind#205 — eliminate six-way CDI config duplication across profiles · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #203 | SC2DebugScenarioRunner for %emulated-sc2 | S | Low | Widen profile gate, depends on #171 (done) |
| #204 | Fix RealSC2Engine.addFrameListener() | S | Low | Benefits both %sc2 and %emulated-sc2 |
| #205 | Eliminate CDI config duplication | S | Low | Move to unqualified defaults |
| #74 | Genericise unit/building definitions | L | Med | Independent |
| #192 | CBR reference implementation | L | High | Blocked on engine Phase 2 (engine#483, #484) |

## Build Note

Build on main is clean. `mvn install` passes (1022 tests). POM now inherits from casehub-parent — `casehub-ledger-memory` and `casehub-qhorus-api` use `${casehub.version}` (not in parent BOM).
