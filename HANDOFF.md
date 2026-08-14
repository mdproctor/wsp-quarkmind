# QuarkMind Handover — 2026-08-14

## Last Session

Brainstormed and specced the QuarkMind restructure — evolving from SC2-only to a multi-world autonomous agent platform. Multi-module skeleton (#272) is complete and closed. Created issues for all Phase 2 worlds (#273–#277) and the core harness extraction (#278).

## Current State

- Branch `issue-272-quarkmind-restructure` has the multi-module skeleton done
- All 7 modules exist and compile: quarkmind-core (empty), quarkmind-sc2 (all current code), plus stubs for town/minecraft/evennia/sonaria/godot-mcp
- #272 is closed

## Immediate Next Step

**#278 — quarkmind-core — extract shared agency harness**

Populate quarkmind-core with the agency framework. This gates all world implementations. Create a new branch `issue-278-quarkmind-core-harness`.

What to build in quarkmind-core:
- `AgencyLoop` — perceive/need/goal/plan/act/reflect, thin wrapper over CaseEngine
- `NeedState` — decaying floats with Eidos disposition modifiers
- `WorldBridge<P, I>` SPI — generic world connection contract
- `IntentQueue<I>` — generic intent buffer
- `NavigationSPI`, `VisibilitySPI`, `SpatialMemory` — spatial awareness contracts
- `InteractionTrigger`, `InteractionPipeline` — trigger→LLM→response pattern
- `MomentDetector` — significant event detection contract
- `LlmRequestQueue` — rate limiting, prioritisation, token budgeting

Design principles:
- SPIs support both strings and rich typed models (Java generics)
- Supports sequential tick loops AND async independent clients — composition not prescription
- Wraps CaseEngine with agency vocabulary
- Extract patterns from SC2 code as reference, but SC2 doesn't implement SPIs yet

## Constraints

- All refactoring via IntelliJ MCP, no subagents
- quarkmind-sc2 tests must stay green
- No behavioural changes to SC2

## Work Queue

```
- [ ] #278 — quarkmind-core — extract shared agency harness ← next
- [ ] #273 — quarkmind-town — Sims-like 3D life simulation
- [ ] #274 — quarkmind-minecraft — Mineflayer bridge + Luanti CI
- [ ] #275 — quarkmind-evennia — MUD agent bridge
- [ ] #276 — quarkmind-sonaria — Roblox/Creatures of Sonaria agent
- [ ] #277 — quarkmind-godot-mcp — Godot EditorPlugin MCP server
```

#278 gates everything. #273–#277 can proceed in parallel after #278.

## References

- Spec: `specs/quarkmind-restructure/2026-08-14-quarkmind-restructure-design.md`
- Decisions: `specs/quarkmind-restructure/decisions.md` (D1–D17)
- Phase 1 plan (skeleton — done): `plans/2026-08-14-quarkmind-restructure-phase1.md`
- Prior work parked: #271 (SC2 opponent ID), #268 (CDI cleanup), #270 (engagement outcomes) — land in quarkmind-sc2
