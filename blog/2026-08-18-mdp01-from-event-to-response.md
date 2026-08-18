---
layout: post
title: "From Event to Response — quarkmind-chat Phase 1"
date: 2026-08-18
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [quarkmind-chat, agency-loop, discord, chat-bot, platform-agnostic]
series: issue-279-quarkmind-discord
---

The chat module has a running agency loop. Not connected to a real Discord server yet — no bot token, no live LLM — but the structural path from "a message arrives on a Discord channel" to "a governed response is dispatched" is complete and tested end to end.

This is quarkmind-chat phase 1: the execution harness. A `DiscordEventSource` adapts the Gateway WebSocket into a blocks `EventSource`. A `DiscordGatewayMessageHistory` accumulates messages locally so we're not hammering the REST API. `ChatWorldBridge` pulls deltas from the history per-tick, tagged with a `WakeReason` — message or heartbeat — so the agency loop knows why it woke up.

The interesting layer is the perception pipeline. Raw messages get classified by attention priority — direct address (@mentions, replies to the bot) always renders verbatim, threads the bot previously participated in get elevated priority, and everything else is ambient. When ambient volume crosses a threshold, `ChatObservationRenderer` compresses it into a summary instead of dumping the full transcript into the LLM context. The compression operates per-channel, not per-thread — a distinction the plan got wrong and implementation caught. Standalone messages each create their own `ConversationThread` of size 1, so per-thread compression never triggers. Obvious in hindsight. Not obvious in a design spec.

The other deviation: `ChatPerceptionBridge` was supposed to live in quarkmind-core alongside the other chat abstractions. It can't — the interface references `ChatPerception` from quarkmind-chat-protocol, which depends on quarkmind-core. Circular. The fix is to put the bridge in the agent module, which already depends on both. The bridge is an integration concern, not a core SPI. The layering actually makes more sense this way.

`ChatAgencyLoop` wires perception into an LLM call and parses the JSON response with Jackson into sealed `ChatIntent` variants — Send, Reply, or React. An `OutputGovernor` rate-limits actions per time window, and `ChatChannelPacing` gates unprompted posts based on channel activity. Three layers of restraint before the bot says anything unsolicited. The design spec's three needs — SOCIAL, CURIOSITY, EXPRESSION — drive autonomous behaviour, with `ChatNeedDefinitions` handling decay rates and satisfaction.

This is the first quarkmind world that uses a direct LLM agency loop instead of CaseEngine blackboard dispatch. SC2 has four specialist plugins coordinated by a case engine — strategy, economics, tactics, scouting — because the reasoning decomposes into independent specialist concerns. A chat character doesn't decompose that way. It perceives a conversation, reasons holistically about how to participate, and acts as one agent. The `AgencyLoop.tick(AgencyContext)` pattern from QuarkVille is the right shape.

Four deferred issues define phase 2: memory integration (#280) wiring neocortex's episodic/semantic/relationship stores into the loop, personality growth (#281) connecting the reflection → disposition evolution chain, multi-character support (#282), and a personality generator wizard (#283). Memory is the one that changes the character from a stateless responder to something that remembers — who it talked to, what it learned, how relationships evolved. That's where the interesting work is.
