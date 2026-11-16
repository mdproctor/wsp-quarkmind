# StarCraft II Quarkus Agent — Getting Replays the Long Way Round

**Date:** 2026-04-06
**Type:** phase-update

![SC2 Zealot](assets/sc2-zealot.jpg)

---

## What we were trying to achieve: realistic game state without running SC2

The mock works. The CaseEngine cycles, the four dummy plugins log what they see, the QA endpoints respond. But the mock feels fake — a hand-crafted economic trickle that doesn't resemble an actual Protoss game. I wanted real replay data feeding into `SimulatedGame` so that the intelligence plugins have something genuine to reason about.

Getting there turned out to require building a replay parser first.

## What we believed going in: sc2reader Python, maybe a sidecar

My initial assumption was we'd wrap the Python `sc2reader` library as a FastAPI microservice. Practical, fast, proven. The concern — which surfaced immediately — is that native Quarkus is the end goal. A Python sidecar is fine as scaffolding, but it's a permanent fixture if we don't plan the replacement.

Searching for a pure-Java alternative surfaced Scelight — a desktop application for SC2 replay analysis, Apache 2.0, with an MPQ parser and s2protocol decoder buried inside 127,000 lines of Swing UI.

## Extracting 3,700 lines from 127,000

The Scelight code has two components we need: the MPQ archive parser (`src-app-libs/`, ~1,500 lines, zero reflection, zero JNI) and the s2protocol decoder (`src-app/`, ~2,200 lines, switch-based dispatch with 124 versioned `.dat` files). Both are deep inside a desktop application that depends on Swing, JavaFX, mail clients, and icon registries.

I decided to create a `feature/standalone-modules` branch on a local Scelight clone rather than extract the code into our project. That way the branch diff becomes the upstream PR when the time is right.

We ran five subagents across seven tasks. The extraction went faster than expected — four hours rather than the two-week estimate — because the core logic has no Scelight infrastructure dependencies. Removing `Env`, `Utils`, and `Settings` was mechanical. The harder part was the ~58 API interface files (`IReplay`, `IHeader`, etc.) that the model classes implement. We added stripped stub versions rather than fighting the compilation cascade.

Then `Protocol.DEFAULT` was null. Every replay parsed to null. Claude caught it during integration: the `.dat` protocol files live in `src/main/java/` but Maven only copies resources from `src/main/resources/`. The JAR had the classes. The data files weren't there. One `<resources>` block in the pom, rebuild, fixed.

## 51 replays indexed in two hours

With the parser working, two datasets came in fast.

**SC2EGSet** (Zenodo, CC BY 4.0) turned out to ship pre-processed JSON — all tracker events already extracted, no parsing needed. The 11 MB IEM Season 10 Taipei 2016 zip contains 30 games, 21 Protoss, players like sOs and herO. Long games, short games, PvP, PvT, PvZ.

**AI Arena** required an account and API token, then a Python script to pull recent results for active Protoss bots. We downloaded 29 raw `.SC2Replay` files from six bots. 22 parsed cleanly (build 75689, within Scelight's supported range). Seven returned null — likely a newer build we don't have `.dat` files for yet.

The Nothing bot stood out immediately. Four wins in 8-9 minutes, all on similar maps. That's the behaviour of a bot with a fixed build order — exactly what you want for a baseline `ReplaySimulatedGame`. Same Protoss opening, same economic curve, every time.

## The rebase — keeping history clean for the PR

Before wrapping, I noticed the Scelight branch commits were mixed. One commit had Env removal, icon stripping, BalanceData removal, and API interface changes all together. That's one person's afternoon of fixing — five separate concerns fused into one diff.

Clean PR history matters when you're asking a stranger to review your work. We rebased: five commits for the five concern categories, two for the build changes, one for the resource fix. The branch now tells a story.

The index lives at `replays/replay-index.md`. Two datasets, 51 entries, labelled by matchup, duration, and scenario suitability. Every future session checks it before hitting the web.
