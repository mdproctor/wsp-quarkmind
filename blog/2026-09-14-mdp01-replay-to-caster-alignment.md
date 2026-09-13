---
layout: post
title: "Building a Commentary Training Dataset from SC2 Replays and Caster VODs"
date: 2026-09-14
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [sc2, commentary, dataset, asr, replay, machine-learning]
---

# Building a Commentary Training Dataset from SC2 Replays and Caster VODs

QuarkMind's commentary pipeline generates play-by-play narration using LLM prompts shaped by game state snapshots. The prompts work, but they're grounded in nothing — there's no corpus of real SC2 caster commentary paired with game state to learn from. LoL has the LoL19-21 dataset (650 games of League game events matched to caster transcripts). SC2 has nothing equivalent.

The gap is worse than "no training data." The existing commentary skews heavily toward early-game events — the moment detection pipeline fires on FIRST_CONTACT, BUILDING_LOST, SUPPLY_BLOCK. A caster narrates the whole arc: the mid-game transition when a player shifts from aggression to macro, the late-game army trades where composition matters more than micro, the endgame resource depletion that turns a positional advantage into a win. I wanted the dataset to correct that bias.

## The matching problem

SC2EGSet gives us 17,930 esports replays as pre-processed JSON — every tracker event (unit born, unit died, player stats, upgrades) with frame-exact timestamps. The data is rich. But there's no link from a replay to the YouTube VOD of that match.

LoL19-21 had it easy: Riot's API provides game IDs that link directly to VODs. SC2 has no such API. The matching problem shifts from "look up the VOD" to "find the VOD from metadata."

The approach that made sense: build a tournament catalog. SC2EGSet covers 55 Premiere and Major tournaments — these all have official broadcast channels with YouTube playlists. Within a playlist, a composite key of two player names plus map name is almost always unique. Match within the playlist, not across all of YouTube. The search space drops from billions of videos to hundreds per tournament.

## Alignment is simpler than it looks

SC2 competitive games run at Faster speed — exactly 22.4 game loops per second. Replays are frame-exact. No drift, no variation. Once you find the game-start moment in the VOD (after the pre-game analysis, player intros, loading screen), the rest is arithmetic:

```
game_frame = (vod_timestamp - game_start_offset) * 22.4
```

For Phase 1, the game-start offset is manual — I watch the VOD and note the timestamp. Phase 2 will automate this with audio fingerprinting (SC2 has a distinctive game-start sound). But the linear mapping after that is exact.

## Event-driven segmentation with phase context

LoL19-21 used fixed 1-minute windows. I wanted something that matched how casters actually work: they react to events, not to a clock. A battle gets dense, rapid-fire commentary. A quiet macro phase gets occasional strategic observations. Fixed windows flatten that natural rhythm.

The segmenter detects events from tracker event patterns — clusters of unit deaths signal battles, new Nexus/CommandCenter births signal expansions, upgrade completions signal tech transitions. Each segment is anchored to an event and extends a configurable window after it. Quiet periods between events get merged into macro/economy segments.

Each segment carries a game-phase tag (opening, early aggression, mid-game, late-game, endgame) based on timing heuristics calibrated from IEM10 games. When a segment spans a phase boundary, it's flagged as a transition — these are particularly valuable because casters often narrate the strategic shift explicitly.

## The pipeline

Six Python scripts, each with a single job:

- `parse_replays.py` — reads SC2EGSet's nested ZIP structure (outer ZIP → data ZIP → JSON per game)
- `extract_state.py` — accumulates tracker events into structured game state snapshots at any frame
- `extract_subtitles.py` — downloads YouTube subtitle tracks via yt-dlp, parses WebVTT
- `catalog.py` — loads the tournament catalog and replay-to-VOD match files
- `align.py` — converts VOD timestamps to game frames using the game-start offset
- `segment.py` — event-driven segmentation with phase tagging

`pipeline.py` orchestrates the lot: for each matched replay, it extracts game state at the midpoint of each segment, pairs it with the aligned commentary text, and writes a structured JSON training example.

The output schema mirrors `GameStateTranslator.toMap()` from the Java side — army composition, economy, tech, buildings, opponent state — so the training input format matches what the commentary pipeline sees at inference time.

## What's next

This is Phase 1 of three. The pipeline framework is built and tested. The next step is the manual curation work — finding VODs for 50 IEM10 Taipei games, recording game-start offsets, downloading subtitles. That produces the first batch of aligned training examples, which can immediately improve the commentary pipeline's prompts with real caster language.

Phase 2 adds automation: YouTube API integration for VOD matching within playlists, audio fingerprinting for game-start detection, scaling to 500 games. Phase 3 covers the full SC2EGSet where VODs exist and integrates the dataset directly into the commentary pipeline as few-shot examples and prompt templates.

The interesting question is whether real caster language — with its mid-game and late-game coverage — changes the character of the generated commentary in a way that pure prompt engineering can't reach. The early-game bias in the current pipeline isn't a prompt problem; it's a knowledge problem. The model doesn't know what late-game SC2 commentary sounds like because nobody has shown it.
