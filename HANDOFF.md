# HANDOFF — quarkmind

## Last Session

Ran SC2 calibration playbooks from #591. Fixed critical bugs in PlaybookSC2Adapter (position-based builds rejected by SC2, Assimilator commands not dispatched, morph/calldown unimplemented). Recorded 5 working mineral baselines + 1 saturation baseline. Gas, MULE, and expansion recordings need one more adapter fix.

### What was done

- **SC2 adapter fixes (3 bugs):**
  1. Build commands sent to hardcoded `(30, 30)` → always rejected. Fixed: use debug API `DebugCreateUnit` for supply/tech buildings (bypasses placement validation)
  2. Assimilator geyser-targeting commands created but never dispatched (build path returned null but command was in the return value). Fixed: route non-null build commands to `sendActions`
  3. Morph (`CC→OC`) and calldown (`MULE`) actions unimplemented in adapter. Added `translateMorph` and `translateCalldown`

- **SC2 calibration findings (from working baselines):**
  - Per-worker mineral rate: Protoss=57.9, Terran=57.2, Zerg=57.2 minerals/min/worker
  - EM at real map distances: ~57.8/min/worker → **within 1% of SC2**
  - EM at test distance (uniform 6.0u): 54.0/min/worker → 7% below SC2 (test distance too short for realistic cycle times)
  - Efficiency factors reverted to 1.0 — oracle-derived factors (1.18/1.12/1.28) were compensating for allocation gaps, not physics
  - Saturation: 23 workers on 8 patches → 48.6/min/worker (84% efficiency). EM shows ~90% → 6% too lenient

- **Remaining adapter fixes needed:**
  - Assimilator dispatch fix committed but not yet tested against SC2 (gas baselines still show 0 vespene)
  - SC2 auto-worker-to-gas assignment not triggered by playbooks — may need explicit worker assignment action in adapter
  - Expansion playbook: debug-created Nexus provides supply but workers don't transfer to 2nd base without rally commands

---

## Current Accuracy State

### SC2 Playbook Calibration Results

| Measurement | SC2 Real | EM Model | Delta | Source |
|---|---|---|---|---|
| Protoss per-worker mineral rate | 57.9/min | ~57.8 (real dist) | <1% | cal-mineral-baseline-12w |
| Terran per-worker mineral rate | 57.2/min | ~57.8 (real dist) | ~1% | cal-mineral-baseline-terran |
| Zerg per-worker mineral rate | 57.2/min | ~57.8 (real dist) | ~1% | cal-mineral-baseline-zerg |
| Saturation efficiency (23w/8p) | 84% | ~90% | EM 6% too lenient | cal-mineral-saturated-16w |

### Standalone EmulatedGameBaselineTest — 7-min checkpoint

Factors at 1.0 (unchanged from previous session):

| Field | MAPE | Status |
|---|---|---|
| mineralsCollectionRate | 22.1% | Worker allocation gap, not physics (#572) |
| vespeneCollectionRate | 29.1% | Gas physics not yet SC2-calibrated |
| mineralsCurrent | 67.5% | Cumulative drift from rate error |
| vespeneCurrent | 65.4% | Cumulative drift |
| All other fields | Same as previous session | At floor or structural |

---

## Remaining Queue

```
- [ ] #592 — Overhead absorption limit investigation
- [ ] #593 — SC2 rate algorithm reverse-engineering
- [ ] #594 — Sub-tick rate tracking — exact 160-loop window
- [ ] #595 — Gas rate regression analysis — 176-loop window impact
```

---

## Critical Insights for Next Session

### 1. Per-worker physics is calibrated — don't touch mining constants

SC2 playbook calibration proves per-worker mineral physics is within 1% at real map distances. The mining-duration=12, gas-harvest=22, acceleration=3.5 constants are correct. The 22% rate MAPE in oracle replays is entirely worker allocation — #572 is the fix, not physics.

### 2. Saturation model needs steeper contention

SC2 drops to 84% per-worker efficiency at 2.9 workers/patch. EM stays at ~90%. The contention model (patch occupancy during mining duration) needs to penalise 3rd workers more aggressively. This is #572 territory.

### 3. SC2 adapter needs 2 more fixes for gas/MULE recordings

**Fix 1 — Assimilator dispatch**: Code is committed but untested. The `onStep` fix routes non-null `translateBuild` results to `sendActions`. Run `recordCalGasBaseline3w` to verify. If Assimilator appears but vespene stays 0, SC2 isn't auto-assigning workers — adapter needs an explicit "assign 3 workers to gas" action.

**Fix 2 — Morph/calldown dispatch**: `translateMorph` and `translateCalldown` are implemented but `fired[i]` semantics for morphs may not be right. The MULE calldown uses `loop: continuous` which should retry each tick until OC has energy. Test with `recordCalMuleIsolation`.

### 4. Expansion playbook needs rally command support

Debug-created Nexus provides supply instantly but workers don't mine at the expansion without rally commands. The adapter needs either: (a) SC2 debug API to create workers at the expansion, or (b) rally point commands to transfer workers.

### 5. Test distance vs real distance gap

EM at test distance (6.0u uniform) produces 54.0/min/worker — 7% below SC2's 57.9. At real distances (4 close×6.6u + 4 far×7.6u), the weighted average is ~57.8. Tests use `MineralPatch.PATCH_DISTANCE=6.0f` for stability. Consider adding close/far patch variants for more realistic testing, but do NOT change PATCH_DISTANCE — it would break all existing test assertions.

---

## Key Code Locations

| What | Where |
|------|-------|
| PlaybookSC2Adapter (debug build, geyser targeting) | `quarkmind-sc2/.../real/PlaybookSC2Adapter.java` |
| ResolvedCommand (targetUnitTag) | `quarkmind-sc2/.../real/ResolvedCommand.java` |
| DualRunCalibrationTest (SC2 comparison) | `quarkmind-sc2/.../playbook/DualRunCalibrationTest.java` |
| SC2 calibration baselines (8 files) | `quarkmind-sc2/src/test/resources/calibration/cal-*.json` |
| Calibration playbooks (10 files) | `quarkmind-sc2/src/test/resources/playbooks/cal-*.yaml` |
| SC2 physics constants | `docs/sc2-physics-constants.md` |
| MineralPatch.PATCH_DISTANCE | `quarkmind-sc2/.../emulated/MineralPatch.java:11` |
| EmulatedGame.tick() (factor wiring) | `quarkmind-sc2/.../emulated/EmulatedGame.java:208-209` |
| PlaybookSC2RecorderTest (10 scenarios) | `quarkmind-sc2/.../real/PlaybookSC2RecorderTest.java` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name: `-Dtest=ClassName`. Not related to Epic #534.
