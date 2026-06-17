---
title: "The attribution errors you don't see until you verify"
date: 2026-06-17
author: mdp
projects: [quarkmind]
tags: [arc42, architecture-docs, casehub]
---

Writing four chapter entries for an architecture document turned out to be mostly a verification problem.

The §9.4 layer entries in ARC42STORIES.MD describe what each integration layer delivered — casehub-qhorus, casehub-ledger, adaptive plugin selection, trust routing. The §9.3 chapter entries are supposed to narrate what each chapter introduced architecturally, drawing on those layer entries as source material. Simple enough in principle.

We wrote the spec first and reviewed it five times. Four of those rounds found real errors.

The most instructive mistake was attributing `GameOutcomeRecorder` to C3 (the ledger integration chapter). The error made sense: both C3 and C5 write to casehub-ledger, both classes are in `io.quarkmind.agent`, both call `outcomeRecorder.record()`. But `GameOutcomeRecorder` observes `GameStopped` — it's a per-game strategy outcome recorder, introduced in the trust routing layer (C5). The actual C3 class is `PluginOutcomeAuditor`, which observes `PluginDecisionEvent` asynchronously and records per-tick plugin decisions.

The spec I'd written also used `OutcomeRecord.of(strategyId, context, verdict)` — three arguments. The real signature has five. The class name wasn't wrong in isolation; it was in the wrong chapter, and the API I cited didn't exist.

The fix: look at what each class actually observes. `@ObservesAsync PluginDecisionEvent` — that's C3. `@Observes GameStopped` — that's C5. The CDI event is a reliable signal in a way that the class name alone is not.

A similar thing happened with `StrategySelector`. I'd placed the `strategySelector.isSelected()` gate in C4's chapter entry, describing it as part of the adaptive dispatch mechanism. C4 is where the `activateIf()` pattern was established. But `StrategySelector` was introduced in the same commit that created `StrategyTrustRouter`, `StrategyTrustObserver`, and two competing strategy implementations. Before that commit there was one `StrategyTask`; the gate had nothing to discriminate between. It belongs to C5.

`git log -- src/main/java/io/quarkmind/agent/StrategySelector.java` gives one commit. That's the only reliable authority. Layer entries are snapshots written at layer close time — they can lag the code.

The §9.2 Layer × Chapter matrix was stale in two places. L2 C4 said Low when the chapter entry said Medium. L4/L6 C5 said Low/Medium when it should have been Medium/High. The matrix predates the chapter entries; when they disagree, the matrix is wrong.

None of this is surprising individually. What struck me is how consistent the pattern was. The errors weren't in the creative parts — the framing, the prose, the structure. They were in the attribution details. Which class does what. Which commit introduced it. Which chapter it belongs to. The things that look like facts because they name real classes and real commits, but turn out to be slightly off in exactly the way that compounds across six related chapters.

The practical lesson: for any named class in an architecture doc, the CDI event it observes or the git commit that introduced it is a more reliable locator than the layer entry that mentions it.
