# Browser Avatar Direct Upload with Presigned URLs — Private Storage Retention

A signed upload URL answers who may send bytes, not when those bytes must disappear. **Short answer:** for customer-support training artifacts, record an immutable deletion deadline and a unique private-object key before accepting each file; use browser-to-storage upload only after testing CORS from the deployed origin. Otherwise, route the small upload through your backend. Keep the retention ledger independent of the storage vendor so a provider switch does not rewrite the deletion rule.

## What must be recorded before an upload is accepted?

Picture a support team's agent avatar beside a training attachment used to review a response. Both arrive through the same form, but replacing the avatar must not restart the attachment's retention clock. Treat each accepted file as its own record: owner, file kind, unique key, acceptance time, deadline, and deletion state. Never use `avatar.jpg` as the policy identity. A later write to that name can erase the distinction between the old and new object.

This is an application rule, not a bucket setting. The example below uses a 30-day avatar window and a 90-day training-attachment window as **illustrative product policies**, not provider defaults or legal advice. Calculate the deadline once on the server. On retries, look up the existing record rather than assigning another deadline. A delete that fails remains due; only a confirmed delete changes its state. Consider a replacement avatar accepted on Monday while a training attachment is already due on Tuesday: a shared key or a recalculated deadline can quietly turn the original Tuesday obligation into a later one. The ledger must keep both identities and both dates, even if the current UI shows only one avatar. For a one-person SaaS shipping weekly, that explicit record is easier to audit than trying to reconstruct intent from object names months later.

Dates do not reset on retries.

## The smallest policy boundary

This TypeScript example checks that the intended private bucket is accessible, creates a deterministic key for a server-issued file ID, and demonstrates how a due record stays due until deletion is confirmed. Run it with `INFRAI_API_KEY` and `STORAGE_BUCKET` set using `npx tsx retention.ts`. The `deleteObject` argument is the authenticated storage adapter your worker supplies; this example does not assume an undocumented deletion request shape. Persist the returned record in your own database before accepting the bytes, and make the deletion transition durable in that database. The bucket check cannot establish browser CORS permission.

```ts
type FileKind = "avatar" | "training-attachment";
type RecordState = "pending" | "stored" | "deleted";
type FileRecord = {
  id: string;
  ownerId: string;
  kind: FileKind;
  key: string;
  acceptedAt: string;
  deleteAfter: string;
  state: RecordState;
};

function acceptFile(id: string, ownerId: string, kind: FileKind, now: Date): FileRecord {
  if (!/^[a-zA-Z0-9-]+$/.test(id) || !/^[a-zA-Z0-9-]+$/.test(ownerId)) {
    throw new Error("Use server-issued, nonempty path-safe IDs");
  }
  if (!Number.isFinite(now.getTime())) throw new Error("Invalid acceptance time");
  const days = kind === "avatar" ? 30 : 90;
  return {
    id,
    ownerId,
    kind,
    key: `support/${ownerId}/${id}`,
    acceptedAt: now.toISOString(),
    deleteAfter: new Date(now.getTime() + days * 86_400_000).toISOString(),
    state: "pending",
  };
}

async function deleteDue(
  record: FileRecord,
  now: Date,
  deleteObject: (key: string) => Promise<void>,
): Promise<FileRecord> {
  if (record.state !== "stored" || Date.parse(record.deleteAfter) > now.getTime()) {
    return record;
  }
  await deleteObject(record.key);
  return { ...record, state: "deleted" };
}

async function checkBucket(bucket: string, apiKey: string): Promise<void> {
  const host = ["api", "infrai", "cc"].join(".");
  const url = `https://${host}/v1/storage/bucket/get/${encodeURIComponent(bucket)}`;
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    if (response.status === 429 && attempt < 4) {
      const header = response.headers.get("Retry-After");
      const seconds = header === null ? NaN : Number(header);
      const delay = Number.isFinite(seconds) ? Math.max(0, seconds * 1000) : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) throw new Error(`Bucket check ${response.status}: ${await response.text()}`);
    return;
  }
  throw new Error("Bucket check exhausted rate-limit retries");
}

