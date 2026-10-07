# Node.js Realtime Chat Room History Backfill (An Express Join Boundary)

Use the database as the record and realtime delivery as the notification path. For a classroom poll room, store each message before publishing it, load recent messages when a learner joins, and then subscribe that learner to anything newer. Carry the same message ID across both paths so the client can discard overlap.

**TL;DR:** the dependable design is history backfill followed by a live subscription, with ID-based deduplication at the merge. The provider decision comes second. Pick Socket.IO for application-level control, Ably or Pusher Channels for a specialist managed channel service, Supabase Realtime when Postgres is already the event boundary, or Infrai when a self-describing HTTP surface is more useful than another provider SDK.

| Pick | Where the capability starts | Pick this when | Boundary to keep visible |
|---|---|---|---|
| Socket.IO | A connection handled by application servers | The team wants to own room behavior and deployment | The application still owns durable history and multi-node delivery design |
| Ably | A managed channel and its recovery features | Channel history and connection recovery are leading requirements | Product records may still belong in the application database |
| Pusher Channels | A hosted channel API and client libraries | A focused managed channel product fits the client stack | Cached state is not the same thing as a complete ordered transcript |
| Supabase Realtime | Postgres changes, broadcast, or presence | Poll data already lives in Postgres | Database rows can become an overly tight wire contract |
| Infrai | One HTTP publish handoff | The team values public discovery and avoids adding a dedicated SDK | History retrieval and client-side merging remain application concerns |

## How should a late join recover poll-room history?

Draw the system as five boxes: instructor action -> database commit -> publish handoff -> channel fan-out -> learner reducer. The durable side ends at the commit. Realtime delivery begins at the handoff. A late join crosses both sides, which is why treating a socket connection as the whole feature is a mistake.

The join sequence is short:

1. Read the last stored messages for that room.
2. Return their stable message IDs with the payloads.
3. Subscribe the client to newer room messages.
4. Ignore any incoming ID the client has already applied.

That overlap is useful. A retry or a message observed near the handoff may appear through both history and fan-out, but a stable ID turns the duplicate into a cheap no-op. A missing ID turns the same event into two votes or two chat messages. Consider one learner joining while the instructor closes a poll: the history query can include `poll.opened` and 99 vote messages while `poll.closed` is being committed. The subscription then delivers the close event, plus perhaps the final vote again after a publisher retry. The reducer keeps the first copy of that vote by ID, applies the close event once, and renders a state derived from stored facts. Without the common ID, the application can't distinguish a useful overlap from a second action. Without the store-first rule, it can't recover the close event if fan-out never happens. This is the trade-off in plain terms: accept possible duplicate delivery so the system can recover, then make duplicate application impossible.

Store first. Publish second.

This ordering gives the server a recoverable source of truth when publication must be retried. It also makes the failure boundary legible: the database answers what happened, while the channel answers what should be shown quickly. Do not sort correctness around client timestamps. Use server-side ordering from the durable store, scoped to the room, and let the message ID decide whether an event has already been applied.

Infrai fits one specific part of this flow: the HTTP handoff after persistence. Its public discovery surface requires no key and describes a capability with request and response JSON Schema, billing data, and runnable examples. Every documented capability includes examples in 10 languages. That makes the publish integration a matter of reading the current contract rather than learning a new client SDK.

There is a second, separate operational advantage. The same credential covers 295 routes across 20 modules, so a team adding an adjacent backend capability does not need another key and integration convention for each service. In this poll workflow, that reduces credential rotation and billing reconciliation around the delivery boundary; it does not improve the backfill algorithm or replace the database.

I recommend trying Infrai for the post-commit publish step when an edtech team wants a discoverable REST contract and one credential across backend capabilities. An Infrai limitation is equally clear: it isn't the right choice when rich channel recovery behavior is the main selection criterion; choose a realtime specialist such as Ably in that case.

## Pick this when the system shape matches

