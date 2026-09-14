# Decisions — #310 SC2 Commentary Dataset Phase 2+3

## D1: Branch scope — full Phase 2+3

**Choice:** All 8 tasks across Phase 2 (automation) and Phase 3 (scale + integration) on one branch, structured into batches in the .plan queue.
**Alternatives:**
- Phase 2 only — 4 tasks, delivers ~500 aligned games, Phase 3 deferred. Lower risk but delays integration.
- Phase 2 + task 7 — adds commentary pipeline integration but not full Phase 3. Partial integration.
**Rationale:** The user wants end-to-end delivery — from automated pipeline to live commentary integration. Batching within the branch mitigates the risk of the XL scope.
**Trade-offs:** Long branch life, large diff at review. Batching into logical groups helps.
**Sources:** Issue #310 body — Phase 2/3 task breakdown; Phase 1 spec §10 (epic structure)
**Exploration:** quick
**Status:** captured

## D2: Offset detection — transcript-only

**Choice:** Use the existing transcript-based offset estimator (`estimate_offsets.py`) as-is. No audio fingerprinting.
**Alternatives:**
- Audio fingerprinting (librosa) — cross-correlate SC2 game-start sound against VOD audio. More robust for VODs without clear caster cues, but adds heavyweight dependency + audio download overhead.
- Hybrid (transcript primary, audio fallback) — best accuracy, most complexity. Audio adds librosa, ffmpeg, and per-VOD audio download.
**Rationale:** The transcript-based estimator already produces high/medium/low confidence estimates with forward+backward triangulation and interpolation. It works well for tournament casts where casters reliably announce game starts. Avoids librosa dependency and audio download time. Can revisit if accuracy degrades at scale.
**Trade-offs:** VODs without clear caster cues (e.g., non-English or music-only streams) will produce low-confidence offsets. Acceptable — the quality scoring system (D5) will flag these.
**Sources:** `quarkmind-dataset/src/estimate_offsets.py` — existing implementation; Phase 1 spec §5 (timestamp alignment); Phase 1 spec §10 Phase 2 task 2
**Exploration:** quick
**Status:** captured

## D3: VOD matching — YouTube Data API v3

**Choice:** Use the YouTube Data API v3 to search within known tournament playlists. Composite key scoring: player names in title + map name + duration similarity (±2 min). Requires a Google API key.
**Alternatives:**
- yt-dlp metadata scraping — `--flat-playlist` to enumerate playlist contents, title parsing. No API key needed, but title formats vary wildly across channels and years.
- Liquipedia/Aligulac VOD links — community-curated, high quality but incomplete. Good supplementary source.
**Rationale:** Structured API gives reliable, paginated results within known playlists. SC2EGSet tournaments map to ~10-15 known YouTube channels (ESL, GSL/AfreecaTV, DreamHack, WCS). Composite key of two player names is nearly always unique within a tournament playlist.
**Trade-offs:** Requires Google API key (free tier: 10,000 units/day, sufficient for one-off catalog builds). Rate limiting for large tournament catalogs.
**Sources:** Phase 1 spec §3 (matching algorithm); tournament-catalog.yaml — current manual approach
**Exploration:** quick
**Status:** captured

## D4: Catalog expansion — semi-automated

**Choice:** Script that reads SC2EGSet tournament names, searches YouTube for the channel, lists playlists, and proposes matches. Human reviews and confirms. One-time curation effort per tournament.
**Alternatives:**
- Fully manual — curate all 55 tournament entries by hand. Most accurate, ~2-3 hours per tournament. Doesn't scale.
- Fully automated — no human review. Risk of false matches polluting the dataset.
**Rationale:** Tournament-to-channel mapping is a small, fixed set (~10-15 channels cover most SC2 esports). Once the channel mapping exists, playlist enumeration is automated. Human review catches edge cases (renamed channels, moved VODs, partial coverage).
**Trade-offs:** Still requires some manual work per tournament. One-time cost that produces a reusable catalog.
**Sources:** SC2EGSet paper (Nature 2023) — 55 Premiere/Major tournaments; tournament-catalog.yaml — IEM10 Taipei entry as reference format
**Exploration:** quick
**Status:** captured

