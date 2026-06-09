# Handover — 2026-06-10

**Head commit (project):** `a372578` — docs: sync CLAUDE.md — add QuarkusSC2Transport tests and sc2/real description
**Head commit (workspace):** `411ef5b` — feat: promote blog entry from issue-185-quarkus-sc2-transport

## What Changed This Session

Closed #185. Replaced ocraft-s2client-bot (Vert.x 3.x + RxJava2 transport) with `QuarkusSC2Transport` — a raw Socket WebSocket implementation (~280 lines) using RFC 6455 framing, `SynchronousQueue` for send/receive pairing, virtual thread game loop with correct quit()/running lifecycle semantics. All 931 tests pass. Ocraft jar patches no longer needed. `ocraft-s2client-protocol` kept as a direct dep (data types only, no transport code, no Vert.x). Also filed #186, #187, #188 for Phase 2 implementation issues found during code review.

## Immediate Next Step

`/work #184` — minor code review findings (stale Javadoc, test name, annotation cleanup) · S · Low. Quick win to clear the trailing review debt before L6 work.

## Cross-Module

*No active blockers.*

## What's Left

- **#184** — Minor review findings (stale Javadoc, test name, annotation cleanup) · S · Low
- **#186** — `connectFallback()` must stop lifecycle chain on failure · S · Med
- **#187** — `onStep` sendActions/sendDebug exception handling + Opus model switch signal · S · Med
- **#188** — macOS-only comment in `resolveSC2Executable()` · XS · Low
- casehub-poc fix (InMemoryCaseFile null) — low urgency, separate repo

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #184 | Minor code review cleanup — Javadoc, test names, annotation | S | Low | **Start here** |
| #158 | Layer 6: trust routing — `TrustWeightedAgentStrategy` | M | High | Foundation unblocked |
| #159 | Layer 7: comparison baseline vs naive AI | M | Med | Unblocked |
| #180 | LLM advisory team on scouting channel | L | High | Depends on Qhorus channel |
| #181–183 | Commentator/Coach LLM + hierarchical summarisation + enemy classifier | L | High | Cascade dependency |

## References

| Context | Where |
|---------|-------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| #185 spec (Revision 9) | `docs/superpowers/specs/2026-06-09-quarkus-sc2-transport-design.md` |
| Transport protocols | `docs/protocols/sc2-joinGame-raw-interface-required.md`, `sc2-createGame-realtime-must-not-be-set.md` |
| Garden entries | `GE-20260609-d6cabc` (TCP probe/accept), `GE-20260609-54d3c7` (BufferedReader frames), `GE-20260609-878c41` (JDK HttpClient hang), `GE-20260609-f0d149` (ImageData byte count), `GE-20260609-f14f75` (SynchronousQueue pairing) |
| Diary entry | `blog/2026-06-09-mdp01-pulling-out-ocraft-vert-x-heart.md` |
| ARC42STORIES.MD | L3 ✅, L4 ✅, L5 ✅, L6/L7 pending (#158, #159) |
