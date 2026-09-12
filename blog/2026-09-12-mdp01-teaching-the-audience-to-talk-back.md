---
title: "Teaching the Audience to Talk Back"
date: 2026-09-12
author: mdp
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [commentary, trust, feedback, workbench]
---

# Teaching the Audience to Talk Back

Commentary had one trust dimension: response latency. Fast commentary scored high. Slow commentary scored low. But a fast commentator that fires at boring moments or gets the facts wrong is worse than a slow one that's accurate and well-timed. The trust routing system couldn't distinguish them.

Two new dimensions change that. Each commentary entry in the workbench now carries four buttons — good timing, bad timing, accurate, wrong. A click sends a WebSocket message to `CommentaryFeedbackHandler`, which records an `OutcomeRecord` against the commentary worker's trust profile. The trust routing system uses these scores alongside latency to select which LLM commentary worker handles the next event.

The design mirrors the coaching feedback path exactly. Coaching already had accept/decline buttons on each advice item, routed through `CoachingAcknowledgmentHandler`. Commentary follows the same pattern: custom event bubbles from the Lit component through `visualizer.js` to the workbench WebSocket, handler records the outcome, trust scoring adjusts future worker selection.

The commentary page switched from `blocks-channel-feed` (a shared CaseHub component that renders messages as a chat feed) to custom rendering — the shared component doesn't support per-message action slots, and feedback buttons need to live inline with each entry. The custom rendering follows the coaching page's existing pattern: a styled list with buttons that disable after the first click.

Accuracy has a higher quality floor (0.4) than timing quality (0.3). A commentator that frequently gets facts wrong is more damaging than one whose timing is occasionally off — factual errors undermine credibility, while timing issues are merely annoying.
