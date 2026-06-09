---
layout: post
title: "Pulling out ocraft's Vert.x heart"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [sc2, transport, websocket, ocraft]
---

The original plan for the real SC2 connection was to use ocraft-s2client-bot — the only Java library that speaks the SC2 Remote API. It did the job for a while: ocraft handles the WebSocket handshake, manages the game loop, and wraps the SC2 protobuf API in reasonably clean Java. What it cannot do is run on Quarkus 3.x without becoming a maintenance nightmare.

ocraft was built against Vert.x 3.x. Quarkus 3.34 bundles Vert.x 4.x. The result was seven API incompatibilities that I patched by bytecode-rewriting five classes in the local Maven repo. Not ideal. The patches were silent — `mvn dependency:purge-local-repository` would wipe them without warning, and the failure mode looked like a connection timeout rather than a classpath problem. More fundamentally, ocraft launches its own Vert.x instance inside a Quarkus application that already has one. Two Vert.x instances, neither aware of the other.

I filed #185 to fix the architecture rather than keep patching. The SC2 protocol itself is not complicated: one WebSocket at `ws://127.0.0.1:8168/sc2api`, binary protobuf frames, strictly sequential request/response. ocraft wraps that in ~2000 lines of RxJava2 observables and Vert.x verticle lifecycle. We could write the transport ourselves in about 200 lines.

We went through nine rounds of design review before writing a line of implementation. That number is higher than usual, but the protocol has threading invariants that are easy to get subtly wrong — specifically around `quit()`: if anything calls `sendSync()` from a thread that isn't the game loop virtual thread, you get concurrent WebSocket senders and two competing consumers on the response queue. The spec documented all of this before we touched the code. Each review round tightened one or two invariants: `running = false` must be first in the finally block (so `AgentOrchestrator.gameTick()` stops dispatching before cleanup); the preamble (game info request + onGameStart callback) must be inside the try block so the WebSocket closes even on startup failure; `quit()` sets a flag and interrupts, it never calls `sendSync()` directly.

Implementation went faster than the spec did. The interesting surprises were in the test server.

I wrote a `FakeSC2Server` — a raw `ServerSocket` that speaks the SC2 binary proto protocol — to avoid needing a real SC2 installation for unit tests. The first version called `server.accept()` once and then served frames. This looked correct until I saw why the real WebSocket connection never arrived: the TCP probe in `tcpProbe()` establishes a real TCP connection to check whether the port is open. That probe triggers `server.accept()` to return. By the time the actual WebSocket client connects, nobody is listening. Two hours gone; the fix is a loop.

The second surprise: I used `BufferedReader.readLine()` to parse the HTTP upgrade headers in the handshake. This works — headers end with `\r\n\r\n`, `readLine()` returns an empty string, loop exits. But `BufferedReader` maintains an 8KB internal buffer. It reads ahead past the header terminator into whatever follows — which is the first WebSocket frame. Those bytes are gone. The fix is to read the HTTP headers byte-by-byte and not touch a `BufferedReader` near a mixed-protocol stream.

The JDK `java.net.http.WebSocket` class turned out to be a dead end for local testing. `HttpClient.newHttpClient()` defaults to HTTP/2 preference. Against a plain `ServerSocket` that only speaks HTTP/1.1, the negotiation never completes. No error, no timeout, just a silent hang. I switched to a raw `Socket` with a manual RFC 6455 handshake — which is also what the production transport ended up using, since it gives complete control and avoids the HTTP client stack entirely.

The frame reader took a bit of thought. A single logical WebSocket message might arrive as multiple frames with `last=false` before the final `last=true`. A `ByteArrayOutputStream` accumulates fragments; only when `last=true` does it offer the complete bytes to the `SynchronousQueue`. The frame reader also has to handle RFC 6455 extended length encoding: sentinel value 126 means read two more bytes for the actual length; 127 means eight. There's a latent bug if you only implement the 1-byte and 2-byte cases — anything over 65535 bytes will silently truncate. The code review caught it.

The final transport is 280 lines: manual WebSocket handshake, masked client frames per RFC 6455, `SynchronousQueue` for response pairing, virtual thread game loop with try/catch/finally that sets `running = false` first, sends `RequestQuit` from the game loop thread (not from the external `quit()` caller), closes the socket, then calls `onGameEnd()`. Two new protocols in the quarkmind docs: `InterfaceOptions.raw = true` is mandatory in `joinGame()` (without it, SC2 never populates the observation), and `realtime` must not be set in `createGame()` (setting it silently ignores every `RequestStep`).

The thing that surprised me most: the spec review was more useful than I expected. By the time we got to implementation, nine rounds had already found the `RequestStep` invariant (without it SC2 freezes after the first observation), the thread safety of `quit()`, the `isConnected()` delegation that ensures the scheduler stops immediately on game end, and the `gameStarted` flag that prevents `onGameEnd()` firing when the preamble fails. The implementation itself was relatively smooth. The spec was the hard part — but the spec hardness was the right kind.
