---
title: "When the Expansion Dies"
date: 2026-09-11
author: mdp
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [scouting, posture, expansion, fog-of-war, design]
---

# When the Expansion Dies

The posture system had a blind spot. Once an enemy expansion was detected, the classification stuck at MACRO forever — even if the enemy sacrificed their expansion for an all-in push. Expansion events were permanent, stored in a list that never evicted. The DRL rule "Expansion: Macro" fired the moment evidence appeared and never retracted.

Worse, the expansion heuristic itself was wrong. It triggered on any enemy unit more than 50 tiles from the estimated enemy base. An attacking army near the player's base — 200+ tiles from the enemy's main — counted as expansion evidence. Every attack made ALL_IN undetectable.

The fix is a two-tier detection system. Tier 1 is the existing unit-distance heuristic, preserved for early-game coverage before expansion buildings are visible. Tier 2 tracks actual enemy base-type buildings (Nexus, Hatchery, Command Center) at non-starting locations. When Tier 2 has data, it overrides Tier 1 — ground truth supersedes proxy.

The interesting design question was fog of war. When an expansion building leaves vision, what do you do? Three options: volatile (only trust current observation — oscillates), timed eviction (expire after N minutes — arbitrary), or sticky-until-contradicted. I chose the third. The cached state persists until we observe the expansion location again AND the building is gone. "I can't see it" means "I don't know," not "it's destroyed." Only "I looked and it's not there" means retract.

The posture bypass is in Java, not DRL. Post-confirmation, posture is deterministic from the confirmed expansion state: expansions alive → MACRO, all retracted → ALL_IN. The DRL expansion rules stay active for pattern classification — historical evidence that the enemy expanded is still useful for strategy archetype detection, even after the expansion is destroyed. Posture and pattern classification intentionally diverge: posture reflects current tactical state, pattern classification reflects historical strategic evidence.

The calibration test confirms the pivot: UNKNOWN → MACRO → ALL_IN, with the transition firing exactly when the building is removed and a friendly unit has vision of the location. MACRO stays stable when the expansion persists — no false retraction across 900 ticks.

Epic #301 is now fully closed. Four issues — cascade verification (#302), ALL_IN calibration (#304), summariser audit (#303), and this expansion sacrifice detection (#305) — all landed in one session. The spatial intelligence layer now tracks real army movement, persists posture classifications, detects timing attacks through the full cascade, and can revert MACRO when the evidence disappears.
