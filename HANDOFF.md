# Handover — 2026-06-07

**Head commit:** `01b66ef` — docs: sync ARC42STORIES.MD — L3 complete, C2/C3/C4 unblocked

## What Changed This Session

**#155 — Layer 3: casehub-qhorus typed inter-plugin messaging (closed):**
- 5 seam types: `ScoutingIntelType`, `ScoutingIntelConsumer`, `ScoutingIntelPayload` (sealed), `ScoutingIntelPreference`, `ScoutingIntelPreferences`
- `ScoutingIntelBroker` computes subscription union at startup; only runs Drools CEP when subscribed
- `DroolsTacticsTask` implements `MessageObserver` + `ScoutingIntelConsumer`; caches intel in `AtomicReference<TacticsIntelCache>`; `canActivate()` gates on cache.threatPosition()
- `DroolsScoutingTask` dispatches STATUS messages on preference-backed thresholds
- `TacticsMessageBridge` CDI bridge needed — `@CaseType` qualifier made DroolsTacticsTask invisible to unqualified `Instance<MessageObserver>` in qhorus
- 4 qhorus surprises filed upstream: qhorus#254, #257, #258, #259
- ARC42STORIES.MD updated: L3 ✅ complete, C2/C3/C4 unblocked

## Immediate Next Step

Start #158 (Layer 6: trust routing) or #155 (Layer 3) — wait, #155 is closed. Next is **Layer 6 (#158)** or **Layer 3's chapter entry (C2)**. Check `ARC42STORIES.MD` §Chapter 2 stub — write the full C2 chapter narrative now that L3 is complete. Then assess whether #158 (needs `TrustWeightedSelectionStrategy` in casehub-engine — not yet implemented) or #171 (SC2 server protocol wrapper, no foundation dep) is the better next branch.

## Open Issues

| # | What | Status |
|---|------|--------|
| #13 | Live SC2 smoke test | Blocked on SC2 |
| #14 | GraalVM native image | Blocked on #13 |
| #74 | YAML unit definitions | Parked |
| #158 | Layer 6: trust routing | Pending (needs casehub-engine `TrustWeightedSelectionStrategy`) |
| #159 | Layer 7: comparison baseline | Pending (depends on prior layers) |
| #171 | SC2 server protocol wrapper | Pending |
| #177 | Strategy/Economics subscribe to ScoutingIntelConsumer | Pending |
| #178 | Dynamic preference hot-reload | Pending |
| #179 | Deprecate NEAREST_THREAT CaseFile key | Pending |

## References

| Context | Where |
|---------|-------|
| Layer 3 spec (revised) | `docs/superpowers/specs/2026-06-06-qhorus-layer3-design.md` |
| Layer 3 diary entry | `blog/2026-06-07-mdp01-layer3-qhorus-intel.md` |
| qhorus surprises garden entries | `GE-20260607-d051f2`, `GE-20260607-a4d78a`, `GE-20260607-1ebb9c` |
| Prior handover | `git show HEAD~1:HANDOFF.md` |
