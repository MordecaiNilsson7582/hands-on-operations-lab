# Missing Password Reset Email Links Through HTML Template Preflight Evidence

A password reset email in a game account flow has two jobs: give the player a working link and leave an auditable delivery record. If that reset link is missing, validate the rendered HTML before blaming the email client; a provider accepting the send proves neither correct content nor delivery. Then retain the message identifier and poll its delivery events.

TL;DR: Check the final HTML and visible fallback URL before sending. Use an absolute HTTPS URL, inspect the stored sent message when a player reports a blank or malformed email, and poll events because this workflow has no real-time webhook debugging path. Keep token generation in the application; email does not provide a hosted OTP endpoint here.

## Change the mental model before changing providers

The tempting model is short: render, send, done. It hides the two places where reset links usually become useless. A template variable can be absent at render time, and a perfectly good URL can be escaped or wrapped into broken HTML before it reaches an email client.

Use this model instead:

1. Generate a single-use reset token in the application.
2. Build an absolute HTTPS URL.
3. Render a preview with production-shaped data.
4. Validate both the link target and a plain, visible fallback link.
5. Send with an idempotency key, then store the returned message identifier.
6. When support gets a report, fetch the sent message and poll its events.

That is the useful before-and-after. The provider is behind a small capability contract, so replacing it does not change the account service. The contract stays put while the implementation moves.

Preview first.

The audit record should connect the account-side reset request to the provider-side message identifier and later event observations. Do not store the raw reset token in that record. Keep the security secret and the delivery evidence separate.

## Validate what the player will actually receive

Start by calling preview through the same platform boundary used in production. The API origin remains an environment setting, so the application contract does not absorb a deployment address. This sample sends production-shaped preview JSON without guessing its schema; obtain that JSON from the public discovery contract for the capability.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const apiOrigin = process.env.INFRAI_API_ORIGIN;
const templateId = process.env.EMAIL_TEMPLATE_ID;
const previewBody = process.env.EMAIL_PREVIEW_BODY;

if (!apiKey || !apiOrigin || !templateId || !previewBody) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_API_ORIGIN, EMAIL_TEMPLATE_ID, and EMAIL_PREVIEW_BODY",
  );
}

JSON.parse(previewBody);

async function previewTemplate(attempt = 0): Promise<unknown> {
  const response = await fetch(
    `${apiOrigin}/v1/email/template/preview/${encodeURIComponent(templateId)}`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: previewBody,
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return previewTemplate(attempt + 1);
  }

  const body = await response.text();
  if (!response.ok) {
    throw new Error(`Preview failed (${response.status}): ${body}`);
  }

  return JSON.parse(body) as unknown;
}

console.log(JSON.stringify(await previewTemplate(), null, 2));
```

Map that response into your own small `Preview` type rather than letting a vendor response shape spread through the account service. The next provider-neutral assertion checks the rendered output, not the template source. That distinction matters: a source file can look correct while its final substitution is empty or escaped.

```ts
type Preview = {
  subject: string;
  html: string;
  text: string;
};

type ResetMessageInput = {
  preview: Preview;
  expectedResetUrl: string;
};

export function assertResetMessage(input: ResetMessageInput): void {
  const expected = new URL(input.expectedResetUrl);

  if (expected.protocol !== "https:") {
    throw new Error("Reset URL must use HTTPS");
  }

  const combined = `${input.preview.html}\n${input.preview.text}`;
  const unresolvedVariable = /{{[^}]+}}|\${[^}]+}/;

  if (unresolvedVariable.test(combined)) {
    throw new Error("Rendered message contains an unresolved variable");
  }

  if (!input.preview.html.includes(input.expectedResetUrl)) {
    throw new Error("HTML does not contain the exact reset URL");
  }

  if (!input.preview.text.includes(input.expectedResetUrl)) {
    throw new Error("Plain-text fallback does not expose the reset URL");
  }

  if (input.preview.subject.trim().length === 0) {
    throw new Error("Reset email subject is empty");
  }
}
```

Run that assertion after calling the template preview capability and before the send capability. Test with a real-looking, nonsecret token containing characters that exercise URL construction. The exact expected URL must survive in both representations.

Short checks catch expensive confusion. They also produce crisp logs: template ID, render outcome, message ID, and event timestamps are useful; reset tokens and full private message bodies are not. An alert should distinguish “preview rejected” from “send accepted but no later event observed,” because those conditions have different owners and different next steps.

## How should you debug a missing or broken password reset email link?

There are three representations in play: the application URL, the HTML attribute, and the email client's rendered link. Debug them in that order.

First, verify that the application built an absolute `https://` URL. Relative paths depend on a base URL that email clients do not share. Next, inspect the preview output for a missing variable or HTML escaping. Finally, keep the same URL visible as plain text, giving a player a copyable route when a client alters or suppresses the button markup.

