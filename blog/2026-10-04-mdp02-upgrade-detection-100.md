---
layout: post
title: "100% Upgrade Detection: When the Obvious Pipeline Lies to You"
date: 2026-10-04
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [sc2, replay-parsing, calibration, signal-processing, upgrade-detection]
---

I started this session expecting to push upgrade detection from 87.7% to 95%. I ended at 100%. The path there was not what I expected — and the lesson is about how aggregate statistics can hide the truth.

## The wrong diagnosis

The first surprise: the main problem wasn't under-detection at all. Running a per-upgrade-type accuracy report revealed massive over-detection. WarpGateResearch showed 91 detections against 48 oracle events — 43 false positives. Charge was +7. PunisherGrenades +12. ShieldWall +15.

Root cause: duplicate abilLink mappings. `ABIL_CONCUSSIVE_SHELLS` (152) and `ABIL_STIMPACK` (165) both mapped PunisherGrenades. `ABIL_CYBERNETICS_CORE` emitted WarpGateResearch, and then the warp-in inference emitted it again because nobody told it the CyberneticsCore path had already fired. Same upgrade, two code paths, double-counted.

Fixing the duplicates and adding emission guards brought accuracy to 97.8%. Clean, deterministic. The remaining 17 missed events were EvolveGroovedSpines (8), EvolveMuscularAugments (7), DarkTemplarBlinkUpgrade (1), and PhoenixRangeUpgrade (1).

## The invisible signal

I declared those 17 events "inherent gaps" — no CmdEvent in the stripped replay data. Claude ran the standard correlation diagnostic and found only ABIL_LARVA as the modal candidate near the missed upgrades. In a Zerg game, Larva spawning fires thousands of times. It's noise, not signal.

I was wrong to stop there.

The breakthrough came from a different diagnostic approach. Instead of statistical correlation across 118 replays, I dumped every single CmdEvent from one specific replay where GroovedSpines was missed — including null-abilLink events that the standard discovery pipeline filters out. Then I filtered by expected research time: GroovedSpines takes 1590 game loops to complete, so the research command should appear exactly 1590 loops before the oracle completion event.

AbiLink 191 showed up at dist=1592 loops. For every single missed GroovedSpines and MuscularAugments upgrade across all replays. The signal was there the entire time — the aggregate correlation was just too noisy to see it.

```java
case ABIL_HYDRALISK_DEN_ALT -> isRace(Race.ZERG) ? switch (idx) {
    case 0 -> upgradeCommand(loop, "EvolveGroovedSpines");
    case 1 -> upgradeCommand(loop, "EvolveMuscularAugments");
    default -> null;
} : null;
```

That pushed accuracy to 99.7%. Two events left.

## The last two

DarkTemplarBlinkUpgrade: abilLink 608 at dist=2717 (expected 2710 — the research time for Shadow Stride). The complication: 608 fires five times in the window because it's the DarkShrine's general interaction abilLink. A first-emission guard solves it — the first 608 in any game is the research; subsequent ones are building interaction.

PhoenixRangeUpgrade: abilLink 69 at idx=2. The existing ABIL_FLEET_BEACON is 71 — same idx, abilLink off by 2. A version-specific shift in the game's ability catalog.

759/759. 100%.

## What this actually means

The 100% is real for this dataset — 118 oracle replays, patch 4.9.3. It does not mean universal. Different patches assign different abilLink values. The 4.10.1 replay pack sits untouched. SC2EGSet on Zenodo has 17,930 tournament replays we haven't touched. Each new patch could introduce abilLinks we've never seen.

What we do have: a diagnostic (`dumpAllCmdEventsNearMissedUpgrades`) that can find them. The standard correlation pipeline is blind to low-frequency signals in noisy event streams. The per-replay temporal filter — "dump everything, filter by expected duration" — cuts through the noise every time. Next step is small samples across many patch versions, testing whether the abilLink mappings hold or need version-specific dispatch.
