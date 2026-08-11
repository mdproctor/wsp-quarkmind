---
layout: post
title: "When the Coach Can't Check Its Own Homework"
date: 2026-08-06
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [coaching, llm, compliance, cbr, onnx]
---

QuarkMind's coaching mode gives real-time advice to a human player. "Build 3 Stalkers." "Expand to your natural." Each piece of advice carries a verification predicate -- a structured check that fires after a time window to see if the player followed through. Count deltas, army centroid movement, expansion placement. Deterministic, fast, runs every tick.

The problem: some coaching advice can't be reduced to a structured check. "Improve your macro." "Apply more pressure." "Be more aggressive." These arrive with no predicate. Today they resolve as NEUTRAL -- no compliance signal, no trust learning, no feedback loop. The coaching system produced advice but can't tell whether it worked.

The obvious move is to add an LLM verification predicate to the sealed hierarchy. The coaching LLM already runs -- why not add a second call as another `VerificationPredicate` permit? We explored this and rejected it. The sealed hierarchy is synchronous and deterministic -- `isSatisfied()` runs on the game tick thread, returns a boolean, completes in microseconds. An LLM call takes two to five seconds. Forcing async behaviour into a synchronous sealed interface means either blocking the game loop or making every predicate async -- both wrong.

Instead, we split the evaluation path. When a non-verifiable commitment's window closes, the evaluator dispatches an async LLM call via `CompletableFuture` on a managed executor and removes the commitment from the map immediately. The worker compares two game state snapshots -- before and after -- and returns a three-value verdict: COMPLIED, PARTIALLY, or IGNORED. The callback maps these to the existing trust outcomes plus a new PARTIAL status.

The interesting engineering was in the cancellation semantics. The commitment is gone from the map the moment the LLM call dispatches, but the callback fires two to five seconds later from a different thread. If the game ends in that window, `withdrawAll()` runs and the callback arrives to find nothing -- or worse, records a trust outcome against a game that's already closed. The fix is an in-flight tracking set: correlationIds go in when dispatched, come out when the callback fires. `withdrawAll()` clears the set. If the callback finds its correlationId absent, it discards silently. Same guard handles supersession -- new advice for the same domain removes the old correlationId from the set before the old LLM call returns.

The game state summarisation is deliberately compact. The LLM doesn't see raw `GameState` objects -- it sees a BEFORE/AFTER/CHANGES text block with grouped unit counts, building counts, and resource deltas pre-computed. The delta is the key: "+3x STALKER, +1x NEXUS, Minerals: -170" lets the LLM assess compliance without reasoning over two full state dumps.

The session also surfaced something unexpected about the ONNX strategy classifier. The neocortex training pipeline shipped -- all three per-matchup models exported, issues closed. But the models were trained on synthetic data, not real replays. The MSC dataset (36k replays) was never downloaded. The vs_zerg model shows 74% top-1 accuracy overall, which sounds passable until you see the per-class breakdown: ROACH_RUSH at 87% carries the average while MUTA_HARASS scores zero on 31 samples and HYDRA_PUSH scores zero on 16. The Protoss model, with nine archetype classes, will be worse. Filed an issue on neocortex for real-data retraining -- that blocks the three-tier cascade and everything downstream.

The CBR epic paints a clearer picture now. ONNX handles "what is the opponent doing?" from training data. CBR handles "what worked for us against that?" from live experience. Replays can't bootstrap CBR meaningfully -- the bot didn't make the decisions being correlated. CBR starts accumulating when the bot plays real games, and ONNX gives it good archetype detection from tick one so the cold-start window is short.
