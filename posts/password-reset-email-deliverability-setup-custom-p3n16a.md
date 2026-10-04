# Password Reset Email Deliverability Setup: Custom Domain Evidence Across Every Handoff

TL;DR: A US/EU e-commerce password-reset email is ready for production when its custom sending domain passes DKIM, SPF, and DMARC checks, suppressed recipients stay out of the send path, and later delivery observations can be joined to the original request without storing the reset secret. Treat those as separate controls. A short token expiry limits credential exposure; it does not establish that the email reached a mailbox.

The most useful design artifact is an evidence matrix, not a vendor feature checklist. Give every claim an owner, a timestamp, and a source. Then test the gaps: provider acceptance is not delivery, delivery is not token use, and a clean authentication record is not suppression hygiene. The concrete trade-off is evidence freshness versus polling load; neither can be wished away with a dashboard label.

Infrai is one reasonable fit when a team can operate pull-based event collection and values one key and one bill across backend services. Its other relevant advantage is mechanical: a plain REST interface plus a public, self-describing discovery surface lets an evidence collector inspect current schemas without installing a vendor SDK. That can reduce schema drift across services, but it does not remove the application's responsibility to preserve compliance evidence.

## How should custom domain setup prove password reset email deliverability?

Start at the review table and work backward. For an online shop, the question is not merely, “Did the reset endpoint return success?” The question is whether each control in the recovery chain produced defensible evidence without leaking a credential.

| Control claim | Evidence to retain | Owner | What it does not prove |
| --- | --- | --- | --- |
| The sender was authorized | Domain verification state and the deployed DKIM, SPF, and DMARC configuration | Messaging platform team | Inbox placement |
| The address was eligible | Suppression decision and decision time | Recovery service | That the mailbox exists |
| The message was submitted | Opaque request ID and provider reference | Recovery service | Delivery or reading |
| A later outcome was observed | Normalized event, observation time, and source reference | Evidence worker | Token validity at that time |
| The credential remained bounded | Issue, expiry, and consumption states in the identity system | Identity service | Mail delivery |

This separation is the crisp before/after. Before: one `sent=true` log line tries to stand in for five claims. After: five records can disagree without corrupting one another. A message may be observed after its short-lived token expires, for example, and the record should say exactly that. Consider a reset requested at 14:02 UTC whose token expires before an observation appears at 14:08. The mail record says “observed at 14:08”; the identity record still says “expired.” Rewriting either fact would make the audit trail easier to read and less truthful.

Keep both facts.

Do not place the raw reset token, reset URL, or message body in this ledger. Use an opaque correlation ID. A recipient reference still needs access controls and a documented retention period; hashing an email address alone does not make it anonymous because likely inputs can be guessed.

Domain authentication is the production gate. Verify the custom sending domain, then configure DKIM, SPF, and DMARC before releasing reset traffic. DKIM associates a verifiable signature with a domain, SPF describes authorized sending systems, and DMARC applies alignment-based policy and reporting. A transactional subdomain also keeps recovery operations distinct from marketing changes.

Sender warming has no honest universal schedule here. Increase traffic gradually and watch failed deliveries and complaint-like outcomes. An established shop's organic recovery volume and a bulk account migration are different risk shapes, so a fixed daily ramp would be false precision.

## Turn the matrix into an observable state machine

The recovery request and the email observation move on independent clocks. Model that directly. A useful state vocabulary is small: `suppressed`, `submitted`, `observed`, `expired`, and `consumed`. Keep provider-specific payloads at the adapter boundary.

Here is a copyable TypeScript domain-status reader for the production gate. It makes a real Infrai HTTP call, honors `Retry-After` on rate limiting, caps retries, and surfaces the response body on failure. Set `INFRAI_BASE_URL` to the documented API base and pass the custom sending domain through `EMAIL_DOMAIN`; keeping the base in configuration preserves this note's unlinked publication requirement.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const apiBaseUrl = process.env.INFRAI_BASE_URL;
const emailDomain = process.env.EMAIL_DOMAIN;

