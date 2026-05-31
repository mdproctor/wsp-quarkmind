# Handover — 2026-05-31

**Head commit (project):** `b921955` — feat(#131): gameTimeSeconds API and time-parameterised mineral income test
**Head commit (workspace):** `e24a270` — archive(issue-131-visualizer-deferred): move plans to attic

## What Changed This Session

- **#131 closed:** All four visualizer deferred sub-tasks shipped. Mineral HUD
  with comma formatting and colour tiers (< 50 critical, 50–149 amber).
  Probe spread — `applyUnitSpread()` distributes co-located sprites in a ring.
  Canvas sprites — `makeResourceMaterial()` factory for geysers and mineral
  patches, replacing solid-colour placeholders. Time-based tests —
  `window.__test.gameTimeSeconds()` + `tickForSeconds()` helper +
  `mineralIncomeScalesWithGameTime` test. Six new Playwright tests. 5 squashed
  commits on project main.
- **1 garden entry:** GE-20260531-b7c259 (Three.js stale pixel after camera
  reposition — `renderer.render()` required before `samplePixel()`).
- **Blog:** 2026-05-31-mdp01-hud-sprites-stale-pixels.md (published).
- **epic-phase-6:** stamped closed at session start.

## Immediate Next Step

Pick up #127 (Phase 5 — EmulatedGame accuracy gaps) or #164 (PlayerState
public API — prerequisite for external race plugins). Run `/work` to start.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #127 | Phase 5 completion — EmulatedGame accuracy gaps | L | High | Main thread; may spawn child issues |
| #164 | PlayerState public API | S | Low | #74 prerequisite; filed this session |
| #155 | Layer 3: casehub-qhorus inter-plugin messaging | L | High | Foundation gate pending |
| #156 | Layer 4: casehub-ledger audit trail | L | High | Foundation gate pending |

## References

| Context | Where |
|---------|-------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| Blog entry (this session) | `blog/2026-05-31-mdp01-hud-sprites-stale-pixels.md` |
| Garden entry | `~/.hortora/garden/tools/GE-20260531-b7c259.md` |
