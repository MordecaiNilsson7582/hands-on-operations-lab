# Node.js Uptime Chart Design: Bounded Metrics Queries for Health Monitoring

Short answer: treat a simple uptime chart as a bounded time-series problem, not a raw-log export. Choose a point budget, aggregate near the metrics store, page with a stable cursor, and trace each deadline before increasing a timeout. This keeps a 30-day view from doing thirty times the client work of a one-day view.

| Query shape | Use it when | What it costs | Trade-off |
| --- | --- | --- | --- |
| Raw checks | A short forensic window needs exact payloads | Rows grow with every check | Too much data for a long chart |
| Fixed time buckets | A dashboard needs an interactive trend | Rows grow with chart resolution | Per-check detail is summarized |
| Rollup tiers | History spans weeks or months | Storage and retention policy | Resolution changes by age |

## What should a Node.js health dashboard measure before changing a timeout?

Start with the clock that expired. A browser deadline, API deadline, metrics-store deadline, and Node.js aggregation deadline are separate budgets. `AbortError` only says that one boundary stopped waiting; it does not identify the slow layer.

Draw the request as words: **chart click -> dashboard API -> metrics query -> bucket pages -> chart points**. Put a timestamp, correlation ID, requested range, bucket width, page count, returned rows, and response bytes at every arrow. If no first page arrives, inspect selectivity, credentials, and backend aggregation. If page one is quick but page 37 is late, inspect cursor progress, retries, and payload growth. If all pages arrive while the event loop stalls, the reducer or JSON serialization is doing too much work.

Use the same severity vocabulary for these diagnostics. RFC 5424 defines eight numerical severities, from Emergency (0) through Debug (7), with lower numbers indicating greater severity. A query deadline can be mapped to a warning in your policy, while malformed configuration can be an error; the RFC does not choose that application mapping for you. Include elapsed milliseconds and the stage name, but never credentials.

Measure first.

## How can pagination and aggregation keep a large time range predictable?

The chart width gives you a practical upper bound. A 900-pixel plot cannot display millions of distinct checks, so transfer only the information the visual can represent. Pick a maximum point count, round the resulting bucket width to an approved interval, and aggregate `healthyChecks` and `totalChecks` near storage. Keep empty buckets as `null`: zero means checks ran and none were healthy; null means no observation.

The cursor is opaque. The backend must provide stable ordering, and the client must stop on `null` rather than guessing a page number. A repeated boundary bucket is safe to merge when the paging contract permits overlap. A page ceiling protects the service from a cursor cycle.

Here is the core loop. `queryPage` is a generic internal adapter; its contract returns bucket summaries, not raw events.

```ts
type Bucket = {
  startMs: number;
  healthyChecks: number;
  totalChecks: number;
};

type BucketPage = {
  buckets: Bucket[];
  nextCursor: string | null;
};

type QueryPage = (input: {
  service: string;
  fromMs: number;
  toMs: number;
  bucketMs: number;
  cursor: string | null;
  signal: AbortSignal;
}) => Promise<BucketPage>;

const BUCKETS_MS = [60_000, 300_000, 900_000, 3_600_000, 21_600_000, 86_400_000];

function chooseBucketMs(fromMs: number, toMs: number, maxPoints = 600): number {
  const minimum = Math.ceil((toMs - fromMs) / maxPoints);
  return BUCKETS_MS.find((size) => size >= minimum) ?? BUCKETS_MS[BUCKETS_MS.length - 1];
}

async function loadUptimeSeries(
  queryPage: QueryPage,
  service: string,
  fromMs: number,
  toMs: number,
  timeoutMs = 8_000,
): Promise<Array<{ startMs: number; uptimePercent: number | null }>> {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);
  const totals = new Map<number, { healthy: number; total: number }>();
  const bucketMs = chooseBucketMs(fromMs, toMs);
  let cursor: string | null = null;
  let pages = 0;

  try {
    do {
      if (++pages > 100) throw new Error("Pagination limit exceeded");
      const page = await queryPage({ service, fromMs, toMs, bucketMs, cursor, signal: controller.signal });
      for (const bucket of page.buckets) {
        const current = totals.get(bucket.startMs) ?? { healthy: 0, total: 0 };
        current.healthy += bucket.healthyChecks;
        current.total += bucket.totalChecks;
        totals.set(bucket.startMs, current);
      }
      cursor = page.nextCursor;
    } while (cursor !== null);
  } finally {
    clearTimeout(timer);
  }

  return [...totals.entries()]
    .sort(([left], [right]) => left - right)
    .map(([startMs, value]) => ({
      startMs,
      uptimePercent: value.total === 0 ? null : (value.healthy / value.total) * 100,
    }));
}
```

This loop bounds output resolution, total wait time, and page count. It does not make an arbitrary raw scan cheap; that is why the adapter should push the grouping into the metrics query. The exact interval list is an operational choice, and your mileage may vary.

## Which troubleshooting tests separate a query problem from a client problem?

Run controlled cases before touching production deadlines. Compare a 24-hour range with a 30-day range using identical filters. After resolution selection, both should stay near the point budget. An empty range should return empty buckets, not invented downtime. A one-bucket range should preserve both counts. A delayed page should abort and clear its timer. A repeated boundary bucket should merge according to the documented contract. A cursor cycle should hit the 100-page ceiling.

It's a small test — and it catches surprising regressions.

I once chased the reducer after 12 requests stopped at `10,000 ms`. Stage timings showed DNS and connection setup completed, but the first page never arrived. The deployment had a region setting with one transposed character. Logging the resolved region beside the stage exposed it; logging the authorization header would have created a new incident. I'm not sure why that client version hid the upstream authentication detail, but the boundary timings were enough to fix the configuration.

Do not add concurrency inside cursor pagination. Page N normally reveals the cursor for page N+1. Concurrency can sit above it for independent services or stable time partitions, with a small queue that respects connection and query budgets. Ten simultaneous long-range requests can turn one slow chart into shared pressure.

## What are the limits of a simple uptime chart architecture?

The catch is that bucketed data cannot answer every forensic question. It is not suitable when an engineer must inspect an exact check payload, reconstruct event ordering, or apply an ad hoc filter to individual records. Keep a raw path for a short, explicit window and reserve the dashboard path for bounded summaries. Stick with a purpose-built analytical workflow when arbitrary slicing is the primary job.

Rollups also require policy: define retention, timezone boundaries, late-arriving data behavior, and how recalculation works. Review response size, rows scanned, aborted requests, and event-loop delay during a controlled rollout. Logging systems can price ingestion and indexed retention separately, so read the current terms for the system you operate instead of inferring cost from latency.

The durable review question is simple: does a 30-day request do roughly the same client-side work as a 24-hour request? If yes, resolution is controlling workload. If no, trace the arrows again.

## References

- RFC 5424, The Syslog Protocol: https://datatracker.ietf.org/doc/html/rfc5424
- Datadog pricing model reference: https://www.datadoghq.com/pricing/
