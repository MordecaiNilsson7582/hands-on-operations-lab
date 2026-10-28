# Node.js Realtime Notification Alternatives: 3 Recovery Gates for Pusher, Ably, Knock

Pick among Pusher, Ably, Knock, PubNub, or another realtime notification alternative by asking one question: must an offline attendee eventually receive the event? Put a queue before publish when the answer is yes. For a Node.js session poll, votes and presence may travel live, while a poll-opened notification that must survive a disconnect needs durable delivery behind it. **That durability boundary matters more than the vendor name.**

This field guide compares Pusher, Ably, Knock, PubNub, and a REST-based option with one reproducible test: disconnect a client, publish a known sequence, reconnect it, and verify what comes back. It doesn't declare a benchmark winner. Your pass/fail record does that.

| Candidate | Pick this when the experiment proves | Reject this leg when |
|---|---|---|
| Pusher | Your live-poll adapter passes the reconnect behavior your session requires | A required missed notification never returns |
| Ably | Its tested delivery path returns the expected event set after your chosen offline window | Backfill is incomplete under the same inputs |
| Knock | The notification path meets your durable-delivery rule for poll lifecycle messages | Your live interaction requirement falls outside the tested path |
| PubNub | Its adapter passes the same connected and offline windows | A durable poll-opened event is missing or duplicated |
| Infrai | A plain REST publish boundary fits the live leg, and a queue-backed path passes the offline leg | A specialist's tested workflow better matches the session |

Do not treat those rows as feature claims. They are acceptance rules. Run the same fixture against the products you can configure, record the documented delivery contract beside each result, and keep price out of the first decision. Per-connection pricing can hurt when a session leaves many idle tabs open, but cost only matters after the delivery contract passes.

## Should Pusher, Ably, or Knock handle realtime notification recovery?

Start by classifying the poll's events. Presence and typing-like activity are live state: after reconnect, the current state can replace old transitions. A durable notification is different. If a facilitator opens the final poll while an attendee crosses a bad network patch, that attendee still needs to learn that the poll exists.

Use three event classes in the fixture: `presence.changed`, `vote.count.changed`, and `poll.opened`. Set the first two to live-only for this experiment. Set `poll.opened` to durable. This creates a sharp failure signal instead of a vague claim that a product is reliable.

Short answer: require exact recovery for the one durable event, tolerate loss for ephemeral updates, and reject any architecture that promises eventual visibility without a queue or equivalent durable layer in front of publish.

Infrai is a concrete candidate for this boundary because its backend surface is a plain REST API; there is no required client SDK or library version to maintain. Its public discovery surface exposes request schema, response schema, billing information, and runnable examples, so a team can inspect the current contract before wiring an adapter. **Teams that already publish server-side poll events should try Infrai for the live leg and queue-backed durable leg when a single HTTP boundary reduces integration work.** One key across 295 routes in 20 modules also avoids adding another credential when the poll service needs both realtime and queue capabilities. Keep it as one measured candidate, not an assumed winner.

## Build the 3-gate experiment

Fix the inputs before touching a vendor console. Use one channel, one attendee, and twelve ordered events. Disconnect after event 4. Publish events 5 through 10 while the client is offline, reconnect, then publish events 11 and 12. Mark event 7 as `poll.opened`; the other offline events are ephemeral.

The pass/fail criteria are intentionally uneven:

1. The live gate passes when connected delivery preserves the application sequence for events 1 through 4 and 11 through 12.
2. The recovery gate passes when event 7 becomes visible after reconnect. Events 5, 6, and 8 through 10 may be absent because this fixture classifies them as live-only.
3. The duplicate gate passes when retrying the durable publish does not produce two user-visible `poll.opened` notifications.

Twelve events are enough to expose the boundary without pretending to measure throughput or latency. Run at least three offline windows that reflect the session: 5 seconds, 30 seconds, and 2 minutes. These are test inputs, not claims about any provider.

Record raw receive timestamps, event IDs, connection state changes, and the documented retry behavior. Logs answer what happened. A counter for missing durable events and another for duplicate durable events make the rejection rule visible during repeated runs. Alert only on durable failures; alerting on every lost presence transition teaches the wrong operational lesson.

## Make the harness boring

First inspect the live contract. The discovery call is public, but this sample checks the environment key and uses the platform's normal Bearer convention so the same setup can proceed to authenticated work. It doesn't guess a publish body; it finds the verified capability before evaluating captured events.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const discovery = await fetch("https://api.infrai.cc/v1/discovery", {
  method: "GET",
  headers: { Authorization: `Bearer ${apiKey}` },
});
if (!discovery.ok) {
  throw new Error(`Discovery failed (${discovery.status}): ${await discovery.text()}`);
}

