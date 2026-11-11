# HANDOFF — quarkmind

## Orientation for Next Session

You are working on EmulatedGame physics accuracy — making the economy simulation match real SC2 tick-for-tick. The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go** when you change constants, find stale values, or discover new physics. It is a living document, not a snapshot.

**Current state:** All mining cycle constants are now data-driven from SC2 per-worker observation data (1366 cargo transitions, 12w economy scenario). The constants represent real animation wait times, not swept compensation values.

| Constant | Value | Source |
|----------|-------|--------|
| Mining duration | 22 loops | Position stability at patch (1 tick) |
| Deposit duration | 7 loops | Return phase (40L) minus walk (33L) |
| Patch docking lockout | 22 loops | Contested vs uncontested cycle delta |
| Approach docking | 11 loops | Uncontested mine cycle (66L) minus walk (33L) minus mine (22L) |

**Calibration after #631 edge-to-edge wiring:**

| Baseline | Delta |
|----------|-------|
| 12w Protoss | **+7.2%** |
| 16w saturation | **-2.0%** |
| 24w saturation | -5.4% |
| 3w gas | +2.8% |
| 6w gas | **+0.7%** |
| Vespene (6w) | -2.8% |

**The 12w +7.2% gap is structural.** EM workers walk straight lines; SC2 workers navigate around each other with path-dependent routing. No constant will close this gap — only movement physics changes will. The previous 0.0% was an artifact of compensating errors.

**What was done this session (#631):**
- Wired `depositTarget()` and `miningTarget()` into the three state machine transition sites (previously used center-of-building positions)
- Walk distances halved: 7.0u center-to-center → 3.5u edge-to-edge
- Recalibrated all mining cycle constants from SC2 observation data (not sweeps)
- Ruled out volume-based mineral transfer — SC2 mining is fixed-duration, all-or-nothing (community 1.99s occupation time + observation confirmation)
- SC2 depletion test written (`MiningAnimationMeasurementTest#measureVolumeBasedTransfer`) — needs no-opponent game mode for clean depletion data (#641)

**What to work on next:**
- **#634 — Worker cycle variance gap.** SC2 cycles span 2-17 ticks (σ≈3+), EM spans 4-9 (σ≈0.5). The variance gap is the root cause of per-tick cadence mismatch (~80%). Requires dynamic worker routing — workers choosing patches based on current occupancy, not fixed assignment.
- **#620 — Mining cycle decomposition.** The phase decomposition infrastructure is in place. Next step: make approach duration state-dependent (zero when patch is idle, longer when queued).
- **#641 — SC2 process not terminated on LOSS exit path.** QuarkusSC2Transport doesn't kill SC2 after test runs that end in LOSS. XS fix.

**Design rules (from CLAUDE.md):**
- Never use compensation constants. If the EM doesn't match SC2, fix the root cause.
- Constants that represent real animation wait times (deposit, mining, lockout) are legitimate — they are fixed durations in the SC2 engine.
- Always use double arithmetic. No integer truncation in the income path.
- Calibrate from controlled SC2 dual-run baselines, not oracle replays.

**Key diagnostic tools:**
- `DualRunCalibrationTest` — full 31-test calibration suite (default profile)
- `MiningAnimationMeasurementTest` — SC2 per-frame mining animation measurement (requires `-Psc2`)
- `WorkerSimulationTest#edgeToEdgeMiningDurationSweep` — mining duration sweep against 12w baseline

---

## Last Session

Wired edge-to-edge walk targets (#631). Three-line fix halved walk distances. Instead of sweeping constants, measured real SC2 phase durations from per-worker observation data (1366 transitions). Set all mining cycle constants to observed values. Multi-worker scenarios improved dramatically (16w: +3.4%→-2.0%, 6w gas: +1.9%→+0.7%). 12w regressed from 0.0% to +7.2% — the honest structural gap from straight-line vs path-dependent walking. Ruled out volume-based mineral transfer. Blog written: "The Methods Nobody Called."

### Accuracy

| Baseline | Mineral Δ | Gas Δ |
|----------|-----------|-------|
| 12w Protoss | **+7.2%** | — |
| 16w saturation | **-2.0%** | — |
| 24w saturation | -5.4% | — |
| 3w gas | +2.8% | -8.0% |
| 6w gas | **+0.7%** | -2.8% |

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | Active | #578...#641 |

---

## Queue

```
[x] #631 — Edge-to-edge walk targets (S/Low) — DONE
[ ] #634 — Worker cycle variance gap (S/Low) — next: dynamic routing
[ ] #620 — Mining cycle decomposition — approach state-dependency
[ ] #641 — SC2 process not terminated on LOSS (XS/Low)
[ ] #633 — Per-race mineral income gap
[ ] #623 — Fix EM delivery timing (M/High)
```

### Key findings

1. **SC2 mining is fixed-duration, all-or-nothing** — 1.99s occupation time, worker takes min(5, remaining) at the end. No volume-based transfer during animation.
2. **SC2 contested cycles add exactly 1 tick (22 loops)** — the patch lockout. Previous EM value of 34 loops over-penalised contention.
3. **Compensation constants vs wait times** — constants that absorb errors elsewhere are forbidden. Constants that represent real animation durations are legitimate. The test: does the value change when you fix something else?
4. **12w +7.2% is the honest gap** — straight-line walks are faster than SC2's path-dependent navigation. Previous 0.0% was compensating errors cancelling out.
5. **Straight-line movement is the cadence ceiling** — SC2 cycles span 2-17 ticks (σ≈3+), EM spans 4-9 (σ≈0.5). Only dynamic routing will add the variance.
