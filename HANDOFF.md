# HANDOFF — quarkmind

## Last Session

Closed #337 (version-aware abilLink dispatch) and #338 (selection-state-aware abilLink discovery). Both landed on main. Filed epic #339 (Full ONNX Coverage) with 5 child issues (#340-#344).

What was built:
- `AbilityProfile` enum + `AbilityDispatch` — N-tier version dispatch with sparse lambda overrides
- 17 HSC_2025 overrides (morph, upgrade, conflict resolution)
- Selection-state-aware discovery test (selectionSize==1 filtering)
- Accuracy: 65.9% → 78.6% (858 → 1023 of 1302 upgrade events across 179 replays)
- Also fixed pre-existing neocortex/ledger API compilation errors

Epic #339 captures the end-to-end path: EmulatedGame fidelity → feature extraction → training data reconstitution → ONNX retraining → AI-vs-AI training. Key insight: stripped replay datasets (SC2EGSet) need reconstitution through EmulatedGame to produce full feature vectors for ONNX training — emulation fidelity is the foundation, not an afterthought.

## Active Branch

None — on main.

## Immediate Next Step

Start #340 (EmulatedGame fidelity baseline) or #341 (feature extraction completeness). Both are Phase 0/1 of epic #339 and can run in parallel.

## References

- Epic: `https://github.com/casehubio/quarkmind/issues/339`
- Diary: `blog/2026-10-03-mdp01-version-dispatch.md`
- Design spec: `specs/issue-337-version-aware-abillink-dispatch/`
- Previous session handover: `git show HEAD~1:HANDOFF.md`
