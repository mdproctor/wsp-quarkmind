---
layout: post
title: "Calibrating the Mining Model"
date: 2026-11-02
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [emulated-game, economy, calibration, replay-validation, mule]
---

# Calibrating the Mining Model

QuarkMind's EmulatedGame has been tracking real SC2 replays with 100% accuracy on units, buildings, and upgrades for weeks. But the economy model has been quietly absurd — reporting mineral collection rates at 7.5% of what actually happens in the game. A base with 16 workers mining happily away would be credited with about 600 minerals per minute. The real number is closer to 1,000.

Two bugs, hiding in plain sight. Then a third.

## The Unit Conversion That Wasn't

The `EconomyTracker` accumulates gross mineral income over a rolling window — 7 ticks, roughly 7 seconds. When it reports the collection rate, it was handing back the raw accumulated value: minerals-per-7-ticks. SC2's own PlayerStats events report collection rates in minerals-per-minute. Nobody had ever converted between the two.

The scale factor is about 8.7×. So right away, the emulated economy was reporting a number an order of magnitude too small — not because the mining model was wrong, but because the output was in the wrong units.

```java
double perTickMineral = (grossMinerals - prevGrossMinerals) / RATE_INTERVAL_TICKS;
mineralRate = (int) Math.max(0, perTickMineral * TICKS_PER_MINUTE);
```

One constant and a multiplication. The kind of bug that survives because both the code and the tests look reasonable in isolation — the EconomyTracker test asserted "greater than zero" for the rate, which it was.

## The Income Rates Were Guesses

With the unit conversion fixed, the collection rate error dropped from 92% to about 35%. Still wrong, but now in a regime where meaningful comparison is possible. The remaining gap pointed at the per-worker income constants themselves.

SC2Data had a three-tier saturation model: the first 8 workers on a mineral line earn full rate, the next 8 earn half, the last 8 earn a tenth. The actual values — 50, 25, and 5 minerals per minute per worker — had been set from community estimates, never calibrated against real game data.

![Three-tier saturation model — before and after calibration](images/three-tier-saturation.svg)

I ran 118 oracle replays through the validation harness. The harness feeds ground-truth worker distributions to the EmulatedGame, so the only free variables were the per-worker income rates. At the 5-minute checkpoint across 168 player runs, the optimal scale factor was 1.554× for minerals and 1.441× for gas.

The calibrated rates — 77.7, 38.8, and 7.8 minerals per minute per worker — sit right where you'd expect from real SC2 mechanics. Community guides typically cite 65–80 minerals/min for the first saturated worker and 35–45 for the second tier. The uncalibrated values had been too conservative by about half.

## The MULE That Didn't Report

With worker income sorted, a subtler problem emerged in the Terran economy. MULEs — the temporary super-miners that Orbital Commands drop onto mineral patches — were earning income but not reporting it.

The `TerranRaceModel.tickPassive()` method added MULE minerals directly to the player's balance via `addMinerals()`. But this call happened after the `EconomyTracker` had already recorded the tick's income. The collection rate saw worker income. It never saw MULE income. For Terran players, the reported rate was systematically low by however many MULEs were active.

The fix changed `tickPassive()` from `void` to `double` — returning passive mineral income so the caller can include it in the rate before recording. Clean interface change: zero for Protoss and Zerg, MULE income for Terran.

Then the MULE constant itself. SC2Data had `172.5 minerals/min` — a community estimate based on the common claim that MULEs mine at 3.45× SCV rate. Calibrating against 83 Terran player runs (115 checkpoints with active MULEs) gave **277.8 minerals/min** — a 1.61× increase. That's 3.57× the calibrated SCV tier-1 rate of 77.7, closer to what replay data actually shows.

## The Result

![Collection rate error before and after calibration](images/income-calibration-results.svg)

At the 5-minute checkpoint, mineral collection rate error dropped from 92.7% to 0.0%. Vespene from 92.3% to 0.2%. Not "better" — effectively solved. The model now tracks real SC2 economy with the same fidelity that the unit production pipeline already achieves.

The accuracy isn't uniform across checkpoints — the 1-minute mark still shows 22% error because the mining start delay and initial worker travel time matter more when the game is young. But from 3 minutes onward, the model converges to within a few percent of ground truth.

## Why This Matters Beyond StarCraft

The economy model isn't a curiosity — it's what makes the entire agent pipeline possible. Every strategic decision the QuarkMind harness makes flows through an economic assessment. "Can I afford this unit?" is the gatekeeper for every production intent. "How fast is my opponent's economy?" drives the scouting evaluation. A model that was 13× too low on income rates was making every downstream decision against a backdrop of artificial poverty.

More practically, the calibration pattern itself is portable. Any game AI that emulates physics offline — for training, for planning, for replay analysis — has the same problem: initial constants are estimates, and estimates drift from reality in ways that compound. The approach here was straightforward: feed ground-truth inputs, measure ground-truth outputs, solve for the constants that minimise divergence. No gradient descent, no learned parameters. A diagnostic test, a scale factor, and an updated constant.

The trade-off is visible too. With correct income rates, the emulated mineral balance now overshoots ground truth by 341% — because we're mining the right amount but not spending it correctly. Army spending in the emulated model doesn't match replay spending patterns. That's the next calibration target: worker-to-resource assignment tracking and spending reconciliation. Income is done; spending is where the balance sheet closes.