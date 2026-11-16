---
layout: post
title: "The Bugs That Cancelled Out"
date: 2026-11-16
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [economy, worker-simulation, emulated-game, calibration, sub-tick, physics]
---

# The Bugs That Cancelled Out

The emulated game's 12-worker mineral income has matched SC2 within 3% for weeks. Every calibration run confirmed it. The physics constants were measured, the collision model was tuned, the approach docking was set from per-worker observation data. The economy worked.

Then I recorded an 8-worker baseline.

At 8 workers --- one per mineral patch, zero contention --- the emulator over-produced by 17%. Not close. Not within tolerance. Seventeen percent faster than the real game, on an economy path with no contention, no collisions, no saturation effects. Just workers walking to patches and back.

The 12-worker accuracy was a lie. Two bugs were cancelling each other out.

## The Measurement

The 8-worker baseline is the simplest economy scenario SC2 offers. Twelve starting workers, kill four via the debug API at tick zero, let eight mine for five minutes. No training, no gas, no buildings. The per-worker rate is unambiguous --- there's exactly one worker per mineral patch, so every cycle is uncontested.

SC2 produced 2,330 minerals. The emulator produced 2,730. Per-worker cycle time: SC2 averaged 117 loops, the emulator 100. Every worker, every cycle, 17 loops faster than real SC2.

