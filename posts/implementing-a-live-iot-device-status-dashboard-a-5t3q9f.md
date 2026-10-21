# Implementing a Live IoT Device Status Dashboard API with Node.js

Short answer: send every device over one authenticated dashboard connection, identify each event with both a device ID and a monotonically increasing stream cursor, and keep a bounded replay log behind that cursor. On reconnect, the browser asks for everything after its last applied cursor. If that cursor has expired, Node.js sends a fresh fleet snapshot and resumes live delivery from the snapshot boundary. Treat typing indicators as disposable signals; treat device state and read receipts as recoverable records.

That split is the design. It prevents a reconnect from turning 10,000 devices into 10,000 subscriptions, while still giving the UI a precise answer to the question operators care about: is this screen current?

Before: a socket says `online`, then drops, and the dashboard guesses what changed. After: one channel carries typed envelopes, the client advances one checkpoint only after applying an event, and reconnect has two explicit outcomes: replay or snapshot. The diagram in words is short. Devices feed an ingest service; ingest commits ordered events; the dashboard gateway fans them into one connection; the browser stores one cursor.

## How should an IoT device status API update a live dashboard?

A B2B device-management dashboard often mixes fleet telemetry with human collaboration. An operator may type a note while another operator acknowledges an alert. Those actions look similar on the wire, but they do not deserve the same recovery policy.

A typing indicator is useful for seconds. Replaying it after a laptop wakes up would manufacture stale activity. Device status is different: missing an `offline` transition can leave a green tile on screen until some later update happens to correct it. A read receipt sits between them. The UI can collapse repeated receipts, yet the latest acknowledgement still matters after reconnect.

That is the trap.

Use a small event taxonomy and make the recovery rule visible in the type system:

```ts
type FleetEvent =
  | { kind: "device.status"; deviceId: string; status: "online" | "degraded" | "offline"; observedAt: string }
  | { kind: "receipt.read"; deviceId: string; alertId: string; readerId: string; readAt: string }
  | { kind: "operator.typing"; deviceId: string; operatorId: string; active: boolean };

type RecoverableEvent = Exclude<FleetEvent, { kind: "operator.typing" }>;

type Envelope<T extends FleetEvent = FleetEvent> = {
  cursor: number;
  event: T;
};
```

This is an explicit trade-off. Persisting every keystroke creates replay work with no correctness benefit, so typing stays live-only. Status changes and receipts enter the ordered log because reconnect must repair them.

## Build the replay boundary before the transport

The important API is not `subscribe(deviceId)`. It is `resume(afterCursor)`. Device IDs belong inside envelopes, which lets a single stream carry changes for the whole authorized fleet. Authorization still happens when the connection opens and when data is selected; multiplexing must not broaden tenant access.

Here is a compact in-memory model. It is deliberately transport-neutral, so the same contract can sit behind a WebRTC data channel or another bidirectional connection. Replace the array with a durable ordered store in production, but preserve the boundary rules.

```ts
type Snapshot = {
  boundary: number;
  devices: Array<{
    deviceId: string;
    status: "online" | "degraded" | "offline";
    observedAt: string;
  }>;
  receipts: Array<{
    deviceId: string;
    alertId: string;
    readerId: string;
    readAt: string;
  }>;
};

type ResumeResult =
  | { mode: "replay"; events: Envelope<RecoverableEvent>[] }
  | { mode: "snapshot"; snapshot: Snapshot };

class FleetLog {
  private nextCursor = 1;
  private readonly retained: Envelope<RecoverableEvent>[] = [];

  constructor(private readonly retentionLimit: number) {}

  append(event: RecoverableEvent): Envelope<RecoverableEvent> {
    const envelope = { cursor: this.nextCursor++, event };
    this.retained.push(envelope);
    if (this.retained.length > this.retentionLimit) this.retained.shift();
    return envelope;
  }

  resume(
    afterCursor: number,
    makeSnapshot: (boundary: number) => Snapshot
  ): ResumeResult {
    const boundary = this.nextCursor - 1;
    const oldest = this.retained[0]?.cursor ?? this.nextCursor;

    if (afterCursor > boundary || afterCursor < oldest - 1) {
      return { mode: "snapshot", snapshot: makeSnapshot(boundary) };
    }

    return {
      mode: "replay",
      events: this.retained.filter(({ cursor }) => cursor > afterCursor)
    };
  }
}
```

The off-by-one detail matters. If the oldest retained event is cursor 501, a client holding 500 can replay safely; a client holding 499 cannot. Guessing here creates silent gaps.

One number changes the answer.

A snapshot also needs a boundary captured with the data. The client installs the snapshot, sets its checkpoint to that boundary, and then accepts events above it. If snapshot rows and the boundary are read at unrelated times, an update can fall into the crack between them. Use a transaction or an equivalent consistent-read mechanism in the backing store.

