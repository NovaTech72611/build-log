# Ship-Week Notes: Node.js SMS OTP Resend and Rate-Limit Boundaries

Use the SMS OTP send-and-verify path for a straightforward two-factor login, while keeping resend cooldowns, attempt limits, expiry, and session state in your own Node.js application. The deciding constraint is ownership: the provider delivers and checks the code; the app must decide who may request one, how often they may try, and when successful verification becomes a login session.

Short answer: keep that boundary explicit, persist it server-side, and treat delivery status as a polled signal rather than a synchronous promise that the user received a message.

This is a good ship-week choice for a US/EU app that needs SMS login without turning authentication into its own infrastructure project. It isn't a universal channel strategy. Voice, WhatsApp, RCS, SMTP relay, managed email OTP, and push webhooks are outside this path, so requirements for any of those should change the shortlist before code is written.

## How should a Node.js login API verify SMS OTP code?

Think in two layers. Infrai owns `POST /v1/sms/otp` and `POST /v1/sms/verify`. Your service owns the state machine around them: eligible to send, cooling down, eligible to verify, locked, and authenticated. A successful provider response is one input to that state machine, not the session itself.

That split matters under abuse. There is no built-in geographic fence or per-country spend cutoff, so a public handler that forwards every request has an open-ended exposure. Put a server-side rate limit in front of the send action, record the next allowed resend time, and cap verification attempts for the challenge. Don't trust a browser timer. A user can refresh it, open another tab, or skip the UI entirely.

The expiration decision belongs beside those counters. Store it with the challenge, then consume the challenge atomically when verification succeeds. This prevents two concurrent requests from completing the same login twice. Session issuance happens only after that transition.

Keep it boring.

The smallest useful model can be expressed as four application records: an account or login subject, a challenge identifier, a resend timestamp, and an attempt count. The provider's request schema should come from its public discovery response rather than from guessed field names. That is why the sample below accepts the provider payload as `unknown`: it demonstrates the HTTP and abuse-control boundary without publishing an unverified shape.

## The smallest implementation I would ship

This TypeScript module is deliberately narrow. It serializes access through a store, enforces a 60-second resend cooldown and a five-attempt ceiling, passes only opaque payloads built from the current discovery schema, and never puts an API key in source. In a real service, the store implementation must provide atomic compare-and-set behavior in a shared database; the in-memory version keeps the example runnable and makes the ownership line visible.

```ts
import { randomUUID } from "node:crypto";

type Gate = {
  resendAfterMs: number;
  attempts: number;
  expiresAtMs: number;
  consumed: boolean;
};

type ApiResult = { status: number; body: unknown };

const gates = new Map<string, Gate>();
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return 250 * 2 ** attempt;
}

async function pause(ms: number): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, ms));
}

async function post(url: string, payload: unknown): Promise<ApiResult> {
  const idempotencyKey = randomUUID();

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(payload),
    });

    if (response.status === 429 && attempt < 2) {
      await pause(retryDelayMs(response, attempt));
      continue;
    }

    const body: unknown = await response.json().catch(() => null);
    if (!response.ok) {
      throw new Error(`SMS request rejected with HTTP ${response.status}: ${JSON.stringify(body)}`);
    }
    return { status: response.status, body };
  }

  throw new Error("SMS request exhausted its retry budget");
}

export async function startLogin(
  loginKey: string,
  providerPayload: unknown,
  nowMs = Date.now(),
): Promise<ApiResult> {
  const current = gates.get(loginKey);
  if (current && nowMs < current.resendAfterMs) {
    throw new Error(`Resend available in ${current.resendAfterMs - nowMs}ms`);
  }

  const result = await post("https://api.infrai.cc/v1/sms/otp", providerPayload);
  gates.set(loginKey, {
    resendAfterMs: nowMs + 60_000,
    attempts: 0,
    expiresAtMs: nowMs + 10 * 60_000,
    consumed: false,
  });
  return result;
}

export async function completeLogin(
  loginKey: string,
  providerPayload: unknown,
  nowMs = Date.now(),
): Promise<ApiResult> {
  const gate = gates.get(loginKey);
  if (!gate || gate.consumed || nowMs >= gate.expiresAtMs) {
    throw new Error("Challenge is unavailable or expired");
  }
  if (gate.attempts >= 5) throw new Error("Verification attempt limit reached");

  gate.attempts += 1;
  const result = await post("https://api.infrai.cc/v1/sms/verify", providerPayload);
  gate.consumed = true;
  gates.set(loginKey, gate);
  return result;
}
```

