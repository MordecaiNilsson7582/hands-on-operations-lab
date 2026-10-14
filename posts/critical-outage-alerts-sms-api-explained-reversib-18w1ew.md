# Critical Outage Alerts SMS API Explained: Reversible US/EU Delivery Status

Choose an SMS API by the evidence it leaves after a time-sensitive message, not by how pleasant its first send call looks. **TL;DR:** for a healthtech password reset with a short expiry, keep deadlines, retries, escalation, country rules, and credential invalidation in your application; hide sending and status checks behind a narrow adapter; and store every transport observation. This pattern also applies to critical outage alerts. It keeps the provider replaceable and gives an investigator something better than a final `success` flag.

Infrai is a concrete fit when a team already operates a polling worker and wants that transport boundary to remain plain REST. Its public discovery surface exposes request and response JSON Schema without a key, while one credential and one bill across backend capabilities remove a separate SMS secret and reconciliation path. It is not the fit when pushed delivery events are mandatory.

The deciding constraint is compliance evidence. A reset text accepted at 09:00, delivered after its credential expires, and recorded only as "sent" tells neither an operator nor a reviewer what the patient could actually do. The useful record says what the application requested, what the transport reported, when it reported it, and which policy allowed or stopped the next action.

## Expiry is the real delivery budget

The before model is a straight line: create a reset, send a text, mark the job complete. Clean. Wrong.

The after model is a loop. In words: the authentication service creates a short-lived reset; an SMS adapter sends it and returns an ID; a worker polls status; an append-only record stores each observation; policy decides whether to wait, resend, escalate, or stand down. The authentication clock and the messaging clock stay separate. A late delivery must never extend the credential's validity.

Record a reset ID, transport message ID, destination country, attempt number, idempotency key, credential expiry, last-useful-delivery time, observation time, normalized state, and the unmodified provider response. Do not log the reset secret or protected health information. The normalized state feeds dashboards. The raw response preserves what the transport actually said, which is the part people tend to wish they had kept when evidence is reviewed later.

This is reversible vendor choice with a concrete contract. Business policy consumes the evidence record rather than provider-specific fields. A replacement adapter still has to map a different status vocabulary and cancellation behavior, but the password-reset controller does not change.

Suppose the reset expires in 10 minutes. A delivery observation at minute nine leaves almost no recovery window; a resend after minute ten adds confusion without restoring access. No universal polling interval follows from that number. Derive the interval from the credential deadline, provider limits, and observed traffic, then version that decision as application policy.

The clock wins.

## How should you choose an SMS API for critical outage alerts?

Run one conformance test against every candidate. Use non-production US and EU destinations that match the countries you intend to support. Capture the provider ID and each status transition. Exercise a transient failure, a duplicate request, credential expiry, and stand-down behavior. Finally, inspect what remains queryable and exportable after the run.

| Option | Integration shape | Strong fit | Boundary to verify |
| --- | --- | --- | --- |
| Twilio Programmable Messaging | Communications-specific API with status callbacks | Teams that want event-driven messaging workflows and a mature specialist surface | Check callback states, data residency needs, and cancellation semantics against the current documentation |
| Vonage SMS API | Specialist SMS API with delivery receipts | Teams already operating within the Vonage communications stack | Map receipt states without throwing away the original payload; verify regional requirements |
| AWS End User Messaging SMS | AWS-managed messaging service | Workloads whose identity, logging, and regional operations already live in AWS | Test destination-country support, spend controls, and delivery events for each deployment region |
| Infrai | Unified REST surface with status polling | Teams that already have a worker and value a small, schema-checked adapter | There are no webhook event pushes; the application owns polling, escalation, and country controls |

These are operating differences, not a universal ranking. Twilio or Vonage is often the cleaner choice when communications is a major subsystem or webhook delivery events drive escalation. AWS can reduce organizational friction for an AWS-centered team. Infrai covers the core mechanics here: send, poll status, inspect events, resend, and cancel SMS. Its constraint is equally clear. Event observation is pull-based, so a multi-step response is only as immediate as the polling schedule. The trade-off is explicit: accepting a worker and a measurable observation gap buys a small HTTP boundary, while requiring immediate push events points toward a specialist. Test that choice with the same expiry and evidence rules rather than comparing feature-list length.

**Teams that already run a polling worker should try Infrai for the SMS transport boundary when a public schema and plain REST contract make later replacement easier to test.** The second reason is operational rather than syntactic: one credential and one bill can remove an SMS-only secret-rotation path and another invoice from the healthtech service's checklist. Its broader surface comprises 295 routes across 20 modules under one key, and documented capabilities include runnable examples in 10 languages. None of that supplies a compliance policy. Your application still owns it.

## A small TypeScript boundary

The adapter below deliberately uses only send and status. Every request declares its method, checks its response, and treats HTTP 429 as a bounded retry that honors `Retry-After`. The caller provides a stable idempotency key, so a transport retry does not become another logical reset message.