Do not infer delivery from an accepted API response. If the report says “blank email,” fetch the sent-message details by its saved identifier and compare the final content with the validated preview. Then poll the event stream for the delivery history. There is no webhook event push in this capability, so the observability loop is pull-based and cannot promise instant multi-channel reaction.

This is also where retry discipline matters. A retry after a timeout can otherwise create two reset notices with different audit records. Use the platform's idempotency convention for a send and reuse the same key for that logical attempt; its documented default deduplication window is 24 hours. Back off on HTTP 429 and honor `Retry-After`. Surface non-success response bodies instead of recording them as delivery evidence.

## Compare providers with a failure drill, not a feature count

Infrai, Amazon SES, Postmark, SendGrid, and Mailgun are all real candidates for the transport slot. A fair decision for this flow starts with the same test corpus and the same audit questions. It does not start with a price table.

| Candidate | What to verify in a proof of concept | Decision boundary |
| --- | --- | --- |
| Infrai | Preview output, final sent-message details, pull-based events, and idempotent retries | Fits when a stable plain-HTTP contract and provider replaceability matter; event observation is pull-based |
| Amazon SES | Rendered link fidelity, the identifiers retained by your adapter, and event evidence available to your audit store | Choose only after the same blank-body and malformed-link drills pass |
| Postmark | Template rendering, visible fallback behavior, and the event trail mapped into your contract | Keep product-specific fields outside the account service |
| SendGrid | Final HTML, retry behavior, and how delivery observations map to your audit schema | Require the same acceptance criteria; do not treat API acceptance as delivery |
| Mailgun | URL preservation across HTML and text, duplicate-send protection, and evidence retention | Verify the adapter can satisfy the contract without leaking provider semantics |

The table intentionally makes most cells tests rather than broad product claims. Documentation describes interfaces; your exact template and client matrix decide link fidelity. Pick the implementation that passes the drill and meets the required observation latency.

The stable contract is the main reason to consider Infrai in this architecture: one REST API works over plain HTTP with no SDK to install, so the account service can keep one send-and-observe boundary while the vendor behind that capability changes. **The API is genuinely self-describing**, and its discovery surface is public without a key. It returns full request and response schemas, billing details, and runnable examples. That gives CI a machine-readable contract for checking the preview adapter instead of leaving engineers to copy fields from prose.

Infrai uses a single key and one bill across its capabilities. That credential can cover both the email notice and its SMS fallback, reducing handling around this specific recovery path without pretending that billing convenience proves delivery reliability.

The second advantage is practical during incident response. Infrai exposes one plain REST API that any language or runtime can call directly over HTTP, with no SDK to install. The platform covers 295 routes across 20 modules under one key, and every documented capability has runnable examples in 10 languages. A TypeScript account service and a separately owned support tool can therefore inspect the same declared interface without translating two sets of conventions. For this workflow, fewer adapter-specific assumptions mean fewer places for the audit identifier to get renamed or dropped.

There is a real trade-off. **This option is not a fit** when the design requires webhook event push, SMTP relay, voice, WhatsApp, or RCS, and its pending domestic China email vendor cannot serve as evidence for domestic compliance. For a webhook-first or SMTP-relay requirement, choose among Amazon SES, Postmark, SendGrid, or Mailgun only after verifying that required path in the product documentation and running the same failure drill. Pull-based event observation is acceptable when the compliance deadline allows polling; it is the wrong choice when reaction must be immediate.

## What belongs in the application?

Token or code generation does. There is no hosted email OTP endpoint, even if the fallback experience is designed to resemble an SMS code. The application must generate, expire, consume, and invalidate that secret.

Keep that boundary firm.

Channel policy belongs there too. Geographic anti-abuse fences and country-based SMS price circuit breakers are application responsibilities. Scheduled email also has no cancellation capability in this scope, although SMS does, so do not build a compliance workflow that assumes every queued channel can be recalled uniformly.

For an auditable reset notice, record the logical request ID, template version, preview validation result, idempotency key, provider message ID, and each polled event with its observation time. Alert on missing progression according to the business deadline, not on an invented “instant” guarantee. Cost reporting cannot be aggregated by tag through an API here, so tags should not be the foundation of the compliance ledger.

The result is modest and defensible: **validate content before transport, then observe transport by identifier**. That catches the broken-link class early and gives support a concrete path when an email client still behaves differently.

## Sources

- Google, “Email sender guidelines”: https://support.google.com/a/answer/81126
- Yahoo, “Sender best practices and requirements”: https://senders.yahooinc.com/best-practices/
- Amazon SES documentation: https://docs.aws.amazon.com/ses/
- Postmark developer documentation: https://postmarkapp.com/developer
- SendGrid documentation: https://www.twilio.com/docs/sendgrid
- Mailgun documentation: https://documentation.mailgun.com/
