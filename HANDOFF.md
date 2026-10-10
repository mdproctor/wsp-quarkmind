# HANDOFF — quarkmind

## Last Session

Closed #398 — Terran SCV production throughput. Fixed a fidelity bug in `drainBuildingQueues()` where queued units started training on morphing buildings (should pause). Added 2nd OC morph to the Terran economy playbook. SCV count went from 40 to 41. Discovered the remaining gap to 48-50 is caused by supply being committed on queue entry rather than training start — 10 supply slots are always locked by unproduced SCVs.

## Immediate Next Step

#399 (M/Med) — supply-on-queue fidelity fix. Move `addSupplyUsed` from queue-entry to training-start in `handleTrain`. Touches every unit training path, so needs careful regression testing.

## References

- `specs/issue-398-terran-scv-production-throughput/` — design spec and decisions
- `blog/2026-10-10-mdp01-supply-on-queue.md` — diary entry
- #399 — supply-on-queue follow-up
- #400 — multi-OC MULE resolution follow-up
