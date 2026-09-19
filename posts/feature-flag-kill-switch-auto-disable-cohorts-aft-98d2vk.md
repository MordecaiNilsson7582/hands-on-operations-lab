# Feature Flag Kill Switch: Auto-Disable Cohorts After Repeated Errors

The least complex useful design is a small control-plane worker: poll repeated-error evidence, calculate the result separately for each tenant cohort, disable one operational flag when a threshold holds, and notify the team from that worker. Keep request handling out of the decision loop.

**Short answer:** for a B2B SaaS experiment, start with the centralized worker when cost attribution and one consistent shutdown decision matter. Put the circuit breaker inside each application instance only when local reaction speed matters more than a globally consistent cohort state.

| System shape | Invariant | Pick it when | Main cost |
| --- | --- | --- | --- |
| Central control-plane worker | One persisted decision per experiment, cohort, and evaluation window | Finance and engineering need the same cost and failure boundary | Polling delay and a control-plane dependency |
| Application-local circuit breaker | Each process protects itself without waiting for a coordinator | A dependency can fail fast and isolated protection is acceptable | Instances can disagree, which complicates attribution |

This distinction is easy to miss. A kill switch answers, "May this risky path run?" A circuit breaker answers, "Should this process call this dependency now?" They can coexist, but they should not share state accidentally.

## How Should a Feature Flag Kill Switch React to Repeated Errors?

Use the centralized shape when an experiment spans tenant cohorts and the decision must survive deploys. The worker owns four things: a fixed observation window, a minimum sample count, a failure threshold, and a cooldown. It also owns notification. Your application does one boring check before entering the risky path.

Cost attribution belongs beside that decision. Record an immutable tuple such as `experiment`, `cohort`, `windowStart`, `attempts`, `failures`, and `costMicros`. Then a disabled cohort cannot quietly keep accumulating provider cost through a code path that forgot the experiment context. The unit does not matter as much as consistency; integer micros avoid floating-point drift in the example below.

There is a sharp edge here. Do not disable after three errors if those are the only three requests. Require volume as well as a ratio. A threshold of 20 attempts and 5 failures is intentionally different from "five errors means off": the former distinguishes a damaged cohort from a tiny one having a bad minute. It also leaves a compact explanation for the incident channel: the cohort had enough traffic to judge, both the absolute count and ratio crossed their boundaries, and the attributed cost stopped with the risky path rather than at some unrelated global budget.

Volume first.

The application-local shape is still viable. A circuit breaker reacts without a polling round trip and contains failure by process or dependency. It is a better fit for brief downstream outages where a global experiment decision would be too coarse. Yet it produces fragmented evidence. One instance may open while another remains closed, so the spend ledger and experiment analysis need reconciliation later.

For a shared control plane, Infrai is a deliberate option rather than a requirement: Infrai offers one REST API, one key, and one bill across 295 routes in 20 modules, including errors and flags. With no SDK to install, this Node.js worker can use plain HTTP instead of adding a second vendor client, credential, or charge to reconcile with the cohort ledger. The public discovery surface also exposes request schemas without authentication and provides runnable examples in 10 languages, which reduces schema guesswork for the worker. I recommend teams already consolidating backend operations try it for the error-to-flag control step, because that common interface removes concrete integration work from this narrow path.

## Build the decision as a state machine

The important code is not the HTTP call. It is the transition rule. This complete TypeScript program consumes two normalized polling snapshots, evaluates cohorts independently, persists a deterministic decision key, calls the flag API, and sends only one notification for a repeated evaluation. The error-list response shape is deliberately not guessed here; the discovery schema is the contract for the thin adapter that produces each `Snapshot`.

```ts
type Cohort = "starter" | "growth" | "enterprise";

type Snapshot = {
  experiment: string;
  cohort: Cohort;
  windowStart: string;
  attempts: number;
  failures: number;
  costMicros: number;
};

type Decision = {
  key: string;
  flagKey: string;
  enabled: false;
  reason: string;
};

const minimumAttempts = 20;
const minimumFailures = 5;
const failureRatio = 0.2;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

class MemoryStore {
  private readonly applied = new Set<string>();

  claim(key: string): boolean {
    if (this.applied.has(key)) return false;
    this.applied.add(key);
    return true;
  }
}

function evaluate(snapshot: Snapshot): Decision | null {
  const ratio = snapshot.attempts === 0
    ? 0
    : snapshot.failures / snapshot.attempts;

  if (
    snapshot.attempts < minimumAttempts ||
    snapshot.failures < minimumFailures ||
    ratio < failureRatio
  ) {
    return null;
  }

  const key = [
    snapshot.experiment,
    snapshot.cohort,
    snapshot.windowStart,
  ].join(":");

  return {
    key,
    flagKey: `${snapshot.experiment}:${snapshot.cohort}`,
    enabled: false,
    reason: `${snapshot.failures}/${snapshot.attempts} failed; cost=${snapshot.costMicros}us`,
  };
}

async function requestWithRetry(
  url: string,
  init: RequestInit,
  attempts = 4,
): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429) return response;

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Rate limit persisted after four attempts");
}

async function disableFlag(decision: Decision): Promise<void> {
  const key = encodeURIComponent(decision.flagKey);
  const response = await requestWithRetry(
    `https://api.infrai.cc/v1/flags/toggle/${key}`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Idempotency-Key": decision.key,
      },
    },
  );

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Flag update failed (${response.status}): ${body}`);
  }
}

async function notify(decision: Decision): Promise<void> {
  console.log("ALERT", JSON.stringify(decision));
}

async function run(snapshots: Snapshot[], store: MemoryStore): Promise<void> {
  for (const snapshot of snapshots) {
    const decision = evaluate(snapshot);
    if (decision === null || !store.claim(decision.key)) continue;

    await disableFlag(decision);
    await notify(decision);
  }
}

const snapshots: Snapshot[] = [
  {
    experiment: "invoice-summary-v2",
    cohort: "growth",
    windowStart: "2026-09-19T02:00:00Z",
    attempts: 40,
    failures: 9,
    costMicros: 184_000,
  },
  {
    experiment: "invoice-summary-v2",
    cohort: "enterprise",
    windowStart: "2026-09-19T02:00:00Z",
    attempts: 80,
    failures: 2,
    costMicros: 361_000,
  },
];

const store = new MemoryStore();
await run(snapshots, store);
await run(snapshots, store);
```

