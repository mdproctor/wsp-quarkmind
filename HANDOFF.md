# QuarkMind Handover — 2026-07-04

## Last Session

Planning session — no code changes. Created the ONNX strategy classifier epic (#208, cross-repo with neocortex) and the full CBR game experience epic (#214, cross-repo with neocortex #86). Research sweep found a decade of SC2 strategy classification literature; neocortex already has `OnnxInferenceModel` + `TextClassifier` — QuarkMind won't build its own ONNX integration.

## Immediate Next Step

Pick new work from What's Next. Run `/work` to start a branch.

## Cross-Module

**engine#625** (TrustWeightedImplementationRoutingStrategy) committed on engine branch `issue-625-trust-impl-routing` — needs to be merged to engine main and published. QuarkMind already consumes it from local ~/.m2.

**neocortex#77** (raw tensor SPI) — prerequisite for QuarkMind's ONNX classifier tier. Not blocking current work.

## What's Left

- quarkmind#74 — genericise unit/building definitions · L · Med
- quarkmind#192 — CBR reference implementation · L · High (unblocked — #207 done)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #192 | CBR reference implementation — mechanical wiring | L | High | Unblocked, first step of CBR epic #214 |
| #74 | Genericise unit/building definitions | L | Med | Independent |
| #181 | Commentator/Coach LLM | L | High | Unblocked by #182 + #180 |
| #183 | Enemy strategy classifier | M | High | Unblocked by #182 |
| #191 | Milestone-based trust scoring | M | Med | Independent |
| #221 | Learning curve observability | M | Med | CBR epic #214, Tier 1-2 |

## Build Note

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*