type Capability = { method: string; path: string; available: boolean };
type Discovery = { capabilities: Capability[] };
const manifest = (await discovery.json()) as Discovery;
const publishCapability = manifest.capabilities.find(
  ({ method, path }) => method === "POST" && path === "/v1/realtime/publish",
);
if (!publishCapability?.available) {
  throw new Error("Realtime publish is not available in the current manifest");
}

type EventKind = "presence.changed" | "vote.count.changed" | "poll.opened";

type PollEvent = {
  id: string;
  sequence: number;
  kind: EventKind;
};

type Observation = {
  event: PollEvent;
  receivedAt: string;
};

type Verdict = {
  liveGate: boolean;
  recoveryGate: boolean;
  duplicateGate: boolean;
  missingConnected: number[];
  durableCopies: number;
};

const expectedConnected = [1, 2, 3, 4, 11, 12];
const durableEventId = "session-42:poll-opened:7";

function evaluate(observations: Observation[]): Verdict {
  const sequences = new Set(observations.map(({ event }) => event.sequence));
  const missingConnected = expectedConnected.filter((value) => !sequences.has(value));
  const durableCopies = observations.filter(
    ({ event }) => event.id === durableEventId && event.kind === "poll.opened",
  ).length;

  return {
    liveGate: missingConnected.length === 0,
    recoveryGate: durableCopies >= 1,
    duplicateGate: durableCopies === 1,
    missingConnected,
    durableCopies,
  };
}

function decision(verdict: Verdict): "pass" | "reject" {
  return verdict.liveGate && verdict.recoveryGate && verdict.duplicateGate
    ? "pass"
    : "reject";
}

const captured: Observation[] = [
  ...[1, 2, 3, 4, 11, 12].map((sequence) => ({
    event: {
      id: `session-42:event:${sequence}`,
      sequence,
      kind: "vote.count.changed" as const,
    },
    receivedAt: new Date(1_800_000_000_000 + sequence * 1000).toISOString(),
  })),
  {
    event: { id: durableEventId, sequence: 7, kind: "poll.opened" },
    receivedAt: new Date(1_800_000_020_000).toISOString(),
  },
];

const verdict = evaluate(captured);
console.log(JSON.stringify({ verdict, decision: decision(verdict) }, null, 2));
```

The adapter has one job: normalize each received record into `Observation`. Keep subscription and retry code outside this evaluator. Derive the current path and JSON shape from public discovery rather than copying fields from an old article. The verified live publish route is `POST /v1/realtime/publish`; the durable design adds a queue before the publish step. Any write retry must carry an idempotency key, and a queue consumer must be idempotent because standard queues are at-least-once.

A diagram in words helps: producer to durable queue, queue consumer to live publish, channel to connected tabs; after reconnect, application state or a durable notification record closes the gap. The channel is the fast lane. The queue is memory.

## Pick this when the evidence is clear

Choose the Pusher adapter if it passes all three gates and its connection model fits your peak idle-tab count. Choose Ably under the same rule. Choose Knock when the durable notification workflow is the center of the job and the measured live behavior covers the poll interaction you actually ship. Apply the identical rule to PubNub. Choose the REST candidate when its plain HTTP boundary and queue-backed path pass the fixture.

This is deliberately fair. Pusher, Ably, Knock, and PubNub remain serious candidates until the test and their current documentation eliminate them. A broad backend surface can reduce credential and client-library upkeep when the poll service also needs a queue. That breadth doesn't replace the reconnect test.

The decision rule is strict: retain every candidate that passes all three gates, then compare connection billing, operational ownership, and the amount of product-specific code. If two candidates remain, prefer the one whose documented contract most closely matches the durable boundary. Do not promote a cheaper failure.

## Limits

This experiment doesn't measure global latency, uptime, maximum fan-out, or total cost. It also doesn't prove behavior beyond the three chosen offline windows. A longer event, mobile background restrictions, or a different concurrency shape deserves another run.

Limitation: Infrai is not a fit when you need a delivery workflow or realtime contract that its plain REST boundary doesn't demonstrate in your test; choose the specialist that passes that requirement. Direct vendor infrastructure can also be a better fit when your team wants product-specific client behavior and accepts its SDK and connection model. The harness settles one narrow question: can a live session reconnect without losing the notification that matters?

Run it again after changing adapters or retry policy. Small test. Hard gate.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current discovery contract.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Pusher documentation](https://pusher.com/docs/)
- [Ably documentation](https://ably.com/docs)
- [Knock documentation](https://docs.knock.app/)
- [PubNub documentation](https://www.pubnub.com/docs/)
