# HANDOFF — quarkmind

## Last Session

Closed #386 (Milestone 1 waypoint) — calibration via CmdEvent-driven validation reached its ceiling. Opened #406 (Milestone 2) — tracker-event-driven physics validation. Branch: `issue-386-calibrate-emulated-physics`. Issue #387 paused on the stack (depends on #386).

### Completed This Session

1. **Protocol: calibration milestones are waypoints** — codified the pattern: each milestone sets a numeric target, reaching it triggers assessment and replanning, not completion. `docs/protocols/calibration-milestones-are-waypoints.md`.

2. **Waypoint assessment for #386** — posted closing assessment documenting: baseline at close, what improved (upgrades 0%→58.3%, buildings 95%→100%, economy error ~200%→12.8%), what didn't (unit accuracy plateaued at 76.7% due to 43% CmdEvent capture ceiling), and the methodology verdict (CmdEvent approach has reached its ceiling).

3. **Created #406** — next milestone: tracker-event-driven EmulatedGame physics validation. Feeds complete inputs from oracle tracker events to isolate physics accuracy from input completeness. Waypoint targets: units ≥95%, buildings ≥99%, upgrades ≥90%, economy MAPE ≤15%.

### Baseline at #386 Close (5-minute checkpoint, 118 oracle replays)

| Metric | Before #386 | After #386 | Milestone 1 Target |
|--------|------------|-----------|-------------------|
| Units | ~13% | 76.7% | ≥80% (waypoint) |
| Buildings | 95% | 100% | ≥97% ✓ |
| Upgrades | 0% | 58.3% | >30% ✓ |
| mineralsCurrent error | ~200% | 12.8% | — |

### Key Finding

The CmdEvent-driven validation conflates input completeness with physics accuracy. At 43% CmdEvent capture rate, we cannot distinguish physics errors from missing inputs. Milestone 2 (#406) shifts to tracker-event-driven validation — feed the emulator 100% of inputs from oracle tracker events, then every remaining discrepancy is a real physics bug to fix.

### Next Steps

1. Start #406 — build `TrackerIntentExtractor` (tracker events → TimedIntents)
2. Create tracker-driven validation harness
3. Calibrate every physics constant against clean measurement

## References

- docs/protocols/calibration-milestones-are-waypoints.md — waypoint pattern protocol
- docs/benchmarks/emulated-game-accuracy-baseline.md — current baseline
- #384 — parent epic
- #387 — paused on stack, depends on #386
- #406 — next milestone
