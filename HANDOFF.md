# Handover — 2026-06-09

**Head commit (project):** `1527e02` — protocol(PP-20260609-38a43e): sc2-stale-process-must-be-killed
**Head commit (workspace):** `b1b702e` — docs(issue-13-live-sc2-smoke-test): mark closed

## What Changed This Session

Closed #13 (SC2 smoke test). SC2 is running — probes mining, economics ticking, scenario runner
spawning Zealots that trigger DEFEND. But ocraft 0.4.21 needs bytecode patching to run on Vert.x 4.x.
Seven incompatibilities fixed across 5 classes (`VertxChannel`, `VertxChannelProvider`,
`S2ClientVerticle`, `Ping`, `OnRequest`). Patches live in the local Maven repo only — fragile.
Also filed #185 to replace ocraft's transport layer entirely with Quarkus-native WebSocket.

## Immediate Next Step

Run `/work #185` — replace ocraft-s2client-api transport with Quarkus-native SC2 WebSocket client.
Plan: vendor `ocraft-s2client-protocol` (protobuf types), write `QuarkusSC2Transport` (~200 lines),
drop the Vert.x 3.x / RxJava2 transport dependency. Eliminates the jar patching entirely.

## Cross-Module

*No active blockers.*

## What's Left

- **ocraft jar patches** — 5 classes patched in `~/.m2/repository/com/github/ocraft/ocraft-s2client-api/0.4.21/`. Lost on `mvn dependency:purge-local-repository`. Re-apply with scripts documented in `NATIVE.md`. This urgency is why #185 is the immediate next step.
- **#184** — Minor review findings (stale Javadoc, test name, annotation cleanup) · S · Low
- **casehubio/parent#201** — quarkmind.md doc update: L3 status, Qhorus dep, Agentic Harness table · S · Low
- casehub-poc fix (InMemoryCaseFile null) — low urgency, separate repo

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #185 | Replace ocraft-s2client-api with Quarkus-native SC2 WebSocket transport | L | Med | **Start here** — eliminates fragile jar patches; unblocks #14 (GraalVM native tracing) |
| #158 | Layer 6: trust routing — `TrustWeightedAgentStrategy` | M | High | Foundation unblocked |
| #159 | Layer 7: comparison baseline vs naive AI | M | Med | Unblocked |
| #180 | LLM advisory team on scouting channel | L | High | Depends on Qhorus channel |
| #181 | Commentator/Coach LLM | L | High | Depends on #182 |
| #182 | Hierarchical event summarisation | L | High | Foundation for #180, #181 |
| #183 | Enemy strategy classifier | M | High | Level 2 in #182 hierarchy |

## References

| Context | Where |
|---------|-------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| #185 issue | casehubio/quarkmind#185 — full rationale in issue body |
| ocraft patch scripts | `NATIVE.md` — procedure for re-patching after `mvn clean` |
| SC2 smoke test blog | `blog/2026-06-09-mdp01-ocraft-vertx-archaeology.md` |
| Garden entries | `GE-20260609-432b2f` (Mac path separator), `GE-20260609-12a3d7` (Vert.x 4.x breakage), `GE-20260609-0b703b` (SC2 single WS), `GE-20260609-6ae928` (System.exit bypass), `GE-20260609-d24a97` (rx vs core writeBinaryMessage) |
| Protocol | `docs/protocols/sc2-stale-process-must-be-killed.md` (PP-20260609-38a43e) |
| ARC42STORIES.MD | `ARC42STORIES.MD` — L3 ✅, L4 ✅, L5 ✅, L6/L7 pending |
