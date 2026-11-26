# HANDOFF — quarkmind

## Orientation

**Terran and Zerg two-base economy calibration complete (#765). SC2 recorder now works for all three races.**

Extended the SC2 calibration baseline infrastructure from Protoss-only to all races. Fixed three recorder bugs exposed by race-specific production mechanics. Recorded controlled SC2 baselines for Terran (OC morph, MULE, Refinery) and Zerg (Larva training, Overlord supply, Extractor).

---

## Changes made (this session)

1. **Playbook YAMLs** — `cal-two-base-economy-terran.yaml` (CC→OC, MULE, 2 Refineries) and `cal-two-base-economy-zerg.yaml` (Hatchery, Overlords, 2 Extractors)
2. **SC2 baselines recorded** — 450-tick recordings. Terran: 76 SCVs, supply 79/102. Zerg: 76 Drones, supply 80/92.
3. **Recorder bug: morph name crash** — `BuildingType.valueOf("OrbitalCommand")` → added `MORPH_BUILDING_NAMES` lookup map
4. **Recorder bug: Zerg Larva targeting** — train commands sent to Hatchery instead of Larva. Added `findAvailableLarva()` with per-tick deduplication.
5. **Recorder bug: premature fired marking** — supply-triggered train steps marked `fired` even when SC2 rejected (insufficient minerals). Added mineral/supply pre-checks on the Larva training path.
6. **Enemy kill sweep** — debug API kills all enemy units per tick, preventing AI disruption of economy recordings
7. **Pinned calibration generalised** — `pinnedCalibrate()` now detects worker type from race. Variable renames: `nexusPositions` → `basePositions`, `sc2Probes*` → `sc2Workers*`.
8. **Filed #774** — Queen training needs SPAWNING_POOL + non-Larva pre-check generalisation

---

## Current state

| Baseline | SC2 workers | EM mineral Δ (pinned) | Dominant gap |
|----------|-------------|----------------------|-------------|
| Terran two-base | 76 SCVs | +36.1% (EM over) | Worker distribution (SC2: [16,0], EM: [20,52]) |
| Zerg two-base | 76 Drones | -46.1% (EM under) | Worker distribution (SC2: [41,0], EM: [21,49]) |
| Protoss two-base | — | -49.4% (existing) | Same pattern |

The pinned divergence is a recorder limitation: SC2's debug API doesn't auto-transfer workers to expansions. The EM's economy model is accurate — the gap is in the test harness.

**All 71 calibration tests pass** (19 SC2PlaybookCalibrationTest + 52 DualRunCalibrationTest).

---

## Next in queue

**#769 — Flow-field worker pathfinding** (scale/complexity unknown). Topic change from calibration.
**#774 — Queen training gap** (S/Low). Follow-up to this session's review finding.

---

## Notes

- **Branch:** `issue-765-terran-zerg-two-base-playbooks` (2 commits, squashed)
- **SC2 was used** — both baselines recorded from real SC2 with enemy kill sweep
- **Zerg Queen training is dead code** — removed Queen steps (no SPAWNING_POOL). Follow-up in #774.
- **Mineral/supply pre-check only on Larva path** — non-Larva one-shot steps still vulnerable (#774 scope)