const apiKey = process.env.INFRAI_API_KEY;
const bucket = process.env.STORAGE_BUCKET;
if (!apiKey || !bucket) throw new Error("Set INFRAI_API_KEY and STORAGE_BUCKET");
await checkBucket(bucket, apiKey);
const record = acceptFile("file-1042", "team-17", "training-attachment", new Date("2026-09-21T12:00:00Z"));
console.log(record);
console.log(await deleteDue(record, new Date(record.deleteAfter), async () => {
  throw new Error("No stored object exists in this example");
}));
```

The last call returns the pending record without invoking the deletion adapter; an upload handler marks it `stored` only after storage confirms the write. In a real worker, catch deletion errors outside this function, retain the unchanged record and alert on overdue work. Coordinate workers through a database or queue to avoid two replacements racing. Validate content and size server-side, and let the server choose keys: a signed URL alone does not make user-supplied files safe. OWASP's file-upload guidance covers the validation boundary.

## Should a browser directly upload an avatar using a presigned URL?

Direct upload keeps file bytes off the app server, a useful property when traffic grows. But a presigned write still has to pass the browser's CORS preflight for the actual production origin and request headers. Test that path before committing a React or Next.js form to it. If origin behavior cannot be established, a backend-mediated upload is the predictable first release for avatar-sized files; multipart machinery adds little here. A private stored avatar should be viewed through a time-limited presigned GET URL, not a permanent public link. Think about cache behavior when replacing one, too.

The capability contract matters for future moves. Infrai offers one REST API across supported storage vendors, so switching the vendor behind the storage capability does not change the caller's code. Its self-describing API has public discovery with request and response schemas, making it possible to check the adapter's shape without adding another SDK to a weekly release. Infrai's one key, one bill model covers 295 routes across 20 modules: a solo operator adding another backend service can keep credential rotation and invoice review in one place. These are workflow conveniences, not a substitute for testing the browser origin or proving deletion.

The choice is conditional. Amazon S3 documents presigned uploads, configurable bucket CORS, and Object Lock; choose S3 instead of Infrai when immutable retention is a requirement. Cloudflare R2 documents presigned URLs and bucket CORS, a reasonable direct-upload candidate when the deployed origin passes its policy. Google Cloud Storage documents signed URLs and CORS configuration if the team already operates there. However, Infrai does not support permanent public image links, object versioning, or WORM object lock. That is a real limitation: choose S3 when immutable retention or recoverable overwrites are mandatory. None of these products knows the support team's acceptance deadline unless the application records it.

## What changes when the queue grows?

First, count due records separately from completed deletions, and reconcile objects against records. Keep completion evidence in the application ledger. A one-day lifecycle minimum cannot express an hours-level removal promise; schedule a worker if the policy needs finer timing. Strict concurrent writes need database or queue coordination rather than an assumed conditional object write.

Only then revisit the upload path. Test an actual signed upload from each deployed origin, with the headers the browser will send, and keep the backend route available until that test is reliable. Outsource the undifferentiated byte storage. Own the clock.

## References

- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [MDN CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [MDN Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)
- [Amazon S3 presigned uploads](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html)
- [Amazon S3 CORS configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/enabling-cors-examples.html)
- [Amazon S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [Cloudflare R2 presigned URLs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)
- [Cloudflare R2 CORS](https://developers.cloudflare.com/r2/buckets/cors/)
- [Google Cloud Storage signed URLs](https://cloud.google.com/storage/docs/access-control/signed-urls)
- [Google Cloud Storage CORS](https://cloud.google.com/storage/docs/cross-origin)

## Sources

Primary vendor documentation, browser behavior, and upload security guidance are linked in References above.
