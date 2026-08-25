---
layout: post
title: "Closing the Loop on CBR Learning"
date: 2026-08-25
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [cbr, commentary, testing, epic-close]
---

# Closing the Loop on CBR Learning

Epic #214 — CBR game experience — had nine child issues closed and all three tiers mechanically complete: retain, retrieve, reuse, structured enrichment, temporal matching, opponent models, observability endpoints. What remained were two definition-of-done items that required cross-cutting integration rather than standalone features. We knocked both out in a session.

The first was a gap in evidence. The learning curve endpoint already computed IMPROVING/DECLINING/STABLE trends by comparing first-10 vs last-10 win rates — clean, straightforward code. But no test exercised the IMPROVING path. The trend detection logic was shipping untested. Three new test methods fixed that: 15 games where late performance dominates early performance, a declining trajectory, and per-matchup breakdown verification.

The second was a gap in connection. The commentary pipeline — the system that generates real-time play-by-play and narrative commentary via LLM — had zero awareness of CBR. It knew about pattern assessments (enemy archetype classifications) and game moments, but nothing about past game experience. The bot could learn from past games, but its own commentator couldn't talk about it.

The fix wired CBR retrieval results into both commentary paths. For reactive commentary (immediate play-by-play), the trigger builder now extracts CBR keys from the CaseContext — whether CBR influenced strategy selection, how many similar past games were found, and what the temporal prediction says. For narrative commentary (contextual arc narration), the NarrativeContextHolder picks up CBR state after each engine settle and includes it in its snapshot. Both paths format the data as a `PAST GAME EXPERIENCE` block in the LLM prompt, letting the commentator say things like "based on 3 similar past games, the bot is choosing a defensive opening."

Code review caught a real bug before it shipped. The reactive and narrative paths assemble their trigger maps differently — reactive builds the map in `CommentaryTriggerBuilder` with direct CaseContext access, narrative accumulates moments over a 45-second window and snapshots strategic context from `NarrativeContextHolder`. We'd added CBR data to both paths, but the narrative path stored it inside a `context` sub-map with keys like `cbr_similar_count`, while the shared consumer (`appendCbrContext`) looked for a `cbrContext` map at the trigger root with keys like `similarCount`. Two data structures, same consumer, zero intersection. The narrative path would have silently produced commentary without any CBR context. The fix promotes CBR fields from the snapshot into a root-level `cbrContext` map in `CommentaryAccumulator.tick()`, matching the reactive format.

This closes the CBR experience epic structurally. The bot retains game outcomes, retrieves similar past games, routes strategy based on what worked before, and its commentator can narrate the learning. What it can't yet do is prove that learning improves outcomes over a multi-game run — the endpoint infrastructure is there, the trend logic is tested, but nobody has run 10+ sequential games against a fixed opponent and watched the win rate curve. That's a simulation harness problem, not a missing feature.
