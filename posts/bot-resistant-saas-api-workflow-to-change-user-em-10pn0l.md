# Bot-Resistant SaaS API Workflow to Change User Email Addresses Safely

A safe email-change API is an account-recovery workflow, not a profile update. **TL;DR: require a fresh login, verify both the current and proposed addresses, store a short-lived pending change, and apply it once through an audited transaction.** For a B2B SaaS that already gates signup with a CAPTCHA, keep that CAPTCHA at the abuse boundary; do not mistake it for proof that the person changing an address owns the account.

| Choice | Takeover resistance | Recovery cost | Decision |
|---|---|---|---|
| Write the new email immediately | Low | High | Reject |
| Confirm only the new email | Medium | High when a session is stolen | Avoid for routine changes |
| Confirm old and new addresses | High | Predictable | Default |
| Support-assisted exception | Depends on evidence and controls | Highest | Reserve for loss of the old mailbox |

The default is dual confirmation.

The runner-up, new-address confirmation plus a delayed old-address alert, belongs only where loss of the old mailbox is common and a separate recovery policy can absorb the risk. That distinction matters more than SDK ergonomics. A one-person SaaS should outsource message delivery and CAPTCHA solving, but keep the state machine, authorization rule, and audit trail inside its own domain.

## How should an API safely change a user email address?

The first criterion is proof continuity. An authenticated session says a credential worked earlier. It does not prove that the person holding the browser still controls either mailbox. OWASP recommends reauthentication after risk events, including changing an email address, and advises rotating or invalidating sessions after reauthentication. A password, passkey, or another enrolled factor can provide that fresh proof.

Then prove control of both endpoints. Send separate, single-use confirmations to the current address and the proposed address. The current-mailbox confirmation limits damage from a stolen session. The new-mailbox confirmation prevents typos and assigning the account to an address the user cannot receive. Do not put the email addresses or a bearer credential with broad scope in the URL. Store only a digest of each random token, compare digests, and expire the request.

Keep the public response boring.

Return the same accepted response when the proposed address is already registered, malformed after normalization, or eligible to proceed; otherwise an attacker gets an account-discovery API. Internally, those cases can take different paths and produce different security events. This is one practical step toward changing the address safely without creating a new enumeration surface.

CAPTCHA has a narrower job. It can add friction to automated signup and repeated recovery attempts, but OWASP treats it as defense in depth rather than a substitute for authentication controls. I would enforce it according to abuse signals at signup and recovery initiation, not place it between the two mailbox confirmations. Every extra challenge is support time competing with the next weekly shipment.

## Recovery paths decide the architecture

The second criterion is what happens when the current mailbox is gone.

This is the hard case. If the ordinary change endpoint silently falls back to one-sided confirmation, an attacker with a stolen session has found the shortest takeover path.

Make recovery a separate state machine with stricter evidence, rate limits, notifications, and review rules. NIST SP 800-63B distinguishes account recovery from routine authentication and requires recovery methods to be managed as authentication mechanisms. That framing is useful even when the product is not seeking a compliance badge: recovery can transfer control, so its evidence cannot be weaker merely because the happy path failed.

For a small B2B SaaS, organization context can help without becoming an automatic override. A verified organization administrator may start an escalation, but should not be able to read tokens or directly replace another member's login identifier. Billing records, support conversation history, and tenant membership may inform a documented manual review. None is mailbox proof by itself. Record who approved the exception, what policy version applied, and when the old address was notified.

This is also where signup design comes back. CAPTCHA may reduce bot registrations, yet a real account created by a human can still be hijacked later. Preserve the separation: signup abuse controls decide whether to create an account; authentication proves a returning principal; recovery restores a lost authenticator; email change moves a login identifier. One endpoint should not perform all four jobs.

## A small state machine beats a clever endpoint

Use an opaque request ID and two independently generated tokens. The API below leaves message delivery behind an interface and makes the commit conditional on the original email still matching. All code paths use the same database transaction boundary. In NodeJS, built-in cryptographic randomness and hashing are enough for this part; the harder practice is enforcing transitions consistently across retries, workers, and browser tabs.

