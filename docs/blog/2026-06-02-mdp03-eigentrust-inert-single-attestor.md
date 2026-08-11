---
layout: post
title: "EigenTrust Has No Work to Do Here"
date: 2026-06-02
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [trust-routing, casehub-ledger, casehub-engine-ledger, adr]
---

The question has been waiting since casehubio/ledger#114 was filed: when L4 and L6
land, should EigenTrust run alongside Bayesian Beta?

No. The math settles it before implementation begins.

EigenTrust propagates trust through a peer attestation matrix — `C[i][j]` is how much
actor `i` trusts actor `j`, normalised from direct interactions. In QuarkMind, the only
attestor is `AgentOrchestrator`. It observes all four plugins and records outcomes. No
plugin attests to any other. The harness row of `C` is the only non-zero row; all four
plugin rows are zero.

With every plugin row zeroed, EigenTrust's dangling-node fallback fires for all four
actors simultaneously on every iteration. Power iteration oscillates rather than
converges. Four actors is well within the pathology zone documented in GE-20260421-09d636
— the exact scenario where the Stanford paper's recommended fallback creates 3-cycles.

So Bayesian Beta only. But the ADR draft got something wrong, and Claude caught it during
review.

The initial version said "Bayesian Beta only — no casehub-engine-ledger dependency." That
is not accurate. `TrustWeightedAgentStrategy` — the routing component that reads trust
scores and dispatches between competing plugin implementations — lives in
`casehub-engine-ledger`. L6 needs the module. What QuarkMind avoids is not
`casehub-engine-ledger` but EigenTrust computation specifically.

The module bundles two independent concerns: `TrustWeightedAgentStrategy` (reads trust
scores, routes dispatch) and `TrustScoreJob` (computes EigenTrust GLOBAL scores nightly).
The strategy dispatches on CAPABILITY scores — Bayesian Beta per-plugin ratios accumulated
by `casehub-ledger`. GLOBAL scores are computed by `TrustScoreJob` but not consumed
anywhere in the capability routing path. EigenTrust runs. Its output goes nowhere. Inert,
not absent.

ADR-0009 records the distinction: L4 activates `casehub-ledger` runtime only (Bayesian
Beta starts accumulating). L6 adds `casehub-engine-ledger` for routing (EigenTrust job
runs, GLOBAL scores produced, not read). Revisit if peer attestation develops or the
actor pool exceeds ~50.