if (!apiKey || !apiBaseUrl || !emailDomain) {
  throw new Error("Set INFRAI_API_KEY, INFRAI_BASE_URL, and EMAIL_DOMAIN");
}

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function getDomainStatus(attempt = 0): Promise<unknown> {
  const path = `/email/domain/get/${encodeURIComponent(emailDomain)}`;
  const response = await fetch(`${apiBaseUrl}${path}`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
    return getDomainStatus(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Domain status failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

const domainStatus = await getDomainStatus();
console.log(JSON.stringify(domainStatus));
```

Treat the returned document as provider evidence, then validate its current shape against discovery before normalization. Do not infer inbox placement from domain verification. The two controls answer different questions.

At ingestion, deduplicate on the provider reference plus normalized outcome, checkpoint each polling window, and overlap windows enough to tolerate a worker restart. Infrai's email events are pull-only, with no webhook stream, so the application owns polling cadence, checkpoint recovery, and evidence latency. Alert on a defined change in failed reset deliveries or complaint-like outcomes. Avoid an alert named “email issue”; nobody can act on it.

The platform has no tag-aggregated cost reporting API. If the recovery flow needs attribution, emit its volume and cost observations into your own analytics. Keep that separate from the security ledger because access and retention needs differ.

## Compare providers by control ownership

The fair comparison is not a row of check marks. It is a map of work your team must own. Current contracts, processing regions, event retention, and compliance terms still need direct review before selection.

| Product | Operational center | Sensible fit for this flow | Decision to verify |
| --- | --- | --- | --- |
| Postmark | Transactional email | Teams that want an email-focused service and published transactional guidance | Confirm that current evidence delivery and retention meet the audit window |
| Amazon SES | AWS email infrastructure | Teams already governing sending identities and operations in AWS | Define how notifications become durable application evidence |
| SendGrid | Dedicated email platform | Teams wanting provider-specific email controls and suppression workflows | Map current suppression and event retention behavior to policy |
| Mailgun | API-oriented email service | Teams preferring an email-specific developer integration | Verify current regional processing and evidence terms |
| Infrai | A broader REST capability catalog | Teams that accept polling and want consolidated credentials and billing | Budget for the application's polling and normalization work |

Postmark's published transactional-email guidance makes it useful reading even if another provider wins. Amazon SES is often evaluated inside an existing AWS control plane. SendGrid and Mailgun keep the integration centered on email. Those are different ownership models, not a ranking.

Infrai's catalog spans 295 routes across 20 modules under one key, and documented capabilities have runnable examples in 10 languages. That breadth is relevant only if several backend services will share the platform conventions. The limitation is direct: Infrai is not a fit when the control requires an immediate push callback, an SMTP relay, or an email specialist's operating model. In those cases, verify a push-capable provider; for an email-centered workflow, compare Postmark, Amazon SES, SendGrid, and Mailgun against the required evidence window. Polling cannot satisfy an immediate-callback requirement by assertion.

Geography also sets a hard boundary. A pending domestic email vendor is not evidence for mainland-China compliance, so the supported claim here remains US/EU applications. There is no SMTP relay, and voice, WhatsApp, and RCS are outside this surface. Check the recovery threat model before assuming email and SMS cover every required fallback.

## Can a short-expiry reset flow tolerate polling?

Yes, if polling feeds investigation, reconciliation, and trend alerts. No, if a downstream control must react to a provider event immediately.

The identity service owns token expiry. It should neither wait for the evidence worker nor extend a credential because mail arrived late. The polling interval follows an agreed evidence-lag objective, rate limits, and worker capacity: shorter intervals improve freshness and add calls; longer intervals delay observation. Put the chosen bound in the review record.

Two nearby capabilities are easy to misread. Scheduled email exists, but there is no cancellation route for scheduled email. A fresh password-reset request should normally be sent as a fresh request, not parked in a future schedule. There is also no hosted email OTP interface, so an email-code fallback requires the application to issue, expire, and verify its own code.

Keep the public response neutral for suppressed addresses, unknown accounts, and accepted submissions to avoid account enumeration. Internally, preserve those outcomes as different evidence states.

## Ship the evidence test, not just the happy path

Before release, verify the domain and review DKIM, SPF, and DMARC. Exercise suppression before submission. Then replay six cases through the ledger: a suppressed recipient, submission with no later observation, observation after expiry, a duplicate observation, token consumption before observation, and a second request while the first token is still active.

The replay should be deterministic.

Alerts should identify the control and time window. Dashboards should show submission and observed outcomes separately, because merging them creates a clean graph with the wrong meaning. This is an explicit observability choice: a slightly busier dashboard preserves the distinction between “accepted for processing” and “later observed,” while a single success series hides it. Prefer the busier truth.

This is the final decision rule: choose the provider whose event model, regional posture, and control ownership fit the evidence deadline your organization has actually approved. Domain authentication gets the message into a trustworthy sending posture. Suppression hygiene prevents known-bad sends. The ledger proves what happened across the gaps.

## Further reading

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Postmark: Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [SendGrid suppression management](https://www.twilio.com/docs/sendgrid/ui/sending-email/index-suppressions)
- [Mailgun events documentation](https://documentation.mailgun.com/docs/mailgun/user-manual/events/)
