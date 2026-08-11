---
layout: post
title: "The Bot That Sees the Whole Board"
date: 2026-07-06
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [dominance, trust-scoring, milestones, fog-of-war]
series: issue-223-multi-factor-dominance
---

Supply count was always a placeholder. It tells you who has more stuff on the field, but nothing about whether that stuff is any good. A player with thirty zerglings and no tech is behind a player with four immortals and a robotics bay, even if the supply numbers say otherwise.

The new `DominanceAssessor` looks at four things: how fast each side is gathering minerals (worker count, not bank — you can't see the enemy's bank), the total resource cost of each army (excluding workers, since they're already counted in economy), how deep each side has gone into the tech tree, and how many bases are running. Each factor produces a score from -1 to +1, and a weighted sum gives the overall dominance signal that milestone attestations use for intermediate trust scoring.

The interesting design question was what to compare. The first draft compared own bank+income against enemy income — which is structurally biased positive because your bank is always ≥ 0 and the enemy's bank is invisible. The design review caught this in round 2. The fix was obvious once named: compare income rates only, both sides using the same formula. Symmetric inputs, symmetric output.

The fog-of-war problem is harder. Every factor compares full-information own state against partial-information enemy state. Early game, you might see three enemy combat units and zero buildings — enough to pass a minimum visibility threshold, but the tech and base factors would peg at +1.0 because they'd be comparing your buildings against zero. The fix is two layers: a combined threshold that returns neutral when total enemy visibility is too sparse, and per-factor guards that return neutral when the specific input list for that factor is empty. Economy returns neutral when no enemy workers are visible, because seeing only combat units tells you nothing about how many probes they have.

Tech tier scoring needed a mapping from `BuildingType` to tier. A Gateway is T1; a Fleet Beacon is T4. The score is `maxTier + 0.1 × breadth` — reaching a higher tier matters more than building wide at the same tier. `BuildingType` values are already race-specific, so no race parameter is needed. Only complete buildings count — a half-built robotics bay doesn't give you immortals.

The `DominanceScore` record carries the overall score plus a per-factor breakdown map. Callers can see *why* the score is what it is — "dominating economically but behind in army value" is a different signal than a single number averaging them out. When milestone attestations eventually activate (blocked on engine#648), this breakdown will give trust scoring richer context than a bare scalar.

The weights are uncalibrated defaults — economy 0.30, army 0.35, tech 0.20, bases 0.15. They're configurable, and the next step is making them phase-adaptive once replay data shows where fixed weights break down. Map control is the missing fifth factor — it needs base-location clustering that doesn't exist yet.
