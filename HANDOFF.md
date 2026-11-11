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

Queue triage session. Marked 11 already-closed issues as done in the `.queue` (had fallen out of sync with GitHub). Added #641 to the queue. Triaged all 9 remaining open issues — all still relevant, none resolved by #631 work.

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | Active | #578...#641 |

---

## What to Work On Next

**Recommended order based on dependency analysis:**

1. **#641 — SC2 process not terminated on LOSS** (XS/Low). Quick fix, unrelated to physics. Blocks clean SC2 test runs and prevents 9.6GB zombie processes.

2. **#616 — SC2 morph via ability API** (S/Med). Independent of the cadence chain. Blocks accurate Orbital/PF morph in calibration playbooks.

3. **Cadence variance chain** (the core remaining physics work):
   - **#620 — Mining cycle decomposition** (M/High). Phase infra built but approach is still a fixed constant (11 loops). Make approach state-dependent: zero when patch is idle, longer when queued. EM uncontested cycle is 62 vs SC2 66 loops (4-loop gap).
   - **#636 — Variable approach docking** (M/High). Core of the cadence gap. Fixed approach (7 loops) can't produce SC2's σ≈3+ variance. Per-patch approach code exists but is disabled (APPROACH_DISTANCE_SCALE=0).
   - **#634 — Worker cycle variance gap** (M/High). EM σ≈0.5 vs SC2 σ≈3+. Depends on #636 — variable approach docking would increase variance naturally. Two approaches tried and failed (#628).
   - **#623 — Fix EM delivery timing** (M/High). Umbrella fix for 80% per-tick mismatch. Blocked by #620 and #636.

4. **Independent side tasks** (can be done in any order):
   - **#633 — Per-race mineral income gap** (S/Med). 1.2% gap for Terran/Zerg — likely different worker/building radii per race.
   - **#639 — vespeneCollectionRate structural floor** (S/Med). 35.2% MAPE insensitive to gas constants — structural EMA/concurrent geyser model issue.

5. **#617 — Foundational tick-alignment audit** (L/High). Parent epic for the cadence chain. Close when #620/#636/#634/#623 land.

## Queue

```
[x] #631 — Edge-to-edge walk targets — DONE
[ ] #641 — SC2 process not terminated on LOSS (XS/Low) — quick fix
[ ] #616 — SC2 morph via ability API (S/Med) — independent
[ ] #620 — Mining cycle decomposition (M/High) — approach state-dependency
[ ] #636 — Variable approach docking (M/High) — core cadence gap
[ ] #634 — Worker cycle variance gap (M/High) — depends on #636
[ ] #623 — Fix EM delivery timing (M/High) — blocked by #620 + #636
[ ] #633 — Per-race mineral income gap (S/Med) — independent
[ ] #639 — vespeneCollectionRate structural floor (S/Med) — independent
[ ] #617 — Foundational tick-alignment audit (L/High) — parent epic
```

### Key findings (carried forward)

1. **SC2 mining is fixed-duration, all-or-nothing** — 1.99s occupation time, worker takes min(5, remaining) at the end. No volume-based transfer during animation.
2. **SC2 contested cycles add exactly 1 tick (22 loops)** — the patch lockout. Previous EM value of 34 loops over-penalised contention.
3. **Compensation constants vs wait times** — constants that absorb errors elsewhere are forbidden. Constants that represent real animation durations are legitimate. The test: does the value change when you fix something else?
4. **12w +7.2% is the honest gap** — straight-line walks are faster than SC2's path-dependent navigation. Previous 0.0% was compensating errors cancelling out.
5. **Straight-line movement is the cadence ceiling** — SC2 cycles span 2-17 ticks (σ≈3+), EM spans 4-9 (σ≈0.5). Only dynamic routing will add the variance.