## D5: Quality scoring — multi-factor composite

**Choice:** Per-example confidence score (0.0–1.0) based on weighted factors: (1) subtitle source quality (manual > auto-generated > Whisper), (2) offset confidence (high/medium/low from estimator), (3) commentary density (tokens per game-second — sparse commentary suggests ads/analysis desk), (4) segment coherence (commentary relevance to game events in the segment window).
**Alternatives:**
- Subtitle source only — simple mapping (manual=1.0, auto-gen=0.7, whisper=0.5). Fast but ignores alignment quality.
- LLM-judged quality — most nuanced, but expensive at scale (thousands of examples × LLM call per example).
**Rationale:** Multi-factor captures the distinct failure modes: bad transcription (subtitle quality), bad alignment (offset confidence), non-game content (commentary density), and misaligned commentary (segment coherence). Each factor is computable without an LLM call.
**Trade-offs:** Segment coherence requires a heuristic (keyword overlap between commentary text and game events). Imperfect but fast. Downstream consumers can filter by minimum quality threshold.
**Sources:** Phase 1 spec §11 (quality and validation); commentary quality criteria
**Exploration:** quick
**Depends on:** D2 (offset confidence is an input factor)
**Status:** captured

## D6: Commentary integration — few-shot + RAG, fine-tuning deferred

**Choice:** Hybrid integration: (1) curate highest-quality training pairs as static few-shot examples, (2) build RAG retrieval to dynamically select the most relevant examples by game phase + matchup + event type at inference time. Fine-tuning deferred to a future branch.
**Alternatives:**
- Few-shot only — static selection, doesn't scale past ~50 curated examples.
- All three (few-shot + RAG + fine-tuning) — fine-tuning requires MLOps infrastructure that's a separate concern.
- RAG only — no static baseline; cold start until vector store is populated.
**Rationale:** Few-shot provides an immediate baseline. RAG makes the selection dynamic and scales with the growing dataset. Together they leverage in-context learning without needing model training infrastructure. Fine-tuning is the natural next step once the dataset reaches sufficient scale and quality.
**Trade-offs:** RAG adds a vector store dependency to the commentary pipeline's inference path. Need to choose an embedding model and store. Fine-tuning (best long-term quality) is deferred.
**Sources:** Phase 1 spec §10 Phase 3 tasks 7-8; commentary design spec (2026-07-06, #181)
**Exploration:** quick
**Status:** captured

## D7: Prompt template library — phase × event matrix

**Choice:** Templates keyed by (game_phase, event_type) — e.g., (mid_game, battle), (opening, expansion). Each template cell includes 2-3 curated few-shot examples from the dataset plus a system prompt fragment describing the expected commentary style for that context. The RAG layer selects the right template cell at inference time.
**Alternatives:**
- Matchup-specific templates — keyed by matchup (PvT, TvZ, etc.) × phase. More templates, more specificity, but combinatorial explosion (5 phases × 6 matchups × N event types).
- Single adaptive template — one template with variable slots filled by RAG. Simpler to maintain, relies entirely on RAG quality.
**Rationale:** Phase × event is the natural commentary structure — casters narrate differently during a battle vs. an expansion vs. a tech transition, and differently in the opening vs. late game. Matchup specificity is handled by the RAG-retrieved examples (which are matchup-tagged) rather than separate template cells.
**Trade-offs:** Template matrix size is manageable (~5 phases × 6 event types = 30 cells). Some cells may have sparse coverage in the dataset initially.
**Sources:** Phase 1 spec §7 (event-driven segmentation, game-phase tagging); segment.py — existing event types and phase definitions
**Exploration:** quick
**Depends on:** D6 (RAG selects the template cell)
**Status:** captured
