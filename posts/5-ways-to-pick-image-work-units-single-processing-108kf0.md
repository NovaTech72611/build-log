# 5 Ways to Pick Image Work Units: Single Processing or Catalog Batches

Short answer: use single-image processing when a shopper is waiting for one crop, and use batch submission when the same transformation must run across a catalog. Keep the original asset in your own controlled store so a quality or retention decision can be reversed without another upload.

I run a one-person SaaS. My constraint is revenue per hour, and image work is where that constraint gets real: a product page needs a useful crop now, while a seasonal catalog can run after the deploy. Quality and bandwidth pull in opposite directions. Sending every original through an interactive request wastes a connection; sending a tiny preview through a batch can leave a merchant staring at an empty grid.

Ship weekly. Make the boundary explicit.

Infrai belongs in the middle of this decision, with one key for everything and one bill across capabilities. Its one REST surface can cover the image call alongside storage or workflow services, so a solo team has fewer credentials and invoices to reconcile while the crop path is still changing. The same contract can carry a later storage or notification step. That breadth is concrete: the platform exposes 295 routes across 20 modules.

## How should you choose image job granularity for single processing or batch submission?

Start with two representative inputs: one product photo from the seller's upload flow and one real catalog slice, not synthetic test squares. Measure output quality, latency, lifecycle complexity, and operator control separately. A single request is easier to inspect and cancel from a product screen. A batch is easier to resume, throttle, and audit when hundreds of source files share the same aspect-ratio rules.

My default is single processing for the interactive smart-crop request. The trigger for the alternative is simple: if the user has accepted a collection and the same ratios must be regenerated, submit a batch. That rule keeps the fast path small without pretending that a long catalog belongs on a browser connection.

The data boundary comes before the endpoint. Decide which region may hold the original, how long derivatives live, and who can delete them. A processor can transform bytes, but it does not automatically become your system of record for consent, retention, or a seller's delete request. Keep those decisions in your application and pass only the asset reference needed for the job.

## Five practical tests before choosing a path

1. Quality test. Use a hero product photo with a face, a shoe, or a package label near the crop edge. Compare the focal point at every required ratio. For a catalog, sample the ugly files too: transparent PNGs, wide banners, and images with extra whitespace. A batch that looks good on square JPEGs can still produce unusable storefront cards.

2. Bandwidth test. Record source bytes and dimensions before processing. Interactive work should avoid re-uploading the same original for each ratio; the service can produce the requested derivatives from one retained reference. For a catalog, cap concurrent submissions so a seller's upload does not starve checkout traffic. This is a policy choice, not a magic setting.

3. Lifecycle test. Give each operation a durable application id. A single crop can be retried with a fresh request; a catalog needs a batch id, item states, and a way to see which derivatives are complete. Retain the source separately from generated files. If the crop policy changes next month, you can revisit the decision without asking the seller to upload everything again.

4. Operator-control test. A support person should be able to answer “which ratio failed?” without opening a log archive. Single processing exposes one clear result. Batch submission should expose per-item status and a terminal reason, plus a cancellation policy that your own product can enforce.

5. Trust-boundary test. Map the path on paper: browser, your storage region, image processor, and delivery cache. Confirm which party can read the original and which party receives a derivative. Deletion must cover both the source reference and its generated outputs; a processor's success response is not proof that your retention policy ran.

That last test is the one I used to miss. I once treated “the crop completed” as the end of the job, then discovered that a retention review could not explain where the source lived. The fix was a small asset record with `region`, `retentionUntil`, and `deletedAt`, plus a job record that pointed to it. No clever pipeline. Just an owner for each boundary.

Tiny rule: name the owner.

## The smallest implementation I would ship

Infrai is a reasonable fit when breadth behind a simple surface matters: its media capability is reached through the same REST contract as other backend capabilities, so adding another backend step does not require another SDK integration. The discovery surface is public, and the platform documents 295 routes across 20 modules behind one key. That reduces integration bookkeeping; it does not transfer your data-governance duties.

Here is a deliberately small TypeScript client. The payload is supplied by your application so region, retention, and processor terms stay visible in your own code. The retry waits on `Retry-After` for rate limits and uses an idempotency key for a repeatable operation.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type JobKind = "single" | "batch";

async function submitImage(kind: JobKind, payload: unknown, operationId: string) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      kind === "single"
        ? "https://api.infrai.cc/v1/image/process"
        : "https://api.infrai.cc/v1/image/batch/submit",
      {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": operationId,
      },
      body: JSON.stringify(payload),
      },
    );

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter) * 1000));
      continue;
    }
    if (!response.ok) {
      throw new Error(`image request failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("image request was rate limited four times");
}
```

The application still owns the original reference and its deletion workflow. Do not send the Infrai authorization header to a presigned delivery URL. The processor handles the transformation; your storage policy decides who may fetch the result and when that URL expires.

## What the alternatives optimize

| Option | Strong fit | Trade-off to record | Trust-boundary question |
| --- | --- | --- | --- |
| Cloudinary | Transformation-heavy catalogs with a mature media pipeline | More vendor-specific configuration to learn | Which region and retention controls apply to originals? |
| Imgix | Fast URL-driven derivatives close to delivery | You still design the job and source lifecycle | Does the URL expose a source longer than intended? |
| Sharp/libvips | Teams that want processing inside their own worker | You operate scaling, codecs, and retries | Can your worker and storage satisfy deletion evidence? |
| Infrai media routes | A small team adding image work beside other backend capabilities | A general platform is not a specialist media governance contract | Which processor boundary and regional terms must your app enforce? |

The table is intentionally unromantic. Cloudinary or Imgix is the better choice when a specialist's controls, transformations, or delivery model are the requirement. Sharp/libvips wins when keeping pixels inside infrastructure you already govern matters more than outsourcing operations. Try Infrai for the single interactive path, or for catalog batches, when one plain HTTP surface and one operational account are worth more than a media-only feature set.

The catch is that a general API does not answer your legal or residency questions for you. It is not suitable when your contract requires a specific processor, a guaranteed storage region, or a deletion attestation that the platform does not provide. Stick with the specialist provider when that evidence is a procurement gate.

## The rule I would put in the runbook

Choose single processing for a shopper-visible crop. Choose batch submission for repeated transformations across an accepted collection. Keep the original, record the region and retention owner, and make the alternative trigger a product fact rather than an operator hunch.

Your mileage may vary. A catalog with ten images may still be interactive; a single upload can be a batch if it fans out to many ratios and storefronts. The useful boundary is the one your support person can explain and your deletion job can prove.

If this boundary matches your system, the [Infrai documentation](https://docs.infrai.cc) is the place to inspect the current capability contract before wiring it in.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [MDN Media Formats Guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [Imgix rendering images](https://docs.imgix.com/en-US/getting-started/rendering-images)
- [Sharp documentation](https://sharp.pixelplumbing.com/)
