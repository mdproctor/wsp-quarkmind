---
layout: post
title: "The Constant That Couldn't Be Wrong"
date: 2026-11-08
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [simulation, calibration, tick-quantisation, sc2-physics, digital-twin, emulated-game, measurement]
series: issue-534-emulated-game-100-accuracy
---

# The Constant That Couldn't Be Wrong

SC2's data editor stores physics constants at normal game speed. Unit speeds are documented at "faster" speed — the 1.4× multiplier that every competitive game runs at. Everyone knows this. Liquipedia reports Probe speed as 3.94 tiles per second. SC2Mapster's data editor shows 2.8125. Multiply by 1.4, get 3.94. Consistent.

Acceleration is also in the data editor: 2.5 units per second squared. Nobody mentions scaling it. Every community resource I found quotes 2.5 directly. So that's what QuarkMind used.

The per-loop velocity change tells the real story. At normal speed (16 Hz): 2.5 / 16 = 0.15625. At faster speed (22.4 Hz): 2.5 / 22.4 = 0.11161. A worker accelerating at 0.11161 per loop takes 35 loops to reach cruising speed. One accelerating at 0.15625 takes 25. Ten extra loops per leg, twenty per round trip, across every worker on every mineral patch for the entire game. That missing ×1.4 scaling halved the simulated mineral income.

Fixing it was one line: 2.5 → 3.5.

Nothing changed.

## The fix that did nothing

The mineral collection rate MAPE stayed exactly where it was. I re-ran the oracle replay baseline — 118 replays, 236 player runs — and the numbers hadn't moved. Workers were accelerating faster, reaching patches sooner, completing cycles more quickly, and the aggregate income was identical.

The simulation processes workers in ticks of 22 game loops. Each tick, a mineral worker can be in one of three states: walking to a patch, mining, or walking home. When it transitions between states — arrives at the patch, finishes mining, arrives at the town hall — it moves to the next state and waits for the next tick to continue.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 720 200" style="max-width:720px;width:100%">
  <defs>
    <marker id="arrow" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6" fill="#4a9eff"/>
    </marker>
  </defs>
  <rect x="20" y="60" width="680" height="80" rx="6" fill="#1a1a2e" stroke="#333" stroke-width="1"/>
  <text x="360" y="45" text-anchor="middle" fill="#999" font-family="monospace" font-size="12">One tick = 22 game loops</text>
  <!-- Walk phase -->
  <rect x="30" y="70" width="200" height="24" rx="4" fill="#2d5a27"/>
  <text x="130" y="87" text-anchor="middle" fill="#8fbc8f" font-family="monospace" font-size="11">walk to patch (19 loops)</text>
  <!-- Arrive - dead zone -->
  <rect x="230" y="70" width="40" height="24" rx="4" fill="#5a2727" stroke="#ff6b6b" stroke-width="1" stroke-dasharray="3,2"/>
  <text x="250" y="87" text-anchor="middle" fill="#ff6b6b" font-family="monospace" font-size="9">dead</text>
  <!-- Next tick label -->
  <line x1="270" y1="60" x2="270" y2="140" stroke="#555" stroke-width="1" stroke-dasharray="4,3"/>
  <text x="280" y="55" fill="#666" font-family="monospace" font-size="10">tick boundary</text>
  <!-- Mining in next tick -->
  <rect x="280" y="70" width="120" height="24" rx="4" fill="#2d4a5a"/>
  <text x="340" y="87" text-anchor="middle" fill="#87ceeb" font-family="monospace" font-size="11">mine (12 loops)</text>
  <!-- Dead zone after mining -->
  <rect x="400" y="70" width="50" height="24" rx="4" fill="#5a2727" stroke="#ff6b6b" stroke-width="1" stroke-dasharray="3,2"/>
  <text x="425" y="87" text-anchor="middle" fill="#ff6b6b" font-family="monospace" font-size="9">dead</text>
  <!-- Next tick boundary -->
  <line x1="450" y1="60" x2="450" y2="140" stroke="#555" stroke-width="1" stroke-dasharray="4,3"/>
  <!-- Return walk -->
  <rect x="460" y="70" width="200" height="24" rx="4" fill="#2d5a27"/>
  <text x="560" y="87" text-anchor="middle" fill="#8fbc8f" font-family="monospace" font-size="11">walk home (19 loops)</text>
  <!-- Dead zone -->
  <rect x="660" y="70" width="30" height="24" rx="4" fill="#5a2727" stroke="#ff6b6b" stroke-width="1" stroke-dasharray="3,2"/>
  <text x="675" y="87" text-anchor="middle" fill="#ff6b6b" font-family="monospace" font-size="9">dead</text>
  <!-- Bottom annotation -->
  <rect x="30" y="110" width="10" height="10" rx="2" fill="#5a2727" stroke="#ff6b6b" stroke-width="1" stroke-dasharray="3,2"/>
  <text x="48" y="119" fill="#ff6b6b" font-family="monospace" font-size="10">= loops wasted at tick boundaries (up to 3 per cycle)</text>
  <text x="360" y="160" text-anchor="middle" fill="#888" font-family="monospace" font-size="11">Before sub-tick chaining: transitions waste remainder of each tick</text>
