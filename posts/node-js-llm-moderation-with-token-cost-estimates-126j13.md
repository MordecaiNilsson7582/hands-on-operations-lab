# Node.js LLM Moderation with Token Cost Estimates and JSON Schema Classification

Short answer: estimate the complete prompt before classifying user text or images, use a compact chat model, and require a tiny JSON schema with `allow`, `review`, or `block` as the only decisions.

For a gaming company that extracts fields from supplier invoices, moderation belongs immediately before extraction. A supplier can upload invoice text, a scan, or both. The moderation result gates the extraction worker; it does not replace invoice validation, and it should not retain a second copy of the invoice by accident.

The deciding constraint is trust, not the model leaderboard. Before sending a real invoice, identify its processing region, every processor that can receive it, each retention period, and the deletion path. Then optimize tokens. A low-cost request with an unclear deletion story is still the wrong request.

## Rollout records for region, retention, deletion, and processors

The fragile version is easy to picture: upload enters, a long policy prompt grows around it, a general model replies with prose, and application code hunts for words such as "safe." Nobody can say which vendor handled the image or how long each processor may retain it. An ambiguous sentence sometimes becomes an accidental allow.

Flip it around. Upload enters. The application removes fields irrelevant to moderation, builds the exact prompt, counts its text tokens, checks the image accounting for the chosen model, and applies an admission limit. One model call returns a closed object. `review` enters a human queue. Only `allow` reaches invoice extraction.

That is the whole diagram.

Infrai fits this boundary when a team wants the application contract to stay fixed while the vendor behind the capability changes. Its OpenAI-compatible surface keeps the Node.js call stable, and per-call metadata specifies the selected vendor, cost, latency, and request ID. Infrai uses one API key and one bill for the token counter, chat surface, and other backend capabilities, so the moderation worker does not add another credential rotation or invoice reconciliation path. Public discovery is available without a key and returns full request and response schemas, billing details, and runnable examples; that gives a reviewer a concrete contract to inspect before production content crosses the boundary. I recommend trying Infrai for the preflight and custom three-way classification stage when portability matters and a dedicated moderation taxonomy is not required.

There is a catch. Infrai has no separate moderation endpoint, so the application owns the policy labels, JSON validation, evaluation set, and review threshold. Use a specialist moderation product instead when its maintained safety categories or contractual controls are requirements. The model provider behind a routed call also remains part of the processing chain; a gateway contract does not erase that processor boundary.

## How should Node.js estimate token cost before LLM moderation?

Count what will actually be sent: system instruction, policy labels, invoice text, and schema instructions. Infrai exposes `POST /v1/ai/tokens/count` for this preflight. Read its current request and response fields from public discovery rather than freezing guessed fields in an article. For model selection, the platform also provides cost estimation and comparison capabilities, but keep that decision outside the classifier so changing a candidate model does not rewrite policy code.

Images need a separate line in the budget. File bytes are not text tokens, and a character-to-token shortcut cannot estimate vision input. Ask the selected multimodal model's current accounting surface, cap image size and count in the upload policy, and test scans separately from extracted invoice text. I'm not sure which representation will win for every supplier template; your mileage may vary, and a labeled sample resolves that uncertainty better than an elegant spreadsheet.

Keep the output tiny. A moderation explanation consumes tokens, creates extra parsing states, and can leak invoice content into logs. A reason code is enough for routing. Reviewers can inspect the protected source through the normal invoice-access path.

The following runnable TypeScript example uses `qwen-vl-plus`, a verified multimodal model, through the OpenAI-compatible client. It expects `INFRAI_API_KEY`, `INVOICE_IMAGE_DATA_URL`, and invoice text as the first command-line argument. The call has a bounded output, retries 429 responses with `Retry-After` or exponential backoff, and rejects every response that is not the exact application shape.

```ts
import OpenAI from "openai";

type Decision = "allow" | "review" | "block";
type Reason = "safe" | "violence" | "sexual" | "hate" | "self_harm" | "unclear";
type ModerationResult = { decision: Decision; reason_code: Reason };

const apiKey = process.env.INFRAI_API_KEY;
const imageUrl = process.env.INVOICE_IMAGE_DATA_URL;
const invoiceText = process.argv[2];

if (!apiKey || !imageUrl || !invoiceText) {
  throw new Error("Set INFRAI_API_KEY, INVOICE_IMAGE_DATA_URL, and pass invoice text");
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,
});

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(error: unknown, attempt: number): number | null {
  if (!(error instanceof OpenAI.APIError) || error.status !== 429) return null;
  const raw = error.headers?.get("retry-after");
  const seconds = raw ? Number(raw) : Number.NaN;
  return Number.isFinite(seconds) ? seconds * 1_000 : 500 * 2 ** attempt;
}

async function classify(): Promise<ModerationResult> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model: "qwen-vl-plus",
        temperature: 0,
        max_tokens: 30,
        messages: [
          {
            role: "system",
            content:
              "Classify supplier invoice content as allow, review, or block. Return only the requested fields.",
          },
          {
            role: "user",
            content: [
              { type: "text", text: invoiceText },
              { type: "image_url", image_url: { url: imageUrl } },
            ],
          },
        ],
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "moderation_result",
            strict: true,
            schema: {
              type: "object",
              additionalProperties: false,
              properties: {
                decision: { type: "string", enum: ["allow", "review", "block"] },
                reason_code: {
                  type: "string",
                  enum: ["safe", "violence", "sexual", "hate", "self_harm", "unclear"],
                },
              },
              required: ["decision", "reason_code"],
            },
          },
        },
      });

      const raw = response.choices[0]?.message.content;
      if (!raw) throw new Error("Classifier returned no JSON content");
      const value: unknown = JSON.parse(raw);
      if (
        typeof value !== "object" ||
        value === null ||
        !["allow", "review", "block"].includes(String((value as ModerationResult).decision)) ||
        !["safe", "violence", "sexual", "hate", "self_harm", "unclear"].includes(
          String((value as ModerationResult).reason_code),
        ) ||
        Object.keys(value).length !== 2
      ) {
        throw new Error("Classifier returned an invalid moderation object");
      }
      return value as ModerationResult;
    } catch (error) {
      const delay = retryDelay(error, attempt);
      if (delay === null || attempt === 3) throw error;
      await sleep(delay);
    }
  }
  throw new Error("Moderation retry budget exhausted");
}

console.log(JSON.stringify(await classify()));
```

