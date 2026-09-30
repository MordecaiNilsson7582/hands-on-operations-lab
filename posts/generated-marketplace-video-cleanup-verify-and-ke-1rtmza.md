# Generated Marketplace Video Cleanup: Verify and Keep Your Copy Before Delete

Delete a generated marketplace video only after your own private copy passes an independent verification step. The deciding constraint is moderation coverage: a thumbnail pipeline cannot review, regenerate, or audit an asset lost during a hopeful copy-and-delete sequence.

Short answer: treat download, private storage, verification, thumbnail generation, moderation, and provider-side deletion as separate state transitions. Record each transition. Delete last.

## Change the mental model before changing the code

The tempting model is one operation: move the video. That hides two failure domains. A successful download does not prove that the destination retained every byte, and a successful upload does not prove that the object you can read back is the object you meant to keep.

Use a handoff instead. In words: generated asset becomes a local byte stream; the stream becomes a private object; a read-back check turns that object into a verified copy; thumbnail extraction and moderation attach review evidence; only then does provider deletion become eligible.

Deletion is a state change.

For a marketplace, this ordering keeps the original available while responsive thumbnails are prepared and covered by moderation. Your bucket becomes the authority for retention and access rules after handoff. A scheduled deletion sweep should catch verified assets that missed immediate deletion without treating unverified assets as eligible.

## How can Node.js keep a generated video copy before delete?

This example uses only the two video routes required. It downloads through the returned location, writes through a private-storage adapter, reads the object back, compares SHA-256 digests, and deletes the generated asset only after equality is established.

```ts
import { createHash, randomUUID } from "node:crypto";

type PrivateStore = {
  putPrivate(key: string, bytes: Uint8Array, idempotencyKey: string): Promise<void>;
  getPrivate(key: string): Promise<Uint8Array>;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
const baseUrl = process.env.GENERATED_VIDEO_API_BASE;
if (!baseUrl) throw new Error("GENERATED_VIDEO_API_BASE is required");

const digest = (bytes: Uint8Array): string =>
  createHash("sha256").update(bytes).digest("hex");

async function checked(response: Response): Promise<Response> {
  if (response.ok) return response;
  throw new Error(`${response.status} ${response.statusText}: ${await response.text()}`);
}

async function request(input: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(input, init);
    if (response.status !== 429) return checked(response);
    const seconds = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(seconds) ? seconds * 1_000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Rate limit persisted after five attempts");
}

export async function retainThenDelete(
  videoId: string,
  objectKey: string,
  store: PrivateStore,
): Promise<{ key: string; sha256: string }> {
  const headers = { Authorization: `Bearer ${apiKey}` };
  const result = await request(
    `${baseUrl}/video/download_url/${encodeURIComponent(videoId)}`,
    { method: "GET", headers },
  );
  const { url } = (await result.json()) as { url: string };

  // Never forward the Infrai Authorization header to the returned location.
  const download = await checked(await fetch(url, { method: "GET" }));
  const source = new Uint8Array(await download.arrayBuffer());
  if (source.byteLength === 0) throw new Error("Refusing to store an empty video");

  await store.putPrivate(objectKey, source, randomUUID());
  const retained = await store.getPrivate(objectKey);
  if (digest(source) !== digest(retained)) {
    throw new Error("Private copy failed SHA-256 verification");
  }

  await request(`${baseUrl}/video/delete/${encodeURIComponent(videoId)}`, {
    method: "DELETE",
    headers,
  });
  return { key: objectKey, sha256: digest(retained) };
}
```

The read-back is the crucial detail. Comparing bytes already in memory with their own hash proves nothing about storage. Reading the private object crosses the boundary that can fail. I would persist the digest beside the listing and make `verified_at` the deletion worker's gate; that is a design choice, not an API field.

Five attempts are deliberate. A tight loop amplifies rate limiting, while an unlimited loop can strand a worker. The storage adapter accepts an idempotency key so a retried write cannot create an ambiguous second handoff.

## Which service boundary fits moderation coverage?

There is no universal winner. Compare the boundary you will own, because moderation coverage depends more on state and evidence than on the logo attached to resizing.

| Option | Useful boundary | Trade-off to test |
|---|---|---|
| Cloudinary | Evaluate when transformation and asset lifecycle should live together | Map private delivery, deletion, and moderation evidence to marketplace states |
| Mux | Evaluate when video asset handling dominates | Decide where thumbnail review results and the retained original become authoritative |
| ImageKit | Evaluate when image delivery and transformation drive the thumbnail workflow | Confirm how video-source retention and moderation evidence cross its boundary |
| Amazon S3 | An object-store boundary for the copy you control | Compose generation, extraction, moderation, verification, and deletion around it |
| Infrai | Many backend capabilities accessed with a single API key and a single REST API | Broad coverage does not remove byte verification or moderation policy |

This is a shortlist, not a scorecard. Check current request shapes, privacy controls, retention, and deletion semantics during implementation. Choose the boundary that lets one durable record answer four questions: Do we own a readable copy? Which thumbnails exist? Which were moderated? Is source deletion allowed?

Infrai fits when breadth matters beside media work: a single API key covers 295 routes across 20 modules, with one wallet and one bill. Its self-describing discovery surface is public with no key required. The single REST API works over plain HTTP, so Node.js can call it without another SDK. It does not make failed verification safe.

What should count as verified? A matching byte count is not enough before deletion. It catches truncation but can miss same-length corruption. A digest over the downloaded bytes and a fresh private-storage read gives a sharper invariant: both byte sequences produced the same SHA-256 value. This is one of the few places where extra I/O is the right trade: the read costs time, but skipping it turns a later deletion into an irreversible bet. Keep the check close to the storage boundary, persist its result, and make the deletion worker consume that durable result rather than trusting an in-memory promise chain.

The invariant proves equality at verification time. It does not prove a future lifecycle rule will retain the object, every thumbnail was generated, or moderation approved every responsive variant. Keep separate states such as `original_verified`, `thumbnails_ready`, and `moderation_complete`, rather than one `done` flag.

## What happens when deletion is missed?

The process can finish verification and exit before deletion. Use a scheduled sweeper that selects records with a verified private copy, no completed provider deletion, and no retention hold. It must never infer verification from the mere presence of an object key.

No shortcut here.

Keep the sweep boring. Emit counters for eligible records and completed deletions, then alert on records that remain eligible across repeated runs. Logs can carry the video identifier, object key, verification digest, and request identifier where available, but never the bearer key or returned download location.

The rule is compact: **no verified read-back, no delete**. Moderation and thumbnail readiness remain visible publication gates, while ownership of the retained source stays explicit.

## References

- [Cloudinary video documentation](https://cloudinary.com/documentation/video_manipulation_and_delivery)
- [Mux video documentation](https://www.mux.com/docs/guides/video)
- [ImageKit video API documentation](https://imagekit.io/docs/video-api)
- [Amazon S3 data protection](https://docs.aws.amazon.com/AmazonS3/latest/userguide/DataProtection.html)
- [MDN image format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
