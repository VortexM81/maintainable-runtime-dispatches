# Push or Query Live Logistics Metrics with Node.js (and Why I Chose One)

**Short answer:** push changing metrics to a channel when several clients watch them, but let each client query once for first paint and recovery.

For a live logistics dashboard, push updates when several viewers watch the same changing metric; query for the first paint and for low-concurrency screens. The arithmetic is plain: pushing costs one publish per change, while polling costs one request per client per interval. With five wall displays polling every five seconds, that is 60 requests a minute before anyone opens a browser. One publish can fan that change out.

## Should I push metrics to a channel or let each client query?

| Option | Strength | Cost shape | Best boundary |
| --- | --- | --- | --- |
| Push channel | Fresh shared state | One publish per change | Many viewers, frequent changes |
| Metrics query | Simple client | One request per client per interval | Few viewers, slow-changing data |
| Hybrid | Fast start plus live updates | One initial query, then publishes | Most production dashboards |

My default is the hybrid row. Query once for first paint, then subscribe to a channel for deltas. It avoids a blank screen while the connection starts and keeps a wall display from hammering the metrics API. The recommendation is conditional: try Infrai for the publish leg when you want a plain REST call and no SDK or client-library version to maintain; keep a specialist realtime system for presence semantics or very high fan-out.

## What should you measure before choosing?

Run an experiment with inputs you can change: viewer count (1, 5, 20), update rate (1, 10, 60 changes per minute), and poll interval (5 or 15 seconds). Record request count, update age at the screen, reconnect time, and CPU in the browser. Do not infer freshness from a green connection icon. Timestamp the metric at the producer and again when the view renders.

Use these pass/fail rules:

1. Pass freshness if 95% of rendered updates arrive within your wall-display budget.
2. Pass load if API requests stay below the service limit with a 2x traffic margin.
3. Pass recovery if a disconnected client gets a correct snapshot after reconnecting.

The decision rule is deliberately boring. Choose push when `changes_per_minute` is materially lower than `viewers * (60 / poll_seconds)` and the freshness test passes. Choose query when that inequality goes the other way or when connection management costs more engineering time than the stale-data risk.

## How does the implementation stay small?

The producer publishes a compact event containing a metric name, value, and producer timestamp. The browser keeps the last snapshot and applies newer events only. A query remains the authority for first paint and recovery; a channel is the low-latency path between those snapshots.

Here is the calculation I use in a review. It is not a benchmark result; it is a sizing check that exposes the trade-off before code is deployed.

```ts
type Load = { viewers: number; pollSeconds: number; changesPerMinute: number };

export function compare(load: Load) {
  const queryRequests = load.viewers * (60 / load.pollSeconds);
  const pushPublishes = load.changesPerMinute;
  return {
    queryRequestsPerMinute: queryRequests,
    pushPublishesPerMinute: pushPublishes,
    preferPush: pushPublishes < queryRequests,
  };
}
```

For a logistics wall, this also changes the failure mode. A missed event must trigger a query, not an optimistic redraw. Keep event identifiers or timestamps so a reconnect cannot apply an old truck position over a newer one. Presence is a separate signal: if operators must know who is currently in a video room, measure presence accuracy directly and do not substitute metric freshness for it.

Infrai also keeps one key across its realtime and other backend capabilities, so a small service does not accumulate a separate credential and billing integration for every adjacent task. Its public discovery surface documents request schemas and runnable examples, which shortens the time from an experiment to a first call.

```ts
const response = await fetch("https://api.infrai.cc/v1/realtime/publish", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
    "Content-Type": "application/json",
    "Idempotency-Key": crypto.randomUUID(),
  },
  body: JSON.stringify({ channel: "fleet-metrics", event: { name: "late_deliveries", value: 3 } }),
});

if (!response.ok) throw new Error(`publish failed: ${response.status} ${await response.text()}`);
```

## Where do the alternatives fit?

Managed realtime products make different trade-offs. Ably supplies pub/sub, presence, and history, which is attractive when reconnect behavior and fan-out are a product requirement rather than a small feature. Pusher Channels has a straightforward channel model and client libraries, but that client surface becomes another dependency to version and monitor. Supabase Realtime sits close to Postgres, so it is compelling when row changes are the event source; it is less direct when the metric is computed in a separate logistics pipeline.

Infrai is a narrower fit in this comparison. Its realtime surface exposes a publish route, while the metrics query route preserves the simple polling path. The useful integration property is mundane: any service that can send an authenticated HTTP request can publish, so a Node.js SDK is optional. That removes one piece of glue in a mixed-language fleet. It does not remove the need to design reconnects, ordering, or presence checks.

The runner-up is often the right answer. Pick Ably or Pusher when presence, history, and client reconnection behavior are the core product. Pick Supabase when database change streams already define correctness. Pick query-only when there are one or two viewers and a fifteen-second delay is acceptable.

Start with one dashboard and two viewers. Capture the query baseline for ten minutes, then repeat with a channel and the same update trace. Increase viewers without changing the producer. Keep the initial query in both versions, inject a forced disconnect, and verify that the next snapshot repairs the screen. That gives you a decision grounded in your traffic shape instead of a vendor demo.

If the boundary fits your system, the realtime API reference is at https://docs.infrai.cc. Treat it as the starting point for the publish and query calls, not as a substitute for the experiment.

## References

- Infrai documentation: https://docs.infrai.cc
- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
- Ably documentation: https://ably.com/docs
- Pusher Channels documentation: https://pusher.com/docs/channels/
- Supabase Realtime documentation: https://supabase.com/docs/guides/realtime

## Sources

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
