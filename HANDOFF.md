# Handover — 2026-06-06

**Head commit (project):** `604b044` — docs: sync ARC42STORIES.MD — stale scan at session wrap
**Head commit (workspace):** `8b976d0` — docs: add blog entry 2026-06-06-mdp01-layer4-ledger-integration

## What Changed This Session

Closed #175 (InMemoryCaseFile null fix in casehub-poc — null guard in `put()`, constructor, `putIfAbsent`, `putIfVersion`) and #156 (Layer 4 casehub-ledger integration). Layer 4 wires all four plugins to casehub-ledger via CDI async events: plugins `fireAsync(PluginDecisionEvent)` on state transitions only, `PluginOutcomeAuditor @ObservesAsync` writes `OutcomeRecord` (confidence 0.7, SOUND verdict). `LedgerLifecycleAdapter` uses `@Any Instance<InMemoryLedgerEntryRepository>` + `isUnsatisfied()` to clear at game-stop without coupling production code to the memory impl.

Key discovery: `Event.fire()` silently does nothing for `@ObservesAsync` observers — must be `fireAsync()`. Also: `Unit` is a record whose `equals()`/`hashCode()` includes `position` (changes every tick) — all transition detection must compare by `unit.tag()` only (new protocol PP-20260606-506f33).

## Immediate Next Step

Run `/work` and start #156's successor: **#158** (Layer 6: trust routing — `TrustWeightedAgentStrategy` wired via `TrustScoreRoutingPublisher` CDI events). `DefaultOutcomeRecorder` is now writing attestations; the trust scorer has data to work from.

## Cross-Module

*No active blockers.*

**Previously blocked by (now resolved):**
- `casehubio/ledger#114` — shipped 2026-06-02. `DefaultOutcomeRecorder @DefaultBean` available.
- `#156` (Layer 4) — closed this session.

## What's Left

- casehub-poc fix (InMemoryCaseFile null) — merged to casehub-poc `docs/engine-reconstruction-v2` branch. Not yet on their main. Low urgency.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #158 | Layer 6: trust routing — `TrustWeightedAgentStrategy` | M | High | Unblocked — `OutcomeRecorder` writing; wire `TrustScoreRoutingPublisher` CDI events |
| #155 | Layer 3: casehub-qhorus inter-plugin messaging | L | High | Foundation gate pending |
| #159 | Layer 7: comparison baseline vs naive AI | M | Med | Unblocked |

## References

| Context | Where |
|---------|-------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| ARC42STORIES.MD | `ARC42STORIES.MD` (project root) — Layer 4 ✅, L5 ✅, L3/L6/L7 pending |
| New protocol | `docs/protocols/unit-record-tag-only-comparison.md` (PP-20260606-506f33) |
| Garden entry | `GE-20260606-0718fd` (@Any Instance<> + isUnsatisfied() technique) |
| Blog entry | `blog/2026-06-06-mdp01-layer4-ledger-integration.md` |
