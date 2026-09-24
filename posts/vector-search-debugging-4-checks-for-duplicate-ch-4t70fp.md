# Vector Search Debugging: 4 Checks for Duplicate Chunks and Idempotent Ingestion

TL;DR: Repeated passages usually mean ingestion created several stored records for one logical chunk. Give every chunk a deterministic ID derived from stable catalog identity and position, upsert that ID, and reconcile each completed run against an expected manifest. Inspect duplicate IDs before touching similarity thresholds. This keeps index growth tied to catalog growth rather than job count.

For a one-person e-commerce SaaS, that matters. Every hour cleaning an index is an hour not spent shipping. My rule is blunt: outsource undifferentiated vector plumbing if useful, but own the identity scheme and ingestion ledger. They define correctness and index cost.

## How should you debug duplicate chunks in vector search results?

Vector retrieval returns stored records that score well against a query. It does not know that two records represent the same product paragraph. Retrieval-augmented generation relies on retrieved external memory; repeated records in that memory can become repeated context [1].

First, check the source boundary. Count logical catalog documents before chunking, then count chunks per stable product revision. A feed can contain the same SKU twice, or a retry can replay a page after a timeout. Normalize around the catalog key first.

Second, check chunk determinism. The same revision must produce the same ordered chunk sequence on every run. Record the source key, revision, chunk ordinal, and content digest. Those fields answer different questions: ownership, version, position, and changed bytes.

Third, inspect storage by logical ID, not similar text. Two passages can legitimately match, such as a returns sentence shared by many products. Conversely, one logical passage may gain a space after feed cleanup. Text equality is evidence, not identity.

Use four checks:

1. Compare unique source keys with expected product keys.
2. Run the chunker twice on one normalized revision and compare ordered IDs.
3. Verify that each logical ID maps to one current record.
4. Compare the completed-run manifest with live IDs for that revision.

Stop there first. Lowering top-k or deduplicating results hides excess records while stale passages remain eligible for retrieval.

Retries happen.

## The smallest ingestion path I would ship

This implementation makes identity independent of the embedding and batch attempt. It assumes the catalog supplies a stable product key and revision. The storage boundary exposes `upsert`, because retries must replace by ID rather than append.

```ts
import { createHash } from "node:crypto";

type ProductRevision = {
  productKey: string;
  revision: string;
  title: string;
  description: string;
};

type VectorRecord = {
  id: string;
  values: number[];
  metadata: {
    productKey: string;
    revision: string;
    ordinal: number;
    contentDigest: string;
  };
};

interface Embedder {
  embed(texts: string[]): Promise<number[][]>;
}

interface VectorIndex {
  upsert(records: VectorRecord[]): Promise<void>;
}

const digest = (value: string): string =>
  createHash("sha256").update(value, "utf8").digest("hex");

function normalize(text: string): string {
  return text.normalize("NFC").replace(/\s+/g, " ").trim();
}

function chunkProduct(product: ProductRevision): string[] {
  const text = normalize(`${product.title}\n${product.description}`);
  const sentences = text.split(/(?<=[.!?])\s+/);
  const chunks: string[] = [];

  for (let offset = 0; offset < sentences.length; offset += 4) {
    const chunk = sentences.slice(offset, offset + 4).join(" ").trim();
    if (chunk) chunks.push(chunk);
  }
  return chunks;
}

async function ingestProduct(
  product: ProductRevision,
  embedder: Embedder,
  index: VectorIndex,
): Promise<string[]> {
  const chunks = chunkProduct(product);
  const vectors = await embedder.embed(chunks);
  if (vectors.length !== chunks.length) {
    throw new Error("Embedding count does not match chunk count");
  }

  const records = chunks.map((chunk, ordinal): VectorRecord => {
    const logicalKey = `${product.productKey}\u0000${product.revision}\u0000${ordinal}`;
    return {
      id: digest(logicalKey),
      values: vectors[ordinal],
      metadata: {
        productKey: product.productKey,
        revision: product.revision,
        ordinal,
        contentDigest: digest(chunk),
      },
    };
  });

  await index.upsert(records);
  return records.map(({ id }) => id);
}
```

