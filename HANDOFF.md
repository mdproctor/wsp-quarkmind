# HANDOFF — quarkmind

## Last Session

Closed #309 (dataset quality). Four bugs fixed in the VOD offset estimator: load pattern regex captured "number" instead of the game number, missing word boundary on "takes the win" matched "takes the wind", conversational "that's it" matched as game-ending, and overlap adjuster created offsets past VOD end. herO vs sOs G4/G5 corrected by 432s/698s. Polt vs Soulkey G5 no longer overruns the VOD. Confidence distribution improved 2→4 high, 15→8 low. Added MIN_COMMENTARY_LEN filter (removed "bye" segment). Pipeline re-run: 235 validated examples, 54 tests, 7/7 validation checks. Created #310 epic for Phase 2+3.

## What's Next

| Item | Scale | Complexity | Notes |
|------|-------|------------|-------|
| #310 Phase 2: automated VOD matching | L | Med | YouTube Data API, expand to 10-15 tournaments |
| #310 Phase 2: game-start audio detection | M | High | librosa cross-correlation for automated offsets |
| #310 Phase 3: full SC2EGSet coverage | L | Med | 72 tournament ZIPs on Zenodo |
| #306 Retrain ONNX strategy classifier | M | Med | Against corrected spatial features |

## References

- Branch: `issue-309-dataset-quality` (merged to main, stamped)
- Epic: #310 (Phase 2+3 — automation, scale, integration)
- Design spec: `docs/specs/issue-249-sc2-commentary-dataset/`
- SC2EGSet on Zenodo: 72 tournament ZIPs, CC BY 4.0
- Diary: `blog/2026-09-14-mdp02-caster-says-gg.md`
