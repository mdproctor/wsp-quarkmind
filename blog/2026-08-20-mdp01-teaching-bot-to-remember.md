---
layout: post
title: "Teaching the Bot to Remember — quarkmind-chat Phase 2"
date: 2026-08-20
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [quarkmind-chat, memory, neocortex, reflection, episodic-memory]
series: issue-279-quarkmind-discord
---

*Continues from [From Event to Response — quarkmind-chat Phase 1](2026-08-18-mdp01-from-event-to-response.md).*

The chat agent has episodic memory. Not the "I stored your name in a database" kind — a generative memory system where conversations become observations, observations get scored for importance, and idle moments trigger reflection that synthesises higher-level insights from accumulated experience. The architecture follows Park et al.'s composite retrieval model: recency × importance × relevance.

Phase 1 gave us the execution harness — perceive, reason, act. Phase 2 gives the agent something to reason *about* beyond the current conversation window. The work landed as three layers.

**ChatMemoryFacade** composes two neocortex stores behind a clean API: `recall()` retrieves memories via a two-stage pipeline — semantic search through `ExperienceQuery`, then personality-weighted re-ranking via `PersonalityWeightedRetrieval`. `ingest()` stores an observation with source references (channel ID, message ID range) so the raw conversation is always traceable. Graphiti graph memory is optional — when deployed, `recall()` merges relationship facts from the knowledge graph; when absent, it falls back to attribute-based participant tags on episodic memories.

The observation model is worth explaining. Each agency tick produces a one-line observation: "Bob asked about ML. I helped." This comes from the same LLM call that decides the action — the response JSON gains an `observation` field alongside `action` and `text`. No extra API call. On WAIT (the most common action), the observation still captures what was perceived. Staying quiet is a meaningful experience.

After ingestion, the observation enters the importance scoring pipeline asynchronously. A LOW-priority LLM request asks "how important was this experience, 0.0 to 1.0?" The score feeds two consumers: `ChatMemoryFacade.scoreImportance()` updates the stored memory, and `IdleReflectionTrigger.accumulate()` tracks cumulative importance toward the reflection threshold. Unscored memories default to 1.0 in retrieval re-rankers — visible until scored, never penalised for async latency.

When the trigger fires during idle heartbeats, `ReflectionOrchestrator.reflect()` queries recent experience memories and passes them to `LlmReflectionSynthesizer`, which asks the LLM for generalised insights. These become reflection-domain memories with importance scaled by level: 0.5 for first-pass synthesis, 0.7 for meta-reflection, 0.9 for deep insight. The full chain — episodic observation → importance scoring → idle trigger → reflection synthesis → stored insight — runs without any CDI framework dependency in the agency loop. The orchestrator and stores are injected; the loop doesn't know what backs them.

The cross-repo work was small but necessary: `updateImportance()` on `CaseMemoryStore` in neocortex, adding a method to update a memory's importance after initial storage. Seven implementations, three contract tests. And `LlmRequest.responseHandler` in quarkmind-core, so the LLM queue can deliver async responses back to callers — importance scoring was the first consumer, but the pattern is general.

What made this straightforward is that neocortex already had every primitive. Semantic retrieval, personality-weighted re-ranking, reflection orchestration, experience queries — all existed. The work was composition: wiring existing stores behind a facade, integrating the facade into the agency loop's tick cycle, and building the one piece the foundation didn't have (an LLM-powered `ReflectionSynthesizer`). Phase 2 is a consumer of the memory platform, not an extension of it.

Next is personality growth (#281) — the chain from reflection → disposition activation → evolution. The reflection trigger we built here is the input. If the agent notices it keeps having deep conversations about ML, that pattern should eventually shift its personality disposition — becoming more curious, more analytically oriented. The memory system gives it something to reflect on. The personality system gives the reflection somewhere to go.
