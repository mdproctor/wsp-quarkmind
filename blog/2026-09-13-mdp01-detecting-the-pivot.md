---
layout: post
title: "Detecting the Pivot"
date: 2026-09-13
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [sc2, strategy, transition-detection, coaching, pattern-classification]
series: issue-260-strategy-transition-det
---

# Detecting the Pivot

QuarkMind's pattern classifier already knows what the enemy is doing. The `CascadingPatternClassifier` tracks cumulative confidence per archetype across a Drools-ONNX-LLM cascade, decaying evidence over time and producing a ranked list of assessments every tick. If the opponent has five Marines and no expansion, the classifier says `TERRAN_MARINE_RUSH` with 0.7 confidence.

What it couldn't do was tell you when they *stopped* doing that.

A Marine Rush that transitions to Bio Timing isn't just a new archetype — it's a coaching moment. "They're adding Medivacs to their Marines. Get Stalkers and Colossus tech." The archetype shift itself carries meaning that neither the old classification nor the new one captures on its own. The coach needs to know the direction of change, not just the current state.

## The Hysteresis Problem

The naive approach — "fire a transition whenever the dominant archetype changes" — breaks immediately. Cumulative confidence decays exponentially (`DECAY_PER_FRAME = 0.99948`), so two archetypes with similar evidence oscillate at the top of the rankings depending on when the last scout returned information. Every noisy flicker would produce a spurious coaching event.

The fix is hysteresis with two gates. A transition fires only when the new dominant's confidence exceeds a minimum threshold (0.4) *and* exceeds the previous dominant by a gap margin (0.15). The gap matters more than the threshold — it's the difference between "archetype B has some evidence" and "archetype B has decisively overtaken archetype A."

I also had to get the state tracking right. The first version updated `prevDominant` on every dominant change, regardless of whether the threshold was met. Claude caught this during the spec review — if a noise-level archetype momentarily leads at 0.05 confidence but doesn't meet the transition threshold, and then a real archetype arrives and overtakes *it*, the transition reports the wrong `from` field. The noise archetype was never coaching-relevant, but it had silently become the reference point.

The fix: `prevDominant` only updates on two occasions — first-dominant detection (initial classification, no transition event) and qualifying transitions that pass both gates. Below-threshold dominant changes are invisible to the tracker.

## Switch Exhaustiveness — A Compiler Safety Net That Wasn't

Adding `STRATEGY_TRANSITION` to `GameMomentType` revealed something worth codifying. Both `CoachingTriggerBuilder.mapMomentToTier()` and `AdvisoryTriggerBuilder.mapMomentTypeToTrigger()` used `default -> null` in their switch expressions. The new enum value compiled without a single warning — it silently fell into the default arm.

Java switch expressions *require* exhaustiveness, but `default` satisfies that requirement. The compiler considers the switch complete and moves on. A developer adding a new enum value in six months would reasonably expect a compile error pointing them to every switch that needs updating. Instead: silence, and wrong runtime behaviour.

We converted both switches to exhaustive form — explicitly listing every unhandled case as `-> null` instead of using `default`. Now any new `GameMomentType` value produces a compile error until explicitly handled. This became a project protocol: never use `default` in switch expressions over project-owned enums.

## Curated Paths

The transition paths themselves live in `strategy-taxonomy.yaml` alongside the existing archetype definitions — fourteen from-to pairs across Terran, Zerg, and Protoss matchups, each carrying a `displayName` and `coachingAdvice` text. Marine Rush to Bio Timing, Macro to Roach Hydra, Gateway Rush to Colossus Push. The detection is independent of the YAML — any dominant-swap that passes the hysteresis gates fires a transition event. The YAML enriches it. Known transitions get curated coaching text; unknown transitions still fire, they just lack the pre-written advice and fall back to counter data.

The coaching pipeline receives both the archetype assessment ("they are playing Bio Timing") and the transition event ("they switched from Marine Rush to Bio Timing"). Both are useful context for the LLM — the first describes the current threat, the second describes the trajectory.

Four consumers receive the event through two delivery paths: coaching and advisory through the existing moment pipeline (`GameMomentType.STRATEGY_TRANSITION` → `CoachingTriggerBuilder` → coaching LLM), workbench and CBR through a CDI observer (`StrategyTransitionPublished`). The moment path adds zero new consumer wiring — it reuses the same trigger-to-tier mapping that handles every other coaching moment. The CDI path adds one observer per consumer, each a thin forwarder.

The gap margin and minimum confidence will need calibration against replay data — 0.15 and 0.4 are educated starting points, and the interaction between decay rate and gap width means the effective sensitivity depends on how quickly evidence arrives. A transition where both archetypes accumulate evidence gradually over thirty seconds behaves differently from one where a sudden tech switch floods the classifier with new evidence in a single tick.
