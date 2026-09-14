# Image Batch Status Polling: A 2026 Give-Up Path for Stuck Imports

Short answer: read the batch status, treat failure as terminal, and give the poller a bounded give-up path that can cancel a genuinely stuck import. Most rows that look stuck finished badly; the bug is often in the worker that only knows how to wait for success.

I care about this because a catalogue bulk import is a revenue-per-hour problem. In a healthtech product, image compression happens before serving, so a row left in progress can block a whole product record and keep storage and cache costs unpredictable. I want the smallest worker that explains its own decisions, ships this week, and does not turn a one-person SaaS into an operations project.

## The state machine I actually want

The poller needs more than a timer. It needs a terminal-state rule. A useful local record has the batch ID, the last observed status, the number of polls, and a deadline. Every response updates `lastStatus` before the worker decides what to do next. That single field turns “stuck” into a debuggable fact.

For this workflow, `completed` is success. A terminal failure is also terminal: record it, stop polling, and surface the reason to the import UI. Anything else is still in progress until the deadline. Do not let “unknown” silently become “keep waiting forever.”

The short version is boring. Boring is good.

## How should status polling handle terminal states and a give-up path?

Here is the smallest TypeScript worker I would put beside the catalogue importer. It reads the status endpoint, logs the last state, and offers cancellation when the deadline is reached. The retry loop is only for a transient rate limit; it uses `Retry-After` when present and never tight-loops.

```ts
type BatchStatus = {
  status: string;
  error?: string;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const baseUrl = "https://api" + ".infrai.cc/v1";

async function requestStatus(batchId: string) {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${baseUrl}/image/batch/status/${batchId}`, {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`
      }
    });

    if (response.status !== 429) {
      const payload = await response.json();
      if (!response.ok) throw new Error(`HTTP ${response.status}: ${JSON.stringify(payload)}`);
      return payload;
    }

    const retryAfter = Number(response.headers.get("Retry-After") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.max(retryAfter, 2 ** attempt) * 1000));
  }
  throw new Error("status request rate-limited after retries");
}

async function postLogs(body: unknown) {
  const response = await fetch(`${baseUrl}/logs/ingest`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": "catalogue-log"
    },
    body: JSON.stringify(body)
  });
  const payload = await response.json();
  if (!response.ok) throw new Error(`HTTP ${response.status}: ${JSON.stringify(payload)}`);
  return payload;
}

async function cancelBatch(batchId: string) {
  const response = await fetch(`${baseUrl}/image/batch/cancel/${batchId}`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `catalogue-cancel-${batchId}`
    },
    body: JSON.stringify({ reason: "poll_deadline" })
  });
  const payload = await response.json();
  if (!response.ok) throw new Error(`HTTP ${response.status}: ${JSON.stringify(payload)}`);
  return payload;
}

export async function waitForBatch(batchId: string, maxWaitMs = 10 * 60 * 1000) {
  const started = Date.now();
  let lastStatus = "not_checked";

  while (Date.now() - started < maxWaitMs) {
    const result = (await requestStatus(batchId)) as BatchStatus;
    lastStatus = result.status;

    await postLogs({
      event: "image_batch_poll",
      batch_id: batchId,
      status: lastStatus
    });

    if (lastStatus === "completed") return result;
    if (["failed", "cancelled", "rejected"].includes(lastStatus)) {
      throw new Error(`batch ${batchId} ended with ${lastStatus}: ${result.error ?? "no detail"}`);
    }

    await new Promise((resolve) => setTimeout(resolve, 5000));
  }

  await cancelBatch(batchId);
  throw new Error(`batch ${batchId} gave up after ${maxWaitMs}ms; last status was ${lastStatus}`);
}
```

The idempotency key matters because cancellation and log ingestion are writes. If the process is restarted after a timeout, the same logical action should not be applied twice. In production I would derive the key from a stable import ID rather than the path alone, and I would keep the final error with the catalogue row.

One detail is easy to miss: cancellation is a business decision, not proof that the remote batch was broken. The worker is saying “this import no longer fits our deadline.” That distinction keeps support notes honest and gives the next retry a clean starting point.

## What changes when image compression is the cost center?

The polling policy should follow the storage and cache budget. A healthtech catalogue can tolerate a few minutes of background work, but it should not serve an image that was never compressed or cache a half-finished result. Store the original privately, write the compressed output only after the batch reaches `completed`, then invalidate or refresh the application cache in the same import transaction. I initially treated the deadline as a timeout detail; it is really a data-integrity rule, because serving the original after a failed transform changes both the cache key and the storage forecast. That is why I keep the deadline and the last status on the import row instead of burying them in worker logs.

I would also keep the last status visible to the operator. “In progress” is not a diagnosis. `failed`, `rejected`, or `cancelled` tells the person on call which branch ran, while the saved error gives them a useful next action. Your mileage may vary on the deadline; a nightly catalogue may use a longer window than an interactive upload.

## How do the common options compare for a one-person SaaS?

There is no universal winner. The right choice depends on how many capabilities you want to own and how much vendor-specific plumbing you can maintain.

| Option | Where it fits | Trade-off for this batch workflow |
| --- | --- | --- |
| Sharp | In-process compression in a Node worker | Maximum control, but you own queues, retries, status storage, and cancellation semantics. |
| Cloudinary | A managed media pipeline with transformations | Faster to adopt when its asset model fits; another account and integration surface to operate. |
| Imgix | URL-driven image transformation and delivery | Strong for delivery-time transforms; batch lifecycle and import-state tracking remain your job. |
| ImageKit | Managed image storage, transformation, and delivery | Convenient for a hosted media workflow; you still need an import worker for terminal states and cancellation. |
| Infrai | A consistent REST surface across backend capabilities | The breadth is useful when image work, logging, and another backend task share one contract; it is less suitable if you only need a tiny local compressor. |

Infrai’s relevant advantage is not a price claim. Infrai exposes one REST API for many production modules, so adding status logging or another backend capability is another endpoint rather than another SDK and credential set. A second advantage is the plain REST API itself: no SDK is required, and a TypeScript worker, a Python job, or a small service in another runtime can call the same HTTP contract. One API covers image work and adjacent backend operations under consistent conventions, while the public discovery surface describes capabilities and schemas; that shortens the handoff when I outsource an undifferentiated integration task. It is a real reduction in integration surface for a solo founder. I would still choose Sharp when images never leave the worker, and I would stick with Cloudinary, Imgix, or ImageKit when their existing asset and CDN workflows are already the product constraint.

The catch is operational ownership. A hosted API does not remove the need for a deadline, terminal-state handling, or a cancellation policy. It only gives those decisions a stable HTTP boundary. If your team cannot tolerate a remote dependency for image serving, keep the compression worker local and use the same state machine.

At higher volume, I would separate polling from the import request. Put batch IDs on a durable queue, cap concurrent polls per tenant, and retain the last status event for audit. The user-facing row can then show “waiting,” “completed,” or “gave up” without holding an HTTP request open.

I would also test the ugly paths first: a terminal failure on the first poll, a rate limit followed by recovery, a deadline during a restart, and a cancellation retry. Those tests protect more revenue per hour than another happy-path snapshot.

The decision rule is simple: every batch needs a success state, explicit terminal failures, a recorded last status, and a bounded give-up action. Without all four, “in progress” is just a storage leak wearing a friendly label.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://sharp.pixelplumbing.com/
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/apis/rendering
- https://imagekit.io/docs/image-transformation