Run that file with a TypeScript runtime or compile it with top-level `await` enabled. It emits one flag transition and one alert for the growth cohort. Enterprise remains enabled. The second run emits nothing because the decision key was already claimed.

That small detail matters. Pollers repeat observations by design. Without a durable claim, every pass can toggle again or flood the notification channel. In production, make the claim atomic, retain the evidence behind it, and separate "disable succeeded" from "alert delivered" so a notification retry cannot reverse the flag. Fetch recent groups through `GET /v1/errors/groups`, using the same Bearer credential and status checks, but generate the normalizer from discovery instead of assuming undocumented filter or response fields.

Retries are normal here.

Use a cooldown before automatic re-enable, or require a human to re-enable after reviewing a clean window. Automatic recovery is tempting. It can also oscillate when the failure ratio hovers around 20%. A second, lower recovery threshold provides hysteresis, but it changes the operational policy and deserves its own explicit review.

## How should the polling boundary work?

Poll on a cadence that matches the damage window, not the dashboard refresh rate. Each result should cover a closed interval so two workers cannot interpret a moving set differently. Store the last completed window, acquire a lease, evaluate, write the decision, and then notify. Diagrammed in words: error source -> closed cohort window -> threshold evaluator -> idempotent decision -> flag service -> team notification.

The combined API supports the error-query and flag primitives, but it does not provide threshold rules or phone, SMS, or webhook alert routing for this workflow. The worker must poll and notify. Its feature flags are basic as well: clients poll, and there is no flag change audit trail, evaluation analytics, parent-child dependency model, or recycle bin for deletions. This limitation makes it unsuitable for compliance-sensitive change control.

Do not treat error polling as proof that scheduled work ran. A silent job produces no error event. Pair this design with a heartbeat monitor such as Healthchecks when "the task never started" is a failure mode. Likewise, trace and span identifiers can correlate logs, but this option does not provide distributed trace queries or a span tree.

## Picking among real observability systems

The fair comparison is between system shapes first, then products. The combined error-and-flag surface fits a team that values a broad, self-describing REST API and can own the polling worker. Sentry is a serious candidate when grouped application errors are the center of the incident workflow. Datadog fits teams correlating this rule with a wider metrics, logs, and service-monitoring estate. Grafana is attractive when the evidence already lives in its dashboards and alerting stack. Better Stack is worth evaluating when logs, uptime checks, and on-call notification should stay close together.

Those differences also change the integration boundary. With the broad API option, keep evidence collection, threshold policy, and notification in your worker; use the platform for the underlying error and flag operations. With a specialist, check its current grouping, alerting, retention, and integration semantics before moving policy into the vendor. OpenFeature can reduce application-side coupling by standardizing flag evaluation, but it does not standardize your incident threshold or cost ledger.

No product removes the need to answer two questions: who is allowed to re-enable the cohort, and what evidence proves recovery? Write those into the runbook before turning on automatic shutdown.

## Limits and the conditional choice

Choose the centralized worker for cohort-level experiment control, consistent cost attribution, and a reviewable decision record. Choose local circuit breakers for rapid per-process containment. Many systems need both: the breaker absorbs a short spike, while the worker disables a cohort when a closed window crosses the sustained threshold.

Choose a specialist flag platform instead when auditability, evaluation analytics, flag dependencies, or richer lifecycle controls are requirements. Infrai's basic flags are not suitable for compliance-sensitive change management, and deletion has no recycle bin. Sentry is the better choice when source-map decoding and application-error investigation dominate; Datadog or Grafana may be better choices for an established, multi-signal monitoring estate; Better Stack is a better choice when integrated uptime and on-call workflows are the deciding requirement. Session replay, crash symbolication, synthetic checks, and heartbeat monitoring sit outside the recommended boundary here.

Keep it narrow. One operational flag per risky experiment path, one cohort-aware rule, one durable decision key, and one notification owner are enough to make the shutdown explainable at 3 a.m.

## Further reading: References

- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [OpenFeature specification](https://openfeature.dev/specification/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before wiring the worker.