I needed to understand where those 17 loops were going. So I recorded the SC2 run with per-tick worker cadence data --- position, action, and cargo state for every worker at every tick --- and decomposed the cycle.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 700 280">
  <defs>
    <style>
      text { font-family: -apple-system, 'Segoe UI', sans-serif; font-size: 12px; }
      .title { font-size: 14px; font-weight: bold; }
      .label { font-size: 11px; fill: #555; }
      .val { font-size: 11px; font-weight: bold; }
    </style>
  </defs>
  <text x="350" y="20" text-anchor="middle" class="title">Mining Cycle Phases (loops per cycle)</text>

  <!-- EM cycle -->
  <text x="10" y="75" class="label">EM (before)</text>
  <rect x="90" y="55" width="120" height="30" fill="#4a90d9" rx="3"/>
  <text x="150" y="75" text-anchor="middle" fill="white" class="val">walk 60</text>
  <rect x="210" y="55" width="46" height="30" fill="#e8913a" rx="3"/>
  <text x="233" y="75" text-anchor="middle" fill="white" class="val">11.5</text>
  <rect x="256" y="55" width="88" height="30" fill="#d94a4a" rx="3"/>
  <text x="300" y="75" text-anchor="middle" fill="white" class="val">mine 22</text>
  <rect x="344" y="55" width="28" height="30" fill="#6ab04c" rx="3"/>
  <text x="358" y="75" text-anchor="middle" fill="white" class="val">7</text>
  <text x="395" y="75" class="val" fill="#333">= 100.5 loops</text>

  <!-- SC2 cycle -->
  <text x="10" y="135" class="label">SC2 (measured)</text>
  <rect x="90" y="115" width="120" height="30" fill="#4a90d9" rx="3"/>
  <text x="150" y="135" text-anchor="middle" fill="white" class="val">walk 60</text>
  <rect x="210" y="115" width="98" height="30" fill="#e8913a" rx="3"/>
  <text x="259" y="135" text-anchor="middle" fill="white" class="val">approach 24.5</text>
  <rect x="308" y="115" width="88" height="30" fill="#d94a4a" rx="3"/>
  <text x="352" y="135" text-anchor="middle" fill="white" class="val">mine 22</text>
  <rect x="396" y="115" width="48" height="30" fill="#6ab04c" rx="3"/>
  <text x="420" y="135" text-anchor="middle" fill="white" class="val">dep 12</text>
  <rect x="444" y="115" width="16" height="30" fill="#888" rx="3"/>
  <text x="475" y="135" class="val" fill="#333">= 117 loops</text>
  <text x="452" y="155" class="label">+2 transition</text>

  <!-- Legend -->
  <rect x="90" y="185" width="14" height="14" fill="#4a90d9" rx="2"/>
  <text x="110" y="196" class="label">Walk (2 legs)</text>
  <rect x="200" y="185" width="14" height="14" fill="#e8913a" rx="2"/>
  <text x="220" y="196" class="label">Approach/dock</text>
  <rect x="320" y="185" width="14" height="14" fill="#d94a4a" rx="2"/>
  <text x="340" y="196" class="label">Mining animation</text>
  <rect x="450" y="185" width="14" height="14" fill="#6ab04c" rx="2"/>
  <text x="470" y="196" class="label">Deposit</text>
  <rect x="540" y="185" width="14" height="14" fill="#888" rx="2"/>
  <text x="560" y="196" class="label">Transition cost</text>

  <!-- Gap annotation -->
  <line x1="372" y1="55" x2="372" y2="47" stroke="#c00" stroke-width="1.5"/>
  <line x1="460" y1="115" x2="460" y2="47" stroke="#c00" stroke-width="1.5"/>
  <line x1="372" y1="47" x2="460" y2="47" stroke="#c00" stroke-width="1.5"/>
  <text x="416" y="42" text-anchor="middle" fill="#c00" class="val">+17 loops</text>
</svg>

The outbound half-cycle --- deposit to pickup --- was 12 loops shorter in the emulator than SC2. The return half was 3 loops shorter. The walk distances were the same. The mining animation was the same 22 loops. The gap was entirely in overhead: the approach phase and the deposit phase.

## The Sub-Tick Chaining Bug

But when I increased the approach docking constant to match the SC2 measurement, the emulator produced the right result at 8 workers but over-corrected at 12 and 16. The constant slowed all cycles equally. Whatever was making the 8-worker cycle too fast had to be something that differentially affected low-saturation scenarios.

I instrumented the simulation to trace per-loop phase transitions. The answer was in `stepOneMineralLoop`:

```java
while (!loopConsumed) {
    if (w.task == MINING_MINERALS) {
        if (currentLoop >= w.phaseEndLoop) {
            w.cargo = harvest();
            w.task = RETURNING_MINERALS;   // transition
            continue;                       // ← zero cost
        }
    }
    if (w.task == RETURNING_MINERALS) {
        w.moveOneLoop();
        loopConsumed = true;               // walk consumes the loop
    }
}
```

The `continue` after the mining-to-returning transition fed back into the while loop without setting `loopConsumed`. The next iteration processed the returning phase, called `moveOneLoop`, and consumed the loop there. One game loop did double duty: it completed the mining phase *and* took the first walk step. The transition itself was free.

The same pattern applied to the depositing-to-mining transition. Two zero-cost transitions per cycle, saving 2 loops directly. But the compounding effect was larger than that, because mid-tick phase transitions let the next phase start with the remaining loops --- effectively shifting all downstream timing earlier by a few loops. The net savings was around 10 loops per cycle.

SC2 doesn't work this way. Each phase transition happens at a game-loop boundary. When a worker finishes mining, the engine processes the completion, and the worker starts its return walk on the *next* game loop. That loop is consumed by the transition --- the worker doesn't get a free movement step.

## The Compensating Bug

This sub-tick chaining affected all saturation levels equally. At 8 workers, it made every cycle 10 loops too fast, producing the 17% surplus. At 12 workers, it did the same --- but the collision speed factor (0.96) was slowing walks down just enough to compensate. Two workers passing each other near a mineral patch would each lose 4% of their speed for that loop. Enough overlapping collisions across a 12-worker mineral line, and the aggregate slowdown matched the sub-tick speedup almost exactly.

The collision model had been calibrated against 12-worker baselines. It was tuned to produce the right total, but it was doing so by compensating for a different bug. Remove the sub-tick chaining and the collision model over-corrects. Keep the sub-tick chaining and the 8-worker rate stays 17% too high. The two bugs were load-bearing against each other.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 620 300">
  <defs>
    <style>
      text { font-family: -apple-system, 'Segoe UI', sans-serif; font-size: 12px; }
      .title { font-size: 14px; font-weight: bold; }
      .label { font-size: 11px; }
      .val { font-size: 11px; font-weight: bold; }
      .axis { stroke: #666; stroke-width: 1; }
      .grid { stroke: #e0e0e0; stroke-width: 0.5; }
      .zero { stroke: #999; stroke-width: 1; stroke-dasharray: 4,3; }
      .bar-before { fill: #d94a4a; }
      .bar-after { fill: #4a90d9; }
      .threshold { stroke: #c00; stroke-width: 1; stroke-dasharray: 6,3; }
    </style>
  </defs>
  <text x="310" y="22" text-anchor="middle" class="title">EM vs SC2 Mineral Income Gap (%)</text>

  <!-- Y axis -->
  <line x1="80" y1="50" x2="80" y2="250" class="axis"/>
  <!-- X axis at 0% -->
  <line x1="80" y1="160" x2="570" y2="160" class="zero"/>

  <!-- Grid lines -->
  <line x1="80" y1="72" x2="570" y2="72" class="grid"/>
  <text x="75" y="76" text-anchor="end" class="label">+20%</text>
  <line x1="80" y1="94" x2="570" y2="94" class="grid"/>
  <text x="75" y="98" text-anchor="end" class="label">+15%</text>
  <line x1="80" y1="116" x2="570" y2="116" class="grid"/>
  <text x="75" y="120" text-anchor="end" class="label">+10%</text>
  <line x1="80" y1="138" x2="570" y2="138" class="grid"/>
  <text x="75" y="142" text-anchor="end" class="label">+5%</text>
  <text x="75" y="164" text-anchor="end" class="label">0%</text>
  <line x1="80" y1="182" x2="570" y2="182" class="grid"/>
  <text x="75" y="186" text-anchor="end" class="label">-5%</text>

  <!-- +5% / -5% thresholds -->
  <line x1="80" y1="138" x2="570" y2="138" class="threshold"/>
  <line x1="80" y1="182" x2="570" y2="182" class="threshold"/>
  <text x="575" y="142" class="label" fill="#c00">+5%</text>
  <text x="575" y="186" class="label" fill="#c00">-5%</text>

  <!-- 8w bars -->
  <rect x="110" y="84.4" width="40" height="75.6" class="bar-before" rx="2"/>
  <text x="130" y="80" text-anchor="middle" class="val" fill="#d94a4a">+17.2%</text>
  <rect x="155" y="141" width="40" height="19" class="bar-after" rx="2"/>
  <text x="175" y="137" text-anchor="middle" class="val" fill="#4a90d9">+4.3%</text>
  <text x="152" y="268" text-anchor="middle" class="label">8w</text>

  <!-- 12w bars -->
  <rect x="240" y="148.2" width="40" height="11.8" class="bar-before" rx="2"/>
  <text x="260" y="144" text-anchor="middle" class="val" fill="#d94a4a">+2.7%</text>
  <rect x="285" y="160" width="40" height="11.9" class="bar-after" rx="2"/>
  <text x="305" y="178" text-anchor="middle" class="val" fill="#4a90d9">-2.7%</text>
  <text x="282" y="268" text-anchor="middle" class="label">12w</text>

  <!-- 16w bars -->
  <rect x="370" y="160" width="40" height="5.3" class="bar-before" rx="2"/>
  <text x="390" y="178" text-anchor="middle" class="val" fill="#d94a4a">-1.2%</text>
  <rect x="415" y="160" width="40" height="7.5" class="bar-after" rx="2"/>
  <text x="435" y="178" text-anchor="middle" class="val" fill="#4a90d9">-1.7%</text>
  <text x="412" y="268" text-anchor="middle" class="label">16w</text>

  <!-- 24w bars -->
  <rect x="500" y="152.1" width="40" height="7.9" class="bar-before" rx="2"/>
  <text x="520" y="148" text-anchor="middle" class="val" fill="#d94a4a">+1.8%</text>
  <rect x="545" y="160" width="40" height="11.9" class="bar-after" rx="2"/>
  <text x="565" y="178" text-anchor="middle" class="val" fill="#4a90d9">-2.7%</text>
  <text x="542" y="268" text-anchor="middle" class="label">24w</text>

  <!-- Legend -->
  <rect x="185" y="282" width="14" height="14" class="bar-before" rx="2"/>
  <text x="204" y="294" class="label">Before</text>
  <rect x="260" y="282" width="14" height="14" class="bar-after" rx="2"/>
  <text x="279" y="294" class="label">After</text>
</svg>

## The Fix

Three changes, applied together:

The transition loop cost: `loopConsumed = true` on both the mining-to-returning and depositing-to-mining transitions. Each transition now consumes one game loop, matching SC2's per-loop boundary processing. The same fix applied to the gas worker loop, which had identical zero-cost transitions.

The approach docking constant: recalibrated from 11.5 to 24.5 loops. The original value was derived from 12-worker observation data --- but the 12-worker measurement was itself contaminated by the sub-tick chaining. With transitions costing their proper loop, the 8-worker cadence data gives a clean measurement of the approach phase: 24.5 loops.

The deposit duration: 7 to 12 loops, measured from the return half-cycle gap.

The collision speed factor: disabled entirely (0.96 to 1.0). With the transition fix providing correct cycle timing, the collision model's compensating role is no longer needed. SC2 workers don't slow down when they overlap --- they path around each other. The speed penalty was approximating path deviation, but it was really approximating the sub-tick chaining bug.

## The Lesson

This is the failure mode that calibration can't catch by definition. You measure a system at one operating point, tune constants to match, and the calibration succeeds. But the constants are absorbing two errors that happen to cancel. The moment you test at a different operating point --- different worker count, different saturation, different contention --- the cancellation breaks and the real errors surface.

The diagnostic that broke the illusion was the simplest possible scenario: eight workers, one per patch, no contention. No collision model, no lockout, no rerouting. Just the raw physics loop doing straight-line walks. That's the operating point where every constant is carrying its own weight, no compensation from other systems. Any error in the base cycle is fully exposed.

The broader principle: calibrate at your simplest operating point first. If the simplest case doesn't match, nothing downstream is trustworthy --- it's just being compensated for.
