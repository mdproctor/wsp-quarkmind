# HANDOFF — quarkmind

## Last Session

Closed 5 issues (#384 epic, #390, #391, #392, #393) — race-specific ability infrastructure for economy playbook calibration. Built AbilityIntent + generalized ability resolution, Chrono Boost (Protoss 50 Probes), MULE calldown (Terran 40 SCVs + 517 minerals), parallel Zerg Larva training (45+ Drones). Squashed 36→9 commits, landed on main via fast-forward merge.

Filed 5 follow-up issues: #394 (vespene income model), #395 (intent verification), #396 (dual-run tick-by-tick calibration), #397 (Zerg Hatchery not building), #398 (Terran SCV count below target).

## Immediate Next Step

#389 — finish HSC XXIX replay download, then #385 — run emulator accuracy baselines. The reconstitution pipeline (#387) and ONNX training (#388) follow once baselines are established.

#396 (dual-run calibration) is the highest-value infrastructure investment — tick-by-tick SC2 vs EmulatedGame comparison with early exit on divergence for faster iteration.

## References

| What | Where |
|------|-------|
| Epic | #384 (emulator → reconstitution → ONNX training pipeline) |
| Ability infrastructure | AbilityIntent, RaceModel.handleAbility/trainingSpeedMultiplier/parallelProduction |
| Playbook engine | SC2PlaybookRunner, SC2DeliveryHandler (7 actions) |
| Economy playbooks | quarkmind-sc2/src/test/resources/playbooks/economy-only-{protoss,terran,zerg}.yaml |
| Replay data sources | CLAUDE.md § SC2 Replay Data Sources |
| Follow-up issues | #394 (gas income), #395 (intent verify), #396 (dual-run), #397 (Zerg expansion), #398 (Terran SCV) |
| Commits on main | `713a0c2f` (9 squashed from 36) |
