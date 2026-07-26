# QuarkMind Handover — 2026-07-27

## Last Session

Branch `issue-243-tactic-taxonomy` closed — #243 landed as `8e80dc2` on main. Full strategy taxonomy infrastructure: two-track detection (hand-authored DRL + generic data-driven pipeline), 20 archetypes across EARLY/MID/LATE, coaching counter enrichment, TimeBasedPhaseResolver. Garden entry GE-20260726-7213f2 captured Drools nested-class codegen gotcha.

## Immediate Next Step

Pick next work from What's Next — #257 (fill taxonomy to 50-90) is the natural continuation, or #258 (dynamic PhaseResolver) for a design challenge.

## What's Left

- #74 — genericise unit/building definitions · L · Med
- Garden push pending — GE-20260630-91be72 + GE-20260726-7213f2 committed locally, push blocked by pre-push hook (auth)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #257 | Fill strategy taxonomy to 50-90 archetypes | M | Low | YAML editing task now — generic pipeline handles detection |
| #258 | Dynamic PhaseResolver — state-based phase detection | M | Med | Replace time-based with game-state heuristics |
| #259 | Visualizer counter overlay | M | Med | Consume CounterInfo API |
| #260 | Strategy transition detection | L | High | Temporal sequencing of archetype detections |
| #261 | Multi-race counter data | M | Med | Matchup-keyed counters |
| #245 | LLM-evaluated compliance verification | M | Med | New VerificationPredicate permit |
| #248 | Coaching acknowledgment UI (DONE/DECLINE) | M | Med | Visualizer overlay |
| #251 | Coaching personality model | M | Med | Epic; adaptive disposition |
| #235 | LLM fallback for novel/ambiguous builds | M | Med | CBR infrastructure in place |
