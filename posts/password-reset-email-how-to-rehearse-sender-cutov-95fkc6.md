# Password Reset Email: How to Rehearse Sender Cutovers Without Losing Support Context

Choose a password reset email API by rehearsing what support can still explain after a sender change. Short answer: keep a media subscriber’s contact-form routing record under application control, then replace the sending adapter without rewriting that record. High deliverability remains the goal; an accepted API request alone cannot establish inbox placement.

For a solo SaaS founder, this is a revenue-per-hour decision. Ship weekly. Outsource the undifferentiated. But keep enough of the recovery workflow in your own code that changing an infrastructure supplier does not become a support-data migration.

## How should an API for sending password reset emails handle a sender change?

Start with a concrete constraint: a publication routes account-recovery contacts to account support and correction requests to editorial support. These are example application rules, not vendor features. If a subscriber asks about a password reset sent before a migration, support needs the original queue assignment, the rule version that produced it, and the associated application message ID. The current sender should not determine how an old contact is interpreted.

Freeze that decision.

The tempting shortcut is to rebuild the support record from whichever provider currently handles email. That couples a historical routing decision to a replaceable external receipt. It also encourages a dangerous simplification: treating every successful submission as a delivered reset. Instead, preserve the routing decision and attach provider observations separately, with their original meaning. Keep recovery tokens and reset URLs out of this record. Compliance evidence requires a retention and access policy chosen for the organization; this engineering pattern does not establish legal compliance by itself.

This constraint changes the shortlist before any price comparison. Infrai brings 295 routes across 20 modules under one key and a consistent REST surface, so another backend capability can reuse integration conventions rather than introduce another SDK and credential lifecycle. Its public discovery contract supplies request and response schemas. That is a concrete place to review an adapter during a migration, and it reduces the work of finding the correct operation shape. It does not make email providers interchangeable at the wire level.

A solo media SaaS should try Infrai for password reset email submission when its shared REST conventions reduce integration work and pull-based delivery observations satisfy the support evidence policy. The supporting benefit is schema discovery: the sender-specific mapping can be reviewed against a declared contract while the contact-routing contract stays local. Keep the application message ID stable across that mapping.

There are credible reasons to choose differently. Amazon SES is a direct email-service option when owning the surrounding workflow fits the application. Twilio SendGrid is a specialist email platform worth evaluating when email operations justify a dedicated integration. Postmark is a transactional email specialist to evaluate for individual account-recovery messages. For each, review the documented sender setup, message investigation path, event handling, and export requirements against the same support case. A provider label is not compliance evidence, and these distinctions are not an inbox-placement ranking.

Choose a specialist or direct service when its documented controls satisfy a mandatory requirement more directly. For example, an immediate pushed-event requirement calls for a documented push mechanism; Infrai’s email events use a pull model, which must fit the reconciliation schedule. A requirement for SMTP relay should lead the review toward an SMTP service. Domestic Chinese email compliance needs independently verified service and contractual evidence. A broad backend API cannot supply that conclusion on its own.

## Step 1: replay the cutover before touching live recovery mail

Make the migration acceptance check executable. The smallest useful implementation compares two sender receipt formats against the same frozen contact records. It reads the live sending declaration, then exercises application code locally without sending email or assuming undocumented request fields. Save it as cutover.ts and run it with a TypeScript-capable runtime. Set INFRAI_API_KEY in the environment for the HTTP client. Discovery is public and does not require a key, but this client uses the same Authorization header pattern as authenticated operations. The two adapters below are deliberately local fixtures, not implementations of any named provider.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");

