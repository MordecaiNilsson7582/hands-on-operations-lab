# Can a Lean Capture API Handle Next.js Backend Exceptions Without Sourcemaps or Replay?

Short answer: for Next.js backend routes with no sourcemaps or replay, use a lightweight error capture API when self-serve exception tracking means one team investigating a deliberately small event envelope; use a full tracker when grouping, assignment, release comparison, and resolution state are the work you need to buy. If structured logs already provide a tested path from alert to cause, add neither.

| Pick this path | Pick it when | You still own | Stop using it when |
| --- | --- | --- | --- |
| Lean error-event API | A few server handlers, one clear owner, explicit context policy | Redaction, deduplication, retention, alert rules, triage handoff | Engineers start rebuilding an issue queue around the endpoint |
| Full exception tracker | Several engineers need shared grouping, ownership, and issue state | Data controls, instrumentation policy, project conventions | The workflow costs more attention than the incidents it resolves |
| Structured logs plus bounded metrics | The existing log path already makes request-level investigation quick | Query quality, retention, cardinality, correlation | Responders repeatedly assemble the same error investigation by hand |

That table is the field guide. “Simplest” doesn't mean the fewest installation steps. It means the shortest dependable path from a thrown value to a useful decision during an incident — including the policy, tests, alerts, and ownership around ingestion.

## How should Next.js backend routes capture exceptions without sourcemaps or replay?

Begin at the catch boundary, not at a product comparison page. A useful boundary knows the stable route template, the operation being attempted, the error class, and a correlation ID. It should not ship the whole request merely because that object is nearby. Cookies, authorization headers, raw query strings, form fields, and request bodies expand the privacy review while making the investigator hunt through more noise.

For a small service, a lean capture API is a good fit when the on-call engineer can answer four questions from a compact event: what operation failed, what kind of failure occurred, which request trail should be opened, and when did it happen? The catch is operational. Ingestion alone doesn't provide sensible grouping, assignment, acknowledgement, retention, or release comparison. A team that needs those shared practices should choose a full exception tracker rather than quietly constructing one from alerts, saved searches, and chat messages.

No replay changes very little for a server-only failure path. Replay explains browser interaction, while a backend handler needs server-side evidence about the operation and its dependencies. No source maps is a sharper constraint: emitted stack locations may be less readable, so stable error classes, an operation name, a deployment identifier when the application already exposes one, and correlation into structured logs become more valuable. Don't compensate by collecting arbitrary context. More payload is not the same as more evidence.

Logs and metrics remain a serious third option. Stick with them when a bounded error counter gets the responder to a searchable, correlated log record in one practiced move. Move to dedicated error events when repeated investigations require the same manual parsing, and move to a full tracker when repeated investigations require shared human state. This frames the choice around the work after capture, where the real difference lives.

Tiny systems count too.

## Draw two lanes before adding exception tracking

Here is the diagram in words. The request enters a handler. Validation handles expected bad input. Application work calls a dependency. An outer boundary catches an unexpected throw, creates a safe envelope, and attempts delivery with a deadline. The original application response keeps its intended semantics. In a second lane, bounded metrics evaluate service health and the responder follows a correlation ID into logs or error events.

Two lanes.

This separation prevents telemetry from becoming a new dependency of the customer path. It also exposes a common blind spot: an HTTP response can report that work was accepted without proving that the promised effect completed. Imagine a route that accepts an export request and returns `202`. The worker later transforms data and stores a file. Counting `202` responses measures acceptance, not completed exports. Exception capture might explain a thrown worker operation, but a completion counter or durable state transition is what detects accepted work that never reached its terminal state. The signals cooperate. They are not substitutes.

Metric labels need the same discipline as event fields. Use a stable route template such as `/api/orders/[id]`, not the raw path for every order. Use a bounded error class, not the full exception message. Prometheus explains why: every unique combination of label values creates another time series, and high-cardinality dimensions can make an instrumentation design expensive and difficult to operate. Request IDs belong in logs or event records, where they support lookup without multiplying metric series.

The browser has another lane as well. Core Web Vitals describe user-visible experience through LCP, CLS, and INP, with the recommended assessment using the 75th percentile. Those measurements answer whether loading, visual stability, and responsiveness are good for users. They don't reveal which server operation threw. Backend exception events don't reveal layout shifts. A product with browser and server surfaces normally needs both questions represented, even when it deliberately uses neither replay nor source maps.

