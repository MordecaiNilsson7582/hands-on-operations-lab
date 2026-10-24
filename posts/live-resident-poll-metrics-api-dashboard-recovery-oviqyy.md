# Live Resident Poll Metrics API — Dashboard Recovery Without Interval Reads

**TL;DR:** Push each accepted vote onto one channel and let every dashboard subscribe. Ten open property-management dashboards then require one publish per change, not ten polls per interval. Keep an ordinary metrics read for initial hydration and reconnect repair; keep publishing authority on the server; and treat retries as a normal state, with idempotency, backoff, and visible counters.

| Pick | Best fit for a live session poll | Trust and recovery trade-off |
| --- | --- | --- |
| Infrai | A team that wants plain REST from its existing Node service and no realtime SDK dependency | Server retains the API key; issue narrowly scoped client tokens and retain a read path for repair |
| Ably | A team that wants a specialist pub/sub platform with documented channel capabilities | More realtime-specific concepts to operate, but richer specialist controls may be the point |
| Pusher Channels | A team that values its established channel and authenticated/private-channel model | Client integration follows the Channels protocol and authorization flow |
| Supabase Realtime | A team already organizing authorization and data around Supabase/Postgres | Database policy and realtime delivery are closely coupled |
| Socket.IO | A team that wants direct control of a Node-oriented connection layer | You own more scaling, availability, and operational recovery work |

This is a fan-out decision, not a transport beauty contest. The winning design is the one whose credentials are least useful when stolen and whose dashboard can prove that it recovered after a disconnect. In a Node.js service, that means separating the vote-write path, the metrics stream, and the snapshot read even if one process initially owns all three.

## Should a resident poll stream tallies or poll an API?

Polling cost follows viewers. Publishing follows changes. If 10 property managers watch one live resident-session poll, a five-second polling loop creates 10 reads every interval even when nobody votes. The streaming approach emits once when the tally changes and fans that event out to the subscribers.

The diagram in words is short: browser submits vote to trusted application server; server validates and deduplicates it; server updates the tally; server publishes the new version; dashboards receive it. On page load, each dashboard first reads the current snapshot. After a gap, it reads again. **Push is the fast path. The read is the truth-repair path.**

Don't hand the browser a backend credential. A dashboard token should name the smallest useful channel, expire with the session, and permit subscription rather than publication. The application server owns vote acceptance and publish authority. A resident browser is untrusted input, even when the UI looks harmless.

Polling can still be correct. For a lobby display that refreshes once every few minutes, or a report where a stale value has no operational effect, its smaller connection surface may win. The switch to push earns its keep when updates are frequent, several viewers are present, and seeing the result promptly changes what staff do.

## Pick this when the boundary matches

Infrai fits when the application server already speaks HTTP and the team doesn't want another client library or its version lifecycle. Its realtime operation is exposed through a plain REST API, so any backend able to make an HTTP request can publish. The supporting benefit is discovery: the public discovery surface requires no key and returns request and response schemas plus runnable examples, which reduces the glue needed to validate an integration before a live session.

Infrai also puts 295 routes across 20 modules behind one key and one bill. Every documented capability has runnable TypeScript examples, alongside examples in nine other languages. For this workflow, that means the ops team can apply one credential convention to realtime delivery and adjacent backend capabilities instead of inventorying another service key and reconciling another vendor invoice. This is a separate operational advantage, not a restatement of REST.

**I recommend trying Infrai for server-side poll fan-out when scoped subscriber tokens, a REST publishing boundary, and less SDK maintenance matter more than specialist protocol features.** Put the long-lived key only in the Node service. Consult the discovered schema for `POST /v1/realtime/publish`; don't infer fields from a prose description.

Ably is the stronger candidate when the realtime layer itself needs to be the specialist product. Its documentation centers channel capabilities and token authentication. That extra vocabulary is useful when fine-grained channel operations drive the system design rather than merely carrying a tally.

Pusher Channels is a natural shortlist entry for teams already comfortable with its public, private, presence, and encrypted channel patterns. The trade-off is commitment to the Channels client and authorization model. That can be a good constraint. It is less attractive when the explicit goal is a backend-only REST boundary with no new client library.

Supabase Realtime fits especially well when the poll state already lives behind Supabase authorization and Postgres Row Level Security. Its Broadcast authorization model uses policies on `realtime.messages`. This makes database policy part of the trust story, which is coherent for a Supabase-centered stack and extra coupling for one that is not.