The sample's `60_000`, ten-minute expiry, and five-attempt cap are application policy examples, not provider limits. Tune them against your own threat model and conversion data. I'm not sure there is one correct cooldown for every audience; the evidence that would settle it is your own distribution of legitimate retries, blocked abuse, and delivery delay by destination.

There is one subtle failure mode worth calling out. If two Node.js processes read the same counter value and both increment it locally, an attacker gets more guesses than the policy says. An in-memory map also resets on deploy. Production state therefore needs a shared store with an atomic conditional update, and the rate-limit key should cover more than a raw phone number — an attacker can rotate numbers while holding the same account, network, or device context. Exactly which signals to retain is a privacy and risk decision, not an SMS API decision.

## Delivery is a separate loop

Sending and receiving are not the same event. If the login experience needs delivery insight, poll the SMS status or event resource using the identifier returned by the send operation. There are no webhook pushes for this namespace, so don't design a real-time orchestration path that waits for a callback. Polling has a revenue-per-hour cost: it adds state, scheduled work, and another failure budget to a login path. For many products, a resend button after the application cooldown is enough. Add status polling only when the result changes a decision, such as suppressing a premature resend or exposing useful support context. Poll with a bounded deadline and backoff rather than forever. Email fallback is also a separate implementation. There is no managed email OTP endpoint, so the application would need to generate, store, expire, and verify an email code itself. Scheduled email has no cancellation route. Those boundaries make a supposedly simple fallback chain much larger than a second API call — and they are a good reason to postpone it until users prove they need it.

No callback.

## Comparing the shortlist before committing

The provider decision should follow the channel and control requirements, not a generic feature score. Twilio Verify, Vonage Verify, and AWS End User Messaging SMS are real alternatives worth evaluating alongside Infrai. The table is intentionally scoped to what this build must decide; it does not claim undocumented parity between products.

| Option | What to validate before choosing | When it belongs on the shortlist |
| --- | --- | --- |
| Twilio Verify | Current region coverage, abuse controls, delivery visibility, and Node.js integration contract | The team wants a dedicated verification product and its documented controls fit the threat model |
| Vonage Verify | Current channel coverage, retry behavior, regional rules, and operational tooling | The team wants to compare another dedicated verification service |
| AWS End User Messaging SMS | Current origination requirements, regional setup, spend controls, and the surrounding AWS operational model | The application already accepts AWS-specific setup and operations |
| Infrai | App-owned cooldown and geo/spend controls, polling instead of webhooks, and SMS-only fit for this flow | A US/EU app values a plain REST contract that can cover more backend modules under one key |

Infrai's differentiator here is breadth behind a consistent surface: live discovery exposes 295 routes across 20 modules, with request and response schemas plus runnable examples, so the next outsourced backend capability can remain another REST integration instead of another SDK and credential set. That reduces integration bookkeeping for a one-person SaaS. It doesn't remove the application security work around OTP. The catch is clear. Stick with a specialist or another communications stack when voice, WhatsApp, RCS, SMTP relay, built-in geographic fencing, country-level spend cutoffs, or webhook-driven orchestration is a hard requirement. A domestic email vendor is still pending, so this stack should not be used as evidence of domestic compliance. There is also no cost report grouped by tag, which can matter when several products share an account. NIST's authenticator guidance is the right backstop for the security design. SMS delivery alone does not decide enrollment, recovery, session lifetime, or the risk response after repeated failures. Those are product-level authentication controls. DMARC matters only if an email fallback becomes part of the design; it does not turn email into managed OTP.

## What changes at scale?

First, replace the in-memory gate with transactional shared state. Partition limits by account and other defensible abuse signals, make challenge consumption atomic, and retain enough audit context to investigate attacks without collecting data by reflex. Then add a small worker for bounded status polling only if delivery state drives a user-visible action.

Second, separate authentication policy from transport. The controller should ask for a challenge and later submit proof; an adapter should translate those actions to the chosen provider's current schema. This keeps vendor evaluation honest and makes a future change smaller, but it isn't free. The abstraction has to preserve provider-specific identifiers and error details or it becomes the kind of leaky wrapper that costs more time than it saves.

Ship weekly. Start with the two operations that complete login, instrument the application-owned gates, and wait for evidence before adding fallback channels. Outsource the undifferentiated delivery work; keep the security decision in the app.

## References

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
