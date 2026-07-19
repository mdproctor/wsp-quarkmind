---
layout: post
title: "QuarkMind — When the Coach Needs a Playbook"
date: 2026-07-19
type: phase-update
entry_type: note
subtype: diary
projects: [QuarkMind]
tags: [coaching, commentary, strategy, tactic-detection, qhorus, trust]
series: issue-230-coach-mode
---

*Part of a series on [#230 — Coach mode](https://github.com/casehubio/quarkmind/issues/230). Previous: [Teaching the bot to read the room](2026-07-12-mdp01-teaching-the-bot-to-read-the-room.md).*

The commentary system narrates. It watches the game, detects moments, and tells you what's happening — a TV presenter calling the action. Coach mode asks a different question: what should the player *do about it*?

The difference isn't as large as it sounds. Same moment detection, same scouting intel, same summarisation pipeline. The architectural split is in the speech act. Commentary dispatches STATUS messages — informational, no obligation. Coaching dispatches COMMANDs — obligation-bearing, tracked through Qhorus commitments. "They're building lots of tanks" is commentary. "Build Void Rays now — they're going mech" is coaching.

## The compliance problem

Declaring "yes, I'll build Void Rays" is cheap. The interesting question is whether the player actually does it. We designed the coach to observe compliance implicitly — watch the game state after issuing advice and check whether the advised units or buildings actually appear. The coach says "build 3 Stargates" and 450 frames later, if three Stargates exist that weren't there before, the advice was followed.

This creates a natural feedback loop. Compliance feeds a trust dimension. Over multiple games, the system learns which coaching personality the player responds to — directive ("build Stargates now") or Socratic ("what do you notice about your supply?"). The coach that gets followed earns more trust and gets selected more often.

The design review surfaced a critical edge case I hadn't considered: what happens when the coach is Socratic and the player's base is under attack? A question wastes seconds the player doesn't have. The fix was a crisis directive override — when the urgency tier is CRISIS, the coach switches to imperative commands regardless of its personality selection. "What do you notice about the Zerglings at your natural?" becomes "Pull probes and build two Shield Batteries NOW."

## The playbook gap

Here's what surprised me. Our pattern classification system — the Drools rules that identify enemy builds — covers exactly ten archetypes. All early-game: marine rush, zergling rush, cannon rush, bio timing, mech push, banshee harass, and four variants of "they're playing macro." The coach can say "they're doing a marine rush" at the three-minute mark. After that, it's flying blind.

I went looking for structured strategy knowledge. What I found was richer than expected.

[TacticCraft](https://arxiv.org/html/2507.15618) (2025) took 50,000 ZvZ replays and used GPT-4 and DeepSeek to classify them into eight tactical archetypes with softmax confidence scores. The methodology is straightforward and transfers to any matchup. [Spawning Tool](https://github.com/StoicLoofah/spawningtool) has a Python library that parses replays into structured build orders — community-maintained, actively used. The [STARDATA classification](https://annals-csis.org/Volume_25/drp/pdf/48.pdf) project identified ninety strategies (thirty per race) in StarCraft: Brood War by combining build order sequences with army composition signatures.

And Liquipedia's [strategy pages](https://liquipedia.net/starcraft2/Strategy) already follow a formal taxonomy: Opening → Transition → Counter → Endgame for every matchup, with strong and weak counter classifications. The structure a coach needs — "they're transitioning to Brood Lords, your counter is Tempest plus Storm" — already exists as structured community knowledge. Nobody has wired it into a game AI.

One thing that *doesn't* exist: a paired commentary-replay dataset for SC2. League of Legends has LoL19-21 — caster transcripts aligned to match events via YouTube ASR. SC2 has the replays ([SC2EGSet](https://www.nature.com/articles/s41597-023-02510-7), 17,930 esports tournament games) but nobody has aligned caster audio to them. The methodology is proven; the dataset isn't built.

## Three pipelines, shared roots

The coaching pipeline ended up as the third LLM system alongside advisory and commentary. Same pattern: trigger builder, worker factory, channel broker, completion callback. Different semantics: COMMAND instead of STATUS, commitment lifecycle instead of fire-and-forget, compliance evaluator instead of deferred advisory evaluator.

The design review was adversarial and thorough — twenty issues across three rounds, seventeen verified fixes applied to the spec. Two findings were architecturally significant beyond what I'd considered: trust routing needs all coaching agents registered under a single capability (not one per personality), and milestone recording would have produced corrupt ledger data in coach mode because the strategy router returns stale IDs when strategy plugins are gated off.

The gap between ten early-game archetypes and the ninety the literature says exist is the most interesting open question. It's not a blocker — the coaching LLM knows SC2 strategy from its training data, and passing raw scouting observations plus game phase context gives it enough to work with. But "we detect TERRAN_MARINE_RUSH at confidence 0.82" is better coaching context than "there are seven marines and no expansion." The coach can name what it sees, explain the counter, and give a specific instruction — all from structured data rather than hoping the LLM's training data covers this particular build.
