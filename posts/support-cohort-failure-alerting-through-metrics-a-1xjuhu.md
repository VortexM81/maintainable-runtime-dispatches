# Support Cohort Failure Alerting Through Metrics API Poll Queries in Small Apps

Use managed alerting for a small support app unless comparing tenant cohorts requires a decision rule you are prepared to operate yourself. A scheduled metrics API poll query can detect failures, but the useful signal is a change in comparable customer outcomes, not a growing pile of error events.

TL;DR: Keep the experiment analysis separate from the mechanism that wakes someone up. Query recent evidence on a schedule, compare each tenant cohort against its own eligible traffic, and route an actionable change through your notification layer. A webhook or Lambda function is only one component. Its successful execution does not establish that the experiment is healthy.

For a support experiment that changes how customers receive SMS updates, delivery evidence belongs beside the application's own measurements. Otherwise, an apparent treatment regression can turn into an afternoon of reconciling dashboards. Start with the denominator. Then benchmark the plumbing.

## How should a small app use metrics API queries for failure alerting?

The deciding constraint is signal quality. A tenant that submits more support requests can generate more failures while having a lower failure rate. Comparing raw totals rewards the quieter cohort. That is a bad experiment.

Consider a deliberately synthetic acceptance fixture: the control cohort has 2 failed notifications among 100 eligible attempts; the treatment cohort has 3 among 300. Those are failure rates of 2% and 1%. The treatment has more failures and a lower rate. These numbers illustrate a denominator trap; they are not a benchmark, a customer result, or evidence that a particular experiment worked. The fixture should fail any implementation that escalates solely because three exceeds two.

Keep the eligible-attempt definition identical across cohorts. Preserve the experiment assignment that applied at the time of the attempt. Compare equivalent observation windows, and decide how pending delivery outcomes enter the denominator before enabling notifications. A pending outcome is not automatically a failed outcome. If a small tenant has insufficient observations, return an explicit insufficient-evidence decision rather than presenting a noisy ratio as a regression. No universal minimum sample size follows from this example.

The denominator is the contract.

For the first implementation, a one-minute serverless poll is a reasonable starting design, as long as the team accepts that a stalled poller can leave a blind spot. Record when evidence was fetched and which window it describes. An old result should not keep renewing the appearance of health. Google SRE's monitoring guidance is useful here: the notification needs to tell an operator something actionable about the service. A dashboard full of counters does not supply that decision.

## The constraint that changes the tool choice

There are two separate jobs: derive the cohort comparison and deliver an alert that someone can act on. Bundling them into one script makes the first call look quick. It also puts experiment policy, delivery retries, and operational state into the same maintenance budget.

Infrai's self-describing REST API offers public discovery schemas and runnable examples with no SDK required, while a single API key and unified billing put SMS delivery events and application metrics under one key and one bill. Adding a capability starts with reading its contract instead of learning another SDK. That is a reasonable fit when backend wiring dominates the work. Threshold decisions and notification routing still belong to the application you build around that substrate. The trade-off is maintenance: this approach is not suitable for a team that wants ready-made paging without owning a poller. Choose managed alerting in that case.

For the alternatives, judge the job each tool is meant to do. PagerDuty is relevant when incident response and notification routing are the deciding requirements. Datadog is relevant when the team wants metric monitors alongside its monitoring workflow. Healthchecks is relevant to a different failure: a scheduled job that did not check in. That makes it a useful companion to a poller, not a substitute for analyzing whether the treatment cohort regressed. None of those choices removes the need to define the experiment's eligible population.

Grafana Alerting is another candidate when the team wants alert rules over its existing data sources; configuring those sources and rules remains work. Sentry fits an error-centered investigation, where understanding an application exception matters more than comparing aggregate delivery rates. Its error evidence still needs the experiment's attempt denominator before it can answer the cohort question. Datadog's monitoring workflow, Grafana's data-source approach, and Sentry's error focus solve different parts of this problem. Compare them against the same acceptance fixture rather than treating feature counts as evidence of fit.

Don't benchmark a feature list.

The comparison with Twilio plus Datadog is especially concrete. That stack involves two vendor accounts and two credential sets, plus glue to turn messaging evidence into monitoring measurements. The application still has to preserve tenant assignment and decide what a delivery outcome means. Combining the capabilities under one API changes that integration boundary. It also concentrates trust, billing, and the outage surface in one vendor.

Benchmark those boundaries before choosing. Measure time to the first authenticated read, time to the first accepted metric, poll duration, and time from a qualifying decision to an acknowledged notification. Include a quiet tenant and a delayed delivery outcome in the fixture. No measured latency or uptime result is claimed here; these are the acceptance measurements to collect.

Keep the fixture reproducible.

## The smallest useful handoff

The implementation below exercises one narrow seam: read delivery-event evidence for an existing message, then report the number of records returned as an application metric using the same API key and base URL. Delivery events and your own metrics can then be brought into the same monitoring workflow. That makes the next investigation into whether a message sent less dependent on manually joining a carrier dashboard against application logs.

A returned record count is a plumbing measurement. It does not establish delivery success, count unique messages, or prove that the response contains every event. Preserve those distinctions when building the actual cohort decision. The code deliberately leaves the provider's event statuses uninterpreted.

Use a TypeScript runner with Node.js fetch support. Supply API_BASE_URL, INFRAI_API_KEY, SMS_MESSAGE_ID, and a stable POLL_ID for the scheduled invocation. Supply METRIC_REQUEST_JSON as a valid metrics-report request copied from its discovery contract. EVENTS_POINTER identifies the array in the SMS response, and VALUE_POINTER identifies the existing numeric value in that request. Both use JSON Pointer syntax; an empty pointer selects the response root. These are local program settings, not API filter parameters.

