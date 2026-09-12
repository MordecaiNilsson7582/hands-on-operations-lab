# Node.js 20 Staging DNS: DMARC Delegation for Production Write Containment

## Short answer

Short answer: use a separately delegated staging zone when the admin console can write records; use a production subdomain only when the same team owns both environments and a narrow write boundary is enforced in code. The deciding evidence is deliverability telemetry, not which layout looks tidier.

In a B2B SaaS product, an internal console often edits customer-facing DNS. A typo in a staging form should never be able to replace the production MX or DMARC record. Think of the design as two doors: delegation decides which DNS authority a credential can reach, and an application policy decides which names that credential may change. The smaller door wins when the two disagree.

| Layout | Pick this when | Main blast-radius property |
| --- | --- | --- |
| Separate delegated zone | Staging has its own account, credential, and mail identity | A staging write cannot address production names |
| Production subdomain | You need one parent zone and can enforce an explicit name allowlist | A policy bug can still reach the parent zone |
| Shared zone with labels only | You are migrating legacy tooling and have a short decommission date | Weak isolation; treat it as temporary |

## Why DMARC evidence changes the boundary

DNS isolation is an email safety problem. DMARC policy is published as a TXT record at `_dmarc.<domain>`, and reports describe authentication results for messages that claim the domain. RFC 7489 defines the policy and aggregate/forensic reporting model; it does not give a staging environment permission to edit a production zone. That permission is yours to design.

For a test tenant, use a domain or subdomain whose From address is never used for real customer mail. Send controlled messages, collect aggregate reports, and compare SPF alignment, DKIM alignment, and the observed source domains. A green application test is not proof of deliverability. The report is the evidence.

I initially treated `staging.example.com` as a harmless label. Then a console role that could update that label also held the parent-zone token. The DNS name looked scoped; the credential was not. That is the failure mode to design out.

Keep the report stream separate too. A staging `rua` destination should land in a mailbox or collector that production operators can ignore without losing an alert. Your mileage may vary on report volume, but the boundary should be testable: a staging change produces a staging report, and a production record remains unchanged.

## How should staging DNS separate a zone or subdomain from production write boundaries?

Start with the authority graph. The root or parent zone delegates `staging.example.com` to nameservers that are administered by the staging account. The console's staging credential is then issued only for that delegated authority. In the subdomain model, the parent zone remains the authority and the console must enforce a write allowlist such as `*.staging.example.com`; the provider credential must offer an equivalent scope.

The implementation below makes the application boundary visible. It rejects apex names, NS changes, and names outside the staging suffix before calling a provider adapter. The adapter is intentionally generic: its job is to translate this validated change into the provider's API, while the policy stays in your codebase and tests.

```ts
type RecordKind = "A" | "AAAA" | "CNAME" | "MX" | "TXT";

type DnsChange = {
  name: string;
  type: RecordKind;
  value: string;
  ttl: number;
};

const STAGING_SUFFIX = ".staging.example.com";

function validateStagingChange(change: DnsChange): void {
  const name = change.name.toLowerCase().replace(/\.$/, "");
  if (!name.endsWith(STAGING_SUFFIX) || name === STAGING_SUFFIX.slice(1)) {
    throw new Error("DNS name is outside the staging boundary");
  }
  if (change.type === "MX" && name === "staging.example.com") {
    throw new Error("Staging apex mail routing is not allowed");
  }
  if (change.ttl < 60 || change.ttl > 86400) {
    throw new Error("TTL is outside the approved range");
  }
}

export async function apply(change: DnsChange, provider: {
  upsert(change: DnsChange): Promise<void>;
}): Promise<void> {
  validateStagingChange(change);
  await provider.upsert({ ...change, name: change.name.toLowerCase() });
}
```

Log the requested name, record type, actor, change ticket, and provider response ID. Emit a counter for rejected writes and a gauge for the age of the last successful staging update. Alert on a rejected write spike and on any production-zone audit event carrying a staging identity. Those signals turn an access-control assumption into an observable contract.

Create the delegated zone first, then publish its NS set at the parent. Verify delegation from outside the CI network with a resolver that is not your local cache. Next, create a canary TXT record and confirm that only the staging identity can change it. Do this before wiring the admin console's UI.

For each release, run three checks: an allowed staging change succeeds, a production name is rejected before provider access, and a DMARC TXT lookup returns the expected policy for both domains. Keep these as integration tests against a disposable zone. A dry-run flag is useful, but it is not isolation; it only exercises code paths.

The negative test should be boring. A production-name request should be rejected with a 403 before the provider adapter runs, and the audit log should carry the same correlation ID as the test run.

The observability loop should answer four questions quickly: what changed, who changed it, which authority accepted it, and did mail authentication remain aligned afterward? Store DNS audit events with a correlation ID that also appears in the mail test and DMARC report records. That gives the on-call engineer one trace instead of three dashboards.

Make that trace useful during a handoff. The change event should retain the normalized DNS name, the pre-change value, the requested value, the actor's role, the ticket reference, and the authority that accepted the update. The mail test record should point back to the same ID, while the DMARC report parser stores the reporting interval and alignment result. With those links, an engineer can start at a failed delivery, find the report row, jump to the message test, and see the exact write request without guessing which environment was active. Keep retention long enough to cover your reporting cadence and release window; the precise period depends on your compliance needs. What matters is that deletion and promotion events remain searchable after a deploy, not buried in a transient application log.

Ship the boundary first.

## Limits and when to choose the other layout

The catch is operational overhead. A separate zone needs delegated nameservers, separate credentials, and a process for promoting records; teams that cannot operate those pieces may create accidental outages during a handoff. Choose the production-subdomain model when delegation is unavailable, but require provider-side record scoping, a deny-by-default policy, and a second approval for MX or `_dmarc` changes.

A subdomain also does not isolate every mistake. Parent-zone administrators can still delete the delegation, and a broad token can still edit unrelated names. Conversely, a separate zone is a poor fit when staging must exercise the exact production domain, because DMARC alignment and reputation signals are intentionally different. In that case, use a production-like test domain and document the gap rather than weakening the production credential.

I am not sure every DNS provider exposes the same granularity for record-level permissions; verify that capability in the provider's current documentation and in an audit-log test. The decision is complete only when the write boundary and the deliverability evidence agree.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://www.rfc-editor.org/rfc/rfc1034
- https://www.rfc-editor.org/rfc/rfc1035
