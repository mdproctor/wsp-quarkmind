# QuarkMind Handover — 2026-08-14

## Last Session

Brainstormed and specced the QuarkMind restructure — evolving QuarkMind from an SC2-only game AI into a multi-world autonomous agent platform. Extensive research into target platforms (Minecraft, Roblox, MUDs, Smallville-style sims), game frameworks, and asset pipelines. Produced a full spec with 17 decisions and a Phase 1 implementation plan.

## What Was Produced

| Artifact | Location |
|----------|----------|
| Design spec | `specs/quarkmind-restructure/2026-08-14-quarkmind-restructure-design.md` |
| Decisions (D1–D17) | `specs/quarkmind-restructure/decisions.md` |
| Phase 1 plan | `plans/2026-08-14-quarkmind-restructure-phase1.md` |
| GitHub issue | casehubio/quarkmind#272 |

## Immediate Next Step

Execute Phase 1 plan — 5 tasks, purely structural refactoring:

1. **Convert to multi-module** — root `pom.xml` becomes parent, all code moves to `quarkmind-sc2/`
2. **Create quarkmind-core** — agency SPIs (WorldBridge, NeedState, IntentQueue, spatial/interaction/moment contracts)
3. **Wire dependency** — quarkmind-sc2 depends on quarkmind-core
4. **Create stub modules** — quarkmind-town, quarkmind-minecraft, quarkmind-evennia, quarkmind-sonaria, quarkmind-godot-mcp
5. **Update docs** — CLAUDE.md, MODULES.md

## Critical Constraints

- **All refactoring via IntelliJ MCP** — type-safe rename, move, find references. No bash file operations on source files. No subagents.
- **SC2 tests must stay green at every step** — run `mvn test` after each task
- **No behavioural changes** — Phase 1 is structural only
- Create branch `issue-272-quarkmind-restructure` before starting

## Architecture Summary

QuarkMind becomes a mono-repo with Maven modules:

```
quarkmind/
├── quarkmind-core/          ← agency framework (SPIs, needs, intents)
├── quarkmind-sc2/           ← current SC2 code (refactored)
├── quarkmind-town/          ← Sims-like 3D town (Godot 4 + Quarkus)
├── quarkmind-minecraft/     ← Mineflayer bridge + Luanti CI
├── quarkmind-evennia/       ← MUD bridge
├── quarkmind-sonaria/       ← Roblox/Sonaria partnership
└── quarkmind-godot-mcp/     ← Godot editor MCP tooling
```

Framework sits on CaseHub foundations (engine, neocortex, ledger, eidos, qhorus, blocks). AgencyLoop wraps CaseEngine with agency vocabulary. SPIs support both string-based and rich typed models. Execution model is configurable — sequential tick loops (SC2) and async independent clients (Town, Minecraft) are both valid compositions.

## Key Decisions (summary)

- **D2:** Multi-module mono-repo (not separate repos)
- **D5:** Extract agency loop, needs/drives, WorldBridge SPI, intent buffer, spatial SPIs, interaction pipeline, moment detection
- **D8:** Each agent is an independent client (client/server, not sequential)
- **D9:** Shared LLM request queue for rate limiting
- **D10:** Needs driven by Eidos dispositions (personality → need decay coefficients)
- **D14:** AgencyLoop wraps CaseEngine (thin wrapper, agency vocabulary over engine internals)
- **D15:** All refactoring via IntelliJ MCP, no subagents
- **D16:** SPIs support both strings and rich models (Java generics)
- **D17:** Phase 1 gates everything; Phase 2 all worlds build in parallel

## Prior Work Context

Before this restructure, QuarkMind was on branch `issue-224-advisory-milestone-attestations` with three issues ready to implement (#271 SC2 opponent ID, #268 CDI cleanup, #270 engagement outcomes). Those can resume after the restructure — they'll land in the `quarkmind-sc2` module.

## References

- Restructure spec: `specs/quarkmind-restructure/2026-08-14-quarkmind-restructure-design.md`
- Phase 1 plan: `plans/2026-08-14-quarkmind-restructure-phase1.md`
- Current QuarkMind CLAUDE.md: project root
- Current module structure: `MODULES.md` in project root
