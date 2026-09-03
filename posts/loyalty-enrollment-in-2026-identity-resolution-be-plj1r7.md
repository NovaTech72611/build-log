# Loyalty Enrollment in 2026: Identity Resolution Before Customer Provisioning

Short answer: resolve a shopper against verified identity signals inside the same transaction that creates the loyalty user; send an existing match into account recovery, quarantine conflicting matches for review, and create a record only when the gate finds no owner. Keep the browser response identical across those branches so enrollment doesn't become an account-enumeration endpoint.

For an e-commerce loyalty program, the hard part isn't the signup form. It is deciding whether `alex@example.com`, a guest-checkout profile, and loyalty number `L-804219` belong to one person without merging two people who happen to share a household inbox. That decision also controls the forgot-password path and the evidence available during an audit.

Ship the gate first.

## How should loyalty identity resolution prevent duplicate user creation?

Treat identity resolution as a command with three outcomes, not as a fuzzy search followed by an unconditional insert. An exact match on a previously verified channel routes to recovery. A conflict between strong signals routes to review. Only a clean miss may continue to creation, and even that miss must be checked again at commit time because two tabs can submit together.

The account record should own authentication state. Orders, points, and marketing preferences should refer to that stable account ID rather than treating an email address as the primary key. Email addresses can change; a stable internal ID lets the recovery process replace a verified channel without rewriting the loyalty ledger.

| Evidence at the gate | Decision | User-facing result | Audit evidence |
|---|---|---|---|
| Verified email fingerprint matches one account | Start recovery for that account | Generic accepted response | Match type, account ID, challenge ID |
| Loyalty number and verified channel point to different accounts | Hold for support review | Generic accepted response | Both pseudonymous references, rule version |
| No strong signal matches | Create inside the transaction | Generic accepted response | Resolution miss, new account ID, rule version |
| Duplicate appears during commit | Use the winning account and route to recovery | Generic accepted response | Uniqueness conflict, winning account ID |

Don't auto-merge on name, postal address, device ID, or an unverified phone number. Those can help a reviewer find a candidate, but they don't prove control. A false negative creates a duplicate that support can reconcile later; a false positive can expose another customer's points, orders, or recovery path. The damage is asymmetric.

OWASP recommends generic authentication responses because response wording and timing can disclose whether an account exists. The same rule belongs here. `We accepted your request` is safer than switching between `account already exists` and `welcome`; the service can still send the rightful owner the appropriate recovery or enrollment message out of band.

## The constraint that changes the design

A lookup-before-insert sequence looks fine in a diagram and fails under concurrency. Request A sees no account. Request B sees no account. Both create one. The fix is a database-enforced uniqueness boundary plus a transaction that interprets a uniqueness collision as another resolution outcome, not as an unexplained signup error.

Consider a concrete checkout sequence. A shopper opens the loyalty enrollment link in a receipt email, then opens the same link on a phone before the first tab finishes. Both requests carry verified email fingerprint `fp_7c91` and loyalty number `L-804219`; each lookup returns no row at roughly the same moment. Without a constraint, the desktop request creates account `acct_301`, the phone creates `acct_302`, and later orders can accrue points to either record. The forgot-password form now has an impossible choice because the same verified channel appears to own two authentication records. With the gate, one transaction inserts `acct_301`. The other insert meets the unique boundary, reads `acct_301`, records a recovery outcome, and returns the same accepted envelope. The shopper sees no branch detail. Internally, the audit trail shows two correlation IDs, one creation, one recovery decision, and a single durable account. This example is intentionally narrow: the unique fingerprint proves control of a channel, not that two people with similar names are the same customer. If `L-804219` already pointed elsewhere, both requests would enter review instead of forcing a merge. That distinction is the safety property the architecture needs to preserve.

There is another constraint: the forgot-password flow must not become a weaker duplicate-account oracle. OWASP's authentication guidance calls for consistent messages and cautions that even different HTTP behavior or processing time can reveal account validity. Return the same `202` shape for new enrollment, existing-account recovery, and manual review. Do the variable work after that response, and rate-limit attempts by more than one dimension so rotating an email address doesn't erase all friction.

This is where an audit log earns its keep — but keep secrets out of it. Record a correlation ID, the rule version, the selected branch, pseudonymous identity references, the actor, and timestamps. Do not record reset tokens, raw passwords, or full recovery answers. The log should explain why the gate chose recovery over creation without becoming a second identity database.

A single operator shipping weekly should resist a large scoring engine here. The revenue-per-hour move is to outsource undifferentiated email delivery and keep the decision policy small enough to test locally: exact verified-channel match, explicit conflict, or clean miss. Your mileage may vary when fraud patterns or regional identity rules demand more signals, but complexity needs evidence.

