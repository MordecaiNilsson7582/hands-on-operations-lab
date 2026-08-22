# Runtime Flags for Node.js Incident Containment and Rollback

**Use a remotely stored feature flag as the production kill switch, and make rollback a monotonic state change rather than a deploy.** An in-memory boolean is simple, but it cannot reliably coordinate multiple Node.js processes during an incident. The practical default is a small authenticated control API, a shared source of truth, local last-known-good state, and an audit event for every attempted change.

| Control pattern | Pick it when | Incident trade-off |
| --- | --- | --- |
| Process-local boolean | One disposable process owns the feature | Fast to build, but each process can disagree and restarts erase the decision |
| Polled shared state | A short propagation delay is acceptable | Easy to reason about; the polling interval defines how long mixed behavior can last |
| Pushed shared state | Coordinated, low-delay changes matter | Faster distribution, with more connection and delivery machinery to operate |

This field guide uses the polled design. It has a boring failure model, which is exactly what an on-call engineer needs at 02:17. The switch disables one risky path while the service stays up; it does not pretend to reverse database writes or replace a deployment rollback.

## How should a Node.js production API use a feature flag kill switch during an incident?

Treat the flag as a tiny control-plane record: `enabled`, a strictly increasing `version`, a reason, and a timestamp. The data plane reads a cached copy before entering the guarded path. The control API changes the shared record. Each application process polls that record and accepts it only when the version is newer than the version already cached.

The diagram in words is short: operator request -> authenticated control API -> shared record -> pollers in every Node.js process -> guarded request path. A second arrow goes from the control API to an append-only audit stream. A third goes from each process to metrics and logs, so an operator can see adoption by version rather than guessing that the button worked.

Versioning matters. Suppose process A observes version 42 with the feature disabled, then a delayed read returns version 41 with it enabled. A plain assignment reactivates the risky path. A monotonic comparison rejects the stale value. No clock comparison is required, and clock skew cannot decide which state wins.

Keep the request path cheap. It should read local memory, not call the shared store for every customer request. Also keep the safe branch real: exercise it in normal tests and deployments. A kill switch whose disabled branch has quietly stopped compiling is an incident amplifier.

## Pick the control pattern that matches the blast radius

A process-local switch is suitable for a command-line worker or a single short-lived task where one owner can stop the entire process. It is not suitable when a load balancer can route traffic to several replicas. In that topology, changing one heap leaves the fleet split.

Polling is the useful middle. The catch is that propagation is bounded by the polling schedule, so it is a poor fit when even a brief mixed state would violate a safety invariant. Pick a pushed design for that requirement, and define how disconnected subscribers recover the latest version after reconnecting. Delivery alone is not enough; convergence is the goal.

A deployment-time environment variable belongs to release configuration, not live incident control. Stick with it when every change should pass through the deployment system and the rollout delay is acceptable. Don't label it a kill switch if changing it requires rebuilding or restarting the fleet.

## Implement the rollback control and the guarded path

The example below keeps storage behind an interface. The route is intentionally internal and pseudonymous. Its version precondition prevents two operators from silently overwriting each other, while the application-side poller prevents an older read from reversing a newer decision.

