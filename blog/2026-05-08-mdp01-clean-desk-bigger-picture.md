---
layout: post
title: "Clean Desk, Bigger Picture"
date: 2026-05-08
type: phase-update
entry_type: note
subtype: diary
projects: [QuarkusMind]
tags: [emulation, visualizer, testing, planning]
---

The panel worked in the tests. The user said it was broken.

Both were true.

We had Playwright tests that clicked on sprites, verified the panel opened, and confirmed the text fields were populated. Those passed. But running the emulated visualizer and clicking a unit showed a blurry full-width image with no text.

Claude found the cause with one search: `canvas { display: block; width: 100vw !important; height: 100vh !important }`. That rule makes the Three.js renderer fill the viewport. It also applies to every canvas on the page — including the 64×64 portrait in the inspect panel. CSS `!important` on a type selector beats a non-`!important` ID rule, so `#up-portrait { width: 64px }` was overridden silently.

The fix was four words: `#up-portrait { width: 64px !important; height: 64px !important }`. No error had appeared anywhere. The portrait just rendered at viewport width, compressed by the flex container, and blurred.

The second panel issue was the HTTP round-trip. Every click fetched `/qa/unit/{tag}` from the server. In `quarkus:dev`, any request after an idle period triggers the live-reload filesystem scanner — consistently adding 1.5 seconds. The browser already had the unit data from the WebSocket broadcast, so we stopped asking: read `lastState` instead, HTTP only as a fallback. The panel went from a two-second delay to instant.

---

Before starting anything larger, I wanted to clear the backlog. Enemy AI and pathfinder had both shipped weeks earlier, but the planning issues were still open. The Three.js migration in April had left the original PixiJS bundle (`pixi.min.js`) sitting unreferenced — 2226 lines loading nothing. The design document still said "PixiJS 8 visualizer" in eight places. The architectural decisions from the last few months — two-pass combat resolution, why `EmulatedGame` and `SimulatedGame` are separate, why Quarkus Flow needs a single `consume()` step — were well-understood but never formally recorded.

Writing the ADRs is the part that forces precision. The "why not the obvious alternative" is harder to articulate than the "what." Documenting that `MoveIntent` clears `attackingUnits` — not because of SC2 semantics exactly, but because kiting requires it — is more useful than the code ever was.

The scouting threshold calibration was also overdue. The Terran 3-Rax detector was triggering in roughly half of normal professional games — a mean of 4.4 Marines by three minutes, threshold at five. We raised it to seven, which eliminates false positives in the IEM10 dataset. The caveat is that we have no rush-game data, so the true positive rate against an actual three-barracks opening is still unvalidated. Calibrated against what we have.

---

The more important conversation was about what we're actually building.

I'd been treating `EmulatedGame` as a good-enough sandbox — plausible SC2 physics, useful for development, no accuracy claim. While discussing combat engagement, I described the real goal: feed a replay into the engine, get back the same game state the real game produced.

Claude searched the project documents and found `docs/roadmap-sc2-engine.md`, written in April. Phase 6. "Apply intents to shadow simulation." The goal was already there, just never actioned. We'd been building toward it without naming it.

The plan: extract real player commands from replay files, apply them to `EmulatedGame`, compare the resulting state against `ReplaySimulatedGame` tick by tick. Start with economics — build timings, train timings, resource accumulation — because those are fully deterministic and should match exactly. The command extraction infrastructure already exists from the replay playback work: `GameEventStream` and `UnitOrderTracker`. What's missing is the comparison harness.

That's where the next session starts.
