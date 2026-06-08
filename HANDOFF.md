# Handover — 2026-06-08

**Head commit (project):** `4fad1f7` — docs: sync ARC42STORIES.MD — stale scan at session wrap
**Head commit (workspace):** `2266937` — docs(issue-177-layer3-scouting-followon): mark closed

## What Changed This Session

Closed #177, #178, #179, #176 — Layer 3 dual-stack redesign. The core architectural change:
`ScoutingIntelBroker` becomes a typed in-memory store (Stack 1, synchronous, for plugins); the
Qhorus channel stays as an advisory surface for future LLM consumers (Stack 2, async, #180/#181).

`DroolsTacticsTask` migrated from `MessageObserver` + `TacticsIntelCache` to reading from the
broker. `DroolsStrategyTask` and `FlowEconomicsTask` added as new `ScoutingIntelConsumer`
implementations. `NEAREST_THREAT` CaseFile key removed entirely (#179) — threat intel flows via
broker only. Hot-reload endpoints added (#178): `POST /qa/scouting/subscriptions/reload` and
`/thresholds/reload`.

Four future issues filed during brainstorming: #180 (LLM advisory team), #181 (Commentator/Coach
LLM), #182 (hierarchical event summarisation), #183 (attack pattern recognition).

## Immediate Next Step

Run `/work` and start **#158** (Layer 6: trust routing — `TrustWeightedAgentStrategy`).
`TrustWeightedAgentStrategy` is already shipped in casehub-engine (confirmed this session — no
longer blocked by the foundation). Wire it via `TrustScoreRoutingPublisher` CDI events.

## Cross-Module

*No active blockers.*

## What's Left

- casehub-poc fix (InMemoryCaseFile null) — on `docs/engine-reconstruction-v2`, not yet on their main. Low urgency.
- Minor review findings → #184 (stale Javadoc in BasicScoutingTask/DroolsScoutingTask, stale test name in AdaptivePluginSelectionIT, annotation cleanup)
- peer-repo doc update → casehubio/parent#201 (quarkmind.md — L3 status, Qhorus dep, Agentic Harness table)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #158 | Layer 6: trust routing — `TrustWeightedAgentStrategy` | M | High | Foundation unblocked — `TrustWeightedAgentStrategy` shipped in casehub-engine |
| #159 | Layer 7: comparison baseline vs naive AI | M | Med | Unblocked |
| #180 | LLM advisory team — trust, latency, personality-based selection | L | High | New; depends on Qhorus channel from this session |
| #181 | Commentator/Coach LLM — real-time narration and coaching | L | High | New; depends on #182 summarisation layer |
| #182 | Hierarchical event summarisation — temporal abstraction layer | L | High | New; foundation for #180 and #181 |
| #183 | Enemy strategy classifier — attack pattern recognition | M | High | New; Level 2 in #182 hierarchy |

## References

| Context | Where |
|---------|-------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| ARC42STORIES.MD | `ARC42STORIES.MD` — L3 ✅ (redesigned), L4 ✅, L5 ✅, L6/L7 pending |
| Spec | `docs/superpowers/specs/2026-06-08-layer3-dual-stack-redesign.md` |
| New protocol | `docs/protocols/scouting-consumer-postconstruct-required.md` (PP-20260608-8584ab) |
| Garden entry | `GE-20260608-2c8739` (CDI proxy package-private cross-package visibility) |
| Blog entry | `blog/2026-06-08-mdp01-two-delivery-paths.md` |
