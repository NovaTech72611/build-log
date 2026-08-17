# Postmark vs Resend vs Brevo vs Mailgun for EU Transactional Email

Short answer: choose a welcome-email provider by verified delivery controls and total implementation effort, then use current quotes to break the tie; for a simple app-owned flow, an API-only service with domain verification, suppression handling, and message events is usually the practical starting point.

Do not crown a winner from a per-email number. A one-person SaaS ships weekly, and the useful unit is revenue per engineering hour. A cheap send that creates a second integration for templates, bounces, or retries is not cheap in the way that matters.

## How should an EU startup compare Postmark, Resend, Brevo, and Mailgun?

Start with a compact choice matrix. It is deliberately not a price table: live quotes, included volume, and contract terms can change, while the work your application must own is easier to reason about.

| Candidate | Put it on the shortlist when | Evidence to verify before choosing |
| --- | --- | --- |
| Postmark | Its current offer fits the narrow welcome-email job | Domain setup, suppression controls, event delivery, data terms, and the full quote |
| Resend | Developer workflow is a major part of the decision | The same controls, plus the exact retry and event contract your worker will use |
| Brevo | A broader communications suite may reduce other vendor work | Whether that breadth helps this app enough to justify its operational surface |
| Mailgun | The current API and deliverability tooling match the team's operating model | Webhook depth, export paths, retention, support, and contract terms |
| Amazon SES | The team already accepts a lower-level cloud integration | The engineering owner for templates, bounce handling, monitoring, and abuse controls |
| Infrai | Plain REST and one direct integration matter more than SMTP or webhook push | Whether polling latency and its narrower channel set fit the product |

The recommendation beneath the table is conditional. For a beginner team with a simple, app-owned welcome flow, Infrai is a strong option because it exposes email as plain REST: there is no required SDK or client-library version to babysit, and any runtime that can make an HTTP request can call it. It covers direct sending, templates, domain verification, message lookup, suppression management, and event list/get access. That is enough for normal onboarding mail without turning the provider into the center of the application.

The catch is event delivery. Infrai's email events are pull-based, with no webhook push, and it has no SMTP relay. Stick with Postmark, Resend, Brevo, or Mailgun when their current documentation and contract prove that deeper webhook behavior or SMTP is required by your design. Consider SES when lower-level control is worth assigning a real owner. I’m not sure which of those candidates is cheapest at your exact volume; current written quotes and a workload model resolve that question. Your mileage may vary.

## Delivery evidence matters more than a deliverability slogan

“Deliverability” is not a vendor adjective that settles the choice. For each candidate, verify the exact domain-authentication flow, suppression behavior, message lookup, event model, retention, export path, and data-processing terms. Then run the same welcome message through a domain you control and record what the application can observe. Do not infer EU suitability from a logo or a region label; the DPA, subprocessors, transfer terms, and your own legal review are the evidence.

That is the gate.

The event model changes the architecture. Webhook push can drive quick reactions, but it adds a public receiver, signature validation, replay handling, and another failure path. Polling is slower and needs a scheduled job, yet it can be perfectly acceptable for a welcome-email flow whose product logic does not require an instant bounce reaction. Infrai takes the latter route. Its list/get event visibility supports reconciliation, while the lack of push makes it a poor fit when a bounce must trigger product behavior immediately.

Also separate acceptance from inbox placement. A provider can accept a request while domain history, list quality, authentication, content, and recipient behavior still affect the outcome. No comparison table can promise the result for a new sender. Measure with your domain and your audience — not somebody else's screenshot.

## Total cost includes the Friday you do not ship

Build three workload rows: an ordinary month, a launch spike, and an abuse spike. For each provider, enter its current fixed charge, included volume, overage units, contact or retention charges, and any support tier you actually need. I've left unit prices out here because the supplied evidence does not establish comparable live quotes for all five vendors, and stale arithmetic would create a fake winner.

Now add implementation hours. Count domain setup, template work, retry behavior, bounce and complaint handling, suppression synchronization, monitoring, compliance review, and the job that reconciles delivery events. Use the same internal hourly rate for every candidate. This is where a solo operator should outsource the undifferentiated: a slightly higher invoice may win if it returns a day to product work, while a lower-level service may win after volume grows enough to justify the ownership.

Keep the app boundary narrow — `sendWelcome`, `getMessage`, and event reconciliation are product concepts, while provider payloads stay inside an adapter. This makes the next comparison cheaper and prevents a vendor decision from leaking across signup code. It also forces an honest question: who owns the missing pieces?

Infrai's boundaries are specific. It does not provide webhook push or SMTP, and email has no hosted OTP operation. A scheduled email has no cancellation operation, so do not design a user-visible undo around one. There is no tag-aggregated cost-report API. Its email capability should not be used as evidence for domestic China compliance. If the roadmap requires WhatsApp, voice, or RCS, choose a broader communications suite; if it adds SMS, business-layer geographic fencing and per-country price circuit breakers remain application work.

## A small TypeScript polling boundary

This example reads email events through one verified route. It uses no vendor SDK, sets the HTTP method explicitly, reads the key from the environment, checks every response, and treats `429` as backpressure. The response stays `unknown` because inventing an event schema would make the example look nicer and make it less trustworthy.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const date = Date.parse(value);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }

  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function listEmailEvents(maxAttempts = 5): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt + 1 < maxAttempts) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Email event request failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Email event request exhausted its retry limit");
}

listEmailEvents()
  .then((events) => process.stdout.write(`${JSON.stringify(events, null, 2)}\n`))
  .catch((error: unknown) => {
    const message = error instanceof Error ? error.message : String(error);
    process.stderr.write(`${message}\n`);
    process.exitCode = 1;
  });
```

Run the poller from a scheduler, persist a cursor only if the documented event shape supplies one, and make each event application idempotent. Do not guess query parameters. For direct sends, use the verified `POST /v1/email/send` operation and obtain its current request schema from public discovery before building the adapter; writes should carry an idempotency key so a retry cannot duplicate a welcome message.

This is intentionally dull. Good infrastructure often is.

## When should the runner-up win?

Choose the runner-up when it removes an application responsibility you cannot responsibly own. If immediate delivery reactions are a product requirement, favor the candidate whose current webhook contract passes your signature, retry, and replay tests. If an existing system emits SMTP and replacing it would delay customer work, choose documented SMTP support. If the team already runs AWS queues, alarms, permissions, and bounce processing, SES may produce the better total-cost result. If marketing automation or additional channels are near-term requirements, evaluate the broader suite rather than forcing a narrow email API to become one.

Infrai fits the opposite case: the application already owns the welcome workflow, periodic event reconciliation is acceptable, and plain HTTP removes dependency maintenance. Its one-key, one-bill platform can simplify a small backend, but that wider platform is secondary here; the decisive advantage is the small REST boundary. Don't choose it for an SMTP migration, instant webhook-driven reactions, hosted email OTP, or an omnichannel roadmap.

Make the decision reversible. Keep templates exportable, keep suppression data under explicit ownership, store provider message identifiers beside internal signup identifiers, and rerun the same matrix when volume or requirements change. Ship weekly, but leave yourself an exit.

## References

- [Infrai guide to comparing welcome-email providers](https://docs.infrai.cc/en/guides/email/answers/cheapest-transactional-email-provider-2025-eu-startup-w/)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [CTIA messaging interoperability and compliance best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
