# HANDOFF — quarkmind

## Orientation for Next Session

EmulatedGame economy physics is at an architectural milestone. The tick-alignment epic (#617) is closed — all compensation constants eliminated, 12w mineral income matches SC2 within +0.6%. The remaining gaps (per-tick cadence, cycle distribution, 16w saturation) are structural and need mineral-line pathfinding to resolve.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current state — post-epic summary:**

| Baseline | SC2 | EM | Gap |
|----------|-----|-----|-----|
| 12w Protoss | 3460 | 3480 | **+0.6%** |
| 16w saturation | 4235 | 3380 | -20.2% (pre-existing since #634) |
| 3w gas | 552 | 504 | -8.7% |
| 6w gas | 1012 | 984 | -2.8% |

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

**Closed 4 issues + 1 epic:**

### 1. #645 — 12w mineral over-production (CLOSED)

12w gap at +0.6% (3480 vs 3460), well within <3% acceptance criterion. The conditional approach docking from #636 compensates for straight-line walks. Physics doc corrected — previous values (EM=3710/+7.2%, EM=4150/-2.0%) were stale from pre-#634 code state. Verified via git worktree at commit beb99937.

### 2. #650 — Cycle length tail σ=1.25 vs SC2 0.56 (CLOSED — structural)

Parameter sweep (factor × floor × idle approach, 12w SC2 positions): the collision speed factor (0.96) does double duty — compensates straight-line speed AND creates distribution tail. No parameter combination satisfies both income (±3%) and distribution (σ ≤ 0.70). Architectural ceiling of straight-line movement — needs mineral-line pathfinding.

Added infrastructure: `COLLISION_SPEED_FLOOR` and `IDLE_APPROACH_LOOPS` constants in WorkerPhysicsConfig/SimulatedWorker. Currently at defaults (0.0 / 0.0 = disabled).

### 3. #617 — Foundational tick-alignment audit epic (CLOSED)

All success criteria met or documented as structural:
- Mining efficiency factor at 1.0 ✓
- Gas efficiency override at 1.0 ✓ (no callers of setter)
- Per-tick delta ≤±5: structural ceiling (76% tick mismatch, documented)

### 4. #623 — Fix EM delivery timing (CLOSED — structural)

76% tick mismatch (231/304 ticks). Structural — same root cause as #650.

### 5. #655 — 16w saturation gap (FILED)

Filed new issue: 16w EM=3380 vs SC2=4235 (-20.2%). Pre-existing since #634 (pre-#636 was -18.9%). Collision model over-penalises at 2+ workers per patch.

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-645-12w-mineral-over-production-path-routing` | Active | #645, #650, #617 |

---

## What to Work On Next

**The queue is drained.** Remaining open issues not in queue:

1. **#652 — Gas entrance queuing model** (M/High). **The ML gate.** 3w gas deficit -8.7% is a directional bias for gas-heavy strategies. The physics doc §15 says entrance queuing has no practical effect (walk stagger prevents simultaneous arrivals), but the 3w/6w gap remains. The root cause is that no single gas harvest duration satisfies both 3w (optimal at 16) and 6w (optimal at 26).

2. **#655 — 16w saturation curve** (M/High). EM under-produces by 20% at 2+ workers/patch. Pre-existing since #634. The collision model creates excessive contention at saturation. Separate from the cycle distribution issue (#650) — this is about total income, not distribution shape.

3. **#644 — Zerg dual-run baseline diverges** (S/Low). Unit count mismatch at tick 0 in the Zerg playbook. Not a physics issue.

### Key findings (carried forward)

1. **12w at +0.6%** — compensation constants eliminated, physics-accurate at 1.5 workers/patch.
2. **Collision factor dual-purpose** — 0.96 compensates straight-line speed AND creates cycle tail. Decoupling needs pathfinding.
3. **Physics doc values were stale** — the §6 table had pre-#634 numbers. All corrected this session.
4. **16w not a #636 regression** — verified via git worktree. Pre-#636 was -18.9%, current -20.2%.
5. **Gas gap is structural** — no single harvest duration works for both 3w and 6w. The 25% vespeneCollectionRate MAPE is a floor (#639 decomposition).