Four sentences is an example policy, not a universal optimum. It is visible and deterministic. A production chunker may use token limits or document structure, but its configuration belongs in the revision contract. Change the policy and the chunk set has changed.

One trap sits in ID design. If the ID includes the content digest, editing a description creates a new record while the old one can survive. That suits immutable history. A live product index usually needs one current slot per logical position, so the example keeps the digest in metadata for diagnosis.

## Retries need a ledger, not optimism

An upsert prevents one ID from multiplying, but it does not prove that a bulk run completed. A process can stop after batch 17, leaving mixed revisions. Keep a run ledger outside the index with the run ID, expected source revisions, expected IDs, and completion state. Mark completion only after every batch acknowledgement and reconciliation.

More bookkeeping buys a clean failure boundary. I would pay it. Weekly shipping depends on rerunning a failed job without creating a cleanup project.

The trade-off is extra state and a second system to reconcile. This approach is a poor fit for an immutable archive where every historical version must remain searchable; content-addressed IDs and explicit version filters are clearer there. It is also unnecessary for a tiny, append-only catalog that is rebuilt atomically from scratch. For a mutable catalog ingested in retryable batches, though, the ledger earns its keep because it separates a completed publication from a pile of acknowledged writes.

Do not delete the previous revision first. Build the new revision under its own IDs, reconcile it, switch the active revision used by retrieval, and then remove superseded IDs. That ordering prevents a partial ingestion from producing empty search. It also makes rollback a metadata decision instead of another embedding run.

Track four counts: source revisions read, chunks expected, records acknowledged, and live IDs observed. One total is weak evidence. If 10,000 writes were acknowledged but only 9,800 unique logical IDs were expected, investigate duplicate input or inconsistent chunking. Those numbers illustrate the comparison; they are not a benchmark.

## Debug the stored set before the ranked set

Start with one affected product key. Rebuild its expected manifest using the exact normalization and chunk configuration attached to the run. Compare expected IDs, acknowledged IDs, and live IDs. Missing IDs indicate an incomplete path. Extra live IDs point to stale revisions, a changed identity rule, or nondeterministic IDs. Repeated content under different valid product keys is a catalog modeling question.

Only after those sets agree should ranking enter the investigation. Capture the retrieved ID, score, product key, revision, ordinal, and digest. Different IDs with matching digests suggest shared boilerplate or duplicated source data. A repeated response ID points downstream of storage identity. Old revisions point to the active-revision filter or cleanup transition.

This order protects the revenue-per-hour calculation. Storage correctness is finite and testable. Similarity tuning against a dirty corpus is open-ended.

For coverage, ingest the same fixture twice and assert that the live ID set does not grow. Change one revision, ingest again, switch the active revision, and assert that retrieval excludes the old one. Finally, interrupt a multi-batch run and verify that the active revision stays unchanged. These tests cover retry, update, and partial failure without binding the system to one database.

## What I would change at catalog scale

At larger volume, I would keep the identity contract and change the mechanics. Batch embeddings within documented limits. Bound concurrency. Persist checkpoints after acknowledged batches. Partition reconciliation by source key range so verification does not load the whole manifest into memory. None of those changes should alter a chunk ID.

Separate two cost questions. How many current chunks does useful retrieval require? How many accidental records did retries and stale revisions create? Better chunk boundaries may change the first number. Deterministic upserts and reconciliation drive the second toward zero. Mixing them makes an architecture review vague.

Result-side collapse can still improve presentation when several passages from one product are relevant. Treat it as ranking policy, not repair. Preserve metadata to cap passages per product after retrieval while keeping raw results observable.

That distinction is easy to miss.

**A bulk job may run many times, but each logical catalog chunk gets one current identity.** Put that invariant in a test, a ledger, and the write interface. Everything else can change when scale or economics change.

## Further reading

1. Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
