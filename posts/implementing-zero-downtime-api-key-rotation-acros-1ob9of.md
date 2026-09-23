# Implementing Zero Downtime API Key Rotation Across 4 Media Rollouts

A media event collector can rotate API keys without downtime when it accepts the old and new keys together, moves callers gradually, and revokes the old key only after both deployment and traffic evidence say it is unused. The important clock is not the nominal rolling-deploy duration. It is the last verified use of the retiring key. That distinction protects billing attribution when delayed play, impression, and completion events arrive after an outage.

TL;DR: publish two key versions, reload them without restarting the verifier, identify usage by key ID, and start the grace timer after the last old-key request. Do not infer success from healthy pods alone. A clean rotation has zero authentication failures caused by the change and no unattributed accepted events.

## How can zero downtime API key rotation work across rolling deploys?

The fragile mental model is short: replace one secret, roll the workload, revoke the old credential. It treats deployment state as proof of traffic state. Those are different signals. A producer may still hold the earlier secret after every collector pod is healthy, especially when a queue is draining after an outage.

Use this model instead: **issue, accept both, migrate, revoke**. In words, the secret store publishes version `media-v43`; every verifier accepts `media-v42` and `media-v43`; producers move to `media-v43`; operators wait until observations show no `media-v42` traffic for the chosen grace interval; then `media-v42` is revoked.

Ship both.

That overlap creates a deliberate trade-off. A longer grace period increases the time in which two credentials are valid. A shorter one increases the chance that a delayed producer is rejected. Choose the interval from measured deployment time, secret propagation time, queue redelivery delay, clock-skew allowance, and an operational buffer. Do not invent a universal number of hours.

Give each version a stable, non-secret key ID such as `media-v43`. Store that ID beside the publisher account. Logs and metrics may contain the ID, but never the secret. Now an accepted event can be charged to the right account while the verifier distinguishes old-key traffic from new-key traffic.

That is the control point.

## Build a reloadable verifier

Keep secret retrieval behind a small interface. This makes the protocol independent of how Kubernetes receives secret material and of any particular secret store. The example expects a loader to return the active set. It hashes presented values before constant-time comparison and never emits them.

```ts
import { createHash, timingSafeEqual } from "node:crypto";

type StoredKey = {
  keyId: string;
  accountId: string;
  sha256: string;
  state: "current" | "retiring";
};
type KeySnapshot = { revision: string; keys: StoredKey[] };
type Verification =
  | { ok: true; keyId: string; accountId: string; state: StoredKey["state"] }
  | { ok: false };

const digest = (value: string): Buffer =>
  createHash("sha256").update(value, "utf8").digest();

export class ReloadableKeyVerifier {
  private snapshot: KeySnapshot = { revision: "unloaded", keys: [] };

  constructor(private readonly load: () => Promise<KeySnapshot>) {}

  async reload(): Promise<void> {
    const next = await this.load();
    if (next.keys.length === 0) throw new Error("refusing an empty key set");
    this.snapshot = next;
  }

  verify(presented: string): Verification {
    const candidate = digest(presented);
    for (const key of this.snapshot.keys) {
      const expected = Buffer.from(key.sha256, "hex");
      if (expected.length === candidate.length && timingSafeEqual(expected, candidate)) {
        return { ok: true, keyId: key.keyId, accountId: key.accountId, state: key.state };
      }
    }
    return { ok: false };
  }

  revision(): string {
    return this.snapshot.revision;
  }
}
```

The empty-set refusal matters. A transient read must not silently turn every request into an authentication failure. Keep the last valid snapshot, report the reload failure, and make readiness reflect whether the process has loaded a valid revision. Fail closed for unknown presented keys.

Attach attribution before accepting an event. The handler below uses generic telemetry contracts, so it does not turn the design into a product setup guide.

