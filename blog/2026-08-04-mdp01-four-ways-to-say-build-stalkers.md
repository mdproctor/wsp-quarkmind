# QuarkMind — Four Ways to Say "Build Stalkers"

**Date:** 2026-08-04
**Type:** phase-update

---

## What I was trying to achieve: a coach that adapts mid-game

V1 of the coaching system had two personalities — directive and Socratic — selected once per game via trust routing. The directive coach says "Build 3 Stalkers now." The Socratic coach asks "What could you build to counter those Roaches?" Both are useful. Neither is right for every moment.

The problem is that games aren't uniform. A player who prefers guiding questions during a calm economic phase still needs direct orders when their base is burning. The personality should shift with the pressure, not stay locked for the whole game.

## What we believed going in: two axes, not one

The original coaching disposition was binary — directive or Socratic. I wanted to expand it to two independent axes: how aggressively the coach responds to urgency (bold vs conservative), and how directly it communicates (tell vs ask). Two axes, four quadrants.

The Eidos personality foundation already had the vocabulary for this. `RISK_APPETITE` maps naturally to urgency response — a bold coach escalates quickly, a conservative one stays measured. `SOCIAL_ORIENTATION` maps to communication style — an independent personality tells, a collaborative one asks. No new foundation types needed. The coaching model just reads what's already there.

## Four quadrants, one agent

The four combinations each have a distinct voice:

**Commander** (bold + independent) — direct orders, imperative voice, urgent. "Build 3 Stalkers now and move your army to their natural." This is the crisis response. When the base is under attack, you don't ask questions.

**Rally** (bold + collaborative) — urgent but guiding. "Your base is under pressure — what could you warp in to defend?" The coach conveys urgency while still letting the player think. A Socratic coach's answer to a military moment.

**Instructor** (conservative + independent) — calm and direct. "You should start +1 weapons from the Forge." Measured advice during quiet moments from a coach that prefers telling over asking.

**Mentor** (conservative + collaborative) — the classic Socratic style. "What tech path do you think would counter those Roaches?" Discovery-oriented, no pressure.

The key design decision: these aren't four separate agents. They're four prompt templates that a single agent traverses as game urgency changes. A Socratic coach (conservative + collaborative) defaults to Mentor during economic moments, shifts to Rally during strategic ones, and promotes all the way to Commander in a crisis. The personality isn't replaced — it's overridden along the urgency axis while preserving the communication style where possible.

This means a player who configures a Socratic coach still gets questions during military moments ("what could you warp in?") but gets direct commands when the Nexus is under attack. The style adapts to the moment without losing its identity.

## The acknowledgment problem

Coaching advice was fire-and-forget. The system would say "Build Stalkers" and then check the game state fifteen seconds later to see if the player actually did it. No way for the player to respond "got it" or "no thanks."

I wanted Accept and Dismiss buttons on each piece of advice. Accept means "I hear you, I'll do it" — the system records the advice as endorsed and stops checking. Dismiss means "I disagree" — recorded as challenged, auto-evaluation stops. The human response is the ground truth. If you told me to build Stalkers and I said no, waiting fifteen seconds to see if Stalkers appeared is pointless.

The tricky part was the qhorus commitment lifecycle. Each piece of coaching advice is dispatched as a COMMAND message with a correlation ID. The commitment state machine expects a response message — DONE or DECLINE — with that same correlation ID to resolve the commitment. But the standard `receiveHumanMessage()` API drops the correlation ID. A garden entry from May had already flagged this: the commitment auto-state-machine never fires for human responses because the correlation ID is null.

The fix was to bypass `receiveHumanMessage()` entirely. The acknowledgment handler calls `MessageService.dispatch()` directly, constructing the DONE or DECLINE message with the correct correlation ID. The compliance evaluator resolves the commitment first (using a CAS operation on the ConcurrentHashMap to prevent race conditions with the auto-evaluator), then the audit trail dispatch follows. If the dispatch fails, the compliance outcome is still recorded — acceptable for a dev-only tool.

The workbench WebSocket, which was previously send-only, now accepts messages back from the browser. The visualizer renders Accept and Dismiss buttons on each coaching item. Clicking one sends a `coaching_response` message with the correlation ID and the response type. The buttons disappear once the compliance resolves — whether by human response or by auto-evaluation, whichever comes first.

## What this enables

The personality model now collects per-agent attribution data. Every coaching commitment records which agent produced it. When a compliance outcome resolves — endorsed, challenged, or superseded — the agent ID is logged alongside the outcome. The data is there for a future trust learning bridge: aggregate outcomes per personality type, feed the aggregates back into personality selection. That bridge isn't built yet, but the instrumentation is.

The four-quadrant model also opens up richer personality configurations. A tournament coach could be configured bold + independent (Commander in all tiers) for aggressive play. A teaching coach could be conservative + collaborative (Mentor by default, Commander only in crisis). The trust routing system can eventually learn which configuration works best for a specific player's skill level and response patterns.

For now, though, the immediate value is simpler: the coach adapts to the moment. When things are calm, it teaches. When things get urgent, it commands.
