---
layout: post
title: "Unlocking 230K Ladder Replays"
date: 2026-09-27
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [onnx, sc2, data-pipeline, pysc2, docker]
series: issue-312-onnx-class-imbalance
---

# Unlocking 230K Ladder Replays

The ONNX strategy classifier has a data problem. TECH_RUSH sits at 28% accuracy on 82 test samples. AIR_SUPERIORITY at 10% on 164 samples. The model works fine on common archetypes — BIO_TIMING at 79%, RUSH at 78% — but the rare strategies that matter most for scouting are the ones it can't identify.

The root cause isn't the model or the training hyperparameters. I tried raising the class weight cap from 15 to 25 and reducing label smoothing from 0.1 to 0.05. Both slightly regressed TECH_RUSH. The problem is simpler: 82 test samples means ~40 unique TECH_RUSH games in the training set. Oversampling those 40 games to 500 copies means the model memorises rather than generalises. More hyperparameter tuning won't fix a data diversity problem.

Tournament replays are 98%+ standard macro play. Pros don't cannon rush each other. Ladder games are where the cheese lives.

## The Blizzard replay wall

Blizzard's API provides anonymised ladder replays in bulk — hundreds of thousands per game version. I downloaded 230K across versions 4.9.3 and 4.10.1 (25GB). Then opened one with mpyq and found this:

```
Original MPQ files:
  replay.attributes.events
  replay.details.backup       ← not replay.details
  replay.game.events
  replay.gamemetadata.json
  replay.initData.backup      ← not replay.initData
  replay.load.info
                              ← no replay.tracker.events
```

Six files instead of fourteen. No tracker events — no unit births, deaths, player stats, or upgrades. The `.backup` suffix on details and initData means anonymised. The tracker events stream doesn't exist at all.

I spent a while trying to load these through SC2's API directly — manual WebSocket connection to the headless client, passing replay bytes, file paths, with and without the `-dataVersion` flag. Every attempt returned `MissingReplay` with no error details. The SC2 client simply wouldn't recognise the stripped format.

## The PySC2 difference

The sc2api.proto file has a comment on the `record_replay` field: "useful for restoring tracker.events." And AlphaStar was trained by supervised learning on these exact replays — the same `.backup` format. So they're loadable. I was doing something wrong.

Claude dug into PySC2's source and found the difference. PySC2's `run_configs` module doesn't just connect to SC2 — it manages the entire launch: extracting the replay version from `gamemetadata.json`, resolving the matching SC2 build, passing `-dataVersion` with the correct hash, setting up temp directories. The manual WebSocket approach skips all of that, and the stripped replay format needs every piece to be exactly right.

```python
rc = run_configs.get(version="4.9.3")
with rc.start(want_rgb=False) as controller:
    req = sc_pb.RequestStartReplay(
        replay_data=replay_bytes,
        options=interface,
        observed_player_id=1,
        record_replay=True,
    )
    controller.start_replay(req)  # Works.
```

The replay loads. SC2 replays it internally, regenerating all the game state that was stripped for anonymisation. `RequestSaveReplay` writes out a new replay with the full fourteen files — including `replay.tracker.events` at 38–51KB per game.

## Docker + headless = batch pipeline

SC2's Linux headless builds are x86_64 only. I'm on an ARM64 Mac. The solution is Docker with `--platform linux/amd64` — QEMU emulates x86_64, SC2 runs inside the container, PySC2 manages the connection.

The setup: download the SC2 headless build matching the replay version from Blizzard's CDN, extract it with the ladder maps, mount into a lightweight Docker container with PySC2 installed. Each worker gets its own SC2 instance on a unique port.

Tested with 10 replays: 10/10 restored, zero failures. Under QEMU emulation it runs at about 43 seconds per replay — too slow for 230K replays. On native x86_64 that drops to 3–5 seconds per replay. With 4 parallel workers, the full 230K would take about two days on a Linux VM.

## What this opens up

230K ladder replays is more data than most SC2 ML projects have ever worked with. SC2EGSet — the largest published tournament dataset — has 17,930 replays. This is 13x that, and it's ladder data where aggressive strategies actually appear.

Even at a conservative 3% TECH_RUSH prevalence in ladder games, that's ~7,000 new TECH_RUSH samples against the current 246. The oversampling problem disappears — the model gets genuine diversity instead of memorised duplicates.

The multiprocessing batch extraction pipeline is already built and tested at 138 replays/second with 10 workers. Once the tracker events are restored, the replays flow straight through the existing training pipeline. The version 4.9.3 replays have an exact SC2 build match (Base75025). The 4.10.1 replays need a compatible build — the 4.10.0 headless is 111 builds behind, which SC2 rejects. That's a version gap to close, but the architecture handles it.

The rare class accuracy targets — TECH_RUSH ≥40%, AIR_SUPERIORITY ≥30% — were set when we thought more data might not be available. With 230K ladder replays in the pipeline, they start to look conservative.
