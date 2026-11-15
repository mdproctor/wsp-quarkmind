# HANDOFF — quarkmind

## Orientation for Next Session

Gas building accuracy fixed (#671). Root cause was not in TrackerIntentExtractor (which was 100% accurate) but in WorkerSimulation's position-based gas slot dedup — geysers at 4.0 units apart on some maps were rejected by a `distance < 5.0` check. Tag-based dedup restored 100% accuracy. Gas rate MAPE dropped from ~30% to 27%. Remaining gap is per-worker gas cycle timing (#675).

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current state — post-#671:**

| Metric | Before #671 | After #671 |
|--------|-------------|------------|
| Gas building count accuracy (5-min) | 86% (144/168) | **100% (168/168)** |
| Gas rate MAPE (5-min) | ~30% | **27.0%** |
| Gas rate MAPE Protoss | — | 27.2% |
| Gas rate MAPE Terran | — | 24.0% |
| Gas rate MAPE Zerg | — | 31.8% |
| Full test suite | 3362 pass | 3362 pass |

**Design rules (from CLAUDE.md):**
- Never use compensation constants. Mining efficiency factor = 1.0.
- Constants that represent real animation wait times are legitimate.
- Always use double arithmetic. No integer truncation in the income path.
- Calibrate from controlled SC2 dual-run baselines, not oracle replays.

---

## Last Session

**Branch:** `issue-671-gas-building-extraction` (4 commits, ready for work-end)

**What was done:**
- Diagnostic proving TrackerIntentExtractor extracts 100% of gas building events (479/479, zero mismatches)
- Pipeline trace identifying the exact gap: buildings created and completed, but gas slots not registered
- Inter-geyser distance analysis: 8.9% of geyser pairs < 5.0u (dominant cluster at exactly 4.0u)
- Fix: `WorkerSimulation.onGasBuildingCompleted` — position-based dedup (`distance < 5.0`) → tag-based dedup (`buildingTag.equals(tag)`)
- Updated `docs/sc2-physics-constants.md` §4 with inter-geyser distance data
- Blog entry: "The Geysers Were Too Close"

**Key finding:** SC2 maps place geysers as close as 4 units apart at the same base. The position-based duplicate check (introduced to prevent double-registering the same geyser) was catching these distinct geysers as false duplicates. The old check was correct in spirit but wrong in threshold — tag-based dedup eliminates the false positive entirely.

**Issues filed this session:**
- #675 Gas rate MAPE at 5-min still above 25% target (open — per-worker gas cycle timing)

---

## Queue State

**Epic:** #607 (Standalone economy accuracy)
**Completed this session:** #671

Open queue items:
1. **#675** — Gas rate MAPE above 25% target (S/Med) ← next
2. **#672** — Gas walk arrival-threshold compensation (S/Med)
3. **#673** — Dual-run 6w gas surplus (XS/Low)
4. #656 — Mineral-line pathfinding (open)
5. #661 — EM base cycle speed edge-to-edge walks (open)
6. #666 — Forward simulation economy mode (open)

---

## Resumption Notes

- Branch `issue-671-gas-building-extraction` ready for `work end`
- CLAUDE.md updated with new diagnostic test listing
- `docs/sc2-physics-constants.md` §4 updated with inter-geyser distance data
- For #675: analyse per-worker gas income rates (GT vs EM rate per gas worker) to quantify per-cycle error, calibrate gas harvesting constants against dual-run baselines
