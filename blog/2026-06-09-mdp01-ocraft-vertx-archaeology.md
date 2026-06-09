# ocraft Archaeology — Getting SC2 Running on a Modern Mac

*2026-06-09*

StarCraft II is installed. The smoke test for #13 should be straightforward: launch with
`-Dquarkus.profile=sc2`, let ocraft connect, watch the game loop tick. It took a full
session to get there, but the game is now running, probes are mining, and the scenario
runner can spawn Zealots that the strategy layer reacts to. Here's what was actually in the way.

## ocraft 0.4.21 on Vert.x 4.x

ocraft-s2client was last meaningfully maintained in 2021. It was built against Vert.x 3.x.
Quarkus 3.x bundles Vert.x 4.x. These two facts collide in seven places:

1. `MessageProducer.setWriteQueueMaxSize(int)` — `MessageProducer` no longer extends
   `WriteStream` in Vert.x 4.x; the method is gone entirely
2. `MessageProducer.writeQueueFull()` — same removal
3. `JsonObject.put(String, String)` / `put(String, Integer)` / `put(String, Long)` —
   all typed overloads collapsed into `put(String, Object)` only
4. `HttpClientOptions.setPipelining(boolean)` — removed
5. `HttpClientOptions.setMaxWebsocketFrameSize(int)` — renamed
   `setMaxWebSocketFrameSize(int)` (capital S in WebSocket)
6. `HttpClient.websocketStream(port, host, uri)` — replaced by `rxWebSocket()`
7. `WebSocket.writeBinaryMessage(Buffer)` on the *core* class — now requires a
   `Handler<AsyncResult<Void>>` parameter; the RxJava2 binding still has the old signature

The errors surface one class at a time as each one loads. Six rounds of diagnosis before
the connection attempt even reaches SC2.

The fix is bytecode surgery: compile patched versions of the five affected classes against
Vert.x 4.x, then replace the `.class` files in the ocraft JAR in the local Maven repo.
Python handles the zip manipulation. Not elegant, but the patches are documented in
`NATIVE.md` and the patch scripts are in `/tmp/` — reapply them if anyone runs
`mvn dependency:purge-local-repository`.

## The Mac Path Separator Bug

Once the Vert.x issues are cleared, the next failure is a `NullPointerException` in
ocraft's `ExecutableParser`, originating from `System.exit(0)`. In Quarkus dev mode,
`System.exit()` is intercepted and rethrown as a non-`Exception` Throwable — so
`catch (Exception e)` misses it entirely, `@Retry` doesn't fire, and `@Fallback` fires
without logging the actual cause. Took a while to find.

The root cause: `getBaseBuildFromGameExePath()` in `ExecutableParser` splits the executable
path using `\\` — Windows backslash — to find the `Base96883` directory segment. On Mac,
the path is `/Applications/StarCraft II/Versions/Base96883/SC2.app/...`. Splitting on `\\`
produces one segment. None start with "Base". Returns null. The subsequent
`null.replaceFirst("Base", "")` NPEs.

The fix: set `System.setProperty("ocraft.game.executable.baseBuild", "Base96883")` before
ocraft's `S2Coordinator.setup()` is first called, so Typesafe Config picks it up as
`customBaseBuild` and the broken path extraction is skipped entirely. The system property
must be set before ocraft's static `OcraftApiConfig.config = ConfigFactory.load()` runs —
class initialization order matters here.

## SC2 Only Accepts One WebSocket Connection

With the path bug fixed, ocraft launches SC2 successfully (`Launched SC2 (...) PID: 5075`)
and starts retrying the WebSocket connection. Twenty retries, no delay between them, all
failing with `Connection closed while handshake in process`. SC2 is running, port 8168
is open — but every handshake fails.

The reason: SC2 in API mode accepts one WebSocket client at a time. ocraft's internal
retry loop (`retry(20)`) fires immediately, reusing the same `HttpClient` pool. SC2 still
has the half-closed previous connection in its table and rejects new handshakes until
the TCP slot is reclaimed. Rapid retries make it worse, not better.

The fix is a 3-second delay between retries in `S2ClientVerticle.connect()`:
```java
.retryWhen(errors -> errors.take(RETRY).flatMap(e ->
    Observable.timer(3, TimeUnit.SECONDS)))
```

Three seconds is enough for SC2 to clean up. Also: always kill stale SC2 processes before
starting a new session — `pkill -9 SC2` — because SC2 holds the port until the OS reclaims
it. A process from a previous session will cause the same silent failure.

## It Actually Works

With all three issues resolved, the connection log reads:

```
[SC2] Connected — coordinator ready
[SC2] Starting game on map: TorchesAIE_v4
Waiting for the JoinGame response.
```

The economics workflow fires on every tick with real SC2 data — probe positions updating,
minerals accumulating, the Nexus at position (124.5, 159.5). The strategy layer reads the
game state and outputs `MACRO`. A `POST /sc2/debug/scenario/spawn-enemy-attack` endpoint
call spawns two Zealots and a Stalker via ocraft's debug API; `enemyArmySize: 6` appears
in the next game state observation, and the strategy shifts to `DEFEND`.

`leaveGame()` terminates SC2 cleanly — `[SC2] Left game` logged, PID 5075 gone.

## What's Next

The JAR patching is functional but fragile. The deeper issue is architectural: ocraft
creates its own Vert.x instance alongside Quarkus's. Two Vert.x event loops for one
WebSocket connection. The right answer is to replace ocraft's transport layer entirely —
vendor `ocraft-s2client-protocol` for the protobuf types, write a ~200-line Quarkus-native
WebSocket client using Quarkus's own Vert.x, and drop the RxJava2 transport dependency
that's causing the version friction. The SC2 protocol is a single WebSocket endpoint with
protobuf request/response frames. It doesn't need a framework.

That's #185.
