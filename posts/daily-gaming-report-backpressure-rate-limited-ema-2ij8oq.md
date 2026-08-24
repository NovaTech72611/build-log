# Daily Gaming Report Backpressure: Rate-Limited Email Queue with Public HTTPS Consumers

A gaming platform's daily report batch has two clocks: the scheduler releases work all at once, while the email provider admits it gradually. Put both clocks in one handler and a provider limit becomes an application outage budget.

Short answer: use cron for the daily kickoff, publish one stable job per report to a queue, and enforce the sending rate in idempotent workers or a public HTTPS push consumer.

That choice keeps the fast path fast. The scheduled request can finish after publishing, while delivery proceeds at the rate the provider accepts. For a one-person SaaS that ships weekly, I care about this boundary because report generation is product work; babysitting a long-lived scheduler isn't. The hard part remains mine: deciding which gaming account receives which report and proving that a retry won't send it twice.

## How should a daily report email queue use a public HTTPS push endpoint?

Treat the system as a pair of budgets. The cron task has a hard execution ceiling of 900 seconds. The consumer has a rate budget imposed by the email provider. A queue transfers work between those budgets without forcing either side to run at the other's speed.

The daily kickoff should calculate a stable identity such as `gaming-report:{accountId}:{reportDate}` and publish a small message containing identifiers, not a rendered report. The queue message limit is 256KB, delayed delivery is capped at seven days, and retention is at most 30 days. An acknowledged message is deleted. Those properties suit dispatch, but they do not make the queue a replayable event ledger.

Then pick the consumer according to network ownership. Push is appropriate when the service already exposes a public HTTPS endpoint. A private internal consumer cannot receive push subscriptions, so use pull workers when the sending process must remain private. This is a deployment choice, not a delivery guarantee: standard queues are at-least-once, and either consumer must be idempotent.

There is no native debounce or throttle primitive. The worker therefore owns concurrency, provider allowances, and retry timing. On a `429`, honor `Retry-After` when present or apply exponential backoff. Keep the same report identity on every attempt. FIFO deduplication covers only a five-minute window, which is too narrow to stand in for a durable uniqueness constraint.

Short bursts are fine.

A common mistake is to confuse queue depth with permission to send. Imagine that 18,400 player-economy summaries become ready at midnight and the provider slows admission. Ten workers can claim ten jobs, but they still share one provider budget; increasing worker count without a shared limiter only creates more `429` responses. The durable record should move through a claim state before the provider call and a sent state afterward, keyed by account and report date. A duplicate delivery that finds `sent` is acknowledged without another email. If strict per-account ordering is required, partition work with that constraint in mind, though idempotency remains mandatory because ordering does not eliminate duplicate delivery.

## Publish through a schema-checked TypeScript boundary

The smallest useful integration does two things: read the live capability description, then call the path declared by discovery. The program below makes a real `POST /v1/queue/publish` request, uses a stable idempotency key, handles `429`, and surfaces other response bodies. It accepts the publish body as JSON because discovery is the authority for its current schema; that keeps the example runnable without inventing queue fields that are not established here.

Before running it, open the public `queue.publish` discovery response, use its request schema and TypeScript example to create a valid body, and place that JSON in `INFRAI_QUEUE_PUBLISH_BODY`. The key stays in the environment.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const rawBody = process.env.INFRAI_QUEUE_PUBLISH_BODY;
const reportId = process.env.REPORT_ID;

if (!apiKey || !baseUrl || !rawBody || !reportId) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_BASE_URL, INFRAI_QUEUE_PUBLISH_BODY, and REPORT_ID",
  );
}

const normalizedBaseUrl = baseUrl.replace(/\/$/, "");
const capabilityUrl = `${normalizedBaseUrl}/discovery/queue.publish`;
const publishUrl = `${normalizedBaseUrl}/queue/publish`;

type Capability = {
  method: string;
  path: string;
  available: boolean;
  params: unknown;
};

async function sleep(milliseconds: number): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, milliseconds));
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return Math.min(30_000, 1_000 * 2 ** attempt);
}

