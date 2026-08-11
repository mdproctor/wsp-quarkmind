---
layout: post
title: "Arc42Stories and a Lighter Ledger"
date: 2026-06-02
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [arc42stories, casehub-ledger, trust-routing, architecture]
---

Two things landed today that change how the next few months look.

The first started with a simple question: do layers 4 and 6 (ledger integration and trust routing) have practical uses for a game agent, or are they just demonstrating the harness pattern for its own sake? I asked Claude to dig into what casehub-ledger actually offers.

The answer was not what I expected. The repo is heavily enterprise — hash chains, DID resolution, JWT VCs, KMS adapters. All compliance-grade machinery. Bayesian Beta trust scoring is in there (ADR-0003), but the full stack bundles cryptographic signing and tamper-evident ledger entries that a game agent simply does not need.

The practical point: QuarkMind does not need tamper-evident records of game AI decisions. Game loop intent logs are not audit artifacts. What we do need is the Bayesian Beta model — outcome history accumulated per plugin, trust scores improving routing over time. So requiring the full cryptographic stack is the wrong gate.

The better design is to let casehub-ledger itself solve this — add a lightweight mode where signing and hashing are optional. Callers opt into simple append semantics; Bayesian Beta scoring still works. I drafted the requirements from QuarkMind's side and we filed casehubio/ledger#114. The implementation approach is deliberately left to ledger R&D. Both #156 and #158 are now blocked on it.

The second thing was issue #166 — bootstrapping `ARC42STORIES.MD` and retiring the tutorial framing that has been in CLAUDE.md since the harness layers were first sketched out. That framing was wrong from the start. QuarkMind is a reference implementation, not a teaching sequence. Arc42stories is the right format: it documents what was actually built and what each layer adds, in reading order, without prescribing a delivery sequence.

Prerequisite 1 of #166 was removing all tutorial language from CLAUDE.md. Done. Then Claude and I wrote the full `ARC42STORIES.MD` — §1 through §13, L1 (conceptual baseline) and L2 (casehub-engine) as complete layer entries, L3 through L7 as stubs pointing at the relevant pending issues.

The most important thing to get right in the document was the CDI displacement pattern. QuarkMind's mechanism is `@CaseType("starcraft-game")` qualification — not `@DefaultBean` displacement like devtown. There is no always-present naive baseline bean. Each layer ships exactly one active implementation per seam, selected by qualifier match. If someone reads the devtown reference and applies it mechanically here, they will build the wrong thing. Getting that documented explicitly was the main architectural work.

The harness/SC2 boundary is also explicit now, which it was not before. `ARC42STORIES.MD` covers the harness only. `EmulatedGame`, `RaceModel`, and `ReplayValidationHarness` are out of scope for the architecture record. They're documented in `DESIGN.md` and belong there.

`epic-saturation-mining` still needs a closure stamp. It's past 14 days.