</svg>

A worker arriving at a mineral patch on loop 19 of a 22-loop tick has 3 loops remaining. Those 3 loops are wasted — the worker stands at the patch doing nothing until the next tick starts. Then it begins mining. When mining finishes mid-tick, the same thing happens: the remaining loops are wasted before the return trip starts.

Three transitions per cycle. Up to one tick wasted at each. At 22 loops per tick and a cycle of roughly 100 loops, that's up to 15% of the worker's time evaporating at tick boundaries. The faster acceleration from the ×1.4 fix saved about 10 loops per cycle — but those 10 loops disappeared into the same tick-boundary gaps. The physics improvement was invisible inside the quantisation noise.

## Chaining within the tick

The movement code already tracked loops precisely — `moveLoops(int)` returns the number of loops consumed before the worker reaches its target. If a worker arrives at loop 19 with 3 loops remaining, those 3 loops are available for the next phase.

The mining phase didn't work this way. It checked whether the current time exceeded the mining end time. If not, it returned the entire remaining tick unused. A 5-loop mining animation starting at loop 19 would report "not done" at loop 19 (correct — end time is loop 24), then check again at loop 0 of the next tick (loop 22): "done" at loop 22 (also correct — 22 > 24 would be true if the end time tracked properly). But the worker consumed 22 loops to complete a 5-loop mining phase.

