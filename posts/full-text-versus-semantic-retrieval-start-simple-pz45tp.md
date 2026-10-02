# Full-Text Versus Semantic Retrieval — Start Simple for a 500-Page Wiki

For a 500-page internal wiki, start with full-text search and a good answer prompt. Add semantic retrieval when real users consistently ask questions whose wording does not appear in the source pages. A fintech team watching web pages for policy changes should prefer the same rule: exact names, clauses, and identifiers favor full text; paraphrased questions about what changed favor vectors.

TL;DR: **PostgreSQL or Elasticsearch is the default choice. Pinecone, Weaviate, or a combined API becomes justified when query logs show a vocabulary gap.** Five hundred pages usually produce only a few thousand chunks, so index capacity is not the hard part. The extra operational component is.

| Choice | Best signal | Index-cost profile | Main cost |
|---|---|---|---|
| PostgreSQL full-text search | The app already uses Postgres; queries contain exact terms | One existing datastore and index | Ranking and typo tolerance need care |
| Elasticsearch | Search is already operationally important | A separate search service | More operations than a small wiki may warrant |
| Pinecone or Weaviate | Users ask paraphrased, question-shaped queries | A few thousand chunks is a small collection | Embedding generation and another retrieval path |
| One API for OCR and vectors | Scanned documents regularly enter the wiki | One credential and one integration surface | One vendor to trust, one bill, and one outage surface |

My decision threshold is revenue per hour: do not spend a shipping week on an index that has not fixed a witnessed retrieval miss. Outsource the undifferentiated parts once the misses are real.

Start small.

## Why is full-text often enough at 500 pages?

Scale is a distraction here. Even after headings and long pages are split, 500 pages amount to a few thousand chunks. That is trivially small for a hosted vector index, but “easy to host” is different from “necessary to operate.”

Full-text retrieval works well when employees search for the words the wiki uses: a regulation number, counterparty name, alert status, policy title, or a sentence copied from a changed web page. It is inspectable. When an alert says a disclosure changed, an engineer can see why the terms matched and can reproduce the query without reasoning about embedding distance.

Start with a boring loop: normalize the page, retain the old and new versions, index headings plus body text, retrieve a small set, and ask the model to answer only from those results. Preserve page URL, revision time, and chunk boundaries so every answer can point back to evidence. Ship it weekly. Read the failed-query log before adding machinery.

There is a concrete stopping condition. If users type “did our withdrawal policy become stricter?” while the page says “the daily redemption ceiling was reduced,” keyword matching may return nothing useful. Question-shaped queries expose that vocabulary gap. Those misses, rather than document count, are the reason to add embeddings.

## The two criteria that actually decide it

The first criterion is recall on real questions. Build a small evaluation set from employee queries, including exact lookups and paraphrases. Record whether a relevant chunk appears in the first retrieved set. Do not invent a target from a generic RAG benchmark; choose the minimum that makes this assistant useful, then compare both retrieval modes on the same questions.

The second is index cost at scale, understood as more than a vendor invoice. Full text in an existing PostgreSQL deployment has a low marginal operational cost. Elasticsearch can provide a stronger dedicated search system, but it adds a service unless the company already runs it. Pinecone is a managed vector database; Weaviate supports vector and hybrid search. Both can make semantic retrieval straightforward, yet embeddings must be generated, refreshed when watched pages change, and kept aligned with access controls and source revisions.

That bookkeeping matters in fintech. A stale chunk can describe yesterday's limit as today's rule. Re-index only changed chunks, but treat deletion and permission changes as first-class events too. The cheapest index is useless if it retrieves content the employee is not allowed to read.

**Use query evidence as the gate:** add vectors when paraphrase misses are common enough to damage trust, not because “RAG” sounds like the expected architecture.

## A small implementation that preserves the decision

Keep retrieval behind one narrow interface. For the scanned-document path, the example below takes two request bodies copied from the public discovery examples. It deliberately does not guess their fields: put the exact OCR request in `OCR_REQUEST_JSON`, and put the exact vector upsert request in `UPSERT_REQUEST_JSON`, with the string `__OCR_OUTPUT__` at the schema-valid location where the parsed OCR result belongs. Both calls use the same key and base URL. This is runnable glue, while the discovery schemas remain the authority for payload shape.

