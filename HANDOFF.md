# HANDOFF — quarkmind

## Last Session

Fixed #327 — `SelectionUnitLinkTracker` state accumulation causing P2:Marine at 178% of oracle. Two fixes landed together:
- Rewrote tracker with tag-based dedup (`TaggedUnit` record pairing addUnitTags with addSubgroups). Prevents unbounded unitLink accumulation.
- Replaced selection-based multiplication with building-count-based (`productionBuildingCounts`), capped at `MAX_MULTIPLICATION=4`. The old selection-based approach mapped Marine unitLink (70) instead of Barracks (42) — discovery via `BarracksUnitLinkDiscoveryTest` confirmed the correct building unitLinks. The old accuracy was two bugs cancelling out.

Validation: P1:Marine 94.7% (≥90%), P2:Marine 109.7% (80-120%). Landed as `c5df8142` on main.

## Immediate Next Step

Continue epic #318 — next open sub-issue is #328 (synthesize morph-based unit events: Baneling, Ravager, BroodLord, Archon).

## References

- Design spec: `specs/issue-327-selection-tracker-corruption/2026-09-29-selection-tracker-tag-dedup-design.md`
- Implementation plan: `plans/2026-09-29-selection-tracker-tag-dedup.md`
- Diary: `blog/2026-09-29-mdp02-two-bugs-make-a-right.md`
- Building unitLink discovery data: `BarracksUnitLinkDiscoveryTest` / `UnitLinkDiscoveryTest` (diagnostic profile)
