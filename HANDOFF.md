# QuarkMind Handover — 2026-06-27

## Last Session

Implemented #171 — EmulatedGame SC2 protocol wrapper. New `%emulated-sc2` profile wraps EmulatedGame in a full SC2 WebSocket/protobuf server so `RealSC2Engine` connects to it identically to real SC2. Six implementation tasks complete (TerrainGrid.toPathingGrid, GameStateToProtobuf, ProtobufToIntent, SC2WebSocketCodec, profile gates + config, EmulatedSC2Server). Three design review rounds refined ocraft validation requirements. 1022 tests green. Pre-close sweep done (forage: 1 new GE + 1 revision, diary written). Branch not yet closed.

## Immediate Next Step

Run `/work-end` on branch `issue-171-emulated-sc2-protocol-wrapper` to close. The pre-close sweep (forage, protocol, diary, update-claude-md) is already done — work-end should skip Step 3b and proceed to code review (Step 3c), then artifact promotion, journal merge, rebase, squash, and push.

## What's Left

- quarkmind#74 — genericise unit/building definitions — trademark removal · L · Med
- quarkmind#192 — CBR reference implementation · L · High (needs engine Phase 2 first)
- quarkmind#203 — SC2DebugScenarioRunner support for %emulated-sc2 · S · Low
- quarkmind#204 — RealSC2Engine.addFrameListener() no-op — visualizer silent in %sc2 and %emulated-sc2 · S · Low
- quarkmind#205 — eliminate six-way CDI config duplication across profiles · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #203 | SC2DebugScenarioRunner for %emulated-sc2 | S | Low | Widen profile gate, depends on #171 |
| #204 | Fix RealSC2Engine.addFrameListener() | S | Low | Benefits both %sc2 and %emulated-sc2 |
| #205 | Eliminate CDI config duplication | S | Low | Move to unqualified defaults |
| #74 | Genericise unit/building definitions | L | Med | Independent |
| #192 | CBR reference implementation | L | High | Blocked on engine Phase 2 (#483, #484) |

## Build Note

Build on branch is clean. `mvn test` passes (1022 tests). Branch has 11 commits ahead of main.
