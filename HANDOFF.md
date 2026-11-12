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

**Major session — #639 fix + #578 branch landing + physics doc revision.**

### 1. #639 — vespeneCollectionRate structural floor (CLOSED)

VespeneRateDecompositionTest decomposes 30% rate MAPE into income vs algorithm components. Root cause: GAS_PER_WORKER_PER_MIN=54 too high (actual ~34). Recalibrated → rate MAPE 30.3%→25.1%. Remaining 25% is structural floor.

### 2. #578 branch landed (28 squashed commits from 79)

The full economy calibration branch (issues #578–#641, 38 issues) landed on main. Includes edge-to-edge walks, collision physics, SC2-observed mining constants, worker stagger, gas entrance queuing, MULE lifetime calibration, and more. Squashed from 79→28 commits (chronological, no reorder). 5 test fixes applied post-landing.

### 3. Issues closed: #639, #611, #620, #633 (4 total)
### 4. Issues filed: #644 (Zerg baseline), #645 (12w over-production), #646 (6w gas verification)

### 5. Physics constants doc — comprehensive revision

Every numeric value cross-referenced against code. Fixed: §3 (approach 7→11, docking 34→22, added deposit/stagger/collision), §6 (fresh dual-run table), §14 (full cycle rewrite), §16 (vespene 35→25.1%), §19 (config block aligned).

### 6. Blog entries

- **"How Wrong Is Your Simulation?"** — revised capstone covering the full calibration arc
- **"The Constant That Explained Everything"** — #639 decomposition methodology

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | Active | #578...#641 |

---

## What to Work On Next

**Recommended order:**

1. **Quick verification tasks:**
   - **#644 — Zerg dual-run baseline diverges** (S/Low). Unit count mismatch at tick 0. Fix playbook or EM Zerg initialization.
   - **#646 — 6w gas baseline not verified** (XS/Low). Run the test, record result, update §6 table.

2. **MULE income validation:**
   - **#613 — MULE income rate validation** (S/Med). Lifetime done (1452 loops). Income rate (277.8 min/min) NOT validated — needs cleaner playbook (requires SC2).

3. **Cadence variance chain** (core remaining physics):
   - **#623 — Fix EM delivery timing** (M/High). Cadence at 75.3%. Needs path-dependent routing.
   - **#636 — Variable approach docking** (M/High). Infrastructure ready (SCALE=0), waiting for collision to produce approach naturally.
   - **#634 — Worker cycle variance gap** (M/High). EM σ≈0.5 vs SC2 σ≈3+.
   - **#645 — 12w mineral over-production +7.2%** (M/High). Straight-line vs path-dependent routing ceiling.

4. **#617 — Tick-alignment audit (Epic)** (L/High). Close when #623/#634/#636 resolve.

## Queue

```
[x] #639 — vespeneCollectionRate structural floor — DONE
[x] #611 — Mining efficiency recalibration — superseded by #623
[x] #620 — Mining cycle decomposition — DONE (mine=22/deposit=7/approach=11)
[x] #633 — Per-race mineral income gap — finding: race-invariant, map geometry
[ ] #644 — Zerg dual-run baseline diverges (S/Low)
[ ] #646 — 6w gas baseline not verified (XS/Low)
[ ] #613 — MULE income rate validation (S/Med) — needs SC2
[ ] #623 — Fix EM delivery timing (M/High) — cadence 75.3%
[ ] #636 — Variable approach docking (M/High)
[ ] #634 — Worker cycle variance gap (M/High)
[ ] #645 — 12w mineral over-production +7.2% (M/High)
[ ] #617 — Tick-alignment audit epic (L/High)
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
