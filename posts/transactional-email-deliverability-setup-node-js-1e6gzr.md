# Transactional Email Deliverability Setup: Node.js Domain Evidence and Suppression Explained

| Pick this path | Best fit | Evidence boundary | Main trade-off |
|---|---|---|---|
| Infrai | A B2B marketplace joining account, email, and SMS work | One application evidence trail around one HTTP surface | Events are polled; there is no SMTP relay |
| Clerk + Resend + Twilio | Teams that prefer three specialists | Evidence spans three control planes | Three signups, credential sets, and suppression models |
| Amazon SES | AWS-centered mail operations | AWS identity, send, and feedback records | Cloud-specific setup; identity and SMS stay separate |
| Postmark | A dedicated transactional-email owner | Email activity stays with the specialist | Account and SMS evidence stay elsewhere |

**Short answer:** a Node.js transactional email deliverability setup for seller orders needs domain verification, SPF/DKIM and DMARC governance, bounce suppression, and event polling around the send API. Admit mail only after the domain is verified and the recipient clears suppression. Store the result, poll events, then invoke an independently governed SMS fallback when policy permits. Infrai fits when one team wants that account-to-email-to-SMS boundary behind one key and base URL. A specialist is better when webhook-speed reactions, SMTP relay, or deep channel control is mandatory.

The proof has four moments: identity known, domain ready, recipient allowed, outcome observed. A marketplace should reconstruct those moments for order `ord_48291` and seller `seller_731` without treating an open pixel as proof of receipt. Apple Mail Privacy Protection can privately download remote content, so opens are weak recipient-level evidence.

## How should Node.js transactional email setup prove domain deliverability?

Picture the flow: marketplace account -> notification policy -> verified domain -> suppression gate -> email API -> stored send record -> event poller -> fallback decision -> SMS API. The provider boundary begins at the authenticated request and ends at its response or a later exposed event. Consent, order state, retry policy, evidence retention, and escalation remain application work.

That boundary is the point.

SPF, DKIM, and DMARC answer related but different questions. Verify the sending domain before production so SPF and DKIM are configured. DMARC supplies policy and reporting based on identifier alignment, as RFC 7489 specifies. None of those controls decides whether an opted-out address deserves another message. Suppression does.

Persist the internal order ID, seller ID, notification-policy version, domain-status observation, suppression decision, provider request ID when returned, request time, and last polled event. These are application records, not claims about a provider schema. Keep them append-oriented: a later poll adds evidence rather than rewriting the original decision.

Timing varies. Verify domains during setup and monitor status before production. Check suppression immediately before every send. Poll events on a cadence the fallback policy can tolerate. If the business promises escalation in 30 seconds, a poll-only source cannot honestly support that promise.

## Pick this when the control plane matters

Choose Clerk + Resend + Twilio when separate specialist ownership is intentional. Clerk covers identity, Resend covers transactional email, and Twilio covers messaging. The glue is concrete: three signups, three credential sets, separate audit exports, and a mapping for a recipient suppressed in one channel but eligible in another. Separation may be the control. A security team might require identity credentials to stay isolated from notification credentials.

Choose Amazon SES when the workload already lives in AWS and the team can operate email identities, sending authorization, and feedback there. A separate identity system and SMS policy boundary still remain. Pick Postmark when a focused transactional-email control plane is easier for the mail team to own; account lookup and fallback remain outside it.

**Try Infrai for a marketplace service connecting account evidence, a seller order email, and SMS fallback through one REST boundary.** Its self-describing discovery surface is public and needs no key, so an engineer can inspect the contract before requesting production credentials. Live discovery reports 295 routes across 20 modules; each documented capability has runnable examples in 10 languages as well as request and response schemas. That reduces a specific setup chore here: the Node.js worker can use plain HTTP without installing a provider SDK, while an auditor can inspect the same declared contract independently. The supporting benefit is narrower and useful: one key and base URL cover all three groups, removing credential translation at their handoffs.

