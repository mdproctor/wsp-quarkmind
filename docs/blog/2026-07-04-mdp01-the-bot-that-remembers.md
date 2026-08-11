---
layout: post
title: "The Bot That Remembers"
date: 2026-07-04
type: phase-update
entry_type: note
subtype: diary
projects: [QuarkMind]
tags: [cbr, neocortex, onnx, strategy, machine-learning]
---

What does it look like when a StarCraft bot learns from its own history? Not just trust scores nudging strategy selection — actual case-based reasoning across games, where the bot remembers how a game played out, what advice it received, and whether the tactics worked.

I've been thinking about this as three tiers of capability, each one pulling the next level of platform infrastructure from neocortex.

## Flat vectors, rich games

**Tier 1 is mostly plumbing.** Flat feature vectors — opponent posture, build order, race context. Store the case at game close, retrieve similar cases at game start, route strategy selection through the retrieved context. The foundation SPIs are built: engine has `ImplementationRoutingStrategy` and `CaseOutcomeObserver`, neocortex has `CaseRetriever` and Qdrant-backed `CaseMemoryStore`. QuarkMind just needs to wire them together.

**Tier 2 is where it gets interesting.** The event summarisation hierarchy already produces L2 moments (FIRST_CONTACT at minute 3.2, BATTLE_WON at minute 5.1), L3 game phases (EARLY_MACRO → MID_SKIRMISH → LATE_PUSH), and L4 narrative arcs. The advisory team already produces trust-scored recommendations from specialised LLM advisors. All of this data exists at game close — it just isn't being written into cases yet.

A Tier 2 case doesn't just say "EarlyPressure won against Zerg rush." It says: "EarlyPressure won because we scouted at 1:30, identified the roach warren at 2:45, the crisis advisor recommended pulling workers (SOUND), and we held the ramp with 3 Stalkers during the EARLY_AGGRESSION phase which lasted 4 minutes before transitioning to MID_SKIRMISH." That's a case the routing strategy can actually learn from — not just *what* worked but *how* and *why*.

This pulls three capabilities from neocortex that don't exist yet: structured case fields (nested objects, not just flat key-value), weighted multi-field similarity (opponent posture matters more than army size ratio), and a case enrichment pipeline SPI so the transformation from raw game events to case features happens automatically at store time.

**Tier 3 is research-grade.** Cases as full game timelines — "find games where the opponent's economy curve looked like this." Dynamic time warping over economy trajectories, edit distance over phase sequences. Predictive CBR: "in similar past games, the opponent's army spike at minute 8 was followed by an all-in at minute 9 — prepare defences now." This is where CBR stops being a lookup table and becomes genuinely anticipatory.

## The ONNX middle tier

Separately, we dug into the SC2 strategy classification literature. A decade of work — from classical [Bayesian build trees](https://alexander-parker.github.io/2019-09-20-Predicting-Opponent-Strategy-in-StarCraft/) through LSTM sequence models to [TacticCraft's tactic tensor](https://arxiv.org/abs/2507.15618) approach from just last month. TacticCraft defines a probability distribution over 8 tactical archetypes validated by Grandmaster players — maps directly to the confidence model in the enemy strategy classifier.

The interesting finding: neocortex already has `OnnxInferenceModel` and `TextClassifier` in production. The ONNX infrastructure for running trained classifiers on the JVM exists. What's missing is a raw tensor input path (the current SPI assumes text → tokenizer → embeddings) and the trained model itself. A small 1D-CNN trained on the [MSC dataset](https://www.mdpi.com/2673-2688/5/4/115) (36k replays), exported to ONNX, would classify enemy strategy in single-digit milliseconds on CPU — sitting between the Drools rules (sub-millisecond, deterministic) and the LLM fallback (500ms+, flexible).

## Remembering the opponent

The opponent modelling piece is particularly compelling. Human players remember specific opponents — "this player always opens 2-rax proxy." With per-opponent case partitioning, the bot can detect adaptation across games: the opponent used roach rush twice, we defended both times, game 3 they switch to macro play. The case history reveals the pattern shift. If the strategy selector can anticipate that, it stops preparing for a rush that isn't coming.

## Measuring learning

The whole thing is measurable. Win rate over a rolling window, strategy selection distribution shifting over time, retrieval hit rate, and the key metric: retrieval influence — how often the retrieved case actually changed the routing decision versus what trust-only would have chosen. If that number stays at zero, the CBR infrastructure is overhead without value.

Two mirrored epics track this: neocortex covers the platform capabilities (structured retrieval, temporal similarity, case enrichment SPI) and QuarkMind covers the game experience (per-plugin learning, opponent modelling, learning curves). Each tier in QuarkMind pulls the next capability tier from neocortex — and the platform capabilities are reusable for AML, clinical, and devtown when those apps reach their own CBR implementations.

The ordering is deliberate: mechanical wiring first, then observability (validates it works), then event/advisory enrichment (cheap — the data already exists), then per-plugin learning, then opponent modelling, then temporal matching last. Each step is independently useful. The bot doesn't need Tier 3 to start getting smarter.
