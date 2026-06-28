# QuarkMind Handover — 2026-06-28

## Last Session

Closed #171 (EmulatedSC2Server). Code review caught tag collisions (unit/building/enemy tags all mapping to the same protobuf tag), null key in WebSocket handshake, and shared mutable state — all fixed. 14 commits squashed to 3, pushed to origin/main. Blog published (3 entries). Build green.

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

Build on main is clean. `mvn install` passes. CI green on main.