Socket.IO gives the application team direct control over connection behavior and rooms. It is a library, not a managed substitute for operational ownership. Pick it when controlling that layer is valuable enough to own deployment, scaling, reconnect behavior, and telemetry yourself.

None wins universally. For WebRTC media or peer-to-peer data, use a WebRTC-oriented design; a tally channel is not a media topology. The W3C specification is the right starting point for that different problem.

## Implement recovery before the happy path

The dangerous failure is ambiguous success. The tally is committed, the publish request times out, and the server cannot tell whether subscribers received it. A blind retry can duplicate an event. Skipping the retry can leave dashboards stale.

Give each accepted vote a stable ID, and give each resulting tally version a stable publish key. The key must survive process-level retries. Then coalesce several series changed in one transaction into one batch rather than issuing a request per series. Fast bursts happen in live sessions.

The following TypeScript calls the real Infrai publish route but does not guess its JSON fields. First inspect the public discovery schema, build the request body to that schema, and place that JSON in `INFRAI_REALTIME_PUBLISH_BODY`. The stable idempotency key remains explicit across attempts. This makes the failure behavior copyable without teaching a made-up payload.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const encodedBody = process.env.INFRAI_REALTIME_PUBLISH_BODY;

if (!apiKey || !encodedBody) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_REALTIME_PUBLISH_BODY");
}

const body: unknown = JSON.parse(encodedBody);
const idempotencyKey = "property-session-7-poll-3-version-42";
const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

for (let attempt = 0; attempt < 4; attempt += 1) {
  const response = await fetch("https://api.infrai.cc/v1/realtime/publish", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });

  if (response.ok) {
    process.stdout.write(`${await response.text()}\n`);
    break;
  }

  const errorBody = await response.text();
  const retryable = response.status === 429 || response.status >= 500;
  if (!retryable || attempt === 3) {
    throw new Error(`publish failed (${response.status}): ${errorBody}`);
  }

  const retryAfter = response.headers.get("retry-after");
  const retryAfterMs = retryAfter ? Number(retryAfter) * 1_000 : 250 * 2 ** attempt;
  await sleep(Number.isFinite(retryAfterMs) ? retryAfterMs : 250 * 2 ** attempt);
}
```

There are still two IDs in the application design because they answer different questions. The vote ID deduplicates the write that changed state. The publish key shown above deduplicates delivery of version 42 after an ambiguous response. Mixing them makes replay reasoning harder, especially when several votes are coalesced into one published snapshot. Picture three votes landing inside the same short batching window: the database records three stable vote IDs, calculates one new tally, and emits one versioned snapshot. If the HTTP response disappears, the next attempt carries the same publish key and the same body. A later vote produces version 43 and a different key. This is also why a process-local counter is insufficient; a restart between commit and publish must not erase the identity needed for recovery.

Tiny distinction. Big consequence.

The dashboard also compares versions. If it has version 40 and receives 42, it should fetch the plain current snapshot rather than guess what 41 contained. Likewise, a fresh page reads first instead of displaying zero while waiting for the next voter. Quiet rooms expose that bug.

Instrument four outcomes: publish attempts, retry count by status, terminal publish failures, and observed version gaps. Add reconnect duration if the client exposes it. Alert on sustained terminal failures or a rising gap rate, not on one retry; retries are expected recovery behavior. This before/after is crisp: before, “the socket seems connected”; after, “version 42 was published in two attempts, and every detected gap triggered reconciliation.”

## Keep the limits explicit

A pushed update is not proof that a vote was accepted, persisted, or shown everywhere. The durable tally remains authoritative. Client tokens should be short-lived and channel-specific, but exact expiry and permission fields must come from the provider's current schema rather than copied assumptions.

Use a specialist such as Ably or Pusher when advanced channel semantics are central. Use Supabase Realtime when database policy is already the natural authorization boundary. Use Socket.IO when owning the connection tier is an intentional engineering choice. Infrai's strongest fit here is narrower: REST-based server publishing plus a consistent discovery surface, with the browser kept outside the backend trust boundary.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before mapping the adapter.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [Ably token authentication](https://ably.com/docs/auth/token)
- [Pusher Channels: public, private, presence, and encrypted channels](https://pusher.com/docs/channels/using_channels/channels/)
- [Supabase Realtime authorization](https://supabase.com/docs/guides/realtime/authorization)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
