---
layout: post
title: "canProduce Was Never Just a Query"
date: 2026-06-02
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [canproduce, playerstate, plugin-seam, racemodel, mule, intent]
---

The last entry ended with `#165` tracked but deferred — `canProduce()` read-only contract was doc-only. Making `PlayerState` public was the right call, but it left the plugin seam in an awkward position: nothing prevented an external race implementation from calling `state.setMinerals(0)` inside `canProduce()` and returning `PROCEED`. The fix looked obvious. Working through the design with Claude, we found the problem ran deeper than the method signature.

`TerranRaceModel.canProduce()` was actively mutating state. MULE calldown had been modelled as `TrainIntent(oc, UnitType.MULE)`. When `canProduce` received it, the implementation spawned the unit directly — `state.addUnit(mule)` — then returned `HANDLED` to tell `EmulatedGame` to skip the training queue. `ProductionResult` had three values: `PROCEED`, `BLOCKED`, and `HANDLED`. The third existed entirely to accommodate this.

The fix to the doc-only constraint was a read-only view type for `canProduce`. But then the MULE calldown had nowhere to write. That's when I realised the design problem wasn't the method signature at all.

MULE calldown is not training. In SC2 it's an Orbital Command ability — instant, energy-based, no queue, no build time. It was modelled as a `TrainIntent` because that was the only available intent, and `HANDLED` existed to paper over the semantic error. Fix the model, not the workaround.

`MuleCalldownIntent` is a record in the sealed `Intent` hierarchy, alongside `TrainIntent`, `BlinkIntent`, and the rest. MULE calldown dispatches through `EmulatedGame.handleMuleCalldown()`, delegates to a new `RaceModel.onCalldown()` default method — Terran overrides it. With the extraction done, `ProductionDecision` needed only two values: `PROCEED` and `BLOCKED`. The sealed interface with a data-carrying `Handled(Consumer<PlayerState>)` variant had no remaining justification. All variants were stateless. An enum was correct.

It's a full-circle moment. `ProductionResult { PROCEED, HANDLED, BLOCKED }` was the original. We planned a sealed interface with `Handled(Consumer<PlayerState>)`. What we shipped was `ProductionDecision { PROCEED, BLOCKED }`. The original enum had the right shape — just one value too many and the wrong method signature. The refactor corrected the method, not the container type.

One thing worth noting about `onCalldown()`: both `EmulatedGame.handleMuleCalldown()` and `TerranRaceModel.onCalldown()` independently validate the OC building type. That looks like redundant defensiveness. It's not. `onCalldown()` is a public extension point; #74 will have external race models calling it. If the method doesn't validate internally, future callers from other contexts bypass the OC check. The inner guard is for the method's own integrity, not for this call path.

`canProduce(PlayerStateView view, ...)` is structurally read-only now. `TerranRaceModel.canProduce()` is a one-liner returning `PROCEED`. 843 tests pass.
