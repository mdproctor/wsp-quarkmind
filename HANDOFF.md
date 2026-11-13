# HANDOFF — quarkmind

## Orientation for Next Session

EmulatedGame economy physics is at an architectural milestone. The tick-alignment epic (#617) is closed — all compensation constants eliminated, 12w mineral income matches SC2 within +0.6%. The remaining gaps (per-tick cadence, cycle distribution, 16w saturation) are structural and need mineral-line pathfinding to resolve.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current state — post-epic summary:**

| Baseline | SC2 | EM | Gap |
|----------|-----|-----|-----|
| 12w Protoss | 3460 | 3480 | **+0.6%** |
| 16w saturation | 4235 | 3380 | -20.2% (pre-existing since #634) |
| 3w gas minerals | 3885 | 3385 | -12.9% |
| 3w gas vespene | 552 | 504 | -8.7% |
| 6w gas minerals | 3375 | 2815 | -16.6% |
| 6w gas vespene | 1012 | 984 | -2.8% |

**Cycle distribution (12w, SC2 positions):**

| Metric | SC2 | EM |
|--------|-----|-----|
| σ | 0.56 | 1.25 |
| Mean | 5.28 | 5.23 |
| 7+ tick cycles | 0% | 12.9% |

**Design rules (from CLAUDE.md):**
- Never use compensation constants. Mining efficiency factor = 1.0.
- Constants that represent real animation wait times are legitimate.
- Always use double arithmetic. No integer truncation in the income path.
- Calibrate from controlled SC2 dual-run baselines, not oracle replays.

---

## Last Session

**Closed 4 issues + 1 epic. Code review. Blog. Filed 6 new issues.**

### 1. #645 — 12w mineral over-production (CLOSED)

12w gap at +0.6% (3480 vs 3460), well within <3% acceptance criterion. The conditional approach docking from #636 compensates for straight-line walks. Physics doc corrected — previous values (EM=3710/+7.2%, EM=4150/-2.0%) were stale from pre-#634 code state. Verified via git worktree at commit beb99937.

### 2. #650 — Cycle length tail σ=1.25 vs SC2 0.56 (CLOSED — structural)

Parameter sweep (factor × floor × idle approach, 12w SC2 positions): the collision speed factor (0.96) does double duty — compensates straight-line speed AND creates distribution tail. No parameter combination satisfies both income (±3%) and distribution (σ ≤ 0.70). Architectural ceiling of straight-line movement — needs mineral-line pathfinding (#656).

Added infrastructure: `COLLISION_SPEED_FLOOR` and `IDLE_APPROACH_LOOPS` constants in WorkerPhysicsConfig/SimulatedWorker. Currently at defaults (0.0 / 0.0 = disabled). Kept for future use when pathfinding decouples income from distribution.

### 3. #617 — Foundational tick-alignment audit epic (CLOSED)

All success criteria met or documented as structural:
- Mining efficiency factor at 1.0 ✓
- Gas efficiency override at 1.0 ✓ (no callers of setter)
- Per-tick delta ≤±5: structural ceiling (76% tick mismatch, documented)

### 4. #623 — Fix EM delivery timing (CLOSED — structural)

76% tick mismatch (231/304 ticks). Structural — same root cause as #650.

### 5. Code review fixes

- Physics doc §6: removed stale "16w -2.0%" paragraph that contradicted new -20.2% text
- Updated 3w/6w gas gap descriptions with current mineral deficit numbers
- MiningCycleDiagnosticTest: removed unused `savedFactor` variable
- WorkerCadenceComparisonTest: wrapped sweep in try/finally for static field cleanup

### 6. New issues filed

| Issue | Title | Scale |
|-------|-------|-------|
| #655 | 16w saturation curve -20.2% | M/High |
| #656 | Mineral-line pathfinding | L/High |
| #657 | Fix diagnostic test cycle measurement | XS/Low |
| #658 | Physics doc stale values audit | XS/Low |
| #659 | 3w gas mineral deficit -12.9% | S/Med |
| #660 | Clean up untracked spec files | XS/Low |

### 7. Blog

"The Constant That Does Two Jobs" — why one collision constant can't fix both income accuracy and cycle distribution. SVG trade-off diagram.

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-645-12w-mineral-over-production-path-routing` | Active | #645, #650, #617, #650 review |

---

## What to Work On Next

**Queue has 10 items scheduled (cleanup → investigation → core gaps → structural):**

| # | Issue | Title | Scale |
|---|-------|-------|-------|
| 1 | #653 | Remove dead APPROACH_DISTANCE_SCALE | XS/Low |
| 2 | #654 | Reset waitingSinceLoop on transitions | XS/Low |
| 3 | #657 | Fix diagnostic test cycle measurement | XS/Low |
| 4 | #658 | Physics doc stale values audit | XS/Low |
| 5 | #660 | Clean up untracked spec files | XS/Low |
| 6 | #659 | 3w gas mineral deficit investigation | S/Med |
| 7 | #652 | Gas entrance queuing model | M/High |
| 8 | #655 | 16w saturation curve | M/High |
| 9 | #656 | Mineral-line pathfinding | L/High |

**Recommended:** Start with the XS cleanup batch (#653, #654, #657, #658, #660) — five quick wins in one session. Then #659 and #652 together (both gas-related). #655 and #656 are the structural work.

### Key findings (carried forward)

1. **12w at +0.6%** — compensation constants eliminated, physics-accurate at 1.5 workers/patch.
2. **Collision factor dual-purpose** — 0.96 compensates straight-line speed AND creates cycle tail. Decoupling needs pathfinding (#656).
3. **Physics doc values were stale** — the §6 table had pre-#634 numbers. Corrected. §3 still has stale σ values (#658).
4. **16w not a #636 regression** — verified via git worktree. Pre-#636 was -18.9%, current -20.2%.
5. **Gas gap is structural** — no single harvest duration works for both 3w and 6w. The 25% vespeneCollectionRate MAPE is a floor.
6. **3w/6w mineral deficit newly quantified** — 3w minerals at -12.9%, 6w at -16.6%. Much worse than gas deficits. May be from gas worker transfer timing (#659).
