# Design Journal — issue-306-retrain-onnx-spatial

## 2026-09-23 — Feature pipeline implementation complete (Java + Python)

Implemented 8 of 10 tasks across both the Java inference pipeline (quarkmind-sc2) and the Python training pipeline (neocortex). The feature vector grows from 269 to 298 per window with three new dimensions: spatial intent (army centroid, distances, spread, proxy detection), temporal deltas (army supply, workers, production, tech buildings between windows), and structural ratios (army/supply, worker saturation, gas/mineral).

Key design insight from brainstorming: the original issue premise was wrong — the feature vector had NO spatial features to "shift." The spatial fix (#298/#300) corrected enemy positions during replay, but the ONNX classifier never used positions. The real opportunity was to ADD spatial features for the first time.

Architecture decisions: single model with modality dropout (not parallel specialists), binary fog-of-war masking for spatial features (centroid × 0.3 is nonsensical), two-phase window assembly (per-tick averaging then per-window deltas), keep 10×30s windows (defer finer resolution).

Also fixed: CBR API migration (CbrCase→CbrRecord), dependency convergence (quarkus-flow, MCP SDK, jackson-databind, jackson-jq), TrustGateService import move.

## 2026-09-14 — Investigation: feature vector is not spatial

Investigated the ONNX strategy classifier feature pipeline. Found that DroolsScoutingTask.buildSnapshot() produces a purely compositional feature vector — 134 features per player: building counts (53), unit counts (53), economic stats (13), upgrade flags (15). Zero spatial features. The issue's premise that "spatial features shifted" was incorrect — there were no spatial features to shift.