async function publish(): Promise<unknown> {
  const discovery = await fetch(capabilityUrl, { method: "GET" });
  if (!discovery.ok) {
    throw new Error(
      `Discovery rejected ${discovery.status}: ${await discovery.text()}`,
    );
  }

  const capability = (await discovery.json()) as Capability;
  if (
    capability.method !== "POST" ||
    capability.path !== "/v1/queue/publish" ||
    !capability.available
  ) {
    throw new Error("queue.publish discovery metadata is unexpected");
  }

  const body = JSON.parse(rawBody) as unknown;
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(publishUrl, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": reportId,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429) {
      await sleep(retryDelay(response, attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(
        `Publish rejected ${response.status}: ${await response.text()}`,
      );
    }
    return response.json();
  }
  throw new Error("Publish remained rate limited after five attempts");
}

publish()
  .then((result) => process.stdout.write(`${JSON.stringify(result)}\n`))
  .catch((error: unknown) => {
    process.stderr.write(`${String(error)}\n`);
    process.exitCode = 1;
  });
```

Infrai fits this boundary when a small team wants scheduling and queueing through plain HTTP. Its unauthenticated discovery surface returns the request schema, response schema, billing information, and runnable examples for a capability; documented capabilities have examples in ten languages. That makes the integration self-describing rather than SDK-dependent. Infrai also puts 295 routes across 20 modules behind one API key and one bill, so using its cron and queue capabilities doesn't require collecting separate vendor keys or reconciling separate invoices for the handoff at month-end.

Don't mistake API consolidation for application correctness. The platform convention supports idempotency keys, but the email side effect still needs the consumer's durable report record and, where the email provider supports it, the same stable provider idempotency key. There is an unavoidable uncertainty window if a provider accepts a message and gives no idempotent way to repeat the request before the local `sent` update. I'm not sure any generic queue choice can erase that ambiguity; the provider contract resolves it.

## Put the side-effect ledger before the retry loop

The database, not queue history, should answer “was this report sent?” Start each delivery with an atomic insert or compare-and-set on the stable report ID. If the record is already `sent`, acknowledge. If another worker owns a live claim, retry later. Only the claim owner may call the email provider.

This ordering matters after process interruption. A read followed by a separate insert permits two workers to observe absence and send. An atomic uniqueness constraint closes that race. After a `429`, release or extend the claim according to the lease design and schedule the same identity again with backoff. After success, mark it sent, then acknowledge the queue message.

No magic.

Cron history is useful for operations but is not this ledger. Paused cron schedules do not backfill missed triggers, execution timing can jitter by seconds, nonstandard expressions such as `L` are unavailable, and only the first 4KB of run output is retained. Reconcile expected daily report IDs against the sent ledger after the delivery window. Republishing a missing stable ID is safe because the consumer applies the same uniqueness rule.

## Where does each scheduler and queue option fit?

The primary decision is how much latency the report can absorb versus how much infrastructure the team can afford to operate. A daily report usually tolerates a drain window, so bounded consumers beat an elaborate workflow system. The comparison changes when orchestration or replay becomes part of the product.

| Option | Good fit | Not suitable when |
|---|---|---|
| Infrai cron and queue | Plain-HTTP scheduling and buffered dispatch with pull or public HTTPS push | The job needs DAG orchestration, fan-out/join, topic broadcast, or Kafka-style replay |
| Celery | A team already runs Python workers and wants its task execution there | Adding and operating the worker stack would displace feature work |
| Airflow | The report belongs to a DAG with explicit orchestration requirements | The only need is pacing an email batch |
| Temporal | The process needs durable, multi-step workflow semantics | A cron kickoff plus one idempotent delivery step is enough |
| Kafka | Long-lived replay and independent consumer groups are product requirements | Jobs should disappear on acknowledgement and simple dispatch is the goal |

The catch with the simple choice is real: there is no native workflow DAG, fan-out/join, topic one-to-many delivery, or native throttle. Stick with Airflow or Temporal when orchestration is the actual problem. Choose Kafka when replay and multiple consumer groups matter. Celery is reasonable when its worker environment is already sunk cost. Infrai is strongest here when a solo operator values the self-describing REST boundary and consolidated operations more than owning a broker and worker control plane.

Push also has a firm boundary. If security policy forbids a public HTTPS receiver, don't force it; use pull consumption. Your mileage may vary on the latency budget because provider quotas and batch size determine the drain time, and no measured throughput is claimed here.

## What changes after the first weekly release?

Start with one queue, a shared rate limiter, bounded concurrency, and four measurements: oldest-message age, attempt count, provider `429` count, and terminal delivery state. These measurements distinguish a provider bottleneck from a slow renderer without pretending that worker count is throughput.

At larger volume, split queues by email provider so one allowance cannot stall another. Partition by tenant only if measured contention or ordering rules require it. Keep message bodies disposable and load report content from the system of record, because the queue's 30-day retention and acknowledge-on-delete behavior are dispatch semantics, not archival semantics.

If delivery delay becomes unacceptable, the next move is not automatically more workers. First raise only the concurrency that fits the provider budget, then reconsider the report window or provider arrangement. More consumers can reduce internal waiting; they cannot manufacture external capacity.

The decision rule is compact: cron decides when work exists, the queue absorbs the burst, the consumer spends the rate budget, and the ledger prevents a retry from becoming a duplicate. That is simple enough to ship and explicit enough to operate.

## References

- https://docs.celeryq.dev/en/stable/getting-started/introduction.html
- https://en.wikipedia.org/wiki/Exponential_backoff
