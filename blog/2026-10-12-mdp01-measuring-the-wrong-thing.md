---
layout: post
title: "Measuring the Wrong Thing"
date: 2026-10-12
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [emulation, validation, methodology, sc2]
---

# Measuring the Wrong Thing

I spent several sessions calibrating EmulatedGame — the physics engine that
simulates SC2 mechanics without needing the real game client. The goal was
straightforward: feed replay data into the emulator, compare output against
ground truth, fix every discrepancy.

Unit accuracy climbed from 13% to 77%. Buildings hit 100%. Upgrades went from
zero to 58%. Each fix was clean — cosmetic tracker events inflating upgrade
counts, case-sensitive map keys silently dropping intents, morph lookups failing
because replay-derived tags never match auto-generated ones. Every fix had a
clear root cause and a measurable improvement.

Then it stopped moving.

The remaining gap was workers, queens, combat units — the bread and butter of
any SC2 game. We'd extract the production command from the replay, feed it to
the emulator, and the unit would appear. Except the Blizzard replay API only
captures about 43% of production commands. The other 57% simply aren't in the
data. No amount of physics calibration fixes missing inputs.

This is the methodological trap: the validation harness conflates two things.
Input completeness — how many commands we feed the emulator — and physics
accuracy — whether the emulator handles each command correctly. At 43% input
coverage, a physics-perfect emulator would still measure around 80%. We'd been
tuning constants and fixing edge cases against a ceiling imposed by the data,
not by the engine.

The fix isn't more calibration. It's a different measurement approach. Oracle
replays contain tracker events — the ground truth record of every unit born,
every building completed, every upgrade finished, with exact timestamps. Instead
of feeding the emulator incomplete commands and blaming it for incomplete output,
feed it the complete production record from tracker events and measure whether
the physics produce the right unit at the right tick.

That isolates the question. Given perfect inputs, is the emulator correct? Every
remaining discrepancy is a real physics bug — a wrong train time, a broken
mechanic, an income model that diverges. No more tuning against noise.

The pattern generalises beyond SC2. Any time you're validating a simulation
against real-world data, check whether your test harness measures the thing you
think it measures. A 77% accuracy number that can't get higher isn't necessarily
a 23% physics problem — it might be a 23% data problem wearing physics clothing.
