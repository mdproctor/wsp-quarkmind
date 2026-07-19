# Design Journal — issue-230-coach-mode

## §Design — 2026-07-19

### Coach Mode Architecture

Designed as the third LLM pipeline alongside advisory and commentary. Key architectural decision: separate pipeline with shared detection layers, not an extension of advisory (which would conflate STATUS/COMMAND speech acts) or a separate game loop (which would duplicate observation infrastructure).

**Six new components in `io.quarkmind.plugin.coaching`:**
- `CoachingTriggerBuilder` — urgency-tiered moment triggering with global cooldown and urgency preemption
- `CoachingSessionSelector` — per-game routing cache (one agent per game, trust-routing invoked once)
- `CoachingWorkerFactory` — directive/Socratic LLM workers with structured output (CoachingAdvice record)
- `CoachingChannelBroker` — COMMAND dispatch to Qhorus with correlationId for commitment auto-open
- `CoachingComplianceEvaluator` — implicit compliance via game state deltas (unit/building count)
- `CoachingEffectivenessTrustRecorder` — new trust dimension for coaching effectiveness

### Key Design Review Findings (3 rounds, 20 issues)

1. **R1-04:** Socratic coaching during crisis is harmful — added crisis directive override
2. **R2-02:** Separate coaching capabilities per personality would make trust routing inoperable — unified to single `coaching` capability
3. **R2-03:** `DispositionAwareRoutingStrategy.select()` is stateless — added `CoachingSessionSelector` to cache routing per game
4. **R2-04:** `MilestoneOutcomeRecorder.onGameStopped()` would record corrupt data in coach mode — gated on `game.mode`
5. **R2-05/R3-02:** Global cooldown with urgency preemption — crisis always fires regardless of active lower-tier cooldown

### Follow-up Epics Created

- #250 Coaching Compliance Evolution (position-based, LLM-evaluated, acknowledgment UI)
- #251 Coaching Personality Model (adaptive intra-game, disposition axes)
- #252 SC2 Strategy Knowledge Infrastructure (full tactic taxonomy, commentary training dataset)