```ts
type FlagState = {
  enabled: boolean;
  version: number;
  reason: string;
  changedAt: string;
};

type ChangeRequest = {
  enabled: boolean;
  expectedVersion: number;
  reason: string;
};

interface FlagStore {
  read(key: string): Promise<FlagState>;
  compareAndSet(
    key: string,
    expectedVersion: number,
    next: FlagState,
  ): Promise<boolean>;
}

interface AuditSink {
  write(event: {
    action: "feature_flag_changed" | "feature_flag_change_rejected";
    key: string;
    version: number;
    enabled: boolean;
    reason: string;
  }): Promise<void>;
}

const key = "checkout-v2";
let local: FlagState = {
  enabled: true,
  version: 0,
  reason: "initial state",
  changedAt: new Date(0).toISOString(),
};

async function refreshFlag(store: FlagStore): Promise<void> {
  const observed = await store.read(key);
  if (observed.version > local.version) local = observed;
}

function runCheckoutV2(input: unknown): unknown {
  return { path: "v2", input };
}

function runStableCheckout(input: unknown): unknown {
  return { path: "stable", input };
}

export function checkout(input: unknown): unknown {
  return local.enabled ? runCheckoutV2(input) : runStableCheckout(input);
}

export async function changeFlag(
  store: FlagStore,
  audit: AuditSink,
  body: ChangeRequest,
): Promise<{ status: 200 | 409; state: FlagState }> {
  const next: FlagState = {
    enabled: body.enabled,
    version: body.expectedVersion + 1,
    reason: body.reason,
    changedAt: new Date().toISOString(),
  };

  const changed = await store.compareAndSet(key, body.expectedVersion, next);
  const state = changed ? next : await store.read(key);

  await audit.write({
    action: changed
      ? "feature_flag_changed"
      : "feature_flag_change_rejected",
    key,
    version: state.version,
    enabled: state.enabled,
    reason: changed ? body.reason : "version conflict",
  });

  return { status: changed ? 200 : 409, state };
}
```

Wire `changeFlag` to an authenticated `PUT /internal/controls/checkout-v2` handler and reject malformed bodies before calling it. Authentication and authorization are separate checks: the caller must be known, and that caller must be allowed to operate this specific control. Keep the endpoint off the public application surface. Rate-limit it too. A kill switch is powerful by design.

The `409` is useful during a production incident. Walk through the race before relying on it: the shared record is enabled at version 41, and two authorized operators both read that state. One sends a disable request with `expectedVersion: 41` and reason `contain checkout errors`; the compare-and-set succeeds, producing disabled version 42. The other operator, still holding the old view, sends an enable request that also expects version 41. That request receives `409` plus the winning state instead of replacing it. Meanwhile, each process may briefly remain on version 41 until its next successful poll; as soon as version 42 arrives, its local comparison accepts the higher version and routes new work to the stable path. If a delayed version 41 response arrives afterward, the comparison ignores it. The operator can now check the fleet-version gauge, request results, and the accepted audit event before making any new change. Re-enabling is a separate decision with `expectedVersion: 42`, a fresh reason, and a new version 43. It should never be a blind retry of the rejected request. This is the crisp before and after the control needs to expose: version 42 contains the risky path; version 43 reopens it only after validation.

## Observe the switch without leaking incident data

Emit one counter for change attempts, labeled by result, plus a gauge for the locally applied version and state. Alert on fleet disagreement when processes report different current versions beyond the intended propagation window. Logs should answer who attempted the change, which flag changed, what version won, and why. They should not copy full request bodies.

Be strict here.

OWASP recommends excluding or masking data such as access tokens, passwords, session identifiers, and sensitive personal data from application logs. Sanitize the operator-supplied reason before logging it, and bound its length. The audit event in the example records the decision, not credentials or customer payloads. Protect logs from tampering and unauthorized access as well; an audit trail is evidence only if its integrity survives the incident.

Test the signals before production. In an integration test, start two data-plane instances, disable the flag at version 42, refresh both caches, and assert that both use the stable path. Then feed one instance version 41 and verify that it remains disabled. Add a conflict test in which two updates expect version 42 and only one succeeds. Finally, verify that the audit sink receives both accepted and rejected attempts without a token or input payload.

Retention has an operational cost, so separate low-volume control audit events from high-volume request diagnostics. Some commercial log plans distinguish ingestion from indexed retention; the linked pricing page is one concrete example of why teams should estimate both. Your mileage may vary because the useful retention period depends on incident review and compliance needs, not a universal number.

## Limits and operating rules

This pattern is not a transaction rollback. Requests already in flight may complete, queued work may still run, and side effects already committed need their own compensating procedure. It is also not suitable when both branches cannot safely run against the same data model during propagation; use a coordinated release or maintenance boundary for that case.

Define ownership before the alert fires: who can disable the path, who may re-enable it, and what evidence is required. Re-enablement deserves the same audit and version checks as shutdown. Fast off. Careful on.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- https://www.datadoghq.com/pricing/