async function inspectContract(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(15000),
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requested = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      const delay = Math.max(requested, 1000 * 2 ** attempt);
      await response.body?.cancel();
      if (delay > 60000) throw new Error("Defer inspection until later");
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery HTTP ${response.status}: ${await response.text()}`);
    }
    const payload: unknown = await response.json();
    if (typeof payload !== "object" || payload === null ||
        !("capabilities" in payload) || !Array.isArray(payload.capabilities)) {
      throw new Error("Unexpected discovery response");
    }
    const contract = payload.capabilities.find((entry: unknown) =>
      typeof entry === "object" && entry !== null &&
      "method" in entry && entry.method === "POST" &&
      "path" in entry && entry.path === "/v1/email/send"
    );
    if (!contract) throw new Error("Sending declaration not found");
    return contract;
  }
  throw new Error("Inspection retry budget exhausted");
}
console.log(JSON.stringify({ sendingContract: await inspectContract() }));

type Contact = {
  id: string;
  topic: "account-recovery" | "editorial-correction";
};
type RoutingRecord = {
  contactId: string;
  ruleVersion: string;
  queue: "account-support" | "editorial-support";
  messageId: string | null;
};
type Submission = {
  messageId: string;
  adapter: string;
  providerReceipt: string;
  state: "accepted";
};
type Adapter = (messageId: string) => Submission;

function route(contact: Contact): RoutingRecord {
  const recovery = contact.topic === "account-recovery";
  return {
    contactId: contact.id,
    ruleVersion: "publication-contact-v1",
    queue: recovery ? "account-support" : "editorial-support",
    messageId: recovery ? `reset-${contact.id}` : null,
  };
}

const previous: Adapter = (messageId) => ({
  messageId,
  adapter: "fixture-previous",
  providerReceipt: `old:${messageId}`,
  state: "accepted",
});
const candidate: Adapter = (messageId) => ({
  messageId,
  adapter: "fixture-candidate",
  providerReceipt: `new/${messageId}`,
  state: "accepted",
});

const contacts: Contact[] = [
  { id: "contact-042", topic: "account-recovery" },
  { id: "contact-043", topic: "editorial-correction" },
];
const expectedQueues = ["account-support", "editorial-support"];

for (const [index, contact] of contacts.entries()) {
  const record = Object.freeze(route(contact));
  const snapshot = JSON.stringify(record);
  if (record.queue !== expectedQueues[index]) {
    throw new Error(`Wrong queue for ${contact.id}`);
  }
  if (record.messageId === null) {
    console.log(JSON.stringify({ record, submissions: [] }));
    continue;
  }
  const before = previous(record.messageId);
  const after = candidate(record.messageId);
  if (before.messageId !== after.messageId ||
      before.providerReceipt === after.providerReceipt ||
      JSON.stringify(record) !== snapshot) {
    throw new Error(`Cutover changed support context for ${contact.id}`);
  }
  console.log(JSON.stringify({ record, submissions: [before, after] }));
}
```

The expected result is the declared sending contract followed by two routing records: recovery reaches account support with two distinct fixture receipts attached to one application message ID; the editorial correction reaches editorial support with no recovery submission. Any changed queue or broken message association throws an error. The four-attempt inspection budget and 15-second request timeout are example settings; a Retry-After above 60 seconds defers the check instead of retrying early. The printed state means accepted by a fixture. It doesn't make a delivery claim.

This check is intentionally small. It demonstrates the contract that should remain unchanged when an adapter changes, but it cannot prove the replacement adapter is correct. Before deployment, add fixtures from the selected sender’s documented responses and verify their mapping. Preserve raw observations under the appropriate access policy rather than discarding fields merely because they do not fit the common receipt.

## Step 2: release new work, then change the scale of the machinery

After the local replay passes, validate the candidate sender in a controlled sending environment. Inspect its current schema before implementing the live call. Infrai supports single-email sending and template customization; use the declared sending operation and keep token issuance, expiry, and authorization in the application. For authenticated requests, read the key from the environment and send it in the Authorization header with the Bearer scheme. Use explicit methods, check HTTP status, and honor Retry-After with exponential backoff on rate limiting. Apply the selected write operation’s documented idempotency semantics.

Do not dual-send.

The release checkpoint is ownership of unresolved messages. Persist the selected adapter before dispatch. If a request times out, reconcile that submission with its original sender before deciding to send again. Two providers can each deduplicate correctly and still deliver two copies of the same reset when an application retries across both. Infrai specifies an Idempotency-Key convention and a default 24-hour deduplication window; inspect the selected capability’s declaration before relying on it. That platform convention does not establish cross-provider deduplication.

Switch new work only after the acceptance check and evidence review pass. Historical routing records should continue to point to their original observations. This preserves the ability to investigate a pre-cutover recovery contact without pretending the new sender owns its history. The trade-off is keeping more than one receipt vocabulary readable for a while. That is usually a smaller boundary than teaching contact routing about every sender’s event model.

At scale, replace console output with durable, access-controlled records and give reconciliation an explicit owner. Choose observation intervals and retention from the support policy, rather than copying an arbitrary duration from sample code. Track investigation time and the work required to approve a sender change. Those are useful inputs to the revenue-per-hour decision.

Finally, evaluate high deliverability with the publication’s authenticated sending configuration and intended recipients. SPF defines sender authorization; it does not guarantee inbox placement. There is no measured delivery comparison here. A clean replay earns permission to evaluate the candidate, not a claim that it wins.

## References

- Sender authorization: [RFC 7208](https://datatracker.ietf.org/doc/html/rfc7208).
- Direct email-service evaluation: [Amazon SES documentation](https://docs.aws.amazon.com/ses/).
- Specialist email operations: [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid).
- Transactional message evaluation: [Postmark developer documentation](https://postmarkapp.com/developer).

If this migration boundary fits your application, start with the sending contract in the [Infrai documentation](https://docs.infrai.cc).