```ts
import { createHash, randomBytes } from "node:crypto";

type ChangeState = "pending" | "applied" | "expired" | "cancelled";

type EmailChange = {
  id: string;
  userId: string;
  oldEmail: string;
  newEmail: string;
  oldTokenHash: string;
  newTokenHash: string;
  oldConfirmedAt: Date | null;
  newConfirmedAt: Date | null;
  expiresAt: Date;
  state: ChangeState;
};

const digest = (token: string) =>
  createHash("sha256").update(token, "utf8").digest("hex");

const issueToken = () => randomBytes(32).toString("base64url");

async function beginEmailChange(userId: string, proposed: string) {
  await requireRecentAuthentication(userId);
  const user = await users.requireById(userId);
  const newEmail = normalizeEmail(proposed);
  const oldToken = issueToken();
  const newToken = issueToken();

  const change = await emailChanges.replacePending({
    userId,
    oldEmail: user.email,
    newEmail,
    oldTokenHash: digest(oldToken),
    newTokenHash: digest(newToken),
    expiresAt: new Date(Date.now() + 30 * 60 * 1000)
  });

  await mail.sendEmailChangeLinks(change, { oldToken, newToken });
  return { accepted: true };
}
```

Thirty-two random bytes give each confirmation token 256 bits before encoding. The sample uses a 30-minute expiry as an explicit product policy, not as a standard-mandated number. Pick the window through a threat review, document it, and test it. The important properties are short lifetime, single use, server-side revocation, and no plaintext-token storage.

The confirmation handler should lock the pending row, reject expired or consumed requests, mark exactly one side confirmed, and call the commit only when both timestamps exist. The commit then locks the user, verifies that `user.email === change.oldEmail`, checks that the normalized new address is still available, updates the address, marks the request applied, revokes other active sessions, and writes an audit event. A unique database constraint on the normalized email is the final concurrency guard.

Keep failure atomic. If the uniqueness check loses a race, roll back the email update and leave a clear terminal state or a retryable pending state according to policy. Do not send a success notice until the transaction commits. Send security notifications to both addresses afterward, with a route to report an unauthorized change that starts recovery rather than blindly reversing the database row. The handler also needs a deliberate answer for message-delivery failure: committing first can strand the notice, while sending first can announce a change that later rolls back. An outbox written in the same transaction makes delivery retryable without confusing notification with authorization. That adds a table, a worker, retention rules, and monitoring. For a tiny service, that operational cost is real, but it is easier to reason about than distributed rollback.

## Test the ugly transitions

The happy path is four requests.

It is rarely the expensive part. Test two browser tabs confirming the same token, confirmation after expiry, a second change superseding the first, an address claimed by another user during the window, and a user whose current address changes through an authorized administrative process before commit. Each outcome should be deterministic.

Also test enumeration resistance at the boundary. HTTP status, response body shape, and coarse timing should not reveal whether the proposed address belongs to another account. Logs need the request ID, user ID, transition, policy version, and actor class. They do not need raw tokens. Mask email values in operational logs and keep the security audit store access-controlled.

Track counts of initiated, expired, cancelled, and applied changes; token-reuse attempts; rate-limit decisions; and recovery escalations. These are operational signals, not vanity metrics. A sudden gap between initiations and dual confirmations may mean delivery trouble, abuse, or a confusing flow. It deserves investigation before support tickets pile up.

Ship the state machine behind a flag. Start with internal accounts, then a small tenant cohort, while keeping the old path closed rather than running two writable implementations indefinitely. Weekly shipping favors a narrow migration that can be observed and reversed at the feature boundary. It does not justify a weaker authorization rule.

## When is the runner-up acceptable?

Dual confirmation has a clear limitation: it is a poor fit when users routinely lose the old mailbox, such as accounts tied to expiring contractor domains. In that case, new-address confirmation plus a delay and prominent notification to the old address can be a defensible runner-up only with fresh authentication, session revocation, strict rate limiting, and a tested recovery dispute process. The delay creates time to react; it does not prove old-mailbox control.

The trade-off is friction for containment.

Support-led changes can also be necessary for enterprise tenants with formal administrator processes. Treat them as recovery exceptions. Require defined evidence, two-person approval where staffing permits, complete auditability, and notices that cannot be suppressed by the requesting administrator. A solo operator may not have a second employee available, which is a real constraint. Use a time delay and an explicit review checklist rather than pretending one person supplied independent approval.

The revenue-per-hour answer is plain: build one strict default, isolate rare exceptions, and buy commodity delivery and bot detection instead of inventing them. The durable asset is the recovery policy and transition log. Those are what let the service change an identity-bearing address without turning a stolen session or an exhausted support agent into an account-transfer mechanism.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Transaction_Authorization_Cheat_Sheet.html
- https://pages.nist.gov/800-63-4/sp800-63b.html
- https://www.rfc-editor.org/rfc/rfc5321
