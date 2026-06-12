---
layout: post
title: "Trust Routing is Degenerate CBR"
date: 2026-06-12
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [casehub, cbr, trust-routing, drools, architecture]
---

The question going in: is QuarkMind actually using CaseHub, or just wearing it as a label? We'd built six layers — connection, physics, the plugin pipeline, scouting, economics, and trust-weighted strategy routing — but I wanted to know how much of the foundation we were genuinely exercising.

The answer, once we pulled the code up, was less flattering than I expected.

`createAndSolve("starcraft-game", data, timeout)` — that's the main touchpoint. It finds all `@CaseType("starcraft-game")` beans, calls `canActivate()` on each, and runs the ones that qualify. That's a plugin dispatch loop. Twenty lines of Java could replace it. The CaseHub routing strategies — `TrustWeightedAgentStrategy`, `SemanticAgentRoutingStrategy`, `CapabilityHealth` probing — are all bypassed because the modules that activate them aren't on the classpath.

Worse: `StrategyTrustRouter`, `StrategySelector`, and `StrategyTrustObserver` — 228 lines of application-layer code — are doing the work that `TrustWeightedAgentStrategy` should own. We reimplemented routing at the application layer rather than wiring the foundation to do it.

That's the core problem. Not that the code is wrong — L6 trust routing works, and the Bayesian Beta four-phase model produces the right selections — but that it's in the wrong place.

We got into a side discussion about Drools. The current RuleUnit API is a wrapper around a `KieSession`, and the classloader isolation it creates is the reason our DRL rules communicate via strings (`"GATEWAY"`, `"STALKER:tag123"`) rather than real types. A new version of Drools is coming that makes RuleUnits first-class rather than a wrapper, which clears the constraint. Removing the Quarkus Drools extension's CDI injection would probably solve it in the interim — the issue is augmentation-time reflection, not anything in Drools itself. We didn't implement it this session, but the path is clear.

The bigger insight came while thinking about what trust routing actually is.

Every agentic system with outcome-based routing is running degenerate CBR — Case-Based Reasoning with two of four steps active. Retain (ledger attestations) and Reuse (trust-weighted routing) are there. Retrieve and Revise are missing.

The distinction matters. With trust scoring and exact-key lookup, the system improves when it encounters identical situations. It cannot generalise. A game against Zerg roach-rush yields no useful signal for a Zerg mass-zergling game, even though the strategic response is nearly identical. The key doesn't match, so the experience doesn't transfer.

Full CBR fixes this with a feature vector and a similarity function. The same pattern applies to AML investigations, clinical adverse events, and PR reviews — all of them maintain outcome histories, none of them retrieve by similarity.

We wrote this up as a platform capability document in casehub-parent, updated the Platform Coherence Protocol, and filed ten issues across seven repos — engine, neural-text, platform, ledger, and the application repos — with a parent epic tracking four implementation waves. The engine issues are the load-bearing ones: a lifecycle hook for Retain, a retriever integration for Retrieve, and an `ImplementationRoutingStrategy` SPI for Reuse. Wave 1 starts with the two that have no inter-CBR dependencies: a structured case-entry type in casehub-platform and a batch scoring API in casehub-ledger.

QuarkMind is the reference implementation. If CBR works at 500ms game-loop tick rate, it works everywhere.