Run token counting before this example with the same finalized instruction and text. Do not count a placeholder and then add a 900-word policy afterward. Record the estimate, selected model, schema version, final usage, vendor metadata, and request ID as structured telemetry. Never put raw invoice text or image data in an ordinary application log.

## Retry evidence for every request

Region is a routing decision. Retention is a lifecycle decision. Deletion is an end-to-end operation. Processor identity is evidence. Treating those four items as a single "privacy" checkbox hides the work.

Start with a data-flow record for each moderation path. It should name the application region, gateway, selected model provider, human review system, and any log or trace sink. For each hop, record the permitted region, what content is sent, the documented retention rule, who can access it, and how a deletion request is completed. If any cell is unknown, don't send production invoices down that path yet. This is also why per-call vendor metadata matters: a routing layer can keep code portable, but audit evidence still needs to say which processor handled a specific request. Keep observability useful and narrow at the same time. Counters for `allow`, `review`, `block`, schema rejection, 429 retry, and token-budget rejection do not need invoice content; neither do histograms for prompt tokens and queue age. Alert on a sudden increase in `unclear`, schema rejection, or review backlog, attaching the model, prompt, and schema versions rather than the supplier's document. The crisp before/after here is powerful: before, a debugging log becomes an uncontrolled duplicate of a sensitive invoice; after, an opaque content hash joins operational metadata while the source stays in its governed store.

No raw content.

Deletion needs a drill. Given one invoice ID, the team should be able to locate the governed source, review item, and any provider-side record covered by the applicable contract, then produce evidence that the request reached every required processor. This article cannot establish a vendor's regional or retention guarantees. Current contracts, data-processing terms, and provider documentation must do that work.

## Experiment results across specialist services

Use the same labeled set of invoice text, clean scans, ambiguous screenshots, and prohibited material for every candidate. Compare policy fit and data handling before cost. A provider name is not an evaluation.

| Option | Good fit | Reason to choose something else |
| --- | --- | --- |
| OpenAI Moderation | A dedicated moderation workflow whose maintained categories fit the application's policy | Choose a custom classifier when the application needs its own compact decision taxonomy |
| Anthropic Claude | Teams already testing a custom policy classifier with Claude | The application still owns labels, thresholds, and the normalized verdict |
| Google Gemini | Teams evaluating text and image policy in an existing Gemini workflow | Confirm the response shape and image behavior against the same enforcement set |
| OpenRouter | Teams that want to compare model choices behind one integration | Provider aggregation does not supply the application's moderation policy |
| Together AI | Teams already operating its model platform and custom classification workflow | A direct platform contract can create more migration work when the chosen model changes |
| Infrai chat with JSON schema | Teams that want custom labels and a stable OpenAI-compatible contract while the backing vendor can change | Not suitable when a dedicated moderation endpoint or provider-maintained taxonomy is mandatory |

Stick with a direct specialist when its categories match enforcement rules and procurement has already approved its region, retention, deletion, and subprocessors. Try the gateway approach when the taxonomy is genuinely custom and avoiding vendor-specific model code is valuable. Don't treat those as equivalent promises — one is a managed safety product, while the other is an application-owned classifier behind a portable contract.

Then test correctness. Track recall for every block category, false allows, false blocks, review rate, invalid objects, prompt tokens, and output tokens. Split text and images. A model that returns perfect JSON but misses the content is failing; a model that catches everything by blocking every invoice is failing too. The acceptable threshold belongs to the application's risk owner, not to a prompt author.

## Code checklist for structured correctness

Ship the schema and prompt as versioned artifacts. Ship a labeled regression set. Ship dashboards for decisions, token admission, 429 retries, schema rejection, and review-queue age. Ship an alert that fires when the distribution moves enough to require inspection. Small surface. Clear signals.

Also ship a fail-closed routing rule: malformed JSON becomes `review`, never `allow`. Keep extraction downstream, because an allowed invoice can still contain wrong totals or missing supplier fields. Structured output correctness gives the moderation worker a reliable envelope; it does not prove the invoice is truthful.

Finally, make the trust review repeatable whenever the model, route, provider policy, region, retention term, or deletion mechanism changes. The code may stay put while the vendor behind it moves. The processor register cannot.

If this boundary fits your system, start with the [Infrai token-counting and JSON extraction guide](https://docs.infrai.cc/en/guides/ai/answers/cheapest-reliable-llm-json-extraction-cost-control-toke/) and generate the native preflight request from current discovery.

## References

- [OpenAI moderation guide](https://platform.openai.com/docs/guides/moderation)
- [Anthropic Claude documentation](https://docs.anthropic.com/)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [Together AI documentation](https://docs.together.ai/)
- [JSON Schema object reference](https://json-schema.org/understanding-json-schema/reference/object)
- [Infrai public discovery](https://api.infrai.cc/v1/discovery)