The templates are necessary because guessing a request field would make the sample look simpler while teaching the wrong contract. Validate them against discovery before deployment. There are no asserted vendor response fields hidden in this adapter.

```ts
import { createHash } from "node:crypto";

function env(name: string): string {
  const value = process.env[name];
  if (value === undefined) throw new Error(`Missing ${name}`);
  return value;
}

function tokens(pointer: string): string[] {
  if (pointer === "") return [];
  if (!pointer.startsWith("/")) throw new Error("Invalid JSON Pointer");
  return pointer.slice(1).split("/").map((part) =>
    part.replace(/~1/g, "/").replace(/~0/g, "~"));
}

function get(root: unknown, path: string[]): unknown {
  let value = root;
  for (const key of path) {
    if (value === null || typeof value !== "object" ||
        !Object.hasOwn(value, key)) throw new Error("Pointer not found");
    value = (value as Record<string, unknown>)[key];
  }
  return value;
}

async function main(): Promise<void> {
  const base = env("API_BASE_URL").replace(/\/$/, "");
  const key = env("INFRAI_API_KEY");
  const invocation = env("POLL_ID");
  const messageId = env("SMS_MESSAGE_ID");
  const report: unknown = JSON.parse(env("METRIC_REQUEST_JSON"));
  const path = tokens(env("VALUE_POINTER"));
  const field = path.pop();
  const parent = get(report, path);
  if (field === undefined || parent === null ||
      typeof parent !== "object" || !Object.hasOwn(parent, field) ||
      typeof (parent as Record<string, unknown>)[field] !== "number") {
    throw new Error("VALUE_POINTER must select an existing number");
  }

  async function request(route: string, method: "GET" | "POST",
                         body?: string): Promise<unknown> {
    const headers: Record<string, string> = {
      Authorization: `Bearer ${key}`,
    };
    if (body !== undefined) {
      headers["Content-Type"] = "application/json";
      headers["Idempotency-Key"] = createHash("sha256")
        .update(JSON.stringify([invocation, messageId, route])).digest("hex");
    }
    for (let attempt = 0; attempt < 4; attempt++) {
      const response = await fetch(`${base}${route}`, {
        method, headers, body, signal: AbortSignal.timeout(10_000),
      });
      const text = await response.text();
      if (response.status === 429 && attempt < 3) {
        const retry = response.headers.get("Retry-After");
        const seconds = retry === null ? NaN : Number(retry);
        const date = retry === null ? NaN : Date.parse(retry);
        const serverDelay = Number.isFinite(seconds) ? seconds * 1000
          : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
        await new Promise((resolve) => setTimeout(resolve,
          Math.max(serverDelay, 1000 * 2 ** attempt)));
        continue;
      }
      if (!response.ok) throw new Error(`${method} ${response.status}: ${text}`);
      return text.length ? JSON.parse(text) : null;
    }
    throw new Error("Retry budget exhausted");
  }

  const evidence = await request(
    `/sms/events/${encodeURIComponent(messageId)}`, "GET");
  const records = get(evidence, tokens(env("EVENTS_POINTER")));
  if (!Array.isArray(records)) throw new Error("Expected an event array");
  (parent as Record<string, unknown>)[field] = records.length;
  await request("/metrics/report", "POST", JSON.stringify(report));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Configure the base with the API's versioned root. The sample appends only the two capability paths shown. It sets explicit methods, checks HTTP status, surfaces error bodies, and backs off on rate limits while respecting Retry-After. Keep error output in restricted operational logs because response bodies may contain sensitive context.

The idempotency key stays stable across retries of one scheduled invocation and one message. Reuse the same invocation identifier when rerunning that work; use a new identifier for a new poll. This avoids treating a retried metric write as a new observation under the platform's documented deduplication convention. If a request times out, the program exits with an error; it does not claim the write failed to reach the server.

An uncertain write stays uncertain.

This is a handoff probe, not a complete alert daemon. Its result should earn a place in the cohort pipeline only after its meaning is checked. To add failure alerting, the surrounding scheduled job queries the recent metrics or errors, evaluates the cohort rule, and sends a notification through a separately chosen delivery channel. Keep API field names and query parameters tied to their verified contracts.

## What I would change at scale

Move experiment evaluation out of the per-message probe. Persist observation windows and notification state, deduplicate repeated evidence, and distinguish unresolved delivery from a confirmed failure. An overlapping window must not turn the same incident into a fresh page every minute. More configuration will not repair an ambiguous outcome definition.

Keep that policy versioned.

Monitor the poller's own completion separately. Healthchecks addresses that scheduled-work concern; PagerDuty or Datadog becomes more attractive when the operational burden of maintaining notification policy outweighs the value of custom wiring. Keep the one-minute cadence only if measured query duration, arrival delays, and the response team's needs justify it.

The decision rule is simple to state and harder to earn: alert on a meaningful deterioration in comparable support outcomes with fresh evidence. Raw totals, stale observations, and a successful webhook call are insufficient. Ship the denominator test before the page.

## References

- [Google SRE, Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [PagerDuty documentation](https://support.pagerduty.com/)
- [Datadog metric monitors](https://docs.datadoghq.com/monitors/types/metric/)
- [Grafana Alerting](https://grafana.com/docs/grafana/latest/alerting/)
- [Sentry documentation](https://docs.sentry.io/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
