# HANDOFF — quarkmind

## Orientation for Next Session

Saturation curve fixed (#655). Post-#655 hygiene complete — 10 issues closed across two batches. The remaining open work is M/High+ economy physics: gas queuing (#652), gas rate floor (#665), pathfinding (#656), cycle speed (#661), forward simulation (#666).

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current state — post-#655 + hygiene batch:**

| Baseline | SC2 | EM | Gap |
|----------|-----|-----|-----|
| 12w Protoss | 3460 | 3540 | **+2.3%** |
| 16w saturation | 4235 | 4185 | **−1.2%** |
| 24w saturation | 3955 | 4025 | **+1.8%** |
| 3w gas minerals | 3885 | 3865 | −0.5% |
| 3w gas vespene | 552 | 504 | −8.7% |
| 6w gas minerals | 3375 | 3350 | −0.7% |
| 6w gas vespene | 1012 | 984 | −2.8% |

**Design rules (from CLAUDE.md):**
- Never use compensation constants. Mining efficiency factor = 1.0.
- Constants that represent real animation wait times are legitimate.
- Always use double arithmetic. No integer truncation in the income path.
- Calibrate from controlled SC2 dual-run baselines, not oracle replays.

---

## Last Session

**10 hygiene issues closed across two batches. Blog revised. No issues remaining below M.**

### Issues closed (this session)

| # | Title | Type |
|---|-------|------|
| #664 | Regenerate standalone accuracy baseline post-#655 | Baseline refresh |
| #653 | Remove dead APPROACH_DISTANCE_SCALE from MineralPatch | Dead code removal |
| #654 | Reset waitingSinceLoop on worker task transitions | Bug fix |
| #657 | Fix diagnostic test cycle measurement — track task transitions | Bug fix |
| #658 | Physics doc stale values audit §3 | Doc fix |
| #660 | Clean up untracked spec files from #572 | Housekeeping |
| #663 | 3w gas calibration table entry — fix mislabeled row | Doc fix |
| #662 | Reroute threshold at 2 workers/patch | Investigation (no code change needed) |
| #659 | 3w gas mineral deficit — already fixed by #655 | Verification close |
| #667 | Reset waitingSinceLoop on expansion transfer | Bug fix (code review finding from #654) |

### Blog

Revised "The Bug That Wasn't There" — added #655 resolution section showing how three targeted physics changes fixed the saturation curve without pathfinding. New before/after SVG.

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-664-regenerate-standalone-baseline` | Closed | #664, #653, #654, #657, #658, #660, #663, #662, #659 |
| `issue-667-waitingSinceLoop-transfer` | Closed | #667 |

---

## What to Work On Next

**Remaining open issues by priority:**

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #652 | Gas entrance queuing model | M | High |
| #665 | Gas rate structural floor — 33.8% MAPE | M | Med |
| #656 | Mineral-line pathfinding | L | High |
| #661 | EM base cycle speed — edge-to-edge walks 25% faster | L | High |
| #666 | Forward simulation economy mode | L | High |

**#652 is the next issue** — gas entrance queuing model. Previous investigation (#639) showed 25.1% vespene MAPE is a structural floor from single harvest duration.

### Key findings (current)

1. **Saturation curve fixed** — 12w +2.3%, 16w −1.2%, 24w +1.8%.
2. **Edge-to-edge walks 25% too fast** — remaining structural issue. Approach docking compensates. Real fix: mineral-line pathfinding (#656).
3. **EM cycle σ=1.56 vs SC2 σ=0.56** — regressed from σ=0.90 post-#655 due to unconditional approach. Root cause is straight-line walks (#661).
4. **Standalone mineral rate MAPE improved** — 17.7%→16.9% at 5-min, 27.4%→24.5% at 7-min.
5. **mineralsCurrent MAPE increased** (71.2→82.2% at 5-min) — expected consequence of removing compensation. Cumulative balance diverges because per-tick delivery cadence doesn't match SC2.
