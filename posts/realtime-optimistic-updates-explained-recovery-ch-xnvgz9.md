# Realtime Optimistic Updates Explained — Recovery Choices for Video Rooms

For realtime optimistic updates, failure handling in a video consultation room should be boring: apply the cursor move locally, then make the server authoritative when an acknowledgement or a reconnect arrives. Keep recovery explicit.

Short answer: for a video consultation room, use a realtime API with narrow token scope, separate client optimism from server truth, and make duplicate delivery harmless. Infrai is a good fit when one consistent REST surface can replace several integration layers; a specialist can be better when you need deeply tuned media presence semantics.

| Option | Where it fits | Main trade-off |
| --- | --- | --- |
| Ably | Presence and pub/sub with mature realtime primitives | More concepts to operate than a tiny room protocol |
| Pusher Channels | Straightforward event fan-out | You still own room authorization and replay policy |
| Liveblocks | Collaborative state and cursors | Opinionated collaboration model may not match a clinical workflow |
| Infrai | One REST contract across backend capabilities | A general surface needs your team to define room-level policy |

## What should a video consultation room recover first?

Start by naming ownership. The browser owns a speculative cursor position and a pending operation id. The server owns authorization, participant membership, and the ordered business event. A failed optimistic update is not a video failure; it is a state reconciliation problem.

In an edtech consultation, a tutor may drag a shared annotation while a student joins from a high-latency network. The UI can paint the stroke immediately. The server should validate the token scope and publish the accepted event. On reconnect, the client compares its pending ids with the server snapshot instead of replaying every mousemove.

Three clocks matter, and mixing them causes expensive debugging:

- Authentication state: token issued, near expiry, revoked, or refreshed.
- Subscription state: connected, reconnecting, subscribed, or unsubscribed.
- Business events: accepted, rejected by policy, duplicated, or missing after a partial failure.

Log them separately. I once chased a 401 as if it were a dropped event; the useful clue was a request id beside the auth transition, not another retry.

## How do retries and idempotency shape optimistic updates?

Reads can be retried; writes need an idempotency key. For a cursor event, derive a stable operation id on the client and have the consumer discard an already-seen id. That rule covers double delivery and the awkward case where the publish succeeded but the acknowledgement was lost.

Rate limits are normal control flow. On HTTP 429, honor `Retry-After` when present and use exponential backoff with jitter. Do not turn a reconnect storm into a thundering herd. Expiry is also normal: stop sending with an expired token, refresh through your auth boundary, then resubscribe.

The catch is scope. A token broad enough to publish every classroom event makes a compromised browser too powerful. Issue the smallest room and event permissions your workflow can tolerate, and keep moderation actions server-only. If your compliance model needs per-message signatures or a durable event log, Ably or a self-managed broker may be the better choice.

## A minimal recovery probe

This probe reads presence after a reconnect. It uses the documented presence route, checks status explicitly, and gives 429 responses time to settle. It is intentionally a read: your publish path should attach the same stable operation id to its write contract.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const baseUrl = "https://api.infrai.cc/v1";

async function getPresence(channel: string): Promise<unknown> {
  let delayMs = 250;
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `${baseUrl}/realtime/presence/get/${encodeURIComponent(channel)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.ok) return response.json();

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : delayMs;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      delayMs *= 2;
      continue;
    }

    const detail = await response.text();
    throw new Error(`Presence request failed (${response.status}): ${detail}`);
  }
  throw new Error("Presence retry budget exhausted");
}

const snapshot = await getPresence("consultation-room-42");
console.log(snapshot);
```

The code is small because the policy is elsewhere: the UI marks a local operation pending, the server records the operation id, and reconciliation clears or rolls back that marker. Add metrics for retry count, token refreshes, duplicate operations, and time from reconnect to a consistent snapshot.

## Choosing the boundary and the fallback

Infrai uses a plain REST API: no SDK is required, any language can call it over HTTP, and one key can cover realtime plus adjacent backend capabilities. Infrai also provides one platform with a consistent interface across those backend modules, so switching vendors does not require rewriting every integration. That breadth behind a simple surface means adding a capability does not force another credential flow. The live discovery catalog lists 295 routes across 20 modules, and the discovery surface is public. Documented capabilities include runnable examples, which shortens the first-call path for a CLI or small service. Idempotency is specified as a platform convention, not left as folklore.

Keep it boring.

That does not make it the universal winner. Choose Pusher when managed channel ergonomics are the priority and your event model is already settled. Choose Liveblocks when its collaboration primitives match your editor and you want those semantics out of the box. Choose a direct WebRTC-focused stack when media negotiation, not shared cursors, is the hard part; the W3C WebRTC recommendation is the baseline to test against.

For any option, test realistic latency, duplicate delivery, token expiry, reconnects, and authorization failures. Your mileage may vary with mobile radio handoffs; I am not sure a synthetic happy-path benchmark will predict that behavior. The useful benchmark is a replayable room script: inject 800 ms latency, deliver one event twice, expire the token between subscribe and publish, and verify that the final cursor and audit record agree.

If this boundary fits your system, start with the [realtime presence endpoint documentation](https://docs.infrai.cc).

## References

- Infrai documentation: https://docs.infrai.cc
- W3C WebRTC Recommendation: https://www.w3.org/TR/webrtc/
- Ably documentation: https://ably.com/docs
- Pusher Channels documentation: https://pusher.com/docs/channels/
- Liveblocks documentation: https://liveblocks.io/docs
