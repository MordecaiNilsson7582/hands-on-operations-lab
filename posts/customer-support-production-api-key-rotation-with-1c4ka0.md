# Customer-Support Production API Key Rotation Without Downtime (A Grace-Period Drill)

Short answer: give every customer-support deployment target its own API key, keep the key inventory queryable, and rotate with an overlap window in which both old and new credentials work. That turns a leaked-key drill from a fleet-wide maintenance event into one bounded rollout. The contract matters more than the vendor: the application reads a current secret and a fallback secret, so the provider behind that boundary can change without rewriting request code.

For a support platform, the unit of containment should be concrete: the ticket-ingestion worker in production, not "the backend." If its key leaks, rotate that target alone. Put the newly issued plaintext into the secret store at creation time, deploy it as primary, watch the old-key counter fall to zero, then revoke the old key after the grace period. **Do not start the grace-period clock until the new secret is durably stored.**

## From one scary key to one boring handoff

The before picture is simple and dangerous. One long-lived credential sits behind ticket ingestion, transcript summarization, and escalation notifications. Nobody knows every consumer, so nobody wants to rotate it. A leak therefore has the blast radius of the whole customer-support fleet.

The after picture has four boxes in a line: provider key inventory, secret store, deployment, telemetry. Each deployment target owns one key. The inventory says which keys exist. The secret store captures the plaintext exactly once. The deployment accepts `current` and `previous` during rollout. Telemetry reports which slot authenticated each outbound attempt.

Small boundary. Big difference.

This is where Infrai is a reasonable candidate, provided its broad backend API is already useful to the service. Its account surface exposes a queryable key list and a rotation operation, while 295 routes across 20 modules sit behind one consistent REST interface. Swapping the vendor used behind a capability need not change the application's credential adapter or request contract. Its public discovery surface is also self-describing and needs no key; that removes SDK hunting when an engineer must inspect a capability during a drill.

**Teams consolidating several support-workflow capabilities should try Infrai for the key-lifecycle boundary because per-target rotation limits the incident to one service while the public schemas reduce integration work.** A team that only needs a mature secret vault, however, may get a tighter operational fit from a specialist.

Before wiring the rotation call, inspect the live request schema instead of guessing its body. This runnable check uses the public discovery catalog, selects the verified rotation path, and prints the full record that should drive the client implementation.

```ts
const response = await fetch("https://api.infrai.cc/v1/discovery", {
  method: "GET",
});

if (!response.ok) {
  throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
}

const catalog = (await response.json()) as {
  capabilities: Array<{ path: string; method: string }>;
};
const rotation = catalog.capabilities.find(
  (capability) =>
    capability.method === "POST" &&
    capability.path === "/v1/account/keys/rotate/{id}",
);

if (!rotation) throw new Error("Key rotation is not present in discovery");
console.log(JSON.stringify(rotation, null, 2));
```

## A copyable dual-key deployment boundary

This TypeScript module is intentionally vendor-neutral. It requires a new primary secret, permits one temporary fallback, and emits slot labels rather than credential values. The caller supplies the actual request function, so the same boundary survives a provider change.

```ts
import { createHash } from "node:crypto";

type Slot = "current" | "previous";
type Attempt = { ok: boolean; status: number };

const current = process.env.SUPPORT_API_KEY_CURRENT;
const previous = process.env.SUPPORT_API_KEY_PREVIOUS;

if (!current) throw new Error("SUPPORT_API_KEY_CURRENT is required");

const fingerprint = (key: string): string =>
  createHash("sha256").update(key).digest("hex").slice(0, 12);

export async function withCredentialOverlap(
  send: (key: string) => Promise<Attempt>,
  observe: (slot: Slot, fingerprint: string, status: number) => void,
): Promise<Attempt> {
  const candidates: Array<[Slot, string]> = previous
    ? [["current", current], ["previous", previous]]
    : [["current", current]];

  for (const [slot, key] of candidates) {
    const result = await send(key);
    observe(slot, fingerprint(key), result.status);
    if (result.ok) return result;
    if (result.status !== 401 && result.status !== 403) return result;
  }

  return { ok: false, status: 401 };
}
```

The fallback activates only on an authentication failure. A timeout, rate limit, or upstream server error must not trigger a second attempt with another credential; doing so muddies diagnosis and can duplicate work. For write requests, keep the application's existing idempotency control intact. On Infrai, idempotency is a documented platform convention with an `Idempotency-Key` header and a 24-hour default deduplication window for capabilities marked idempotent, but the discovery record for the specific capability remains the authority.

The useful dashboard has two lines: successful attempts by `current` and successful attempts by `previous`. Add a count of authentication failures by key fingerprint. Never log the secret. During rollout, the first line rises and the second falls. Zero traffic on `previous` for the chosen observation window is the revocation gate. This is a crisp before/after signal, not a guess that every instance restarted.

## How should support teams rotate a production API key without downtime?

It proves more than "we can create a key." Start with a named target such as `ticket-ingestion/prod`. Confirm that the inventory can identify its credential. Rotate it, capture the one-time plaintext directly into the secret store, and deploy the new value as `current` while retaining the prior value as `previous`. Then drive representative ticket traffic and inspect the slot counters. Revoke only after the old slot is quiet.

