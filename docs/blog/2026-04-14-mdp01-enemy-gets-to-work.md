---
layout: post
title: "The Enemy Gets to Work"
date: 2026-04-14
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [emulation, enemy-ai, tdd, subagent-development]
---

The plan from April 10th didn't survive.

I'd designed a Terran enemy with workers, production buildings, and a separate `EnemyAI` class — the kind of opponent that mirrors what the player bot faces. Eleven tasks standing by. Four days later I came back to start and stripped it down to essentials.

The complication was scope creep masquerading as completeness. SCVs, Command Centers, Barracks — these make the enemy economy *feel* real, but they don't make it harder to beat. What matters is: minerals accumulate, units train, armies attack. The mechanism can stay abstract.

What we built is a single record:

```java
public record EnemyStrategy(
    List<EnemyBuildStep> buildOrder,
    boolean loop,
    int mineralsPerTick,
    EnemyAttackConfig attackConfig
) {}
```

No workers. No buildings. No `EnemyRace` enum. Minerals accumulate at a fixed rate per tick, units pop out of the build order when affordable, and when the army hits a threshold — or a timer fires — they attack. The strategy is swappable at runtime: load from YAML at startup, or POST a new JSON body to `/qa/emulated/config/enemy-strategy`.

The staging area was the right call. Trained units wait at spawn rather than trickling toward the nexus one at a time. When the attack fires, they move as a wave. That one design choice makes the enemy feel like it's massing instead of dribbling.

## Burst attacks, not damage per tick

Parallel to the economy work, we rewrote how combat damage is applied. E3 used flat damage per scheduler tick — every 500ms, each unit in range loses HP. SC2 doesn't work that way. Units have attack animations with specific cooldowns between strikes.

The fix is per-unit cooldown maps in `EmulatedGame`. Absent key means ready to fire immediately. After each attack, the cooldown resets to `SC2Data.attackCooldownInTicks(type)`. The two-pass simultaneous resolution from E3 stays intact; only the "does this unit fire this tick?" check changes.

One bug came out with it. A unit given a `MoveIntent` after an `AttackIntent` kept firing indefinitely — `attackingUnits` was never cleared by a move command. That's now fixed. It had been there since E3 and made it this far uncaught.

## Eight subagents, eight tasks

We used subagent-driven development — a fresh Claude per implementation task, two review passes each (spec compliance, then code quality). Eight tasks total.

The reviews were productive. The code reviewer caught that I'd added `@JsonProperty` annotations to the new domain records for Jackson serde. The existing records — `Unit`, `Building`, `GameState` — have none. Jackson 2.12+ reads constructor parameter names directly when `-parameters` is set, which Quarkus enables by default. The annotations were noise. We stripped them.

The same reviewer caught a gap: the spec promised YAML file loading but `jackson-dataformat-yaml` wasn't in `pom.xml`. Added before merging.

What the subagents didn't do: create GitHub issues. The issue-workflow instructions weren't in their prompts, and CLAUDE.md automatic behaviours don't propagate to subagent context — each starts fresh. Ten commits landed with no `Refs #N` footers. We created the issues retroactively and rewrote the commit history with a cherry-pick loop to link them. Next time, the issue numbers go into the subagent prompts before the first line of code.

272 tests. The enemy has a job.
