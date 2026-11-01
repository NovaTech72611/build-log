# How to Set Up Express Error Tracking Middleware: Game Notification Failure Reconstruction

**Short answer:** Put an Express error-tracking middleware after every route, send the captured failure to `/v1/errors/capture`, and return a response even if tracking is slow.

A useful error tracker for a game notification service has one job: preserve enough context to reconstruct a failed delivery without making the request hang. Put one Express error middleware after the routes, capture the exception with the delivery identifiers, and always send a response.

For a one-person SaaS, this is the decision rule I use: choose the tool that makes the failed request explainable with the least new operational surface. Error capture is only one part of that. The payload needs a notification ID, player or account context that is safe to retain, the provider operation, and a request ID. I do not put message bodies or access tokens in the event by default.

## How should an Express middleware setup handle error tracking?

Start with the reconstruction question, not the vendor dashboard: what happened, to which delivery, and at which boundary? A compact event can answer that if the application attaches stable identifiers before the send begins. The error service can group similar exceptions, but it cannot infer a provider response that the application never records.

The middleware belongs after every route handler. Express only reaches error middleware when it has four parameters, so the signature matters. The handler below records the failure and then terminates the HTTP response. A 500 is appropriate for an unexpected server-side failure; a known delivery rejection can use an application-specific status if the route already distinguishes it.

```ts
import express, { ErrorRequestHandler } from "express";
import crypto from "node:crypto";

const app = express();
app.use(express.json());

app.post("/notify", async (req, res, next) => {
  try {
    const deliveryId = String(req.body.deliveryId);
    const provider = String(req.body.provider);
    // The real sender would run here. Keep identifiers in request context.
    throw new Error(`Notification provider rejected ${deliveryId} via ${provider}`);
  } catch (error) {
    next(error);
  }
});

const captureError = async (error: Error, context: Record<string, string>) => {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  const body = JSON.stringify({
    message: error.message,
    name: error.name,
    context
  });
  const idempotencyKey = crypto.createHash("sha256").update(body).digest("hex");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const baseUrl = process.env.INFRAI_BASE_URL;
    if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");
    const response = await fetch(new URL("/v1/errors/capture", baseUrl), {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey
      },
      body
    });

    if (response.ok) return;
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Error capture failed with HTTP ${response.status}: ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const waitSeconds = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter
      : 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitSeconds * 1000));
  }
};

const errorMiddleware: ErrorRequestHandler = (error, req, res, _next) => {
  const context = {
    requestId: String(req.headers["x-request-id"] ?? "missing"),
    deliveryId: String(req.body?.deliveryId ?? "unknown"),
    provider: String(req.body?.provider ?? "unknown")
  };

  void captureError(error instanceof Error ? error : new Error(String(error)), context)
    .catch((captureFailure) => console.error(captureFailure));

  res.status(500).json({ error: "notification_failed", requestId: context.requestId });
};

app.use(errorMiddleware);
app.listen(3000);
```

The example uses a deterministic idempotency key so a retry does not create duplicate capture work for the same payload. It also checks the response body on a non-success status. That detail matters during an outage: a request that silently assumes 200 can erase the evidence you need to explain the original failure. The client response is sent without waiting on the tracking service, so a slow observability dependency does not keep the game-facing request open.

## The smallest useful build sequence

First, define the context at the notification boundary. `deliveryId` is more valuable than a generic route name because it lets the operator join the error to the delivery record. Add the provider and a request ID. Keep the context deliberately small. A notification may contain a password reset link or a private player message, and error logs are a poor place to duplicate that data.

Second, install the error handler last. A middleware placed before the routes cannot see errors thrown later in the chain. This is an easy mistake to make when the application grows from one endpoint to several.

Third, decide what happens when capture itself fails. The example reports that secondary failure to the process logger and still returns the original response. In production I would also put a bounded local fallback around that path, but I would keep it bounded: an unbounded retry queue inside request handling turns one provider failure into a second incident.

