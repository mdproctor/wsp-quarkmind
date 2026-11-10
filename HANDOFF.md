# HANDOFF — quarkmind

## Last Session

Closed 3 issues (#611, #612, #613). Filed 8 issues (#617-624). Foundational tick-alignment audit identified as top priority — 80% of per-tick mineral deliveries mismatch SC2. CLAUDE.md updated with priority and design rules. Physics constants doc revised throughout.

### Key finding: per-tick delivery cadence mismatch (#618)

Diagnostic showed 242/304 ticks (80%) have mineral delivery mismatches in a controlled 12w economy-only scenario. The efficiency factor (1.006) forces convergence at tick 304 but individual ticks are wrong (±24 minerals oscillation). Workers deliver on different ticks (±1-2 from SC2). SC2 deltas are clean multiples of 5; EM has non-5-multiples from the factor artifact.

**Root cause:** EM worker delivery cadence doesn't match SC2. The stagger model (uniform `i×6` loops) is an approximation of SC2's natural desynchronization. The efficiency factor compensates at aggregate level but 80% of individual ticks are wrong.

**Fix path (issues #621-624):**
1. Instrument SC2 observation for per-worker delivery events (#621) — next
2. Record per-worker SC2 12w delivery baseline (#622)
3. Fix EM delivery timing to match SC2 per-tick cadence (#623)
4. Remove compensation factor, revert to 1.0 (#624)

### Changes this session

1. **Mining efficiency unification (#611):** All three race models set to 1.006 (was Protoss=0.997, Terran/Zerg=0.984). SC2 dual-run shows both Protoss and Terran produce 3460 minerals at tick 304 — no race-specific difference. Old per-race factors were oracle sweep noise.
2. **§6 baseline refresh (#612):** Added gas baselines (3w: −2.9%, 6w: +3.6%), 24w saturation (+5.9%), updated 16w (4235/4519, +6.7%). Table now covers all 7 dual-run baselines.
3. **MULE lifetime (#613):** Calibrated to 1452 loops (64.8s) from SC2 dual-run. Was 1434 loops (64.0s) — 1.3% short. MULE income rate (277.8 min/min) needs cleaner playbook for validation.
4. **CLAUDE.md priority (#617):** Added "Current Priority: Foundational Tick-Alignment" section with design rules (compensation constants forbidden, double arithmetic, dual-run calibration, exact 5-mineral deltas).
5. **Delivery cadence diagnostic (#618):** New test `deliveryCadenceDiagnostic()` in DualRunCalibrationTest comparing per-tick deltas.

### Accuracy summary

**Dual-run (controlled SC2):**

| Baseline | SC2 | EM | Delta |
|----------|-----|-----|-------|
| 12w Protoss (tick 304) | 3460 | 3460 | 0.0% |
| 12w Terran (tick 304) | 3460 | 3460 | 0.0% |
| 16w saturation (tick 304) | 4235 | 4519 | +6.7% |
| 24w saturation (tick 304) | 3955 | 4190 | +5.9% |
| 3w gas (tick 304) | gas 552 | gas 536 | −2.9% |
| 6w gas (tick 304) | gas 1012 | gas 1048 | +3.6% |

**Per-tick accuracy:** 80% of ticks have delivery mismatches (±24 mineral oscillation). This is the foundational issue to fix.

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | 22 commits, active | #578, #605, #603, #602, #606, #604, #609, #598, #608, #610, #611, #612, #613, #618 |

---

## Queue — foundational issues first

```
[ ] #621 — Instrument SC2 observation for per-worker delivery events (S/Med) ← next
[ ] #622 — Record per-worker SC2 12w delivery baseline (S/Low)
[ ] #623 — Fix EM delivery timing to match SC2 per-tick cadence (M/High)
[ ] #624 — Remove mining efficiency compensation factor (S/Low)
[ ] #619 — Worker initialization desynchronization (S/Med)
[ ] #620 — Mining cycle decomposition (M/High)
[ ] #614 — Expansion transfer delay analysis (S/Low) — needs cleaner playbook
[ ] #579 — Technology spending timing (S/Med)
[ ] #580 — foodUsed MAPE (S/Med)
[ ] #581 — Army spending death-timing (S/Med)
[ ] #582 — Economy spending reporting-lag (S/Med)
[ ] #615 — Gas worker allocation modeling (M/Med)
[ ] #616 — SC2 morph via ability API (S/Med)
```

---

## Issue Map

### Epics

| # | Title | Status |
|---|---|---|
| 607 | Standalone economy accuracy | Active — spending issues deferred for foundations |
| 617 | Foundational tick-alignment audit | Active — top priority |

### Open — foundational (priority)

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|---|---|---|---|---|
| 621 | Instrument SC2 observation for per-worker delivery events | S | Med | — | Extend ObservationTranslator for orders/buffs |
| 622 | Record per-worker SC2 12w delivery baseline | S | Low | #621 | Run with instrumentation, save baseline JSON |
| 623 | Fix EM delivery timing to match SC2 per-tick cadence | M | High | #622 | Core fix — make workers deliver at same ticks |
| 624 | Remove mining efficiency compensation factor | S | Low | #623 | Revert factor to 1.0, verify convergence |
| 619 | Worker initialization desynchronization | S | Med | #618 | Match SC2's natural worker desync |
| 620 | Mining cycle decomposition | M | High | #618 | Decompose lumped mining=12 into explicit phases |

### Open — deferred

| # | Title | Scale | Complexity | Notes |
|---|---|---|---|---|
| 614 | Expansion transfer delay analysis | S | Low | Baseline confounded — needs cleaner playbook |
| 579 | Technology spending timing | S | Med | InProgress/Current timing boundary |
| 580 | foodUsed MAPE | S | Med | 6.1% vs target 2-3% |
| 581 | Army spending death-timing | S | Med | 7% MAPE |
| 582 | Economy spending reporting-lag | S | Med | 5.8% MAPE |
| 615 | Gas worker allocation modeling | M | Med | Replace 1.20× override with proper allocation |
| 616 | SC2 morph via ability API | S | Med | Research — why MORPH_ORBITAL_COMMAND rejected |

---

## Key Code Locations

| What | Where |
|------|-------|
| Mining efficiency factor | `ProtossRaceModel.miningEfficiencyFactor()` (all 3 race models return 1.006) |
| MULE lifetime | `SC2Data.MULE_LIFETIME_LOOPS` (1452) |
| Delivery cadence diagnostic | `DualRunCalibrationTest.deliveryCadenceDiagnostic()` |
| ObservationTranslator | `quarkmind-sc2/.../real/ObservationTranslator.java` — needs extension for orders/buffs |
| Domain Unit record | `quarkmind-sc2/.../domain/Unit.java` — no order/cargo fields yet |
| CalibrationSnapshot | `quarkmind-sc2/.../playbook/CalibrationSnapshot.java` — aggregate only |
| SimulatedWorker | `quarkmind-sc2/.../emulated/SimulatedWorker.java` — has rich per-worker state |
| WorkerSimulation | `quarkmind-sc2/.../emulated/WorkerSimulation.java` — tick logic |
| SC2 physics constants | `docs/sc2-physics-constants.md` |
| Priority & design rules | `CLAUDE.md` — "Current Priority: Foundational Tick-Alignment" section |

---

## SC2 API capability (for #621)

The SC2 API provides per-worker data that ObservationTranslator currently ignores:
- `u.getOrders()` — ability ID, target, progress (HARVEST_GATHER = mining, HARVEST_RETURN = carrying)
- `u.getBuffs()` — includes CARRY_MINERAL_FIELD_MINERALS (worker carrying resources)
- `u.getCargoSpaceTaken()` — cargo state
- `u.getAssignedHarvesters()` / `u.getIdealHarvesters()` — town hall/geyser worker counts

The EM's SimulatedWorker has equivalent rich state (WorkerTask enum, position, cargo, assignedPatch, currentSpeed) but this isn't surfaced through GameState or CalibrationSnapshot.

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name.
- **MULE divergence at tick 179** — SC2 baseline has 2 MULEs, EM has 0. MULE calldown in playbooks not working in EM.