## The smallest working gate

The following TypeScript keeps transport details out of the policy. `resolveOrCreate` runs in a transaction, and the store owns the unique index on `verifiedEmailFingerprint`. The email has already passed a proof-of-control challenge before this function runs.

```ts
type Enrollment = {
  verifiedEmailFingerprint: string;
  claimedLoyaltyNumber?: string;
};

type Account = { id: string };

type Resolution =
  | { kind: "recover"; account: Account }
  | { kind: "review"; candidateIds: string[] }
  | { kind: "created"; account: Account };

interface IdentityStore {
  transaction<T>(work: (tx: IdentityStore) => Promise<T>): Promise<T>;
  findByVerifiedEmail(fingerprint: string): Promise<Account | null>;
  findByLoyaltyNumber(number: string): Promise<Account | null>;
  insertAccount(input: Enrollment): Promise<Account>;
  appendAudit(event: AuditEvent): Promise<void>;
}

type AuditEvent = {
  correlationId: string;
  ruleVersion: "2026-01";
  outcome: Resolution["kind"];
  accountIds: string[];
};

async function resolveOrCreate(
  store: IdentityStore,
  input: Enrollment,
  correlationId: string,
): Promise<Resolution> {
  return store.transaction(async (tx) => {
    const byEmail = await tx.findByVerifiedEmail(
      input.verifiedEmailFingerprint,
    );
    const byNumber = input.claimedLoyaltyNumber
      ? await tx.findByLoyaltyNumber(input.claimedLoyaltyNumber)
      : null;

    let result: Resolution;
    if (byEmail && byNumber && byEmail.id !== byNumber.id) {
      result = { kind: "review", candidateIds: [byEmail.id, byNumber.id] };
    } else if (byEmail || byNumber) {
      result = { kind: "recover", account: (byEmail ?? byNumber)! };
    } else {
      result = { kind: "created", account: await tx.insertAccount(input) };
    }

    const accountIds =
      result.kind === "review"
        ? result.candidateIds
        : [result.account.id];

    await tx.appendAudit({
      correlationId,
      ruleVersion: "2026-01",
      outcome: result.kind,
      accountIds,
    });
    return result;
  });
}
```

The HTTP handler should not serialize `Resolution` back to the caller. It maps every successful policy branch to one neutral response, then dispatches the correct private notification.

```ts
type Accepted = { status: "accepted"; correlationId: string };

async function enroll(request: Enrollment): Promise<Accepted> {
  const correlationId = crypto.randomUUID();
  const resolution = await resolveOrCreate(store, request, correlationId);

  await dispatchPrivateNextStep(resolution, correlationId);
  return { status: "accepted", correlationId };
}
```

One detail stays below this interface: simultaneous inserts must converge. In a relational store, enforce a unique constraint on the fingerprint and translate a duplicate-key collision by reading the winning account inside a fresh transaction, then choosing `recover`. That is ordinary control flow for this command. Test it with two promises released by the same barrier; assert one account, two accepted responses, and two audit events.

Also test the unpleasant branches. A loyalty number owned by account A plus a verified email owned by account B must never merge automatically. Repeating the same request should not create more users. Browser-visible bodies should match byte for byte across creation and recovery, while a timing test should allow for noisy infrastructure without tolerating an obvious branch-specific delay.

## What changes when the program grows?

At larger scale, split proof collection from identity policy, version every rule, and put notification work on a durable queue. Add a review console with least-privilege access and an explicit merge operation that records who approved it. Keep the immutable points ledger separate from mutable profile data so an approved account link doesn't rewrite earning history.

The catch is that exact-match gating is not suitable when one person is expected to own several program identities, when families intentionally share an address, or when phone-number reassignment is common in the target market. In those cases, keep ambiguous identities separate and invest in a reviewed linking flow. A probabilistic identity graph may help rank candidates at very high volume, but it should not silently authorize password recovery or a points transfer.

There is a cost to the conservative design: more support reviews and some duplicates survive. That is preferable for an early e-commerce product because a review queue is visible and reversible, while an incorrect merge crosses an authorization boundary. Revisit the threshold using observed review volume, confirmed merge errors, and recovery completion data — never by chasing a cleaner-looking account count.

For a solo SaaS, this is enough to ship. The valuable artifact isn't an elaborate matcher; it is a narrow gate with database enforcement, neutral responses, repeatable tests, and an audit trail that explains every account recovery path.

## References

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html

## Further reading

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