Before selecting a tool, write one sentence for each lane: “this signal pages us when ___,” and “the responder opens ___ next.” If the blanks are vague, adding a richer tracker won't repair the operating model. If the blanks are precise and the only missing piece is a safe error envelope, a lean endpoint may be exactly enough.

## Keep the implementation boundary boring

The route should depend on a tiny capture port, not a vendor SDK or a guessed ingestion URL. An adapter can implement that port later. This keeps the handler copy-pasteable, makes the data contract visible, and lets tests inspect every outbound field without network access.

```ts
type ErrorEvent = {
  occurredAt: string;
  route: string;
  operation: string;
  errorClass: string;
  message: string;
  correlationId: string;
};

type ErrorCapture = {
  send(event: ErrorEvent, signal: AbortSignal): Promise<void>;
};

function normalizeError(value: unknown): Error {
  return value instanceof Error ? value : new Error("Unknown thrown value");
}

function redactMessage(message: string): string {
  return message
    .replace(/[\w.+-]+@[\w.-]+/g, "[redacted-email]")
    .slice(0, 240);
}

export async function runObservedRoute(
  work: () => Promise<Response>,
  capture: ErrorCapture,
  correlationId: string,
): Promise<Response> {
  try {
    return await work();
  } catch (value: unknown) {
    const error = normalizeError(value);
    const event: ErrorEvent = {
      occurredAt: new Date().toISOString(),
      route: "/api/orders/[id]",
      operation: "update-order",
      errorClass: error.name,
      message: redactMessage(error.message),
      correlationId,
    };

    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 800);

    try {
      await capture.send(event, controller.signal);
    } catch {
      // Preserve the application's original error semantics.
    } finally {
      clearTimeout(timeout);
    }

    throw error;
  }
}
```

The adapter's transport, authentication, retry policy, and endpoint stay outside the route wrapper. That's deliberate — the exact protocol is a deployment choice, while the event contract and failure semantics are application choices. An error path should be legible at a glance.

Test it aggressively. Pass a fake `ErrorCapture` and assert that the route is a template, the operation is bounded, an email address is removed, and an unknown thrown value is normalized. Make the fake reject and verify that `runObservedRoute` still throws the original application error. Make it wait for cancellation and verify the deadline. Then test a sensitive request fixture at the integration boundary, because field-level unit tests won't catch a future adapter that serializes extra context.

Deployment should be narrow first: one representative handler, one owner, one alert, and a fixed review window. Compare event volume with the bounded metric for that handler. Check that every test event has a useful next step and that no sensitive fixture appears. I'm not sure there is a universal event-volume threshold that proves the design is ready; traffic shape, failure rate, and on-call capacity vary too much. The evidence that resolves that uncertainty is local: actual volume, label counts, alert quality, and a timed test investigation.

This is where self-serve either becomes real or collapses into “data exists somewhere.” A developer should be able to trigger a synthetic throw, find the event, follow its correlation ID, and name the owning alert without asking a platform specialist. Fast. If that exercise needs shared issue state or repeated manual grouping, the experiment has found a workflow requirement rather than an ingestion requirement.

## Know the limits before rollout

A lean API is not suitable when several teams need automatic grouping, assignment, acknowledgement, release comparison, and durable resolution state. Choose a full tracker for that job and budget for data-policy review. A full tracker is a poor fit when a tiny route set has one owner, the extra workflow will go unused, and existing logs already carry the investigation. In that case, keep the smaller path.

Structured logs plus metrics are not suitable when every exception investigation starts with a fragile handcrafted query or when responders cannot connect an alert to one request trail. Add a deliberate error event contract before adding more dashboards. Conversely, don't add another event store merely to duplicate evidence that the current pipeline already presents quickly and reliably.

The final acceptance check is concise: a synthetic exception produces a sanitized event; capture delay doesn't alter the original error; bounded metrics avoid request-specific labels; asynchronous work has a completion signal; and an engineer can go from alert to evidence without tribal knowledge. Source maps and replay can be useful in other systems, but neither is required to build this server-route loop.

Choose the smallest operating model that passes that check. The endpoint is the easy part.

## References

- Prometheus, “Instrumentation”: https://prometheus.io/docs/practices/instrumentation/
- web.dev, “Web Vitals”: https://web.dev/articles/vitals