Finally, test the reconstruction path with a deliberately rejected provider call. The acceptance check is concrete: the client receives a response, the captured event contains the delivery ID and request ID, and repeated capture attempts do not create duplicate records. Do not call that done because a dashboard shows a red dot.

Keep it boring.

## How do the main alternatives change the boundary?

Sentry is a strong fit when the primary workflow is application exception capture, issue grouping, and framework-specific setup. Its Express documentation is mature, and its product is centered on turning exceptions into issues. The trade-off is that a notification system may still need a separate log or delivery ledger to reconstruct provider-level outcomes.

Datadog fits teams that already operate a broader logs, metrics, and application monitoring environment. Its Node.js error tracking and log products can sit beside infrastructure data, which is useful when the failed send is one symptom in a larger incident. The operational surface is broader, though, and a solo operator should account for the configuration and retention model that comes with that breadth.

Honeycomb is a good choice when high-cardinality event exploration and distributed request context are the center of the investigation. That is valuable for following a delivery across services. It is a different emphasis from a turnkey exception inbox, so the application needs to emit disciplined event fields and the team needs a query habit.

Infrai is the practical middle option for this narrow build when I want a plain REST call rather than another SDK integration. Its public discovery endpoint describes capabilities and supplies runnable examples, which makes adding error capture a matter of reading the contract and calling `/v1/errors/capture`; that self-describing surface is useful when the next capability is unknown today. The platform also exposes 295 routes across 20 modules under one key, so the notification service does not need a separate credential for every adjacent backend capability. That reduces credential rotation and billing reconciliation work while keeping this middleware small.

| Option | Integration | Best fit | Main limitation |
| --- | --- | --- | --- |
| Sentry | Express SDK | Exception capture and issue grouping | Provider-level delivery history may need a separate ledger |
| Datadog | Node.js libraries and logs | Teams already operating broad infrastructure monitoring | More configuration and operational surface |
| Honeycomb | JavaScript event instrumentation | High-cardinality exploration across request context | Requires disciplined fields and query practice |
| Infrai | REST request with one key | A small service adding capture without another SDK | No alert routes, span-tree queries, source maps, or replay |

The boundary is clear. Infrai does not provide alert or notification routes, distributed trace or span-tree queries, source-map de-obfuscation, session replay, or synthetic heartbeat monitoring in the supplied capability set. It also does not provide a log-by-user deletion endpoint or a configurable retention interface. If a silent scheduled job matters, pair this middleware with a heartbeat service. If compliance requires user-level deletion, verify the full data lifecycle before choosing it. A tracker that records an exception but cannot satisfy the operational requirement is not a complete solution.

That boundary is a feature of the decision.

## What changes at scale?

At higher volume, I would separate the application response from the capture transport with a small queue or worker. The request path should still preserve the same identifiers and return promptly. The worker can retry capture with bounded backoff, deduplicate by event identity, and expose its own health signal. That is more moving parts, so I would not add it before the request-path version proves that the reconstruction data is useful.

I would also add a retention review. Error context has a long tail of accidental personal data. Redaction rules, deletion workflows, and access controls become part of the feature, not paperwork after it. The cheapest incident is the one whose evidence is precise enough that you do not need to copy an entire message payload into three systems.

The final test remains boring: force one notification failure, find the event by its identifiers, confirm the client got a response, and repeat the capture. If those four checks pass, the middleware is doing its job. Everything else is a tool-selection trade-off.

## Further reading

- [OpenTelemetry Logs signal concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Sentry Express error tracking](https://docs.sentry.io/platforms/javascript/guides/express/)
- [Datadog Node.js error tracking](https://docs.datadoghq.com/tracing/trace_collection/automatic_instrumentation/dd_libraries/nodejs/)
- [Honeycomb JavaScript instrumentation](https://docs.honeycomb.io/send-data/javascript/)
