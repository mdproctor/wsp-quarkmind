---
layout: post
title: "When One Character Isn't Enough — Multi-Agent Chat Architecture"
date: 2026-08-25
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [multi-character, blocks, inner-life, drives, discord, architecture]
series: issue-279-quarkmind-discord
---

*Continues from [Personality That Grows](2026-08-20-mdp02-personality-that-grows.md).*

The first version of quarkmind-chat ran one Discord bot with one personality. One agentId, one set of needs, one system prompt, one memory store. It worked. Then the question arrived: what if you want *two* characters in the same server?

Not two instances of the same bot — two different characters with different personalities, different memories, different social drives, seeing each other's messages and occasionally responding to each other. Luna the empathetic listener and Rex the analytical problem-solver, coexisting in the same Discord channels, each with their own inner life.

This is a harder problem than it looks.

## The Single-Character Assumptions

A single-character agent bakes identity into the loop. The agency loop holds an `agentId`, a `systemPrompt`, idle tick counters, reflection timestamps — all as instance fields. This is fine when there's one character. With two, every field becomes a question: whose counter? Whose reflection? Whose memory?

The refactoring pattern is straightforward once you see it. Extract per-character state into a `CharacterContext`:

```java
public class CharacterContext {
    private final String agentId;
    private final String tenantId;
    private final String systemPrompt;
    private final Supplier<AgentDescriptor> descriptorSupplier;
    private final BotIdentityDetector identityDetector;
    // ...
}
```

The agency loop becomes stateless. Each tick receives its character context through the `AgencyContext` map — the same mechanism already used for perception. A `ChatCharacterManager` holds N contexts and routes each character's Discord events to the shared loop with the right context. One loop instance, N characters.

WackyManor — a murder-mystery mansion where AI characters scheme, steal, and pull each other aside for private conversations — proved this pattern months ago. Its `ScenarioOrchestrator` manages a dozen characters using shared CDI services with `agentId`-scoped operations. The quarkmind-chat adaptation follows the same spine.

## Two Paths to Speech

A chat character has two fundamentally different reasons to produce output. It can *respond* — someone mentioned it, asked it a question, replied to its message. Or it can *initiate* — it has something it wants to say, unprompted, because its internal state has crossed a threshold.

These two paths have different requirements. The reactive path needs conversation-aware context: thread history, @mention parsing, memory recall. The proactive path needs motivation evaluation: should I speak? What drives me to speak? Is it socially appropriate right now?

The blocks `InnerLifeOrchestrator` handles the proactive path. It maintains per-agent state internally — event buffers, drive profiles, civility constraints — and returns a simple sealed result:

```java
public sealed interface InnerLifeTick {
    record Silent(@Nullable String reason) implements InnerLifeTick {}
    record Initiated(String content, @Nullable String channelHint,
                     double motivationScore) implements InnerLifeTick {}
}
```

On a heartbeat wake, the loop calls `innerLifeOrchestrator.tick(descriptor, channelContext)`. If the orchestrator decides the character is motivated enough to speak, it returns `Initiated` with content generated from its own internal LLM call. If not — `Silent`. The character stays quiet and waits.

The reactive path stays in the agency loop. Someone posts a message, the loop builds context, recalls memories, calls the LLM, and produces a response. But it also feeds the conversation events into the InnerLifeOrchestrator via `observe()` — so the proactive path has data to work with later. Without this coupling, the inner life would be starved of observations and would never initiate.

## Drives Instead of Needs

The original design had three mechanical needs: SOCIAL (decays when idle), CURIOSITY (spikes on interesting topics), EXPRESSION (builds over time). Simple decay-and-satisfy counters.

The blocks `DriveOrchestrator` replaces these with four drive axes drawn from Self-Determination Theory: CURIOSITY, COMPETENCE, AFFILIATION, AUTONOMY. The difference is more than naming. Each drive is backed by a specialised `DriveSource` that evaluates real agent state — the `CuriosityDrive` checks memory hygiene signals, `AffiliationDrive` monitors user model staleness, `CompetenceDrive` tracks strategy learning outcomes. Drives compose with mood state and narrative modulation to produce a `DriveProfile` that feeds into the proactive path's motivation evaluation.

For the reactive path, drive state enriches the LLM prompt. Instead of `Needs: SOCIAL=75, CURIOSITY=40, EXPRESSION=60`, the prompt carries a richer motivational picture that shapes how the character responds — a character with high AFFILIATION drive responds differently than one driven by AUTONOMY.

## The Shared Pacing Problem

With one character, rate limiting is simple. With two characters watching the same channels, both might want to respond to the same message. Without coordination, they respond simultaneously — two messages appearing within milliseconds of each other, breaking the illusion.

`CivilityConstraint` is a composable chain of constraints that the `InnerLifeOrchestrator` evaluates before any proactive initiation. `MinimumGapConstraint` enforces a time gap between actions. `MaxPerWindowConstraint` caps actions per time window. `ConsecutiveInitiationCooldownConstraint` backs off when the character has been talking without getting responses. For multi-character, a single `MinimumGapConstraint` instance shared across characters prevents the collision — if Luna just spoke, Rex waits.

## Per-Character Gateway

Each Discord bot account is a separate WebSocket connection to the Discord Gateway. This isn't a design choice — it's a Discord API requirement. One token, one session, one connection. Multi-character means N connections, N message buffers, N identity detectors. Each character's `DiscordGatewayMessageHistory` accumulates all messages including from other characters — so Luna sees Rex's messages as external conversation, naturally.

The identity detector ensures a character doesn't perceive its own messages as external events. When Luna posts, her event listener accumulates the message into history (so Rex can see it) but doesn't fire a wake event for Luna herself.

## What This Enables

The architecture is deliberately scoped as a foundation. The `CharacterContext` + stateless loop + shared services pattern enables multi-character, but the full end-to-end wiring — CDI lifecycle, Quarkus `@ConfigMapping`, per-character Gateway startup — is a deployment concern layered on top. What's in place: any number of characters can share one agency loop, one memory store, and one LLM queue, with per-agent isolation guaranteed by `agentId` keying in every orchestrator's internal state.

The next step is the personality generator wizard — an LLM-powered tool that helps create character descriptor YAML from a natural language description. "Describe your character" becomes a structured personality definition with disposition axes, style profile, goals, and constraints. Configuration-driven deployment means a new character is a YAML file and a bot token, not code.
