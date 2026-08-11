---
layout: post
title: "The Second Red"
date: 2026-06-20
entry_type: note
subtype: diary
projects: [casehub-quarkmind]
tags: [ci, websocket, testing, async]
---

CI went red again on the first push to main.

The dependency resolution failures were gone — scelight and casehub-core inlined, the casehubio/* GitHub Packages repository configured, CI green on the branch. But the push to main triggered a new failure: `GameStateWebSocketTest.jsonContainsShieldsAndMaxShields` timed out after 3 seconds.

Obvious fix: 3 seconds is tight for CI. We bumped it to 10.

Wrong. The next run hit the same timeout — 10.01 seconds exactly — on a different test method. Same symptom, different test. Something was wrong with the whole class.

The actual problem: `WebSocket.buildAsync(uri, listener).join()` looks synchronous. `.join()` blocks until the future resolves. What resolves is the *client-side* handshake — receiving the 101 upgrade response. The Quarkus server-side `@OnOpen` fires afterward, on a separate thread. On a fast local machine the gap is effectively zero. On a GitHub Actions runner under load, `engine.observe()` fires in that gap, before `sessions.add()` has run.

The broadcaster has this guard:

```java
if (suppressed || sessions.isEmpty()) return;
```

Correct for production — don't broadcast to nobody. But in the test, when the broadcast fires into an empty session set, the message is dropped silently. The test polls for the message that was never sent, waits the full timeout, then fails. No error, no warning, nothing in the logs.

The fix: `waitForSession()` in the broadcaster, called in the test's connect helper right after the handshake:

```java
broadcaster.waitForSession(2000); // spins 10ms at a time until sessions.size() > 0
```

The session registers within a few milliseconds. The cap handles slow runners. The test now has a real synchronisation point instead of a timeout guess.

The 10-second timeout increase was a red herring. It made the test wait longer for a message that was never going to arrive.
