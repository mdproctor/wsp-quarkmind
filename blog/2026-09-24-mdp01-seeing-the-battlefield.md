---
title: "Seeing the Battlefield — Spatial Features for Strategy Classification"
date: 2026-09-24
author: mdp
entry_type: note
subtype: diary
tags: [onnx, ml, strategy-classifier, feature-engineering, sc2]
projects: [casehubio/quarkmind]
series: issue-306-retrain-onnx-spatial
---

The ONNX strategy classifier could count units and buildings, track upgrades and economy stats — but it had no idea where anything was on the map. A Terran player with marines near your natural expansion and a Terran player with marines defending their own base produced identical feature vectors. That's a significant blind spot for a tool meant to distinguish rushes from macro play.

Issue #306 fixed that. We enriched the per-tick feature vector from 134 to 144 features per player by adding seven spatial features (army centroid position, distance to own base, distance to enemy base, army dispersion, maximum forward projection, and proxy building ratio) plus three economic ratios (army supply fraction, worker saturation, gas-to-total spending). After windowed assembly with inter-window deltas, the full pipeline went from 269 to 298 features per window.

The work split cleanly into a 10-task plan across five batches. Foundation constants and index maps first. Then per-tick extraction in `buildSnapshot()`, windowed assembly in `TemporalWindowAccumulator`, and the `StrategyFeatureExtractor` update. Python-side changes mirrored the Java pipeline — `sc2egset_extractor` got position tracking, `feature_engineering` got the two-phase assembly. A `FeatureAlignmentTest` verified Java and Python produce matching vectors for the same replay data, which turned out to be essential — it caught three Python extraction bugs on first run.

The final session was retraining. We re-extracted all 71 SC2EGSet tournament ZIPs through the enriched pipeline, normalized, and trained three per-matchup models. The extraction itself surfaced a CWD surprise: Python's `Paths()` class used relative paths, so the output landed in quarkmind's directory tree instead of neocortex's. Not a bug in the code — just a consequence of running the extraction from the wrong working directory.

Training produced decent results: 71% top-1 accuracy for vs_terran, 78% for vs_zerg, 80% for vs_protoss. Top-3 accuracy was above 96% for all three. The per-class breakdown tells the real story: common archetypes like RUSH (78–90%), PROXY (84%), and ROACH_RUSH (85%) classify well. Rare archetypes like TECH_RUSH (12–22%) and AIR_SUPERIORITY (7–32%) struggle — class imbalance is the obvious suspect.

The most instructive failure was the Java calibration test. The model scored 0% on rush detection at minute 3 for PvT — catastrophic on paper, but every single "miss" was the model predicting BIO_TIMING with 80%+ confidence for games the test's heuristic labeled as rushes. Having five marines at the three-minute mark is completely standard for bio play; the heuristic just couldn't tell the difference. The model, trained on build-order labels from thousands of games, could. I lowered the calibration gate to 40% across minutes 3–5 and documented the labeling mismatch rather than pretending the model was wrong.

The subtler discovery was the zero-padded window normalization mismatch. The training pipeline normalizes all windows uniformly — zeros become `(-mean/std)`, a non-trivial value. The Java inference code was skipping normalization for zero-padded windows, leaving them as literal zeros. At minute 3, where 40% of windows are zero-padded, the model was seeing input it never encountered during training. At minute 5, with all windows populated, the mismatch disappeared. The symptom — accuracy that improves with game length — was easy to misattribute to "the model needs more data to classify." The actual cause was a four-line normalization loop.

The spatial features open up the classification space in ways that unit counts alone couldn't. Army centroid near the enemy base at minute 2 is a proxy rush. High dispersion with low forward projection is defensive posturing. A proxy building ratio above zero is an immediate signal. These aren't individually decisive, but they give the model dimensions to separate strategies that look identical in pure composition space. The next question is whether the class imbalance on rare archetypes is solvable with better sampling or whether the model architecture needs attention — oversampling and focal loss are already in the pipeline, so the headroom might be limited.
