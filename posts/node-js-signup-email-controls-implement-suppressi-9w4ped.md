# Node.js Signup Email Controls: Implement Suppression List and Bounce Handling

Short answer: implement a local suppression list in front of every Node.js transactional email API call, including welcome and signup verification messages. Record unsubscribe choices and permanent bounce events there, process delivery events idempotently, and keep temporary failures eligible for a bounded retry. Do not ask a remote suppression endpoint for permission during the signup request. That adds another network dependency to the path that is supposed to help a new customer get in.

For a one-person SaaS, this is mainly a reliability decision. A verification link is useful only if it arrives, yet repeated sends to a known bad destination damage the same delivery channel. I want the signup endpoint to enqueue quickly, a worker to make one deterministic eligibility decision, and delivery events to improve the next decision. That is small enough to ship this week and plain enough to operate alone.

## How should a Node.js API implement welcome email suppression and unsubscribe handling?

The constraint is awkward: a customer who cannot verify an account will contact support, but the support queue cannot repair an email that keeps bouncing. Retrying everything is not reliability. It is repeated failure with more load.

An unsubscribe and a bounce do not mean the same thing. A user can decline optional welcome mail while still needing an account-security message. A permanent delivery failure says the address is not currently usable for either stream. A temporary failure says try later, within a limit. Flatten those signals into one Boolean and the system will either send mail the recipient declined or suppress mail required to finish signup.

That distinction is the design.

Google's sender guidance reinforces the operational side: authenticate mail, keep spam rates low, and make unsubscribing easy for subscription messages. The application still needs a local policy that maps each message purpose to the recipient states that block it.

**The gate belongs at send time, not only when the job is created.** A bounce event may arrive after enqueueing but before a retry. Checking again in the worker closes that gap.

## Model state by purpose

Use events as the audit trail and a projection as the fast decision surface.

| State | Verification link | Optional welcome email | Next action |
|---|---:|---:|---|
| Active | Send | Send | None |
| Marketing opt-out | Send | Suppress | Preserve purpose-specific consent |
| Permanent failure | Suppress | Suppress | Require address correction |
| Temporary failure | Bounded retry | Bounded retry | Wait before another attempt |

This table is application policy, not provider behavior. Put it in tests. The distinction matters because an opt-out applies to a purpose, while an unusable mailbox applies to the delivery channel. Legal requirements vary by jurisdiction and message content, so product and counsel must settle the exact classification. Code should make that decision visible rather than burying it in a template name.

Store normalized address identity, message purpose, state, source event ID, and event time. Treat the raw address as sensitive data. Keep it out of logs. If an operator corrects a typo, preserve the prior event instead of erasing why delivery stopped.

Remote provider suppression can remain a second guard. The local ledger is still the application decision point because signup should not depend on a vendor-specific read, and purpose-specific consent belongs beside product state.

It has a cost.

The local list is a poor fit when several independent applications send to the same recipients but cannot share state with low latency: each copy can drift, and a late unsubscribe may cross an application boundary. In that case, use one shared policy service or make a centralized delivery layer authoritative. A tiny product with one sender has the opposite trade-off. Running a separate service adds deployment work and another outage surface without improving the decision, so a transactional database projection beside account state is usually the smaller choice. Neither layout removes the need to reconcile remote bounce suppression with local consent.

## The smallest Node.js implementation

The useful boundary is three interfaces: a recipient-state store, a queue, and a mail transport. This TypeScript leaves storage and delivery adapters outside the policy.

```ts
import { randomBytes } from "node:crypto";

type Purpose = "signup_verification" | "optional_welcome";
type RecipientState =
  | "active"
  | "marketing_opt_out"
  | "permanent_failure"
  | "temporary_failure";

type Job = {
  id: string;
  recipientId: string;
  to: string;
  purpose: Purpose;
  token: string;
};

interface StateStore {
  get(recipientId: string): Promise<RecipientState>;
}

interface MailTransport {
  send(input: {
    idempotencyKey: string;
    to: string;
    subject: string;
    text: string;
  }): Promise<void>;
}

function blocked(state: RecipientState, purpose: Purpose): boolean {
  if (state === "permanent_failure") return true;
  return state === "marketing_opt_out" && purpose === "optional_welcome";
}

export async function deliver(
  job: Job,
  states: StateStore,
  transport: MailTransport,
): Promise<"sent" | "suppressed"> {
  const state = await states.get(job.recipientId);
  if (blocked(state, job.purpose)) return "suppressed";

  const link = new URL("/verify", "https://accounts.example.com");
  link.searchParams.set("token", job.token);

  await transport.send({
    idempotencyKey: job.id,
    to: job.to,
    subject: "Verify your account",
    text: `Finish signup: ${link.toString()}`,
  });
  return "sent";
}

export function createToken(): string {
  return randomBytes(32).toString("base64url");
}
```

Store only a digest of that random token, make redemption single-use, and enforce expiry in the database transaction that verifies the account. The transport idempotency key is the stable job ID, not a new value per worker attempt. An adapter must map it to the chosen transport's duplicate-send protection.

Delivery events enter through a separate authenticated endpoint. Verify the signature exactly as the transport documents before converting the payload into an internal event. Apply the source event ID under a uniqueness constraint. Duplicate webhook delivery then becomes a no-op.

Do not infer a permanent failure from an arbitrary error string. The adapter owns that translation because event taxonomies differ. Quarantine unknown values for inspection rather than guessing a state that may block a customer.

## Test the failure path

The happy path needs one test. State transitions deserve most of the attention. Test that a marketing opt-out still permits signup verification, a permanent failure blocks both purposes, and the same event applied twice produces one state change. Then force this ordering: enqueue, receive a permanent-failure event, run the worker. The stale job must be suppressed.

Return success to an authenticated event sender only after the event is durably recorded. Track counts for queued, sent, suppressed by reason, temporarily failed, permanently failed, and jobs older than the normal delivery window. Those counters answer the support question that matters: is one customer mistyping an address, or is the delivery path unhealthy?

One trap is easy to miss. A browser refresh can request the link twice. Deduplicate active requests for a short application-defined window, or invalidate earlier tokens and state that behavior clearly. Either rule can work. Accidental ambiguity cannot.

Keep it explicit.

## What I would change at scale

I would keep the policy boundary and replace the storage mechanics. At modest volume, a relational table with a unique event ID and a transaction around projection updates is enough. Higher event volume may justify an append-only stream plus an asynchronously rebuilt projection, but that adds replay tooling and lag. It should earn its keep.

I would also separate delivery reputation by message stream, add synthetic signup checks through controlled inboxes, and alert on latency percentiles rather than raw send acceptance. Acceptance only proves that the next system took the message. It does not prove that the user received it.

The revenue-per-hour choice is straightforward: own the recipient policy, token lifecycle, tests, and observability. Outsource the undifferentiated transport behind a narrow adapter. Ship the small version weekly; do not build a mail platform to deliver one verification link.

**Reliability comes from refusing the wrong send as deliberately as making the right one.** A local, purpose-aware gate and idempotent event ingestion provide that control without tying signup to a specific delivery vendor.

## Further reading

- Google, Email sender guidelines: https://support.google.com/a/answer/81126
- Twilio, SMS character limits and segmentation: https://www.twilio.com/docs/glossary/what-sms-character-limit