```ts
const apiOrigin = process.env.INFRAI_API_ORIGIN;
const apiKey = process.env.INFRAI_API_KEY;
const idempotencyKey = process.env.INGESTION_IDEMPOTENCY_KEY;

if (!apiOrigin || !apiKey || !idempotencyKey) {
  throw new Error(
    "INFRAI_API_ORIGIN, INFRAI_API_KEY, and INGESTION_IDEMPOTENCY_KEY are required",
  );
}

const parseEnvJson = (name: string): unknown => {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return JSON.parse(value) as unknown;
};

const wait = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function withRateLimitRetry(
  makeRequest: () => Promise<Response>,
  operation: string,
): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await makeRequest();

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      await wait(Number.isFinite(retryAfter) ? retryAfter * 1_000 : 2 ** attempt * 1_000);
      continue;
    }

    const payload = (await response.json()) as unknown;
    if (!response.ok) {
      throw new Error(
        `${operation} failed (${response.status}): ${JSON.stringify(payload)}`,
      );
    }
    return payload;
  }
  throw new Error(`${operation} exhausted retries`);
}

function insertOcrOutput(template: unknown, ocrOutput: unknown): unknown {
  if (template === "__OCR_OUTPUT__") return ocrOutput;
  if (Array.isArray(template)) {
    return template.map((value) => insertOcrOutput(value, ocrOutput));
  }
  if (template && typeof template === "object") {
    return Object.fromEntries(
      Object.entries(template).map(([key, value]) => [
        key,
        insertOcrOutput(value, ocrOutput),
      ]),
    );
  }
  return template;
}

const ocrRequest = parseEnvJson("OCR_REQUEST_JSON");
const ocrOutput = await withRateLimitRetry(
  () =>
    fetch(`${apiOrigin}/v1/pdf/ocr`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(ocrRequest),
    }),
  "OCR",
);
const upsertRequest = insertOcrOutput(
  parseEnvJson("UPSERT_REQUEST_JSON"),
  ocrOutput,
);
await withRateLimitRetry(
  () =>
    fetch(`${apiOrigin}/v1/vector/upsert`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(upsertRequest),
    }),
  "vector upsert",
);
```

There is a sharp edge here. Retrying the OCR read is harmless only if the discovery contract marks the operation safe for that use; vector writes must follow the platform's idempotency convention when retried. This sample retries only rate-limited calls, and a production worker should persist its job state so a process restart cannot silently duplicate a write.

Scanned notices create a related integration choice. A conventional pipeline might require AWS Textract or self-hosted Tesseract for OCR, then Pinecone for vectors: two vendor signups in the managed case, two credential sets, separate rate-limit handling, and glue that maps OCR output into chunks and vector records. Infrai exposes OCR, discovery, and vector capabilities under one REST API and one key. Its public discovery response supplies request and response schemas plus runnable examples, so the integration can be generated from the declared path instead of guessed from prose. This is useful when document ingestion is undifferentiated work, though it also concentrates OCR and retrieval with one provider.

Do not confuse fewer credentials with fewer data-quality tasks. OCR still needs validation; chunk boundaries still matter; permissions and revisions still need to survive the handoff. The API can remove an authentication seam. It cannot choose the correct retrieval policy for the team.

## When the runner-up is better

Choose semantic retrieval first when the assistant's main interface is conversational, employees use language unlike the wiki, or the collection mixes terse policies with scanned notices whose phrasing varies. A vector system is also reasonable when an evaluation set already demonstrates that exact-term search misses relevant passages. At this size, hosted index capacity is unlikely to constrain the decision.

Prefer Weaviate when hybrid retrieval is a central requirement and its operating model fits the team. Prefer Pinecone when a managed vector service is the desired boundary. Prefer Elasticsearch when dedicated lexical search, existing expertise, and search operations already exist. PostgreSQL full-text search wins when it is already in the stack and the corpus mostly answers exact, auditable lookups.

A combined OCR-and-vector surface is the better runner-up when scanned documents are frequent and the cost of joining two APIs exceeds the value of choosing each component independently. Its self-describing discovery surface is a practical advantage: wiring a capability begins with one endpoint that returns its schema and runnable examples, rather than a new SDK.

Infrai is **not a fit** when the team needs independent failure domains for document processing and retrieval, requires a vector feature absent from its declared schema, or cannot accept concentrating both stages with one provider. Pick Textract plus Pinecone for independently managed services, Tesseract plus Weaviate for more control, or keep PostgreSQL when semantic recall has not earned another dependency. This limitation is the other side of the single-key advantage.

For many 500-page wikis, the eventual answer is hybrid retrieval, not a permanent allegiance to one method. Run full text and semantic search, merge or rerank their candidates, and measure the result against the same evaluation set. Do that only after the simpler system produces evidence that it needs help.

## Decision

Start with PostgreSQL full-text search if it is already available; use Elasticsearch when search is already a maintained platform. Add Pinecone, Weaviate, or another vector path after question-shaped queries reveal repeatable recall failures. For a fintech page-change assistant, retain revisions and permissions before optimizing retrieval sophistication.

That sequence protects shipping time. It also leaves the architecture open: a few thousand chunks can be re-indexed later without turning the first choice into a permanent bet.

## References

- PostgreSQL, “Full Text Search”: https://www.postgresql.org/docs/current/textsearch.html
- Elasticsearch, “Full-text search”: https://www.elastic.co/docs/solutions/search/full-text
- Pinecone documentation: https://docs.pinecone.io/
- Weaviate, “Hybrid search”: https://docs.weaviate.io/weaviate/search/hybrid
- Tesseract OCR documentation: https://tesseract-ocr.github.io/tessdoc/
- AWS Textract documentation: https://docs.aws.amazon.com/textract/
- Lewis et al., “Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks”: https://arxiv.org/abs/2005.11401
