---
layout: post
title: "Wiring playerResult to L6 Trust Routing"
date: 2026-06-12
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [sc2, trust-routing, concurrency, quarkus]
---

L6 trust routing shipped two weeks ago. The machinery was correct — Bayesian Beta model, four-phase maturity, designated fallback tiebreaker. The problem was that `GameOutcomeRecorder` always wrote `AttestationVerdict.SOUND`. Every game, win or loss, the same neutral signal. The trust model accumulated observations but learned nothing. To fix that, I needed to wire the SC2 API's `playerResult` to `GameStopped` — and then discovered the problem was considerably larger than that.

## Natural game ends didn't fire GameStopped

The SC2 API sends `status = ended` on the final observation frame when a game finishes. `QuarkusSC2Transport.gameLoop()` breaks out of its observation loop when it sees this, but the final observation response — which contains `playerResults: List<PlayerResult>` with each player's VICTORY/DEFEAT/TIE outcome — was simply discarded. That's the first problem.

The second: `GameStopped` was only ever fired by `AgentOrchestrator.stopGame()`, which is a manual call. When the game ended naturally, the scheduler saw `isConnected() == false` and stopped dispatching, but `GameStopped` never fired. `GameOutcomeRecorder` never ran. The L6 trust pipeline was effectively dead for any naturally completed game.

Enriching `GameStopped` to carry a `GameResult` (WIN/LOSS/TIE/UNKNOWN) meant rebuilding how game-end detection works in `AgentOrchestrator`. The scheduler now tracks the connected→disconnected transition with a `volatile boolean engineWasConnected`, firing `GameStopped` via a private `fireGameStoppedOnce()` when it first sees `isConnected()` flip false. `gameActive`, an `AtomicBoolean` armed in `startGame()`, provides the CAS guard to ensure exactly one `GameStopped` fires regardless of whether the game ended naturally or was manually stopped.

## The second-order bug

My first implementation put `gameActive.set(true)` in `gameTick()`'s connected branch, not in `startGame()`. That introduced a race.

`stopGame()` calls `transport.quit()` — async: sets a flag, interrupts the game loop thread, returns. `running` stays true for several scheduler frames while the loop exits. If a `gameTick()` fires in that window and sees `isConnected() == true`, it re-arms `gameActive`. Then when `running` finally goes false and the natural-end path fires `fireGameStoppedOnce()`, it finds the guard re-armed and fires a second `GameStopped`.

Claude caught this in review. The fix: `gameTick()` never writes `gameActive` — owned solely by `startGame()` and the CAS in `fireGameStoppedOnce()`. `engineWasConnected` handles the transition detection; `gameActive` handles the exactly-once guarantee. They do different things and must stay separate.

## Happens-before in the transport

There's an ordering requirement in `gameLoop()`'s finally block. `callback.onGameEnd(result)` must run before `running = false`. Within the game loop thread, program order guarantees `lastOutcome` is set before `running` is written. The volatile write to `running` then establishes happens-before with the scheduler thread's volatile read — so `lastOutcome` is visible to `gameTick()` by the time it observes `!isConnected()`. Reverse the order and you get a race where `gameTick()` reads UNKNOWN instead of the actual result.

## The cross-repo collateral

Running `@QuarkusTest` blew up with:

```
DeploymentException: MessageLedgerEntry declares persistent fields (channelId,
commitmentId, ...) but does not override domainContentBytes(). These fields
are not hash-protected.
```

`casehub-ledger`'s Quarkus deployment module validates every `LedgerEntry` subclass at augmentation time — any subclass with `@Column` fields must override `domainContentBytes()` to include them in the Merkle leaf hash. Nothing in `LedgerEntry`'s Javadoc hints at this. The base implementation returns `EMPTY_BYTES`, so unit tests pass silently; `@QuarkusTest` augmentation fires the validator. The fix is an override returning all fields pipe-joined — same format as `canonicalBytes()`.

That went into casehub-qhorus as a separate fix before the quarkmind tests would run.

---

The verdict mapping itself — WIN→ENDORSED, LOSS→CHALLENGED, TIE→SOUND, UNKNOWN→skip — is the simple part. The plumbing to get there was the work.
