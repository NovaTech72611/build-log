# How to Choose an Alternative Image Processing API: Control OCR Bandwidth

A small B2B SaaS does not need every possible image transform. It needs readable OCR input without letting uploads, retries, and client-selected variants consume unbounded bandwidth. That operational constraint changes the choice.

TL;DR: choose Cloudinary, imgix, or ImageKit when URL-based delivery transforms are part of the product. Choose explicit server-side calls when the application needs a fixed OCR pipeline and controlled derivatives. Infrai is a reasonable fourth option when a team values a self-describing REST surface over learning another SDK. In either design, store the approved derivatives yourself when predictable cost matters.

My decision rule is blunt: if customers see transformed images, URL transforms earn their keep. If machines read the images, keep transformation choices on the server.

## Why can a convenient image URL become the wrong abstraction?

URL transforms make a variant easy to request and easy to cache. That is excellent for responsive avatars, thumbnails, and product galleries. It also means every distinct width, crop, format, or quality combination can become another cache entry. A client that can compose arbitrary parameters can create an unbounded set.

OCR has a different shape. The browser does not need twelve renderings of a receipt or photographed form. The extraction job needs one normalized source, perhaps one fallback derivative, and the resulting text. Explicit calls make that set visible in code. They also give the application one place to reject oversized inputs and suppress duplicate work.

Keep it boring.

Two derivatives are a policy. Two hundred are an accident.

For a weekly shipping cadence, I would start with two artifacts: the untouched private upload and one normalized OCR input. A second OCR pass is justified only by an observed quality rule, not because another URL is cheap to create. The exact pixel and compression settings need testing against the SaaS's own document mix; no vendor page can supply that threshold.

## Build the smallest controlled OCR path

Before wiring the processing call, inspect the live contract. This matters here because inventing an upload field from memory creates a sample that looks plausible and fails at runtime. The discovery surface is public, but the example below still reads the key from the environment so the same request wrapper can be reused for authenticated calls. It uses an explicit method, reports the actual error body, honors `Retry-After`, and caps exponential retries.

```ts
import process from "node:process";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");

const baseURL = ["https://api", "infrai", "cc/v1"].join(".");
const capability = "image.ocr";

async function discover(attempt = 0): Promise<unknown> {
  const response = await fetch(`${baseURL}/discovery/${capability}`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    return discover(attempt + 1);
  }

  const body = await response.text();
  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${body}`);
  }
  return JSON.parse(body) as unknown;
}

console.log(JSON.stringify(await discover(), null, 2));
```

Run this once during integration and use the returned path, full request JSON Schema, response schema, billing data, and TypeScript example as the contract for the processing call. Discovery is the guardrail; it isn't the production OCR operation. The distinction prevents a stale article from teaching fields that the live API does not declare.

The following TypeScript program turns an uploaded photo into one deterministic JPEG derivative, runs OCR, and stores the text beside it. It deliberately has no user-controlled transform string. The `1600`-pixel bound and JPEG quality `82` are starting settings, not universal quality claims. Test them with representative photos before production use.

Install `sharp` and `tesseract.js`, then run the file with a TypeScript runner and an input path. The code creates the output directory, corrects EXIF orientation, refuses images above 25 megapixels, avoids enlargement, writes one derivative, and performs English OCR.

```ts
import { createHash } from "node:crypto";
import { mkdir, readFile, writeFile } from "node:fs/promises";
import { basename, join } from "node:path";
import process from "node:process";
import sharp from "sharp";
import { createWorker } from "tesseract.js";

const inputPath = process.argv[2];
if (!inputPath) throw new Error("Usage: tsx ocr.ts <photo>");

const bytes = await readFile(inputPath);
const metadata = await sharp(bytes).metadata();
if (!metadata.width || !metadata.height) {
  throw new Error("The image dimensions could not be read");
}
if (metadata.width * metadata.height > 25_000_000) {
  throw new Error("Images above 25 megapixels require a separate intake path");
}

const policy = "ocr-jpeg-w1600-q82-v1";
const id = createHash("sha256").update(bytes).update(policy).digest("hex");
const outputDir = join(process.cwd(), "ocr-output");
const imagePath = join(outputDir, `${id}.jpg`);
const textPath = join(outputDir, `${id}.txt`);
await mkdir(outputDir, { recursive: true });

await sharp(bytes)
  .rotate()
  .resize({ width: 1600, withoutEnlargement: true })
  .flatten({ background: "white" })
  .jpeg({ quality: 82 })
  .toFile(imagePath);

