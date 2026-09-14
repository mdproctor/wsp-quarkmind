# HANDOFF — quarkmind

## Last Session

Two issues this session. Landed #249 (commentary training dataset pipeline framework) — 6 Python modules in `quarkmind-dataset/`, 28 tests, design spec, diary entry. Then started #307 (IEM10 VOD curation) — found all 7 ESL YouTube VODs covering 30 games, downloaded audio, began Whisper ASR transcription (3/7 complete at session end, 4 running in background).

Built `estimate_offsets.py` to find game-start timestamps from transcript cues automatically, and `curate_iem10.py` to batch-update the match YAML.

## Resume — #307

1. Check `/tmp/iem10-subs/` for 7 VTT files — if missing (reboot cleared `/tmp/`), re-run Whisper
2. Run `python3 src/curate_iem10.py` to estimate all 30 game-start offsets
3. Copy VTTs to a persistent location
4. Run end-to-end pipeline → validate output → `work end`

## References

- Branch: `issue-307-iem10-vod-curation`
- `.plan` — queue state
- `JOURNAL.md` — design journal
- `quarkmind-dataset/src/curate_iem10.py` — batch offset estimation
- `quarkmind-dataset/catalog/matches/2016_IEM_10_Taipei.yaml` — 30 matches (offsets TBD)
- `docs/specs/issue-249-sc2-commentary-dataset/` — design spec + decisions
