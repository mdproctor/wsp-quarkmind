---
layout: post
title: "When Your Tests Pass but Your Numbers Lie"
date: 2026-11-18
author: Mark Proctor
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [quarkmind, sc2, emulated-game, calibration, dual-run, testing]
---

# When Your Tests Pass but Your Numbers Lie

The emulated game had good numbers. 12-worker Protoss mineral income: -0.3% vs SC2. 24-worker saturation: +0.8%. 16-worker oversaturation: -2.8%. All within the range where you nod and move on.

Then we built a catalogue of every calibration recording — a table of all SC2 baselines, their EM counterparts, and the deltas. The kind of infrastructure that feels like busywork until it catches something.

It caught something.

## The dual-run framework

The idea is simple: record a game in real SC2, capture a per-tick snapshot of minerals, vespene, units, and buildings. Then run the same playbook against the emulated game and compare tick by tick. Any divergence in unit counts triggers an early exit — the economy comparison only matters if the structural state matches.

There are fourteen single-run recordings now, covering mineral baselines at three saturations (8, 12, 16, 24 workers), three races, gas configurations, MULE income, expansion transfer, and worker trip timing. Each recording has a matching playbook that drives the EM through the same actions: train workers, build structures, assign gas workers, expand.

Some of these playbooks build structures — Pylons, Nexus, Supply Depots. The EM can't place buildings at real SC2 map positions (it doesn't model map geometry for placement), so the test framework uses `skipBuildings`: buildings get created instantly at a dummy position via `debugCreateBuilding()`. The building appears, supply is added, and the test continues.

The problem is what `debugCreateBuilding` doesn't do. It doesn't deduct minerals.

## Free buildings

A Pylon costs 100 minerals. A Nexus costs 400. A Supply Depot costs 100. When the test framework skips building placement, it was also skipping the bill. Every playbook that built structures was giving the EM free minerals that SC2 spent.

Those free minerals didn't sit idle. With continuous worker training (`train: PROBE, loop: continuous`), every unspent mineral goes into the next worker, which mines more minerals, which trains more workers. The inflation compounds.

The 24-worker saturation playbook builds a Nexus (400) and several Pylons. That's 800+ free minerals cascading through continuous training. The 16-worker playbook builds fewer structures — less inflation. The MULE isolation playbook builds three Supply Depots and a Barracks: 550 minerals not being deducted.

This is why the numbers looked good. They weren't.

## What the catalogue revealed

Economy-only recordings — 12w Protoss, 8w Protoss, Terran, Zerg — don't build anything. Their numbers are real. The 12w baseline at -0.3% is genuine accuracy.

But every recording that uses `skipBuildings` was flattered. When we added mineral cost deduction as an option and re-ran:

![skipBuildings inflation: free vs costed buildings](images/2026-11-18-free-buildings-inflation.svg)

The 24w result flipped from +0.8% (EM slightly over SC2) to -4.3% (EM significantly under). The MULE isolation delta tripled from -7.4% to -22.9%. The 3w gas comparison — which looked like a comfortable +2.9% — dropped to -10.1%.

The EM's true accuracy is worse than the catalogue showed. The test framework was hiding the gap.

## Why this was invisible

Each test passed. The structural checks (unit counts, building counts, upgrades) were clean. The economy deltas were within ranges that felt plausible. Without a catalogue comparing recordings side-by-side, there was no signal that the skipBuildings tests were systematically optimistic while the economy-only tests were not.

The pattern only becomes visible in aggregate: every recording that builds structures shows a better delta than every recording that doesn't. That's not noise — it's a systematic bias. But you can't see a systematic bias by looking at one test at a time.

## The fix and what it opened up

We made cost deduction opt-in (`deductBuildingCosts` parameter) rather than changing the default, because the existing calibration was done against the free-building baseline. Rewriting that calibration history is a separate effort. The parameter exists for new recordings and for anyone who wants to see the honest numbers.

We also filled the four remaining gaps in the catalogue this session: Zerg mineral baseline (+0.3%), MULE isolation (-7.4% with the original free-building numbers), worker speed (-1.1%), and expansion transfer — which turned out to be structurally incomparable because instant construction gives supply too early, snowballing 30+ extra workers by tick 304.

The expansion transfer gap led to a separate finding: the EM doesn't just skip building costs, it skips construction time. Supply from a Nexus arrives immediately instead of 71 seconds later. For economy-only comparisons, that doesn't matter. For multi-base economy with continuous training, it distorts everything.

## What infrastructure catches that tests don't

A passing test tells you the code does what the test expects. A calibration catalogue tells you whether your expectations are consistent with each other. The free-buildings bug wasn't in the simulation physics — the physics runs fine. It was in the test harness, inflating results for a subset of recordings. No single test could detect it because no single test was wrong. The pattern across recordings was wrong.

The catalogue is a table in a markdown file. The recordings are JSON snapshots. The dual-run framework is a 200-line test method. None of this is sophisticated. But the combination — a systematic, browsable comparison of every calibration point — made a bias visible that had been hiding in plain sight across months of work. The good numbers weren't wrong. They just weren't what they claimed to be.