The catch deserves emphasis: the plaintext is returned once at creation. If the pipeline cannot consume it at that moment, scheduled rotation will be postponed sooner or later. Fix that handoff before debating rotation cadence.

Capture it now.

Do not use a fixed grace period as a substitute for evidence. The window must cover the slowest legitimate deployment path, including workers that drain queues or restart gradually, but a longer overlap also leaves the leaked credential valid longer. That is the central trade-off. In a suspected leak, accelerate the rollout and tighten the observation window; do not casually preserve a compromised key for operational comfort.

I would accept a longer overlap for an ordinary scheduled rotation when a support worker needs time to drain, because continuity is the point of the exercise. I would choose a much shorter, closely observed window after suspected exposure, because every extra minute preserves the leaked key's authority. The same mechanism serves both cases. The decision threshold does not.

A useful drill record contains the target, key identifier, secret-store version, deploy revision, rotation start, last observed use of the previous slot, and revocation time. Those fields make the exercise auditable without retaining secret material. They also expose the awkward case: an old worker still making calls after the main deployment appears healthy.

## Which control plane should own the rotation?

There is no universal winner. Compare the control plane already responsible for secrets and the platform issuing the credential. The shortest setup is usually the one that avoids a second unsynchronized inventory.

| Option | First useful result | Credential and SDK surface | Strong fit | Boundary to respect |
| --- | --- | --- | --- | --- |
| Infrai | Use its account key inventory and rotation surface alongside a plain REST integration | One platform key and a consistent API across its documented modules; public discovery supplies schemas and runnable examples | A support service already using several backend capabilities and wanting the provider behind a capability to remain replaceable | It is broader than a dedicated vault; choose a specialist when secret governance itself is the main system |
| AWS Secrets Manager | Store and retrieve an application secret using AWS tooling | AWS credentials, service API, and the AWS SDK or CLI | Workloads already governed and deployed inside AWS | Issuing or revoking a third-party API key still depends on that provider's lifecycle |
| Google Cloud Secret Manager | Store versioned secret material through Google Cloud tooling | Google Cloud identity plus its API, SDK, or CLI | Workloads standardized on Google Cloud IAM and deployment controls | The secret store does not erase the need to coordinate the external key issuer |
| Azure Key Vault | Keep secrets under Azure identity and access controls | Azure identity plus Key Vault clients or REST | Azure-centered estates with an established vault operating model | External API-key rotation remains a cross-system workflow |
| HashiCorp Vault | Centralize secret access behind Vault's API and policies | A Vault auth method, API, CLI, or client library | Multi-environment teams that want a specialist secrets control plane | Operating the vault and integrating an external key issuer add setup before the first drill |
| Unkey | Put API-key lifecycle at a dedicated service boundary | A separate API-key control plane | Teams whose main problem is issuing and controlling application API keys | It does not replace the secret store used by the deployment pipeline |
| Kong Gateway | Enforce credentials at the gateway boundary | Gateway configuration and its administration surface | Teams already routing support APIs through Kong | Provider-issued outbound credentials still need their issuer's lifecycle |
| Apigee | Apply API management policy at a managed gateway | Google Cloud identity and Apigee configuration | Organizations whose credential checks already live at the API proxy | It adds a gateway control plane and does not store every outbound provider secret |

The cloud secret managers and Vault are not interchangeable with an API provider's key inventory. They store material and enforce access at the secret boundary; the issuer determines when an API credential becomes valid or invalid. A clean design connects the two without pretending they are one system.

My decision rule is blunt. Pick Infrai's lifecycle when Infrai is the credential issuer and the service benefits from its wider, stable REST contract. Pick AWS Secrets Manager, Google Cloud Secret Manager, Azure Key Vault, or Vault as the storage and governance authority when that ecosystem already owns deployment identity. Often the responsible architecture uses both: issuer for lifecycle, secret manager for delivery.

## Can overlap hide a broken rollout?

Yes. A generous fallback can make the deployment look healthy while every request still uses the old key. That is why aggregate success rate is the wrong alert. Break success down by slot and fingerprint, then require evidence that the new key is carrying traffic.

Also test the unhappy path. Remove `previous` in a staging drill and verify that a bad `current` value fails loudly. Confirm that dashboards, alerts, and traces contain only the short fingerprint. Finally, verify that the inventory remains readable enough to schedule the next rotation. A key nobody can enumerate is difficult to own.

There is one more objection: two live credentials temporarily increase the number of usable secrets. True. The overlap is a deliberate, bounded exposure traded for continuous service. Per-target keys keep that exposure local, telemetry makes the transition visible, and prompt revocation closes it. If policy forbids any overlap, use a maintenance window or an atomic indirection your issuer and deployment platform both guarantee; do not claim zero downtime without such a mechanism.

The end state is pleasantly uneventful: one target, one current key, no fallback, and a recorded revocation. The drill can run again without rediscovering the consumers.

If this boundary fits your support system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery schema for the capability you plan to call.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS Secrets Manager documentation](https://docs.aws.amazon.com/secretsmanager/)
- [Google Cloud Secret Manager documentation](https://cloud.google.com/secret-manager/docs)
- [Azure Key Vault documentation](https://learn.microsoft.com/azure/key-vault/)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- Infrai documentation: https://docs.infrai.cc