```ts
type Counter = { add(value: number, labels: Record<string, string>): void };
type MediaEvent = { eventId: string; kind: "play" | "impression" | "complete" };

type Dependencies = {
  verifier: ReloadableKeyVerifier;
  accepted: Counter;
  rejected: Counter;
  persist: (event: MediaEvent, accountId: string) => Promise<void>;
};

export async function ingest(
  authorization: string | undefined,
  event: MediaEvent,
  deps: Dependencies
): Promise<number> {
  const prefix = "Bearer ";
  if (!authorization?.startsWith(prefix)) {
    deps.rejected.add(1, { reason: "missing_bearer" });
    return 401;
  }

  const result = deps.verifier.verify(authorization.slice(prefix.length));
  if (!result.ok) {
    deps.rejected.add(1, { reason: "unknown_key" });
    return 401;
  }

  await deps.persist(event, result.accountId);
  deps.accepted.add(1, { key_id: result.keyId, key_state: result.state });
  return 202;
}
```

Notice what is absent: the presented credential. Also avoid event or account identifiers in metric labels because their value set grows without a useful bound. They belong in protected structured logs or traces. The metric needs key version and outcome; the durable event record needs account attribution.

## How do you know the grace window is over?

Do not ask whether the rollout finished. Ask whether the retiring key is still authenticating work. The revocation gate combines several observations:

1. Every verifier reports the intended secret revision and accepts both versions.
2. Every known producer reports the new version ID without exposing the key.
3. Authentication failures remain at their normal level during migration.
4. Accepted requests using the retiring ID reach zero for the full grace interval.
5. Queues and retry paths have drained, or their maximum delay is included in that interval.

One metric tells a crisp story: accepted requests grouped by `key_id`. Before migration, `media-v42` carries the traffic. During migration, the two series cross. After migration, `media-v42` becomes zero while `media-v43` continues. That is the before-and-after diagram in numbers. Alert if the old series reappears, unknown-key rejections rise, or verifier revisions diverge across replicas. For a concrete review, inspect consecutive 15-minute buckets rather than one instant: a single empty bucket may mean the producer is quiet, while a run that spans the measured replay delay is meaningful. The duration is an example of a viewing interval, not a universal grace period; the actual revoke threshold still comes from the system's measured delays.

Watch the crossing.

The billing check is separate. Compare accepted event counts with persisted, account-attributed event counts over the same interval. A security rotation that drops attribution is still a failed change for this system. Duplicate delivery should use the existing event ID and idempotency policy; rotation must not create a second billing identity for one account.

Test transitions, not only steady state. Send an event with the old key before overlap, both keys during overlap, and the old key after revocation. Repeat while reloading and while a simulated backlog drains. Assert the response, recorded account ID, key-version metric, and absence of secrets in captured logs.

## What if a producer returns after revocation?

It should fail authentication and trigger a specific operational signal. Quietly restoring the retired key would erase the value of revocation. Preserve enough non-secret evidence to identify the producer, then use the credential-recovery process to issue a fresh version.

This objection often exposes a missing inventory. If nobody can say which encoders, ad services, partner feeds, or replay workers own a credential, no grace duration can make rotation dependable. Record an owner, account binding, creation time, state, and last-used time for each key ID. OWASP's secrets guidance covers lifecycle handling including creation, rotation, revocation, and expiration; the inventory makes those actions operable.

Emergency compromise response has a harder edge. A scheduled rollover can afford overlap. A credential believed to be exposed may require immediate revocation, accepting availability impact to stop unauthorized use. Write two runbooks because their risk decisions are opposite.

## Ship the protocol, then shorten uncertainty

Start with the four phases and a conservative, measured grace interval. Run a staged rotation with non-production credentials, then rotate one limited production account while watching version usage, rejection reasons, replica revision, queue delay, and attribution completeness. Expand only after each gate is observable.

Once the process is repeatable, reduce uncertainty instead of chasing the smallest interval. Faster propagation, bounded retries, a complete producer inventory, and reliable last-used telemetry can justify a shorter window. The revoke action should remain explicit and auditable.

Zero downtime is the visible result. **Correct attribution is the acceptance criterion.**

## Further reading

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
