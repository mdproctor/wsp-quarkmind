# HANDOFF — quarkmind

## Orientation for Next Session

You are working on EmulatedGame physics accuracy — making the economy simulation match real SC2 tick-for-tick. The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go** when you change constants, find stale values, or discover new physics. It is a living document, not a snapshot.

**Current state:** All mining cycle constants are now data-driven from SC2 per-worker observation data (1366 cargo transitions, 12w economy scenario). The constants represent real animation wait times, not swept compensation values. The 12w +7.2% gap is structural — straight-line walks vs SC2's path-dependent routing. No constant will close it.

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

**#639 — vespeneCollectionRate structural floor diagnosis and fix.**

Wrote `VespeneRateDecompositionTest` diagnostic to decompose the 30% vespene rate MAPE into three independent components:
- **Cumulative income error** — does EM mine the right total gas?
- **Rate algorithm error** — does EMA misrepresent the income?
- **Worker count estimation** — are we assigning the right number of gas workers?

**Key finding:** Rate(EMA) and Rate(SimpleWindow) produce identical MAPE at 7-min (30.3% each). The EMA algorithm (fixed three days ago) is correct but is not an accuracy lever — the rate tracks the income, not the smoothing method.

**Root cause:** `GAS_PER_WORKER_PER_MIN=54` was too high. EM's actual per-worker gas rate is ~34 gas/min. Dividing GT rate by 54 systematically under-estimated worker count, causing -114 gas bias per replay. Sweep across 118 oracle replays confirmed 34 is optimal (income MAPE floors at 15.2% in the 30-44 range).

**Fix:** Recalibrated GAS_PER_WORKER_PER_MIN from 54 to 34 in all 6 test harness files. Rate MAPE dropped from 30.3% to 25.1%. Remaining 25% is structural floor from per-worker gas harvest physics approximation, integer worker count rounding, and PlayerStatsEvent cadence mismatch.

**Docs updated:** `economy-structural-floor.md` §10, `sc2-physics-constants.md` §4/§16/§18, CLAUDE.md diagnostic test list.

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | Active | #578...#641 |

---

## What to Work On Next

**Recommended order based on dependency analysis:**

1. **#616 — SC2 morph via ability API** (S/Med). Independent of the cadence chain. Blocks accurate Orbital/PF morph in calibration playbooks. Needs SC2 for investigation.

2. **Cadence variance chain** (the core remaining physics work):
   - **#620 — Mining cycle decomposition** (M/High). Phase infra built but approach is still a fixed constant (11 loops). Make approach state-dependent: zero when patch is idle, longer when queued. EM uncontested cycle is 62 vs SC2 66 loops (4-loop gap).
   - **#636 — Variable approach docking** (M/High). Core of the cadence gap. Fixed approach (7 loops) can't produce SC2's σ≈3+ variance. Per-patch approach code exists but is disabled (APPROACH_DISTANCE_SCALE=0).
   - **#634 — Worker cycle variance gap** (M/High). EM σ≈0.5 vs SC2 σ≈3+. Depends on #636 — variable approach docking would increase variance naturally.
   - **#623 — Fix EM delivery timing** (M/High). Umbrella fix for 80% per-tick mismatch. Blocked by #620 and #636.

3. **Independent side tasks** (can be done in any order):
   - **#633 — Per-race mineral income gap** (S/Med). 1.2% gap for Terran/Zerg. EM physics are correctly race-invariant — gap likely from map geometry per starting position. Needs SC2 calibration testing.
   - ~~**#639 — vespeneCollectionRate structural floor** (S/Med).~~ DONE — 30.3%→25.1% MAPE. Remaining 25% is structural floor (gas harvest physics, integer rounding, cadence mismatch).

4. **#617 — Foundational tick-alignment audit** (L/High). Parent epic for the cadence chain. Close when #620/#636/#634/#623 land.

## Queue

```
[x] #631 — Edge-to-edge walk targets — DONE
[x] #641 — SC2 process not terminated on LOSS — DONE (this session)
[ ] #616 — SC2 morph via ability API (S/Med) — needs SC2
[ ] #620 — Mining cycle decomposition (M/High) — approach state-dependency
[ ] #636 — Variable approach docking (M/High) — core cadence gap
[ ] #634 — Worker cycle variance gap (M/High) — depends on #636
[ ] #623 — Fix EM delivery timing (M/High) — blocked by #620 + #636
[ ] #633 — Per-race mineral income gap (S/Med) — needs SC2 calibration
[x] #639 — vespeneCollectionRate structural floor (S/Med) — DONE: 54→34, 30%→25% MAPE
[ ] #617 — Foundational tick-alignment audit (L/High) — parent epic
```

### Key findings (carried forward + new)

1. **SC2 mining is fixed-duration, all-or-nothing** — 1.99s occupation time, worker takes min(5, remaining) at the end.
2. **SC2 contested cycles add exactly 1 tick (22 loops)** — the patch lockout.
3. **Compensation constants vs wait times** — constants that absorb errors elsewhere are forbidden. Constants that represent real animation durations are legitimate.
4. **12w +7.2% is the honest gap** — straight-line walks are faster than SC2's path-dependent navigation.
5. **Straight-line movement is the cadence ceiling** — SC2 cycles span 2-17 ticks (σ≈3+), EM spans 4-9 (σ≈0.5).
6. **Worker physics are race-invariant** — Probe/SCV/Drone all use 3.94 u/s speed, 0.375 collision radius. Nexus/CC/Hatchery all 2.5 radius. Per-race mineral gap (1.2%) is not from unit physics.

### Physics constants doc discrepancy

Section 6 of `docs/sc2-physics-constants.md` says "SC2 dual-run shows Protoss and Terran produce identical minerals (3460)" but the calibration table in the same section shows Terran at 3955/-19.5%. One of these values is stale — verify in next session.
