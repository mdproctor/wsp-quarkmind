# Handover — 2026-06-04

**Head commit (project):** `0a4e882` — docs(#174): replace stale BasicStrategyTask refs with DroolsStrategyTask
**Head commit (workspace):** `966cdf3` — feat: promote blog from issue-172-173-174-sx-fixes

## What Changed This Session

Closed S/XS batch: #172 (NEAREST_THREAT null guard), #173 (Assimilator dispatch handler), #174 (stale doc refs). All three on single branch `issue-172-173-174-sx-fixes`, closed and merged to main.

Key discovery: `InMemoryCaseFile` and `CaseContextImpl/MapCaseFile` have opposite null write semantics — put(key, null) always inserts in `InMemoryCaseFile`, making `contains()` return true even when value is null, then `get()` throws NPE via `Optional.of(null)`. Filed casehub #175. Garden entry GE-20260604-cf25cd submitted. Protocol PP-20260603-049dd0 updated with explicit non-null clause.

Also: ARC42STORIES.MD synced at session start (L5 status, ledger#114 resolved). Handover skill updated with arc42 stale scan step in wrap checklist (cc-praxis, synced).

## Immediate Next Step

Run `/work` and start #156 (Layer 4: casehub-ledger integration) — `DefaultOutcomeRecorder @DefaultBean` is available; wire with `casehub.ledger.trust-score.routing-enabled=true`. See handover References for capability tag note.

## Cross-Module

*No active blockers.*

**Previously blocked by (now resolved):**
- `casehubio/ledger#114` — shipped 2026-06-02. `DefaultOutcomeRecorder @DefaultBean` available. Capability tag must match what `TrustScoreCache` reads — use `OutcomeRecord.of(actorId, subjectId, capabilityTag, verdict, confidence)`, not `ofGlobal()`.

## What's Left

- casehub #175 — `InMemoryCaseFile.get()` uses `Optional.of()` → NPE for null values; fix: `Optional.ofNullable()` · XS · Low (casehub side, not quarkmind)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #156 | Layer 4: casehub-ledger integration | L | High | Unblocked — `DefaultOutcomeRecorder` available |
| #158 | Layer 6: trust routing | M | High | Unblocked — wire `TrustScoreRoutingPublisher` CDI events |
| #155 | Layer 3: casehub-qhorus inter-plugin messaging | L | High | Foundation gate pending |
| #175 | InMemoryCaseFile.get() NPE for null values (casehub) | XS | Low | casehub fix; quarkmind orElseThrow() already in place |

## References

| Context | Where |
|---------|-------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| ARC42STORIES.MD | `ARC42STORIES.MD` (project root) |
| Ledger OutcomeRecorder config | `casehub.ledger.trust-score.routing-enabled=true`; capability tag must match `TrustScoreCache` key |
| Garden entries (this session) | `GE-20260604-cf25cd` (InMemoryCaseFile null semantics) |
| Protocol updated | `docs/protocols/nearest-threat-conditional-write.md` (PP-20260603-049dd0) |
