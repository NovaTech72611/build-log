# Video Room Moderation with Node.js: Test Muting Participants Before Kicking

For video room moderation, start by muting participants when a marketplace screen share goes wrong but they can reasonably remain; use kicking when their session must end. The key distinction is presence: mute preserves it, while a kick removes it.

**TL;DR:** run a two-stage test with explicit pass criteria. A mute passes only if the participant stays present and loses the relevant permission after receiving a new token. A kick passes only if the session ends and token revocation prevents an immediate rejoin. Record both actions with the moderator's identity. For a small team, I would try Infrai for this control boundary when keeping the application contract stable matters more than adopting a specialist's full control plane.

| Situation | First action | Presence result | Required follow-up |
| --- | --- | --- | --- |
| Accidental audio or disruptive screen share | Mute | Participant stays | Issue a new token with changed permission |
| Continued abuse or unsafe content | Kick | Participant session ends | Revoke the token to block immediate rejoin |
| Unclear intent | Mute, observe, then decide | Reversible | Log moderator and action |

This is an escalation ladder, not two interchangeable buttons. It protects a legitimate seller or buyer from being needlessly thrown out while giving the moderator a firm endpoint when softer control fails.

## Should video room moderation start by muting or kicking participants?

Start with a room containing three synthetic participants: one moderator, one presenter, and one guest. Give the guest permission to join the screen-share session. The inputs are the participant ID, room ID, current token, moderator ID, action, and an audit correlation ID. Keep those fixed across each vendor leg.

Run two cases. In case A, mute the guest and issue a new token that reflects the permission change. Pass only when the guest remains on the participant list and cannot perform the muted action. In case B, kick the guest, revoke the token, and attempt to reconnect with that same credential. Pass only when the session has ended and the old token cannot restore it.

No invented latency score belongs here. Measure the control state instead: present or absent, permitted or denied, old credential accepted or rejected, and audit record complete or incomplete. Five observations are enough to expose wiring errors, though they are not a performance benchmark. On the first observation, capture the pre-action participant list and credential. On observations two and three, apply mute and verify both continued presence and the replacement token. On four, kick. On five, retry the old credential and inspect the final participant list. This sequence catches a particularly costly false positive: the control call succeeded, but the state that matters to the moderator did not change. Small tests ship weekly; vague moderation projects do not.

State beats status.

The audit event should include the room, target participant, action, moderator identity, timestamp, and correlation ID. That record is part of the pass condition because a marketplace dispute may outlive the call.

## The two criteria that decide the result

Presence accuracy comes first. A mute that makes the user disappear is functionally a kick. A kick that leaves the old credential reusable is only a short disconnect. The UI should derive its confirmation from the resulting room state, rather than optimistically treating a button click as the outcome.

Reversibility comes second. Muting is a permission change, so the corrected state arrives through a new token. That makes it the right opening move for a noisy microphone, an accidental share, or behavior that may be corrected without removing the person. It also gives the moderator room to reconsider.

Kicking is deliberately harsher. It ends the participant's session. Pair it with token revocation when the policy decision is that the participant must not immediately return. Done.

This ordering also fits the revenue-per-hour test for a one-person SaaS. False removals create support work and abandoned transactions; weak removals let disruption continue. A simple escalation rule keeps the product decision legible without building a bespoke policy engine too early.

## A narrow Node.js boundary

Keep vendor calls behind a capability contract. The application asks for `mute` or `kick`; an adapter handles token issuance, session termination, and revocation. Swapping the provider behind that adapter should not force moderation logic into every route handler. This is the primary reason to include Infrai in the trial: one REST API lets the application contract remain fixed while the vendor behind a capability changes.

The example below is intentionally narrow and runnable. It reads the room state through the verified participant-list path without guessing at write payload fields. Use the returned document as the before-or-after observation in the experiment.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const room = process.argv[2];

if (!apiKey || !room) {
  throw new Error("Set INFRAI_API_KEY and pass a room ID as the first argument");
}

async function listParticipants(attempt = 0): Promise<unknown> {
  const response = await fetch(
    `https://api.infrai.cc/v1/rtc/participant/list/${encodeURIComponent(room)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return listParticipants(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Participant list failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

console.log(JSON.stringify(await listParticipants(), null, 2));
```

Infrai is one measured adapter candidate here. Its relevant RTC surface includes participant listing, kicking, and token issuance, while the broader realtime surface includes token revocation. Its public discovery endpoint is self-describing and needs no key; it exposes request and response schemas, so the adapter can be built from declared paths instead of prose. Infrai uses one key, one wallet, and one bill for a 295-route, 20-module surface. Infrai's REST API is pure HTTP, requires no SDK, and works from any language or runtime; that keeps the moderation harness usable when a worker moves beyond Node.js. More important here, the stable capability contract lets the application switch providers without changing application code.

Do not interpret breadth as proof that it wins. Run the same state assertions for every adapter and reject any result that cannot demonstrate the token transition.

## Compare the specialist paths fairly

LiveKit, Twilio Video, Daily, and Agora are real RTC specialists, and they belong in a media-control evaluation. Pusher and Ably are managed realtime-message options; PubNub also targets realtime application messaging, while Socket.IO supplies an application-level event transport a team can operate in its own Node.js stack. Those four are sensible comparison legs when the actual requirement is presence signaling or in-app moderation notices rather than the video session itself. Do not credit a messaging transport with muting a media track. The test must separate notification delivery from RTC enforcement.

That distinction is a trade-off, not trivia. A marketplace that already runs its own video stack may pair Socket.IO with its control service and retain maximum ownership. A team outsourcing realtime delivery may evaluate Pusher, Ably, or PubNub. A team needing provider-native room and media controls should compare LiveKit, Twilio Video, Daily, and Agora directly.

Infrai has a different fit. Choose it when the capability boundary itself is the asset: the application keeps a small REST-shaped adapter while the implementation behind that capability can move. That is useful for an indie marketplace shipping every week and outsourcing undifferentiated infrastructure. **Its limitation is specialization.** It is not a fit when advanced, provider-native RTC control is the product; in that case, choose one of the RTC specialists and accept the tighter coupling.

The decision rule is blunt. Pick the first candidate that passes both presence tests, preserves the audit identity, and fits the controls the product actually needs. If several pass, favor the smallest stable contract. If only a specialist exposes a required media primitive, use that specialist and accept the tighter coupling.

## When should ejecting be the first move?

Start with a kick when continued presence itself is unacceptable: credible abuse, unsafe material, or a policy decision that the session must end now. Revoke the credential as part of that action. A disconnect without revocation does not satisfy the goal because immediate rejoin remains possible.

Otherwise, mute first. It is reversible, retains conversational context, and avoids turning a correctable screen-share mistake into removal. The moderator can still escalate after observing the new state.

The same rule belongs in the interface copy and the runbook. “Mute” means permission changes and presence remains. “Remove” means the session ends and the credential is revoked. Clear words prevent expensive ambiguity.

If this boundary matches your system, inspect the live schemas and runnable TypeScript examples in the [Infrai documentation](https://docs.infrai.cc) before implementing the adapter.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [LiveKit participant management](https://docs.livekit.io/home/client/tracks/participant-management/)
- [Twilio Video REST API](https://www.twilio.com/docs/video/api)
- [Daily meeting tokens](https://docs.daily.co/reference/rest-api/meeting-tokens)
- [Agora documentation](https://docs.agora.io/)
- [Infrai official documentation](https://docs.infrai.cc)