## Apply events without letting duplicates corrupt the view

Reconnect races are normal. The old connection may deliver a late packet while the new connection is replaying. The browser should therefore accept duplicates and reject anything at or below its applied cursor. Keep mutation and checkpoint advancement together.

```ts
type DashboardState = {
  cursor: number;
  statusByDevice: Map<string, string>;
  readAlerts: Set<string>;
  typingByDevice: Map<string, Set<string>>;
};

function applyEnvelope(state: DashboardState, envelope: Envelope): void {
  if (envelope.cursor <= state.cursor) return;

  const event = envelope.event;
  if (event.kind === "device.status") {
    state.statusByDevice.set(event.deviceId, event.status);
  } else if (event.kind === "receipt.read") {
    state.readAlerts.add(`${event.deviceId}:${event.alertId}:${event.readerId}`);
  } else {
    const operators = state.typingByDevice.get(event.deviceId) ?? new Set<string>();
    event.active ? operators.add(event.operatorId) : operators.delete(event.operatorId);
    state.typingByDevice.set(event.deviceId, operators);
  }

  state.cursor = envelope.cursor;
}

function installSnapshot(state: DashboardState, snapshot: Snapshot): void {
  state.statusByDevice = new Map(
    snapshot.devices.map(device => [device.deviceId, device.status])
  );
  state.readAlerts = new Set(
    snapshot.receipts.map(
      receipt => `${receipt.deviceId}:${receipt.alertId}:${receipt.readerId}`
    )
  );
  state.typingByDevice.clear();
  state.cursor = snapshot.boundary;
}
```

Notice what the snapshot does not restore: typing. Good. A reconnect clears those indicators, and fresh operator activity recreates them. The snapshot does restore the latest material state and acknowledgement records.

For a DataChannel implementation, make transport pressure observable. The WebRTC specification exposes `bufferedAmount`, defines a low-buffer threshold, and allows ordered delivery to be selected. Those are transport controls, not a recovery protocol. The application cursor remains necessary because closing and recreating a channel does not explain which business events the browser applied.

The referenced standard is WebRTC 1.0.

## Test the gap instead of the happy path

A unit test that opens a connection and sees one green tile proves almost nothing. Exercise the boundary: apply an event, disconnect, append several recoverable events, discard a typing event, reconnect from the saved cursor, and compare the final state with a clean snapshot. Then repeat with a cursor just outside retention.

Three assertions catch most contract mistakes:

1. Replay contains every retained recoverable event above the requested cursor, in cursor order.
2. An expired or impossible cursor returns a snapshot, never a partial replay.
3. Applying the same replay twice produces the same dashboard state and cursor.

Test slow consumers too. Set a queue ceiling per connection. Once a browser falls behind, stop adding replaceable status updates to an unbounded queue; close the connection and let the normal snapshot path recover it, or coalesce pending status by device while preserving the cursor contract. The correct choice depends on whether intermediate transitions have audit value. For a current-state dashboard, coalescing is often acceptable. For an incident timeline, it is not.

Ship telemetry with the feature. Record reconnect outcome (`replay` or `snapshot`), requested cursor, returned boundary, replay count, replay age, connection queue depth, and snapshot duration. Alert on a sustained rise in snapshot fallbacks or queue-limit disconnects. Those signals turn a vague complaint such as "the board looked stale" into a boundary you can inspect.

Keep labels bounded. Device IDs belong in logs and traces, not as metric labels across a large fleet. Aggregate metrics by tenant tier, region, or outcome, then use a sampled trace or structured log to find the individual device.

## What about per-device ordering and acknowledgements?

A single stream cursor gives the browser a clean recovery checkpoint, but it does not replace domain timestamps. Include `observedAt` on device reports and reject an older observation when devices can upload delayed measurements. The stream cursor answers "what did this dashboard consume?" The observation time answers "when did the device report this state?" They solve different problems.

Read receipts deserve a stable identity such as alert, reader, and device. That makes retries harmless. If the product only displays "read by anyone," collapse records when building the view; do not weaken the ingestion identity just because the UI is compact.

There is also a team trade-off. One fleet stream is easier for frontend code and reconnect bookkeeping, while the gateway must filter authorization carefully and manage fan-out pressure. Per-device channels move complexity into connection churn and checkpoint sets. For a dashboard whose main action is scanning the whole fleet, one channel and one cursor make the failure model much easier to teach, test, and operate.

Start with the recovery invariant: after replay or snapshot, the browser matches an authorized current view at a declared cursor. Transport selection comes next. No mystery state.

## References

The transport behavior discussed above is defined by the primary WebRTC specification. The replay, snapshot, idempotency, retention, and observability rules are application-level design choices in this example.

## Further reading

- W3C, WebRTC 1.0: Real-Time Communication Between Browsers: https://www.w3.org/TR/webrtc/
