# Design Journal — issue-307-iem10-vod-curation

## 2026-09-14 — VOD curation infrastructure

### What happened
- Found all 7 IEM10 Taipei 2016 series VODs on ESL Archives YouTube channel
- No YouTube subtitles available (2016 uploads) — pivoted to Whisper ASR
- Installed openai-whisper in a venv, added `transcribe_whisper()` and `download_audio()` to `extract_subtitles.py`
- Downloaded all 7 audio tracks (~4.7 GB WAV)
- Transcribed 3/7 VODs with Whisper base (remaining 4 running in background)
- Built complete replay-to-VOD mapping — all 30 replays matched to their series VOD with game numbers
- Built `estimate_offsets.py` — automatically finds game-start timestamps from transcript cues (load cues + GG back-calculation)
- Built `curate_iem10.py` — batch script to estimate all offsets and update the match YAML

### Decisions
- Whisper base model for speed over accuracy — good enough for caster speech. Can upgrade to large-v3 later if quality is insufficient.
- Transcript-based offset estimation rather than manual annotation — search for "loaded into game" and "GG" cues, validate against known replay durations.

### What's next
- Check if background Whisper transcription completed (VTT files in `/tmp/iem10-subs/`)
- If not, re-run transcription (files are in `/tmp/` — may be cleared on reboot)
- Run `curate_iem10.py` to populate all 30 game-start offsets
- Copy VTT files from `/tmp/iem10-subs/` to a persistent location
- Run the end-to-end pipeline to produce first batch of training examples
- Validate output, then `work end`