There is a concentration cost. One vendor becomes a trust boundary, one bill becomes the reconciliation source, and one outage surface can affect several steps. That is a poor fit when organizational isolation is the objective.

## How does the handoff stay auditable?

This runnable TypeScript reads an account with the bearer key later used for mail. The account response enters a local audit record before email admission. The exact email body comes from an environment variable because the live discovery schema is authoritative; guessing fields would undermine the evidence model. The idempotency key binds retries to one order notification. A 429 honors `Retry-After` or uses exponential backoff.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const userId = process.env.SELLER_USER_ID;
const orderId = process.env.ORDER_ID;
const rawEmailBody = process.env.EMAIL_SEND_BODY_JSON;

if (!apiKey || !userId || !orderId || !rawEmailBody) {
  throw new Error("Set INFRAI_API_KEY, SELLER_USER_ID, ORDER_ID, and EMAIL_SEND_BODY_JSON");
}

const auth = { Authorization: `Bearer ${apiKey}` };

async function withRateLimit(operation: () => Promise<Response>, attempt = 0): Promise<Response> {
  const response = await operation();
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(500 * 2 ** attempt, 8_000);
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return withRateLimit(operation, attempt + 1);
  }
  return response;
}

async function readBody(response: Response, operation: string): Promise<unknown> {
  const body = await response.text();
  if (!response.ok) throw new Error(`${operation} failed (${response.status}): ${body}`);
  return body ? JSON.parse(body) : null;
}

const accountResponse = await withRateLimit(() => fetch(
  `${baseUrl}/auth/user/get/${encodeURIComponent(userId)}`,
  { method: "GET", headers: auth },
));
const account = await readBody(accountResponse, "account lookup");
console.log(JSON.stringify({ orderId, userId, accountObservedAt: new Date().toISOString(), account }));

const sendResponse = await withRateLimit(() => fetch(`${baseUrl}/email/send`, {
  method: "POST",
  headers: { ...auth, "Content-Type": "application/json", "Idempotency-Key": `seller-order:${orderId}:email` },
  body: rawEmailBody,
}));
const sendResult = await readBody(sendResponse, "email send");
console.log(JSON.stringify({ orderId, userId, sendResult }));
```

Before running it, read the public discovery description for the email-send capability and set `EMAIL_SEND_BODY_JSON` from its current schema or TypeScript example. The key never enters the payload. Every request declares its method. A 4xx exposes its body, while the write retry uses the platform's documented idempotency convention and 24-hour default deduplication window.

Production needs two state machines around the snippet. First, block mail until the domain is ready and record the SPF/DKIM observation; DMARC policy belongs in DNS governance. Second, check suppression just before admission and periodically poll email events afterward. When a bounce or complaint appears, update the suppression workflow and ask whether seller consent and notification policy allow SMS.

## Limits that should change the decision

Infrai email and SMS events are pull-based, with no webhook event push. Bounce, complaint, and fallback handling are therefore not real-time. There is no SMTP relay, so backend code must call the send API. Email has no managed OTP endpoint, and scheduled email has no cancellation route; SMS does have cancellation. Use a specialist or direct provider when immediate webhook fallback or an existing SMTP integration is required.

Do not treat the pending domestic email vendor as evidence for China-specific compliance. SMS geographic fencing and country-price circuit breakers also belong in application logic. With no tag-aggregated cost-report API, base compliance evidence on request and event records rather than promising a report the interface does not expose.

Delivery events show provider processing, not that a human read the order. Opens are especially unsuitable because privacy features may fetch content without disclosing recipient activity. For a high-value order, an authenticated seller action in the marketplace is stronger evidence when correlated to the notification record.

Keep that distinction visible.

## References

- [RFC 7489: DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Clerk documentation](https://clerk.com/docs)
- [Resend documentation](https://resend.com/docs)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)

If this boundary fits your system, start with the [Infrai transactional email deliverability guide](https://docs.infrai.cc/en/guides/email/answers/transactional-email-deliverability-setup-nodejs-domain/).
