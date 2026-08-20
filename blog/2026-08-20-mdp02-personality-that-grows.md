---
layout: post
title: "Personality That Grows — Wiring Disposition Evolution in quarkmind-chat"
date: 2026-08-20
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [quarkmind-chat, personality, eidos, disposition, evolution, reflection]
series: issue-279-quarkmind-discord
---

*Continues from [Teaching the Bot to Remember](2026-08-20-mdp01-teaching-bot-to-remember.md).*

Memory gives the bot a past. Personality evolution gives it a trajectory. The D6 design spec laid out a three-layer personality system — prompt-driven character (Layer 1), mechanical needs (Layer 2), and reflection-driven growth (Layer 3). Layers 1 and 2 were already working. Layer 3 was the missing chain: reflections accumulate, but nothing connects them to the disposition system that shapes how the character behaves.

The full pipeline reads: idle ticks accumulate importance from scored memories → `IdleReflectionTrigger` fires → `ReflectionService` produces insight summaries from episodic memories → each insight gets classified against the character's disposition function terms → `DispositionSignalStore` records which personality trait was activated → `DispositionHealth` probes for threshold crossings → `DispositionEvolution` evaluates whether to evolve or dampen.

Every link in that chain already existed in the foundation. The eidos API has `DispositionSignalStore`, `DispositionHealth`, and `DispositionEvolution` — the full health-check-to-evolution pipeline with four evolution types (dominant-auxiliary swap, dominant replacement, auxiliary replacement, structural reorganisation). Neocortex has `ReflectionService` and `ReflectionSynthesizer`. What was missing was the bridge between them: how does a reflection insight become a disposition activation signal?

The answer is a decorator. `DispositionAwareReflectionSynthesizer` wraps the existing `LlmReflectionSynthesizer` and intercepts insights at the point they're produced — before they go to the store. Each insight gets fed to `LlmReflectionDispositionActivator`, which submits an async LLM request at LOW priority asking: given this reflection and these personality function terms, which term does it most strongly activate? When the LLM responds, the activator records the activation in `DispositionSignalStore`.

I chose the decorator over two alternatives. A CDI observer on `ReflectionRecorded` would be clean in a full Quarkus app, but quarkmind-chat-agent is plain Java — no CDI container. A store-then-retrieve round-trip (reflect → store → fetch by ID → classify) would work, but `CaseMemoryStore` has no `getById` method, and adding one to avoid a decorator felt wrong. The decorator intercepts at the source — zero plumbing, zero round-trip.

The `PersonalityEvolutionPipeline` in quarkmind-core bundles the health → evolution → decay cycle. Three lines of logic: probe the descriptor, evaluate if pending, decay on dampened. `ChatAgencyLoop` calls it on every idle heartbeat. When evolution produces `Evolved`, the loop updates the activator's profile so subsequent classifications use the new personality terms.

The classification is async and best-effort. A failed LLM call means no activation recorded — not a system error. This matters because the LOW-priority queue may be contended with importance scoring. Over time, enough activations accumulate to cross the thresholds configured in `DefaultDispositionHealth`. The character's personality drifts toward the traits its experiences reinforce — an extroverted bot in a community that values empathy will gradually strengthen its empathetic function term. Evolution happens on the timescale of weeks, not ticks.

What this opens up: the full Generative Agents reflection → personality feedback loop is now wired end-to-end. The next question is multi-character (D9 v2) — per-character identity isolation, memory boundaries, and the architectural changes that implies for a system currently scoped to one character per server.