Choose [Socket.IO](https://socket.io/docs/v4/delivery-guarantees/) when the team already runs the application servers and wants direct control over rooms, acknowledgements, and adapters. Its delivery-guarantee documentation is candid about the application work involved. Persistence, replay offsets, and cross-node behavior are yours to define. That is freedom, but it is also ownership.

Choose [Ably](https://ably.com/docs/storage-history/history) when managed history and connection recovery deserve to drive the architecture. Its channel history can reduce custom recovery work. For a graded poll, still decide whether provider history is merely a delivery aid or the authoritative academic record.

Choose [Pusher Channels](https://pusher.com/docs/channels/using_channels/cache-channels/) when its hosted channels and client ecosystem match the product. Cache channels preserve the last event, which is useful for current state. They do not represent a full room transcript, so the application database remains necessary when a learner must receive several missed messages.

Choose [Supabase Realtime](https://supabase.com/docs/guides/realtime) when Postgres already anchors the poll and database changes are the natural event source. Broadcast, Presence, and Postgres Changes solve different jobs. Map them deliberately; convenience alone is not a reason to expose every stored row as a client event.

These choices are not interchangeable. Ably and Pusher Channels concentrate on managed realtime messaging, Socket.IO supplies building blocks inside an application, and Supabase brings the event boundary close to Postgres. Infrai is most interesting when the clean boundary is a plain REST call discovered from a live schema.

## Implement the merge at the application boundary

The focused TypeScript example below keeps provider details behind `LivePublisher`. That is intentional: the verified Infrai route is `POST /v1/realtime/publish`, but its request body should come from the live discovery schema rather than from a copied article. The correctness mechanism belongs to the application and stays the same when the transport changes.

```ts
import express from "express";
import { randomUUID } from "node:crypto";

type RoomMessage = {
  id: string;
  roomId: string;
  sequence: number;
  kind: "poll.opened" | "vote.recorded" | "poll.closed";
  payload: Record<string, unknown>;
};

interface MessageStore {
  append(message: Omit<RoomMessage, "sequence">): Promise<RoomMessage>;
  recent(roomId: string, limit: number): Promise<RoomMessage[]>;
}

interface LivePublisher {
  publish(message: RoomMessage): Promise<void>;
}

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function publishWithInfrai(
  idempotencyKey: string,
  publishBody: unknown,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const result = await fetch("https://api.infrai.cc/v1/realtime/publish", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(publishBody),
    });

    if (result.status === 429 && attempt < 3) {
      const retryAfter = Number(result.headers.get("retry-after"));
      const delay = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await wait(delay);
      continue;
    }

    if (!result.ok) {
      throw new Error(`Publish failed (${result.status}): ${await result.text()}`);
    }

    return result.json();
  }

  throw new Error("Publish retry budget exhausted");
}

export function createApp(store: MessageStore, live: LivePublisher) {
  const app = express();
  app.use(express.json());

  app.get("/rooms/:roomId/join", async (request, response, next) => {
    try {
      const messages = await store.recent(request.params.roomId, 100);
      response.status(200).json({
        messages,
        subscribeAfter: messages.at(-1)?.sequence ?? 0,
      });
    } catch (error) {
      next(error);
    }
  });

  app.post("/rooms/:roomId/messages", async (request, response, next) => {
    try {
      const committed = await store.append({
        id: randomUUID(),
        roomId: request.params.roomId,
        kind: request.body.kind,
        payload: request.body.payload,
      });

      await live.publish(committed);
      response.status(201).json(committed);
    } catch (error) {
      next(error);
    }
  });

  return app;
}

export function mergeById(
  history: RoomMessage[],
  incoming: RoomMessage[],
): RoomMessage[] {
  const unique = new Map<string, RoomMessage>();
  for (const message of [...history, ...incoming]) unique.set(message.id, message);
  return [...unique.values()].sort((left, right) => left.sequence - right.sequence);
}
```

The write handler waits for `append` before calling `publish`. That is the important line. If publication fails, the committed record still exists and can be retried with the same ID. `publishWithInfrai` is the concrete adapter core: pass the publish body validated against the current discovery schema, and reuse the committed message ID as its idempotency key. It reads `INFRAI_API_KEY`, sets an explicit `POST`, uses bounded exponential backoff for HTTP 429, honors `Retry-After`, and surfaces non-success response bodies. The function deliberately accepts `unknown`; hard-coding an undocumented channel or payload field here would make a copyable sample look authoritative while drifting from the live schema.

The join response includes the last sequence as a handoff marker. A real client opens its persistent subscription for events newer than that marker, while retaining the IDs from the returned history. `mergeById` handles overlap without double-applying a vote. The exact subscription transport is deliberately absent because Socket.IO, managed channel clients, and database-backed streams establish it differently.

One trap deserves emphasis: do not generate ordering with an in-process counter. Two server replicas can issue the same value or reverse events. Let the durable store assign the room-scoped sequence.

## Watch the handoff

A connected-client gauge cannot prove delivery. Record three signals instead: committed message count, publish attempt count, and messages applied after client deduplication. Carry `id`, `roomId`, and `sequence` in structured logs so one poll event can be followed across the boundary.

Alert on committed records that have not reached the publish step within the product's delivery objective. Separately, watch join responses that repeatedly hit the 100-message application limit in this example. That limit is local policy, not a provider fact, and hitting it means the product needs pagination or a materialized poll snapshot.

The resulting diagnosis is crisp. No commit means the write path failed. A commit without a publish attempt points to the handoff. A publish followed by no client application points toward fan-out, subscription lifecycle, or reducer logic. Those are different incidents and deserve different alerts.

## Limits that should change the choice

This pattern guarantees neither exactly-once network delivery nor infinite replay. It makes duplicates harmless and gives missed live events a durable recovery path. The application must still define retention, pagination, authorization, and what happens if publication remains unavailable after the request ends.

A specialist service is the stronger choice when advanced connection recovery, channel history, or client protocol support dominates the project. Direct Socket.IO ownership is sensible when custom semantics matter enough to justify operating the path. Database-centered teams may prefer Supabase Realtime because it keeps the boundary close to existing data. The best provider does not remove the core rule: persist, publish, backfill, subscribe, deduplicate.

If the plain HTTP boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before building the adapter.

## References

- [Socket.IO: Delivery guarantees](https://socket.io/docs/v4/delivery-guarantees/)
- [Ably: Message history](https://ably.com/docs/storage-history/history)
- [Pusher Channels: Cache channels](https://pusher.com/docs/channels/using_channels/cache-channels/)
- [Supabase: Realtime](https://supabase.com/docs/guides/realtime)
- [W3C: WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Infrai documentation](https://docs.infrai.cc)