The fix: after `moveLoops` returns, calculate how many loops remain in the tick. If mining would complete within those loops, complete it immediately and chain into the return trip. Same pattern at every transition.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 720 200" style="max-width:720px;width:100%">
  <rect x="20" y="60" width="680" height="80" rx="6" fill="#1a1a2e" stroke="#333" stroke-width="1"/>
  <text x="360" y="45" text-anchor="middle" fill="#999" font-family="monospace" font-size="12">Same tick = 22 game loops — sub-tick chaining</text>
  <!-- Walk phase -->
  <rect x="30" y="70" width="200" height="24" rx="4" fill="#2d5a27"/>
  <text x="130" y="87" text-anchor="middle" fill="#8fbc8f" font-family="monospace" font-size="11">walk to patch (19 loops)</text>
  <!-- Arrow showing chain -->
  <line x1="230" y1="82" x2="238" y2="82" stroke="#4a9eff" stroke-width="2" marker-end="url(#arrow)"/>
  <!-- Mining chained -->
  <rect x="240" y="70" width="30" height="24" rx="4" fill="#2d4a5a"/>
  <text x="255" y="87" text-anchor="middle" fill="#87ceeb" font-family="monospace" font-size="9">3</text>
  <!-- Tick boundary -->
  <line x1="270" y1="60" x2="270" y2="140" stroke="#555" stroke-width="1" stroke-dasharray="4,3"/>
  <text x="280" y="55" fill="#666" font-family="monospace" font-size="10">tick boundary</text>
  <!-- Mining continues -->
  <rect x="280" y="70" width="80" height="24" rx="4" fill="#2d4a5a"/>
  <text x="320" y="87" text-anchor="middle" fill="#87ceeb" font-family="monospace" font-size="11">mine (9 loops)</text>
  <!-- Arrow showing chain -->
  <line x1="360" y1="82" x2="368" y2="82" stroke="#4a9eff" stroke-width="2" marker-end="url(#arrow)"/>
  <!-- Return walk chained into same tick -->
  <rect x="370" y="70" width="160" height="24" rx="4" fill="#2d5a27"/>
  <text x="450" y="87" text-anchor="middle" fill="#8fbc8f" font-family="monospace" font-size="11">walk home (13 loops)</text>
  <!-- Next tick -->
  <line x1="540" y1="60" x2="540" y2="140" stroke="#555" stroke-width="1" stroke-dasharray="4,3"/>
  <!-- Continue walk -->
  <rect x="550" y="70" width="60" height="24" rx="4" fill="#2d5a27"/>
  <text x="580" y="87" text-anchor="middle" fill="#8fbc8f" font-family="monospace" font-size="11">6 loops</text>
  <!-- Arrow -->
  <line x1="610" y1="82" x2="618" y2="82" stroke="#4a9eff" stroke-width="2" marker-end="url(#arrow)"/>
  <!-- Deposit chained -->
  <rect x="620" y="70" width="60" height="24" rx="4" fill="#5a4a2d"/>
  <text x="650" y="87" text-anchor="middle" fill="#daa520" font-family="monospace" font-size="9">deposit+</text>
  <!-- Bottom annotation -->
  <line x1="230" y1="100" x2="270" y2="100" stroke="#4a9eff" stroke-width="2" marker-end="url(#arrow)"/>
  <text x="280" y="105" fill="#4a9eff" font-family="monospace" font-size="10">chains carry remaining loops into next phase</text>
  <text x="360" y="160" text-anchor="middle" fill="#888" font-family="monospace" font-size="11">After: zero dead loops — every transition chains immediately</text>
</svg>

This one change made the acceleration fix visible. The mineralsCollectionRate MAPE dropped — but it didn't drop to where it should have been. It dropped *too far*. The simulation was now producing 65% more income than real SC2 at the 1-minute checkpoint.

## The calibration that follows

With tick boundaries no longer absorbing errors, every constant had to be re-examined. The mining duration — previously set to 5 game loops based on an empirical sweep — was producing the right MAPE only because the tick boundary overhead was compensating for a too-short animation. With that overhead gone, 5 loops was far too fast.

A sweep across 118 oracle replays at 13 different mining durations found the new optimum: 12 loops. But 12 loops is 0.54 seconds, and community sources cite mining at roughly 1 second. So I connected to a real SC2 instance via the protobuf API, started a game, and tracked a Probe's position every single game loop for 3000 frames.

The stationary period at mineral patches — the actual mining animation, position unchanged frame to frame — was 5 loops. Median: 5 loops. Mean: 6.1 loops. Not 1 second. Not 12 loops. Five.

The 7-loop gap between measured reality (5 loops) and the calibrated constant (12 loops) isn't error. It's the sum of everything the simulation doesn't model: 2-3 loops of deposit time at the town hall (visible in the measurement data as stationary periods at the Nexus position), 2-3 loops of approach and docking delay at the mineral patch, and occasional path contention between workers. The mining constant absorbs these because the simulation treats deposit as instant, approach as immediate, and pathing as obstacle-free.

## The lesson is two-layered

The acceleration constant was wrong. Fixing it alone changed nothing. The simulation granularity was too coarse to express the improvement — the tick boundary quantisation ate every loop the faster acceleration saved.

This is a general pattern in any tick-based simulation. A physics constant and the simulation's time resolution interact multiplicatively. If the resolution is coarse enough, the constant doesn't matter — any value within a range produces the same output. You calibrate the constant, find an optimum, and the optimum is an artefact of the quantisation, not the physics.

The corrected acceleration was hiding inside the quantisation. The corrected mining duration was compensating for the quantisation. Both were "calibrated" — both matched the oracle MAPE — and neither reflected reality. Only when the quantisation was fixed did the physics constants become meaningful.

If you're building a simulation — game AI, a digital twin, a physics model — and your calibration sweeps show wide plateaus of identical output, check your time resolution. The constant isn't wrong. The simulation can't see it.