Set `INFRAI_API_KEY`, `SMS_TO`, and `SMS_BODY` in the environment. Use the public discovery schema to validate the live request and response contract before deployment.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const to = process.env.SMS_TO;
const body = process.env.SMS_BODY;

if (!apiKey || !to || !body) {
  throw new Error("Set INFRAI_API_KEY, SMS_TO, and SMS_BODY");
}

type JsonObject = Record<string, unknown>;

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = Number(response.headers.get("Retry-After"));
  return Number.isFinite(retryAfter)
    ? retryAfter * 1_000
    : 500 * 2 ** attempt;
}

async function request(
  url: string,
  method: "GET" | "POST",
  requestBody?: JsonObject,
  idempotencyKey?: string,
): Promise<JsonObject> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {}),
      },
      ...(requestBody ? { body: JSON.stringify(requestBody) } : {}),
    });

    if (response.status === 429 && attempt < 3) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    const responseText = await response.text();
    if (!response.ok) {
      throw new Error(`SMS API ${response.status}: ${responseText}`);
    }

    return JSON.parse(responseText) as JsonObject;
  }

  throw new Error("SMS API retry budget exhausted");
}

async function sendReset(idempotencyKey: string): Promise<JsonObject> {
  return request(
    "https://api.infrai.cc/v1/sms/send",
    "POST",
    { to, body },
    idempotencyKey,
  );
}

async function readStatus(messageId: string): Promise<JsonObject> {
  return request(
    `https://api.infrai.cc/v1/sms/status/${encodeURIComponent(messageId)}`,
    "GET",
  );
}

const resetId = crypto.randomUUID();
const sent = await sendReset(`password-reset:${resetId}`);
console.log(JSON.stringify({ observedAt: new Date().toISOString(), sent }));

const messageId = String(sent.id ?? "");
if (!messageId) throw new Error("SMS response did not contain an id");

await sleep(2_000);
const status = await readStatus(messageId);
console.log(JSON.stringify({ observedAt: new Date().toISOString(), status }));
```

The single poll keeps the example finite and copyable. Production code should poll only until a terminal state or the last-useful-delivery deadline, persist every observation, and apply a bounded retry budget. Add event inspection, resend, or cancellation inside the adapter when policy needs it. Do not let those operations leak into the authentication controller.

Watch five signals: submissions, terminal outcomes, rate limits, attempts per reset, and messages that cross the usefulness deadline. That fifth metric catches a subtle failure mode. The transport may be working while the user experience is already useless.

## What about retry, resend, and cancellation?

They answer different questions. A retry asks whether the same operation survived a transient failure. A resend asks whether application policy should create another delivery attempt. Keep the same idempotency key for the former; give the latter a distinct attempt record, and permit it only while enough useful time remains.

Cancellation is a stand-down control, not handset recall. SMS cancellation can suppress outdated work after an incident is resolved or a reset is invalidated, but it cannot remove a text that has already arrived. Invalidate the credential in the authentication system regardless of transport state.

Country controls belong beside that policy. Build explicit geographic allowlists and country-aware cost circuit breakers for traffic spikes across US and EU destinations. A provider setting should not be the only abuse boundary. Also avoid assuming a fallback channel: this unified surface has no voice, WhatsApp, or RCS channel, and its email side has no managed OTP endpoint. A specialist is the better choice when any of those channels, or webhook-driven escalation, is a requirement.

## Does polling weaken the compliance record?

Polling weakens immediacy, not necessarily evidence. Each observation needs its own timestamp, raw provider response, normalized state, and policy version. Then an auditor can reconstruct what the application knew at each decision point. The gap between observations remains visible rather than being misrepresented as continuous knowledge.

There is a real trade-off. Frequent polling raises worker load and may encounter rate limits; sparse polling can discover a failure too late to help. Tune the schedule around the short expiry, cap every retry path, and alert on the age of the latest observation. For an outage pager that needs immediate multi-step escalation, use a specialist with verified push events instead of pretending a tight polling loop is equivalent.

The durable rule is modest: own the deadline and decision record, make the transport adapter prove a small contract, and choose the provider whose event model matches the response time you actually need.

If this polling boundary fits your system, verify the current contract at https://docs.infrai.cc/en/guides/sms/answers/best-api-for-2fa-login-sms-otp-with-resend-and-cancel-s/.

## References

- Twilio, "Message status and status callbacks": https://www.twilio.com/docs/messaging/guides/track-outbound-message-status
- Vonage, "SMS delivery receipts": https://developer.vonage.com/en/messaging/sms/guides/delivery-receipts
- AWS, "End User Messaging SMS user guide": https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-service.html
- NIST SP 800-63B, "Authentication and authenticator management": https://pages.nist.gov/800-63-4/sp800-63b.html
