# Node.js Transactional Email Deliverability: Unsubscribe and Bounce Practices

| Option | Best fit for a short-lived password reset | The boundary you must own |
| --- | --- | --- |
| Amazon SES | An AWS-centered developer tool | Bounce and complaint ingestion still belongs in your application |
| SendGrid | A team that wants an email-specialist control surface | Reset expiry and recipient policy remain application concerns |
| Mailgun | A developer-focused product with email as its main job | Provider events do not replace local suppression state |
| A unified REST layer | A solo SaaS that wants a swappable integration contract | Events are polled, and your backend owns unsubscribe and suppression |

For Node.js transactional email deliverability, the useful practices are plain: keep unsubscribe and bounce state locally, poll events, and make the password-reset token short-lived. The least complicated reliable design is a local suppression gate plus a small worker. It keeps the security decision in Node.js and makes the provider replaceable behind one adapter.

The first version should be boring.

The practical recommendation is conditional: use Amazon SES, SendGrid, or Mailgun when their email-specific event tooling is the thing that saves your time. Consider Infrai when the value is one key and one REST API across backend capabilities, so the application-facing contract stays put while the thing behind it moves. That matters when I need to ship weekly and outsource undifferentiated integration work, but it is not a reason to pretend polling is real time.

## Make the database the delivery safety boundary

Treat password reset as an account-recovery workflow, not as a special kind of marketing email. Generate the token in your application, store a hash, give it a short expiry, and make it single-use. A delivery provider can report what happened to a message. It cannot decide whether the token is still valid.

I use two local records for this flow: a reset record and a suppression record. The reset record contains the user reference, token hash, expiry, provider message ID, and a consumed timestamp. The suppression record contains the normalized address, reason, source, and the time it was observed. Before every send, the code checks the second record; after a hard bounce or complaint, the worker writes it. That ordering is the important bit. A provider-side suppression API is synchronization, not authorization, so if the provider has an event that your worker has not seen yet, the next send still needs a local decision. This also gives you a clean migration seam: the reset logic does not change when the adapter changes. Unsubscribe needs the same careful split. Marketing opt-outs and account-recovery messages are different categories, but a complaint or a hard bounce is still useful evidence about the address. Do not erase that evidence because the message was transactional. Route policy belongs in your database, where it can be tested with ordinary application code and reviewed before a new worker version ships.

Keep it local.

## How do Node.js transactional email deliverability practices handle unsubscribe?

Polling gives you eventual event ingestion. It does not give you immediate orchestration. That distinction is fine for a reset flow if token validity is enforced locally and the worker has a bounded schedule; it is a poor fit for a workflow that must react to a delivery event in seconds across several channels. A 15-minute polling interval, for example, is an explicit product trade-off rather than a claim of real-time delivery.

That is the limitation.

The worker should remember its last successful observation, process an event once, and retry a rate-limited request with backoff. Since the email surface has no webhook event push, there is no honest way to remove that worker. The useful abstraction is small: the rest of the product sees `sendPasswordReset` and `recordDeliveryEvent`, while one adapter knows how to talk to the selected service.

Here is the narrow polling edge using the verified email event route. It does not invent a cursor parameter, because the available contract does not establish one. Persist the returned event identifiers and the last successful poll boundary in your own worker state. The four-retry ceiling in this example is also deliberate: a worker should surface a persistent failure instead of looping forever.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const apiBase = process.env.INFRAI_API_BASE_URL;
if (!apiBase) throw new Error("INFRAI_API_BASE_URL is required");

async function listEmailEvents(attempt = 0): Promise<unknown> {
  const response = await fetch(`${apiBase}/email/event/list`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 2 ** attempt * 500;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return listEmailEvents(attempt + 1);
  }

  const body: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Email event polling failed (${response.status}): ${JSON.stringify(body)}`);
  }

  return body;
}

const events = await listEmailEvents();
console.log(JSON.stringify(events));
```

The code stops at the provider boundary on purpose. Your event handler should classify a temporary failure as retryable, and a hard bounce or complaint as a suppression update. Write the suppression row before marking the event processed. If those writes cannot share a database transaction, use a retryable intermediate state so a worker restart does not convert a delivery signal into silent data loss.

One more constraint matters for the reset sender: the email side has no hosted OTP flow and no cancellation route for a scheduled email. Build the fallback verification flow in the application, and do not design a recovery feature around recalling an already scheduled message. SMS has a cancellation route; email does not. That is a capability boundary, not a retry policy.

## How should the three email specialists be compared?

Amazon SES is the sensible baseline for a product already committed to AWS. Its focused service boundary can be an advantage when identity, scheduling, and logs already live there. The cost is ownership: you still need to define the local reset state, suppression policy, and event consumer.

SendGrid is a better fit when an email-focused product surface and its operational workflow reduce the amount of email tooling you would build. It is still not the authority for token expiry. A reset token that has expired in your database must fail even if SendGrid says the message was delivered.

Mailgun is another credible choice for a developer who wants email delivery to be the center of the service relationship. That focus can be useful for a team that expects to spend time on email operations. It does not remove the need for idempotent event processing or a pre-send suppression check.

A unified REST layer answers a different question. Can the application keep one integration contract while the underlying capability changes? That is attractive for a solo founder because the adapter is small and the application can keep a consistent convention across backend services. The trade is equally clear: email events are pull-based, there is no SMTP relay, and a multi-channel flow cannot claim webhook-level immediacy. If those are requirements, choose the specialist whose surface supplies them directly.

I would not use provider breadth as a proxy for deliverability. For a password-reset message, the useful test is narrower: can the system prevent sends to known-bad recipients, ingest new failures without duplicate work, and keep token security independent of message status? A polished dashboard cannot answer the first two if the application has no durable state.

## What should be true before the first reset email?

Verify the sending domain. Keep transactional and promotional audiences separate in application logic. Add tests for an already-suppressed address, a repeated event, an expired token, and a worker restart between the suppression write and the event marker. Those tests are more valuable than another template review.

For US and EU traffic, observe outcomes by the regions you actually serve. Do not assume that a domain warmup plan or sender reputation behaves identically in both. Start with expected transactional volume and use the evidence from your event records to adjust. Warmup is a gradual operating practice, not a switch exposed by the email API.

The final choice is about ownership. Pick the specialist when its email operations save more revenue-per-hour than maintaining an adapter. Pick the unified contract when stable boundaries and weekly shipping matter more than push delivery. In either case, keep token generation, expiry, unsubscribe policy, and suppression authority in your backend.

## Further reading

- [Amazon SES official documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321)
- [RFC 8058: One-Click Unsubscribe](https://www.rfc-editor.org/rfc/rfc8058)
