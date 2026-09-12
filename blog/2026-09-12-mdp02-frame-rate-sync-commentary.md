---
layout: post
title: "The Frame Rate Insight — Synchronized Replay Commentary"
date: 2026-09-12
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [replay, commentary, llm, synchronization, game-loop]
---

# The Frame Rate Insight — Synchronized Replay Commentary

Commentary at 1x replay speed works fine. The LLM takes a couple of seconds, the game advances at real-time pace, and the narration trails the action by a small margin. Noticeable but tolerable.

At 4x, it breaks. A reactive commentary fires when a battle starts. By the time the LLM responds two seconds later, the game has advanced sixteen frames. The battle might be over. An enthusiastic "The enemy is at the gates!" arrives after the gates are already rubble.

The fix sounds obvious: pause the replay, wait for the LLM, resume. Replays are immutable — pausing costs nothing. But the interesting part wasn't the fix. It was working out what "pause" means inside a game loop, and what to do about speed.

## The scheduler as a mutex

Quarkus fires `gameTick()` every 500ms with `ConcurrentExecution.SKIP`. If the previous tick is still running, the scheduler doesn't fire a new one — it just skips. I'd been treating this as a concurrency safety measure. It's also a synchronization primitive.

If `GameTickExecutor.execute()` blocks for three seconds waiting on an LLM call, the scheduler simply doesn't fire during those three seconds. No explicit pause flag, no locking, no state machine. The tick takes longer, so fewer ticks happen. The game freezes at the exact frame being narrated, the commentary appears, then the next tick resumes.

`CompletableFuture.runAsync(() -> execute(triggers, type), Thread::startVirtualThread)` keeps the LLM I/O on a virtual thread. `.get(15, TimeUnit.SECONDS)` blocks the scheduler thread with a bounded wait. On timeout, the commentary is skipped and the replay continues — a missed line is far less harmful than a frozen game.

## The frame rate analogy

The speed multiplier was dead code. `AgentOrchestrator` had a `speedMultiplier` field with a getter, setter, and QA endpoint — but `gameTick()` never read it. Wiring it up raised a question: at 8x speed, should the full agent pipeline run eight times per scheduler fire?

The pipeline costs 100–200ms per run (Drools rules, CaseHub signal and settle). Running it eight times would take 800–1600ms — well past the 500ms tick budget.

The insight came from thinking about video. You can watch at 24fps and it's smooth. You can watch at 6fps and still follow everything that happens. The comprehension threshold is much lower than the rendering rate.

Same principle: at 8x, call `engine.tick()` eight times to advance the replay, but only `observe()` and run the pipeline on the final state. The agent watches at a lower framerate. Moment detection cooldowns (110+ frames for reactive commentary) are far longer than the eight-frame sampling gap. Nothing is missed.

## Three modes, runtime switchable

The sync model has three modes — `full` (pause for both reactive and narrative commentary), `reactive-only` (pause for immediate reactions, fire-and-forget for periodic summaries), and `none` (current behaviour). Full is the default in `%replay`. A QA endpoint lets you switch at runtime without restarting.

At 8x with `full` sync, narrative pauses add significant overhead — a 3–5 second LLM call every 30–45 game-seconds means roughly half the wall-clock time is paused. That's the "chapter marker" experience: the game freezes, the narrator summarises, the game resumes. If that's too slow, `reactive-only` drops the narrative pauses and keeps the play-by-play synchronised. The viewer picks the tradeoff.
