---
layout: post
title: "DTW Doesn't Do What You Think"
date: 2026-08-24
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [cbr, dtw, temporal-matching, time-series, neocortex]
series: issue-222-temporal-replay-matching
---

# DTW Doesn't Do What You Think

The temporal CBR pipeline is wired end-to-end now. A game timeline sampled every 30 seconds — minerals, workers, army supply — gets stored as a TimeSeries feature at game close. During live games, a periodic query finds similar past games by DTW similarity, extracts what happened next in those games, and writes a prediction to the CaseFile. The prediction includes trend direction (economy growing? army spiking?), predicted next phase, and a confidence score weighted by consensus across the top-5 matches.

The interesting part wasn't the pipeline. It was the assumption I got wrong about DTW.

## The assumption

DTW (Dynamic Time Warping) aligns two time series by finding the optimal warping path through a cost matrix. I assumed this alignment path would tell me *where* the current game sits in a longer past game. Query with the first 3 minutes of a live game against a stored 10-minute game, and the alignment maps each query observation to its best match in the stored game. The last alignment pair gives you the alignment point — "you are here" — and everything after that point is your lookahead window. The prediction.

That's how subsequence DTW works. It is not how standard DTW works.

## What actually happens

Standard DTW forces full endpoint alignment. The cost matrix starts at `(0,0)` and the optimal path always ends at `(n,m)` — the last element of both sequences. The backtrace starts there. So the last query observation (minute 3) always maps to the last case observation (minute 10). There is no data after that point. The lookahead window is empty. Every single stored case gets skipped.

The math is unambiguous. Both endpoints are anchored by construction. A 5-observation query against a 20-observation case produces an alignment where observation 5 maps to observation 20. The DTW score tells you how similar the trajectories are. The alignment path tells you nothing useful about where one sequence sits inside the other.

## The fix

Separate the two concerns. DTW answers "which past games look like this one?" Timestamps answer "where are we in those games?"

```java
private static int findClosestTimestampIndex(
        List<Map<String, FeatureValue>> timeline, 
        double targetMinute) {
    int closest = 0;
    double minDiff = Double.MAX_VALUE;
    for (int i = 0; i < timeline.size(); i++) {
        double minute = ((FeatureValue.NumberVal) 
            timeline.get(i).get("minute")).value();
        double diff = Math.abs(minute - targetMinute);
        if (diff < minDiff) { minDiff = diff; closest = i; }
    }
    return closest;
}
```

After DTW retrieval ranks the top-5 similar games, find the observation in each stored game closest to the query's current timestamp. Everything after that is the lookahead. The prediction aggregates trends across all lookaheads, votes on the next phase, and computes confidence from consensus.

This works because the timelines use real-time timestamps. A game at minute 3 with minerals ramping linearly will find past games with similar ramp shapes, and the timestamp puts us at minute 3 in those games. The DTW score filters for trajectory similarity. The timestamp does the spatial alignment. Clean separation.

## The schema upgrade

The `phase_sequence` field — previously a flat `CategoricalList` scored by Jaccard — is now a `DiscreteSequence` with edit distance and a semantic substitution matrix. EARLY_MACRO to MID_SKIRMISH costs 0.2 (different families). EARLY_AGGRESSION to MID_SKIRMISH costs 0.6 (both combat). The matrix captures that "aggressive early then skirmish mid" is closer to "rush then fight" than to "macro then turtle."

DTW at game-loop scale is fast. The calibration benchmark shows 0.016ms per 8x20 observation pair with Sakoe-Chiba band constraint. At 500 stored cases, the entire retrieval is sub-millisecond. No pruning needed at this scale.

The temporal prediction sits at the end of the plugin chain — after scouting, strategy routing, tactics, economics, and summarisation. It doesn't feed back into any of those yet. A follow-up will wire at least one consumer, probably strategy adaptation or the coaching pipeline. For now, it validates the infrastructure: neocortex's DTW and EditDistance machinery works, the storage path is correct, and the query produces structured predictions from real game data.
