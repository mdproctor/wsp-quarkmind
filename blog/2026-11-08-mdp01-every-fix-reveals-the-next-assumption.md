---
layout: post
title: "Every Fix Reveals the Next Assumption"
date: 2026-11-08
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [calibration, spending-model, economy, ground-truth, starcraft2, digital-twin]
series: issue-534-emulated-game-100-accuracy
---

# Every Fix Reveals the Next Assumption

*Continues the [100% accuracy series](2026-11-07-mdp04-when-physics-isnt-the-answer.md).*

The standalone EmulatedGame tracks every unit, building, and upgrade at 100% accuracy against 118 professional StarCraft II replays. The economy is another story. vespeneUsedCurrentTechnology was 42% wrong. mineralsCollectionRate was 14% off. vespeneCurrent was 72% wrong at the seven-minute mark. Three distinct problems, each hiding behind the one in front.

## SC2 doesn't categorise spending the way you'd expect

The technology spending field — `vespeneUsedCurrentTechnology` — was 42% MAPE at the seven-minute checkpoint. Every upgrade and research cost was being recorded as technology spending. Forge upgrades, weapon levels, Blink research, Stimpack — all of it routed to the technology bucket alongside tech building costs.

That felt right. Research is technology. SC2's own protocol field is literally called `vespeneUsedCurrentTechnology`.

It's wrong. SC2 puts research spending in the **army** category, not technology. The technology field tracks only the vespene cost of tech buildings that are currently alive.

The evidence was unambiguous. We stripped research from the emulator's technology total and compared buildings-only against ground truth. MAPE dropped to 1.1% at five minutes. Zerg replays were the clearest signal: ground truth reported zero technology spending at five minutes, while the emulator showed 200 gas from completed upgrades. That 200 gas wasn't in any of SC2's technology fields — it was in army.

![SC2 spending categories: where research actually goes](images/spending-categories.svg)

Reclassifying research from technology to army fixed tech spending immediately — 42% down to 15.4%. But it shifted the problem. Army vespene MAPE went from 7.4% to 35.4% because research costs now sat alongside unit costs, and the timing didn't match ground truth's snapshot cadence.

That timing mismatch has its own explanation. SC2's PlayerStatsEvent fires every 160 game loops — roughly seven seconds. Items that complete between the last event and the measurement snapshot show up as "current" in the emulator but still as "in progress" in ground truth. This creates a structural floor of 10–15% MAPE on spending fields that no code change can eliminate. The real game and the emulator agree on the totals; they disagree on which seven-second window each item landed in.

## The workers that never stopped

With spending reclassified, the dominant remaining error was mineral income. The emulator was earning too many minerals per player — 971 per Protoss player at seven minutes, 552 for Terran. Worker counts matched ground truth perfectly (tracker events inject every worker at the correct time), so the issue was per-worker income rate, not worker count.

The clue was in the building placement model. `WorkerTask.BUILDING` existed in the code as a worker state that earns zero income. It was never assigned. Every worker mined continuously, even while supposedly constructing a building.

In real SC2, a Terran SCV stays at the building site for the full construction time. A Supply Depot takes 21 seconds. A Barracks takes 46 seconds. With 22 buildings per Terran player by seven minutes, that's roughly 770 worker-seconds of paused mining — the equivalent of 1.8 workers offline for the entire game.

We added race-specific building pauses: Terran SCVs pause for the full build duration, Protoss Probes pause for about five seconds (place the building and walk back). Terran's mineral income error dropped from 552 to 436. The building pause explained almost all of Terran's over-earning.

Protoss barely moved — 971 down to 899. Twenty-two buildings at five seconds each is only 110 worker-seconds. The real gap for Protoss is about three workers' worth of mining income that disappears to activities the tracker events don't capture: scouting Probes sent to the enemy base, workers repositioning between tasks, idle time between building placement and mineral line return. The emulator can't see these pauses, so it mines with all workers all the time.

![Per-race income error: building pauses vs untracked activity](images/income-error-decomposition.svg)

## Gas workers don't auto-saturate

The last assumption was gas worker assignment. The emulator assigned three workers to every gas building on completion — standard macro for a mid-game economy. But real players don't always run three on gas. Aggressive openers might run zero or one gas worker. Players pull workers off gas to rush minerals for units. Some delay gas saturation until they need tech.

The result: the emulator had nine gas workers when the real player had three, tripling the gas rate and producing 28.5% vespeneCollectionRate MAPE at five minutes.

We estimated gas worker count from ground truth's vespene collection rate and synced it each tick. Gas rate MAPE dropped from 28.3% to 20.1%. vespeneCurrent MAPE dropped from 57.4% to 39.4%.

But the freed gas workers had to go somewhere. They went to the mineral line. Six workers pulled off gas returned to mining minerals, and the mineral income — already over-producing due to the untracked worker activities — got worse. mineralsCurrent went from 71.5% to 79.7%.

Each fix exposed the next assumption. Spending categories were wrong, so we fixed them. Building pauses were missing, so we added them. Gas workers were over-assigned, so we synced them. And each correction shifted the error to a different field, because the mineral mining model still doesn't know that real players have three workers doing something other than mining at any given moment.

## The measurement floor

The accuracy ceiling for economy fields is not 0%. PlayerStatsEvent reporting lag creates a 10–15% structural floor. Low mineral banks amplify MAPE through small denominators — Protoss players averaging 506 minerals at seven minutes produce higher MAPE from the same absolute error than Terran players averaging 686. Worker activity that tracker events can't observe costs roughly three workers' worth of income per player.

The next step is a per-race mining efficiency factor — a scalar on mineral income calibrated from replay data to account for the fraction of workers actually mining at any given moment. Not a uniform rate multiplier (that was tried earlier and doesn't work because the direction reverses between checkpoints), but a worker availability discount that absorbs the structural gap between "all workers mine" and "real players don't do that."

The emulator now tracks every unit, building, and upgrade at 100%. Spending categories match SC2's actual classification. Workers pause during building construction. Gas workers follow the real player's assignment, not a default. What remains is the gap between a simulation that models what workers do and a real game where workers also do things the replay doesn't record.
