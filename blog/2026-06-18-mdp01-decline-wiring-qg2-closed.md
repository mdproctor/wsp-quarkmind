---
layout: post
title: "DECLINE wired: QG2 finally closed"
date: 2026-06-18
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [qhorus, speech-act, decline, testing, caseengine]
series: issue-200-docs-and-decline
---

The branch closes four issues — three small doc fixes and one that needed thinking.

\#200 was a pair of wrong dependency claims in ARC42STORIES.MD's sequencing rationale. The C3→C4 explanation said "adaptive binding selection reads ledger attestations" when C4 actually uses `requires()` key-presence and broker state gates; ledger data is what C5 reads. Four sentences, two commits, done. \#196 and \#197 were similarly fast: the plugin-guide had `activateIf()` re-checking a key already covered by `requires()`, which I fixed with a proper example using `strategySelector.isSelected()` and `broker.current(POSTURE).isPresent()`. The null-drop in `CaseFileContext.set()` took one Javadoc comment.

\#199 took more.

Quality Goal 2 has been in the architecture since the beginning: a plugin outside its scope declares it structurally — no silent no-op. The DECLINE speech act in casehub-qhorus is the mechanism. Wiring it is not straightforward because there is no COMMAND infrastructure in QuarkMind's game loop — `createAndSolve()` is a direct call that returns before plugins even finish executing.

To have DECLINE, you first need COMMAND. DECLINE is a commitment reply: it needs `inReplyTo` pointing to the prior COMMAND's message ID, plus a matching `correlationId`. So the question became: what is the COMMAND, and when does it fire?

The approach we settled on: a new `PluginDispatchBroker` CDI bean evaluates each registered plugin's activation state from the tick's `caseData` snapshot, before `createAndSolve()` runs. When activation state changes — first tick, or a plugin transitions active↔inactive — it sends COMMAND+DONE or COMMAND+DECLINE on the `quarkmind-plugin-dispatch` channel. Transitions only, not every tick.

Timing mattered. `createAndSolve()` submits the control loop to a cached thread pool and returns before plugins run. Reading from the CaseFile after the call means evaluating activation against data a worker thread is concurrently writing — a real race. The `caseData` map is immutable by the time `recordTick()` runs, so we use `MapCaseContext(caseData)` for the evaluation and the race disappears.

`MapCaseContext` is worth noting independently: a read-only `CaseContext` backed by a `Map<String, Object>`. Tests that used to construct a CaseFile through `InMemoryCaseFileRepository` just to get a `CaseFileContext` can use `new MapCaseContext(Map.of(...))` instead — no transaction, no factory.

One subtlety in the implementation: `priorActivation` (the in-memory per-plugin activation state) and `lastDispatchedId` (a DB cursor for test isolation) must stay consistent with committed DB state. If `MessageService.dispatch()` throws mid-loop and the transaction rolls back, the in-memory state has to be unchanged — otherwise the next tick sees "no change" and suppresses the signal. The fix is collect-then-apply: accumulate updates in local collections, apply to the `ConcurrentHashMap` only after all dispatches succeed.

Two other things came up that weren't in the plan.

`TestActorIdentityProvider` had been broken since the ledger SPI migration. `ActorIdentityProvider` moved from `io.casehub.ledger.runtime.privacy` to `io.casehub.ledger.api.spi`, and `tokeniseForQuery()` return type changed from `String` to `Optional<String>`. The error shows as `cannot find symbol` on the import, then a second error on `tokeniseForQuery` that looks unrelated. Two independent breaking changes in one migration; the second only surfaces after you fix the first.

There was also a pre-existing ordering failure in `AdaptivePluginSelectionIT`. `DroolsScoutingTask` runs on the CaseEngine worker thread after `gameTick()` returns, and on test boundaries it can finish and update `ScoutingIntelBroker` after `@BeforeEach`'s `clearLatest()` has already run. A 50ms settle in `@AfterEach` before the clear is enough — plugins complete well within that.