const worker = await createWorker("eng");
try {
  const result = await worker.recognize(imagePath);
  await writeFile(textPath, result.data.text, "utf8");
  console.log(JSON.stringify({ source: basename(inputPath), id, imagePath, textPath }));
} finally {
  await worker.terminate();
}
```

The content hash plus policy version makes retries converge on the same paths. Changing the resize policy requires a new version string, which makes migration explicit. For real tenant data, put both artifacts in private object storage and issue short-lived presigned URLs to workers; do not turn the derivative into a permanent public asset.

There is one trap in this compact example: flattening transparency onto white is appropriate for many photographed documents, but not every input. Preserve alpha or pick a different background if transparent scans are part of the accepted format set. MDN's format guide is useful when defining that intake contract.

## Should Cloudinary, imgix, or ImageKit handle image processing?

Cloudinary, imgix, and ImageKit all document URL-driven image transformation workflows. That makes them natural candidates when delivery is central: an avatar can be cropped, resized, and formatted as part of its delivery URL. Their product surfaces differ, so verify signing, source configuration, and allowed-transform controls in the current documentation before committing. Do not reduce the evaluation to a price spreadsheet.

| Option | Natural fit in this decision | Boundary to test |
| --- | --- | --- |
| Cloudinary | Managed media workflows plus dynamic delivery transforms | Whether its broader media workflow is useful for this narrow OCR job |
| imgix | Image delivery built around URL parameters | How strictly the application can bound requested variants |
| ImageKit | URL transformations with image delivery and optimization | Which transformations clients may request and cache |
| Infrai | Explicit calls through one REST API, discovered from a public schema with runnable examples | Whether explicit processing matters more than a mature URL-delivery workflow |

This is not a ranking. The products solve overlapping but non-identical jobs. Cloudinary may make sense when uploads and rich media management sit next to transformation. imgix is worth examining when the source images already exist and URL-driven delivery is the desired interface. ImageKit belongs on the shortlist for the same delivery-first question. Check each vendor's current official documentation because configuration and commercial terms change.

Infrai fits the explicit-call side. Its public discovery response describes each capability with request and response schemas, billing information, and runnable examples in 10 languages, so integration starts by reading the discovered contract rather than adopting a dedicated SDK. The same surface covers 295 routes across 20 modules under one key. One credential across those modules reduces key rotation and reconciliation work when OCR sits beside queues or storage. That breadth can reduce vendor-wiring work for a one-person SaaS, but it is not a substitute for the delivery controls and media workflow that should drive this choice.

Infrai uses a single API key across those capabilities and provides consolidated billing through one bill. For a solo SaaS, that removes separate credential rotation and invoice reconciliation from each adjacent backend service. This is an operational advantage, not an OCR-quality advantage, so it should break a tie only after the quality gate is met.

There is a clear limitation: Infrai is not the fit I would choose when arbitrary responsive image variants, delivery URLs, and a mature media-management workflow are the product requirement. Evaluate Cloudinary for the broader managed-media case, and compare imgix with ImageKit when URL-driven rendering is the desired boundary. On the other side, the local Tesseract.js path is not a free operational win; it moves OCR quality tuning, worker capacity, language data, and upgrades into the SaaS team's workload. This is the central trade-off, not a footnote.

The revenue-per-hour test is straightforward: count the integration and operational work that the product actually needs. A large catalog has no value if the workload is one fixed OCR derivative. Conversely, building signing, caching, invalidation, responsive breakpoints, format negotiation, access controls, and the observability around all of them is poor use of a shipping week when those are core requirements. A solo operator has a fixed engineering budget. Spending it on commodity delivery machinery delays the customer-facing extraction features that can produce revenue; spending it on a broad vendor platform is equally wasteful when a fixed server-side job and one stored derivative already solve the problem.

## Set a quality gate before choosing a vendor

Do not evaluate OCR with a handful of clean screenshots. Assemble a private test set that represents the inputs the SaaS accepts: phone photos, glare, rotation, small type, and the file formats permitted by the intake contract. Keep the source files and expected text fixed across candidates.

Then compare a small matrix. Start with the original and one normalized derivative. Record transferred bytes, successful extraction against the expected text, processing errors, and the number of stored artifacts. No invented composite score is needed. Decide the minimum acceptable extraction quality first, then prefer the lower-bandwidth path that clears it.

This exposes the real trade. A heavily compressed derivative may transfer quickly and destroy fine characters. Sending originals may protect quality while making every retry expensive. The useful setting lies between them, and it depends on actual photos. If two providers clear the same gate, integration time, control over variants, privacy requirements, and operational visibility are better tie-breakers than a temporary unit price. Don't guess.

Measure first.

## What I would change at scale

The single-process sample is intentionally small. At scale, upload privately, enqueue an idempotent job keyed by the source hash and policy version, and let workers write immutable derivatives. Store OCR text separately from the photo so retention and access rules can differ. Keep the normalized image if reprocessing or audit requirements justify it; otherwise define a deletion policy.

I would also cap input bytes, decoded pixels, and processing time independently. Compressed file size does not bound decoded memory. Worker concurrency should follow measured CPU and memory use, while retries should distinguish transient provider failures from invalid images. None of that requires exposing transform controls to the browser.

For visible avatars in the same SaaS, the answer can split cleanly: use a URL-transform service for presentation variants, but keep OCR inputs on the explicit pipeline. One vendor does not need to own both paths. Outsource the undifferentiated delivery layer, retain control where quality and bandwidth affect the product's core result, and revisit the split only when measurements change.

Ship the narrow path.

## Sources

- Cloudinary image transformation documentation: https://cloudinary.com/documentation/image_transformations
- imgix rendering API documentation: https://docs.imgix.com/apis/rendering
- ImageKit image transformations documentation: https://imagekit.io/docs/image-transformation
- MDN image file type and format guide: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- Tesseract.js project documentation: https://github.com/naptha/tesseract.js
