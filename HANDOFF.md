# HANDOFF — quarkmind

## Orientation for Next Session

EmulatedGame economy physics is at an architectural milestone. The tick-alignment epic (#617) is closed — all compensation constants eliminated, 12w mineral income matches SC2 within +0.6%. The remaining gaps (per-tick cadence, cycle distribution, 16w saturation) are structural and need mineral-line pathfinding to resolve.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current state — post-cleanup summary:**

| Baseline | SC2 | EM | Gap |
|----------|-----|-----|-----|
| 12w Protoss | 3460 | 3480 | **+0.6%** |
| 16w saturation | 4235 | 3380 | -20.2% (structural) |
| 3w gas minerals | 3885 | 3385 | -12.9% (saturation — see below) |
| 3w gas vespene | 552 | 504 | -8.7% (structural floor) |
| 6w gas minerals | 3375 | 2815 | -16.6% (saturation) |
| 6w gas vespene | 1012 | 984 | -2.8% |

**Design rules (from CLAUDE.md):**
- Never use compensation constants. Mining efficiency factor = 1.0.
- Constants that represent real animation wait times are legitimate.
- Always use double arithmetic. No integer truncation in the income path.
- Calibrate from controlled SC2 dual-run baselines, not oracle replays.

---

## Last Session

**Closed 7 issues. Code review fix. Blog. Key diagnostic finding on gas mineral deficits.**

### 1. XS cleanup batch (5 issues)

- **#653** — Removed dead `APPROACH_DISTANCE_SCALE` field, dead formula branch in `MineralPatch.computeApproachDocking()`, and two sweep tests (`cadenceWithDistanceScale12w`, `sweepApproachDistanceScale12w`)
- **#654** — Reset `waitingSinceLoop=0` in `pauseWorkerForBuilding()`, `onWorkerScouting()`, both gas worker transfer paths, and TRANSFERRING transition (code review catch). Prevents stale wait times triggering immediate reroutes
- **#657** — Fixed `ContentionCalibrationDiagnosticTest` cycle measurement. Was attributing deliveries to all workers with `MINING && cargo==0`; now tracks per-worker `MINING→RETURNING` transitions
- **#658** — Physics doc stale values audit: fixed 10 stale values across §1b, §2, §3, §14, §19. SC2 σ 0.58→0.56, EM σ 0.90→1.25, +3.8%→+0.6%, per-tick cadence 75.3%→76%
- **#660** — Committed two untracked design specs from #572 worker activity modelling

### 2. #659 — 3w gas mineral deficit (CLOSED — diagnostic)

**The 3w/6w gas mineral deficits are the saturation curve problem, not gas worker transfer timing.** Both playbooks train probes continuously, reaching 23 probes (supply-capped). With 3 on gas, 20 mine minerals at 2.5 workers/patch — where the collision model over-penalises. The 12w baseline never trains probes (stays at 12, 1.5/patch), which is why it's accurate.

Evidence: SC2 gains +425 minerals from 3w gas (extra probes offset gas workers). EM loses -95. The 520-mineral swing is from saturation, not gas mechanics.

### 3. Code review

Code review caught a missed `waitingSinceLoop` reset on the TRANSFERRING transition (expansion worker transfer). Same pattern as the scouting/building/gas fixes — fixed in a follow-up commit.

### 4. Blog

"The Bug That Wasn't There" — how the 3w gas mineral deficit looked like a gas worker transfer bug but turned out to be the saturation curve. SVG diagram mapping all three deficits onto one accuracy curve.

### 5. #652 — Gas entrance queuing (CLOSED — already investigated)

The 2D sweep in #639 proved entrance queuing has no practical effect. Infrastructure exists (`GasSlot.ENTRY_LOCK_DURATION`). The 3w gas vespene gap (-8.7%) is from per-geyser walk-distance rate interaction — no single harvest duration works for both 3w and 6w.

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-645-12w-mineral-over-production-path-routing` | Active | #653, #654, #657, #658, #659, #660, #652 |

---

## What to Work On Next

**Queue has 2 items remaining:**

| # | Issue | Title | Scale |
|---|-------|-------|-------|
| 1 | #655 | 16w saturation curve -20.2% | M/High |
| 2 | #656 | Mineral-line pathfinding | L/High |

**#655 is the single structural issue.** It's the root cause of the 16w (-20.2%), 3w gas mineral (-12.9%), and 6w gas mineral (-16.6%) deficits. The collision model (speed factor 0.96, collision radius 0.75) and patch docking overhead (22 loops) create excessive contention at >2 workers per patch. Fixing this unblocks #656 (pathfinding) — which decouples income accuracy from cycle distribution accuracy.

### Key findings (carried forward)

1. **12w at +0.6%** — compensation constants eliminated, physics-accurate at 1.5 workers/patch.
2. **Collision factor dual-purpose** — 0.96 compensates straight-line speed AND creates cycle tail. Decoupling needs pathfinding (#656).
3. **3w/6w gas mineral deficit = saturation** — not gas worker transfer. Playbooks train 23 probes → 2.5 workers/patch (confirmed #659).
4. **Gas entrance queuing has no effect** — 2D sweep proved it (#639, #652). 25.1% vespene MAPE is structural floor.
5. **16w not a #636 regression** — verified via git worktree. Pre-#636 was -18.9%, current -20.2%.
6. **Physics doc fully audited** — all stale values corrected (#658). SC2 σ=0.56, EM σ=1.25.
