# Node.js Send Bulk Event Notifications: Email Batches with Queue Workers

Send a receipt only after payment settlement, but design the system so an operator can prove what happened to every order. The useful unit is an evidence chain: settlement event, notification intent, attempt, remote acceptance, and terminal state. Batch size and transport are implementation details inside that chain.

| Operating choice | Pick this when | Evidence you must retain |
| --- | --- | --- |
| Database outbox with a poller | Order volume is steady and the order database is the natural audit surface | Claim lease, attempt record, and next eligible time |
| Dedicated queue worker | Bursts make polling latency or database contention unacceptable | Queue receipt, visibility timeout, and handoff event |
| Scheduled sweep | A bounded dispatch window is acceptable for a non-urgent receipt | Sweep watermark and reconciliation result |

The decision rule is simple: choose the path your on-call team can replay and explain, then measure whether its backlog violates the delivery target.

## How should Node.js send bulk event notifications by email?

Start with a notification intent keyed by `order_id`, channel, and template version. Keep `ready_at`, attempt count, lease expiry, the last error class, and a remote message identifier when the transport returns one. A unique constraint on that key prevents a retry from creating a second intent. It does not prevent a remote service from accepting the same request twice, so the business record needs an idempotency decision as well.

The log line should carry the order identifier, notification identifier, channel, template version, and a redacted destination. Never put the receipt body or a full phone number in logs. Those fields let an engineer start with a complaint and walk backward without copying customer data into a ticket.

That trail is the product. The message is only one output.

The limitation is fundamental: no worker can prove a handset received an SMS or a person opened an email.

Three timestamps tell a better story than a single success counter: settlement time, enqueue time, and accepted-send time. Record them for both email and SMS. DKIM signs email messages, but signing does not prove that a recipient read the message; it is one checkpoint in the chain, not the outcome.

## How do batch workers preserve that chain?

Claim work briefly, release the database transaction, call the network, and record the result. Holding a transaction open during SMTP or SMS I/O turns a slow dependency into a lock queue. Releasing it creates a duplicate window, which is why the intent key and attempt history matter.

```ts
type Channel = "email" | "sms";
type Job = { id: string; orderId: string; channel: Channel; payload: string };

interface Sender {
  send(job: Job): Promise<{ messageId?: string }>;
}

async function drain(db: any, sender: Sender, now = new Date()) {
  const jobs: Job[] = await db.claimReady({
    limit: 40,
    leaseUntil: new Date(now.getTime() + 30_000),
    before: now
  });

  for (const job of jobs) {
    try {
      const result = await sender.send(job);
      await db.markSent(job.id, result.messageId ?? null, new Date());
    } catch (error) {
      const retryable = classify(error) === "transient";
      await db.markAttempt(job.id, {
        status: retryable ? "ready" : "dead",
        nextAt: retryable ? new Date(Date.now() + 15_000) : null,
        error: String(error)
      });
    }
  }
}

function classify(error: unknown): "transient" | "permanent" {
  return /timeout|429|5\d\d/i.test(String(error)) ? "transient" : "permanent";
}
```

The number 40 is a starting probe, not a promise. Increase it only after checking lock duration, provider response time, and memory. Keep email and SMS batches independent. SMS length depends on GSM-7 or UCS-2 encoding; a receipt that looks short in JavaScript can become multiple segments. Calculate encoded length before enqueueing and preserve the segment count in the attempt record.

I keep the polling interval visible in the runbook. A ten-second interval adds nearly ten seconds of waiting in the quiet case, while a very short interval spends database reads for no work. A dedicated queue is a poor fit for a small team that cannot rehearse visibility timeouts and dead-letter recovery. A scheduled sweep is a poor fit when customers expect a receipt immediately after settlement. Those are boundaries, not ranking claims.

## Which failures should the test plan rehearse?

Test the awkward interval after a remote accept and before `markSent`. Kill the worker there, let the lease expire, and verify that reconciliation finds the ambiguous attempt instead of silently creating a new receipt. Test a duplicate settlement event, a malformed destination, a rate-limit response, and a process restart with 40 claimed jobs. The expected result is a visible state transition for each case.

The ugly test is the valuable one. It failed once in a staging drill because the cleanup job treated an expired lease as a fresh intent; the fix was to preserve the original intent key and append an attempt row. That rule also makes migrations safer: deploy the new worker beside the old one, read both schemas, and remove the old writer only after a full reconciliation window.

For scheduled polling, the clock should trigger a drain, not decide what is true. Expired leases become claimable, and a reconciliation sweep searches for settled orders without a terminal notification state. A queue has equivalent obligations under different names: visibility timeout, retry policy, and dead-letter handling need the same tests.

Small batches are easier to inspect. Large batches drain faster but make a bad payload fan out quickly.

## What should the dashboard show, and where does it stop?

Chart ready, leased, sent, and dead counts by channel. Add oldest-ready age, attempt-count percentiles, lease expirations, and the time from settlement to accepted send. Alert on a rising oldest age and a sudden permanent-error ratio. Keep a daily sample of provider responses and reconcile accepted message IDs with local terminal states.

These measures prove what your system attempted and recorded. They cannot prove handset delivery or that an email was read. That limit belongs in the runbook, beside the replay procedure and the retention policy. Observability is successful when an engineer can explain one order without guessing.

## References

- https://datatracker.ietf.org/doc/html/rfc6376
- https://www.twilio.com/docs/glossary/what-sms-character-limit
- https://nodejs.org/api/timers.html
