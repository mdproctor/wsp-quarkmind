# HANDOFF — quarkmind

## Orientation for Next Session

You are working on EmulatedGame physics accuracy — making the economy simulation match real SC2 tick-for-tick. The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go** when you change constants, find stale values, or discover new physics. It is a living document, not a snapshot.

**Current state:** All mining cycle constants are data-driven from SC2 per-worker observation data (1366 cargo transitions, 12w economy scenario). Post-#636: conditional approach docking (0 loops idle / 11 loops contested) and wait-then-reroute (22-loop threshold) reduce the 12w mineral surplus from +7.2% to **+3.8%**. The 0-approach for idle patches compensates for straight-line walks being faster than SC2's path-dependent routing — documented in physics doc as an intentional trade-off, not physics-accurate.

| Constant | Value | Source |
|----------|-------|--------|
| Mining duration | 22 loops | Position stability at patch (1 tick) |
| Deposit duration | 7 loops | Return phase (40L) minus walk (33L) |
| Patch docking lockout | 22 loops | Contested vs uncontested cycle delta |
| Approach docking | 0 or 11 loops | 0 if patch idle on arrival, 11 if worker waited (#636) |
| Reroute threshold | 22 loops | Wait before scanning for idle patches (#636) |

**Calibration post-#636:**

| Baseline | Delta |
|----------|-------|
| 12w Protoss | **+3.8%** (was +7.2%) |
| 16w saturation | **-2.0%** |
| 3w gas | -8.7% (gas entrance queuing — see #652) |
| 6w gas | -2.8% |

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

**#636 — Variable approach docking with contention-based re-routing.**

### 1. #636 — Variable approach docking (COMPLETE)

Replaced fixed 11-loop approach/docking with contention-based model:
- **Conditional approach:** 0 loops when patch is idle on arrival, 11 loops when worker waited at occupied patch
- **Wait-then-reroute:** workers waiting >22 loops scan for idle patches, reroute if travel time < remaining wait
- 12w mineral surplus improved from +7.2% to +3.8% (3590 EM vs 3460 SC2)
- 22 reroute events per 304 ticks in 12w diagnostic
- 3361 tests pass, zero regressions

### 2. Code review findings fixed

Unused import removed, diagnostic cycle measurement corrected (was double-counting deliveries), physics constants doc §3 and §6 updated for post-#636 numbers.

### 3. Issues filed: #652 (gas queuing), #653 (dead APPROACH_DISTANCE_SCALE), #654 (waitingSinceLoop reset)

### 4. Physics constants doc — comprehensive update

§3 table: approach row annotated conditional, reroute threshold added. SC2 observed cycle table: EM column updated (55L uncontested). Conditional approach entry: clarified that 0-approach compensates for straight-line walks. §6 dual-run table: marked pre-#636, annotated post-#636 improvement.

### 5. Blog entry

- **"The Compensation That Isn't"** — honest account of why the improvement works for the wrong reason, and the ML training readiness assessment

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | Active | #578...#641 |

---

## What to Work On Next

**Recommended order (from .queue):**

1. **#645 — 12w mineral over-production path routing** (M/High). Remaining +3.8% gap from straight-line walks vs SC2 path-dependent routing. May need mineral-line navmesh.

2. **#650 — Cycle length tail EM σ=0.90 vs SC2 σ=0.58** (M/High). 7+ tick collision stacking tail. Contention model may have partially addressed this (fewer walking workers = fewer collisions). Re-measure.

3. **#652 — Gas entrance queuing model** (M/High). **The ML gate.** 3w gas deficit −8.7% is a directional bias for gas-heavy strategies. Fix the concurrent geyser access model.

4. **#653 — Remove dead APPROACH_DISTANCE_SCALE** (XS/Low). Cleanup.

5. **#654 — Reset waitingSinceLoop on task transitions** (XS/Low). Cleanup.

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
