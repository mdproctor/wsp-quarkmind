---
title: "The Workbench Problem"
author: mdp
date: 2026-08-03
entry_type: note
subtype: diary
tags: [visualizer, workbench, websocket, architecture, observability]
status: draft
---

# The Workbench Problem

The visualizer started as a full-screen Three.js canvas — units moving, buildings rising, minerals depleting. It showed you the game. What it couldn't show you was the thinking.

QuarkMind has four intelligence pipelines running underneath: scouting classifies enemy archetypes, strategy routes responses through CBR, coaching generates real-time advice, and compliance tracks whether the human followed it. All of that reasoning flows through CDI events, Qhorus channels, and CaseFile keys — none of it visible. The visualizer rendered the game world while the intelligence lived entirely in log files.

This is a general problem in agentic systems. The agent acts, but the reasoning behind the action is opaque. You can watch your Stalkers blink forward, but you can't see that the scouting pipeline classified a zergling rush at 87% confidence and the counter recommends gateway units with a wall-off. That information exists — it just has nowhere to go.

## The shape of the fix

The obvious answer is a text panel: bolt a div onto the side of the canvas, pipe some JSON into it, done. I started there, but the more I thought about it, the less it held up. Every future output type — coaching advice, strategy routing explanations, compliance verdicts — would need its own bolted-on panel, fighting for z-index and screen space.

What I actually needed was an application shell. The Three.js canvas isn't the application — it's one pane in it. The workbench owns the frame, the toolbar, the status bar, the panel layout. The 3D view renders inside its allocated cell. Adding a new intelligence surface means adding a page, not rearchitecting the overlay stack.

This is the same shift that game editors made years ago. Unity and Unreal don't render full-screen — the scene viewport is one panel among many. The inspector, the hierarchy, the console all live as siblings in a managed layout. For a living lab whose job is to demonstrate an agentic harness, having the intelligence visibly flowing alongside the game is arguably more valuable than the game rendering itself.

## Selection correlation is where it gets interesting

The real design challenge isn't layout — CSS grid handles that. It's the link between what you see in the 3D view and what you read in the panels.

When the scouting pipeline detects a zergling rush and recommends Stalkers as a counter, clicking "Stalker" in the counter panel should highlight every Stalker in the 3D view with a pulsing green ring. Clicking a specific Stalker in the 3D view should populate the unit detail section with its HP, shields, and position. The two views are correlated — the same selection model drives both.

This bidirectional correlation is what makes the workbench useful rather than decorative. A text dump of intelligence data is a log file with better formatting. A correlated view — where clicking navigates across the text/spatial boundary — is an instrument you can actually use to understand what the agent is doing and why.

The selection model is simple: a shared JS object with `subscribe`, `set`, and `clear`. Two selection modes — tag selection (click a specific unit) and type selection (click a unit type name in the panel). Listeners registered by both the 3D renderer and the panel pages. A re-entrancy guard prevents loops. When the selected unit dies, `syncUnits()` detects the missing sprite and clears the selection automatically.

## Two WebSockets, two concerns

The existing `/ws/gamestate` pushes spatial data every frame — unit positions, building states, resource counts. Intelligence events are different: they fire when something *changes*, not every tick. A pattern assessment might update once every few hundred frames. Coaching advice might fire three times in a game.

A dedicated `/ws/workbench` channel carries intelligence events only. The server observes CDI events from the four pipelines, enriches pattern assessments with counter recommendations from `StrategyTaxonomy`, and pushes typed JSON to connected clients. On reconnect, the server replays its latest-per-type snapshot so the client doesn't start blank.

The two WebSockets don't couple. They meet only at the selection model — when a GameState frame updates a tag-selected unit's live stats in the detail panel.

## What it enables

The workbench is infrastructure, not a feature. Once it exists, every intelligence pipeline has a place to surface its output. The coaching page shows advice with live compliance status. The strategy page shows CBR routing decisions. The pattern page shows detected archetypes with clickable counter recommendations.

And there's an interesting follow-on: if the workbench is the application shell, the Three.js canvas is just one possible renderer for the game view. In `%sc2` profile, the real StarCraft window could replace it — the workbench panels appear as a transparent Electron overlay alongside the actual game. The human plays SC2 normally; the intelligence panels float beside it. That's [#267](https://github.com/casehubio/quarkmind/issues/267) — the canvas and real SC2 are already swappable via profiles, so the workbench infrastructure doesn't care which one fills the viewport.

For anyone building observable AI systems — game AI or otherwise — the pattern generalises. The agent's reasoning needs a surface. That surface needs to correlate with the agent's actions. And the correlation needs to be bidirectional: click the action to see the reasoning, click the reasoning to see the action. A log file gives you one direction. A workbench gives you both.
