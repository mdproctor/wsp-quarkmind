# HANDOFF — quarkmind

## Orientation for Next Session

Saturation curve fixed (#655). 16w from −20.2% to −1.2%, 24w from −13.4% to +1.8%. The remaining structural gap is edge-to-edge walks being ~25% faster than SC2 path-dependent routing (#661). Unconditional approach docking (11.5 loops) compensates but doesn't model the actual path deviation.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current state — post-#655:**

| Baseline | SC2 | EM | Gap |
|----------|-----|-----|-----|
| 12w Protoss | 3460 | 3540 | **+2.3%** |
| 16w saturation | 4235 | 4185 | **−1.2%** |
| 24w saturation | 3955 | 4025 | **+1.8%** |
| Terran | 3885 | 3865 | −0.5% |
| 6w gas minerals | 3375 | 3350 | −0.7% |
| 6w gas vespene | 1012 | 984 | −2.8% |

**Design rules (from CLAUDE.md):**
- Never use compensation constants. Mining efficiency factor = 1.0.
- Constants that represent real animation wait times are legitimate.
- Always use double arithmetic. No integer truncation in the income path.
- Calibrate from controlled SC2 dual-run baselines, not oracle replays.

---

## Last Session

**#655 — 16w saturation curve fix. Three-part fix, code review, blog, 3 follow-up issues filed.**

### #655 — 16w saturation curve (−20.2% → −1.2%)

Three changes fix the saturation curve:

1. **Lockout bypass for waiting workers** — workers at a patch bypass the 22-loop lockout when the active miner departs. SC2 has near-zero handoff time.
2. **Unconditional approach docking (11.5 loops)** — all mining starts pay approach, not just contested ones. Replaces #636's conditional model which over-penalized saturated patches.
3. **Collision speed cap (2-collision max per loop)** — caps compound collision penalty. At 16w, workers with 5+ overlapping neighbours were losing 15-20% speed per loop.

Results: 12w +2.3% (was +0.6%), 16w −1.2% (was −20.2%), 24w +1.8% (was −13.4%).

### Code review

Added dedicated unit test for collision speed cap (24w efficiency >45%). Fixed physics doc — removed duplicate calibration table entry.

### Follow-up issues filed

- **#661** — EM base cycle speed: edge-to-edge walks 25% faster than SC2 paths (L/High)
- **#662** — Reroute threshold ineffective at 2 workers/patch (S/Med)
- **#663** — 3w gas calibration table entry missing from physics doc (XS/Low)

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-655-16w-saturation-curve` | Active | #655 |

---

## What to Work On Next

**Follow-up issues from #655:**

| # | Issue | Title | Scale | Complexity |
|---|-------|-------|-------|------------|
| #661 | EM base cycle speed — edge-to-edge walks 25% faster than SC2 | L | High |
| #662 | Reroute threshold ineffective at 2 workers/patch | S | Med |
| #663 | 3w gas calibration table entry missing | XS | Low |
| #656 | Mineral-line pathfinding | L | High |

**#661 is the structural root cause** of the remaining cycle speed error. Edge-to-edge walking (3.2u for close patches) is ~25% shorter than SC2's path-dependent routing around mineral lines. The unconditional approach docking (11.5 loops) compensates but doesn't model the physical path deviation. Fixing this would make the approach docking unnecessary. #656 (pathfinding) is the implementation path.

### Key findings (updated)

1. **Saturation curve fixed** — 12w +2.3%, 16w −1.2%, 24w +1.8%. Three-part fix: lockout bypass, unconditional approach, collision cap.
2. **Edge-to-edge walks are 25% too fast** — the remaining structural issue. Approach docking compensates. Real fix: mineral-line pathfinding (#656).
3. **Collision compound penalty was the 16w killer** — workers with 5+ overlapping neighbours lost 15-20% speed per loop. Cap at 2 effective collisions fixes it.
4. **Gas entrance queuing has no effect** — 2D sweep proved it (#639, #652). 25.1% vespene MAPE is structural floor.
5. **Physics doc updated** — §6 calibration table, approach docking description, saturation model summary all reflect post-#655 state.
