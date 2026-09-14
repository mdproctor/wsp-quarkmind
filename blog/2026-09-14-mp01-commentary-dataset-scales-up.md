---
layout: post
title: "From 30 Games to All of SC2EGSet"
date: 2026-09-14
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [sc2, commentary, dataset, rag, pipeline]
---

# From 30 Games to All of SC2EGSet

Phase 1 of the commentary dataset gave us a proof of concept: 30 IEM10 Taipei games aligned to caster VODs, 235 training examples pairing game state snapshots with real commentary. The pipeline worked end-to-end but everything was manual — VOD matching, game-start offset annotation, tournament catalog curation.

Phase 2+3 automates the bottlenecks and connects the dataset to QuarkMind's live commentary.

The most interesting decision was what *not* to build. The original spec called for audio fingerprinting (`librosa` cross-correlation against SC2's game-start sound) to detect where each game begins in a VOD. Turns out the transcript-based estimator we built in Phase 1 — parsing caster cues like "loaded into game number three" and "GG", then triangulating forward and backward — already handles this well enough. It produces high/medium/low confidence estimates with interpolation for gaps. Audio fingerprinting would have added a heavyweight dependency for marginal accuracy gains on a batch pipeline that runs once per tournament.

VOD matching is the piece that actually needed automation. Each replay needs to find its YouTube VOD from a tournament playlist. We built a composite key scorer: player names in the title (40 points), map name (20), duration similarity (20), stage match (10), game number (10). Anything above 60 is accepted; 40-60 flagged for review. The YouTube Data API searches within known tournament playlists — we map the ~10 major SC2 esports channels (ESL, AfreecaTV, DreamHack, WCS) to their playlists, and the scorer does the rest.

Quality scoring was straightforward but important for downstream filtering. Four weighted factors: subtitle source quality (manual > auto-generated > Whisper), offset confidence from the estimator, commentary density (tokens per game-second — catches ads and analysis desk segments), and segment coherence (keyword overlap between commentary text and game state). A composite score from 0.0 to 1.0. Anything above 0.7 is suitable for few-shot prompt examples; above 0.4 goes into the RAG index.

The integration architecture hit one interesting snag during self-review. The plan had `FewShotRetriever` loading a FAISS index directly from Java — except FAISS is a Python library. The fix was clean: use FAISS at build time in the Python pipeline for curation assistance (finding similar examples, selecting diverse few-shot sets), but at runtime the Java `FewShotRetriever` loads pre-indexed JSON files and matches by structured metadata. Filter by game phase and event type, rank by matchup similarity and quality score. For a dataset of a few thousand examples, brute-force filtering over pre-loaded JSON is sub-millisecond. No embedding model needed at runtime, no vector store dependency on the hot path.

The `CommentaryWorkerFactory` integration is the payoff. The factory already builds `ChatRequest` messages with a system prompt and user message. We added `UserMessage`/`AiMessage` pairs between them — real caster commentary as few-shot examples, matched to the current game context. The `CoachingWorkerFactory` already had inline examples in its system prompt, but the commentary integration uses true multi-turn few-shot for better in-context learning.

The prompt template library organises examples into a 5×4 matrix: game phase (opening through endgame) crossed with event type (battle, expansion, tech transition, macro economy). Not every cell will have high-quality examples — `endgame × expansion` is rare in competitive SC2 — but the structure means the retriever always looks in the right place first.

SC2EGSet has 55 tournaments on Zenodo. The batch pipeline processes them sequentially, catches exceptions per tournament so one bad archive doesn't stop the run, and aggregates statistics into a coverage report. The realistic yield depends on VOD availability — many older tournament channels have removed content — but even 40% coverage across 17,930 games would give us thousands of aligned examples. That's a different order of magnitude from 235.
