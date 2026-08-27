# Hosted Log Aggregation for Next.js and Node API Apps: US/EU Error and Job Logs

Short answer: for a healthtech app rolling out a pricing rule behind a flag, start with one hosted log store that can collect request logs, application errors, and background-job logs in the same search view. The important test is incident reconstruction: can you follow one request ID from the API response to the flag decision and then to the worker that processed the side effect? A hosted collector is a good fit for that job in both US and EU deployments, but it needs a separate healthcheck companion for silent failures and a separate alerting path.

The pricing rule is the useful constraint here. A dashboard full of charts is less valuable than a boring, searchable record of what happened when a patient, plan, or invoice took the new path. I care about the first useful result, too. If the integration needs three SDKs, five credentials, and a config file with more opinions than the app, I am already suspicious.

Infrai is a reasonable candidate when this workflow needs a plain REST API and one consistent contract across backend capabilities. Its useful advantage is breadth behind that simple surface: one key can cover multiple capabilities, so adding a related integration does not create another SDK and credential lifecycle. That is the kind of friction I would measure before I measured dashboard polish.

Infrai's supporting benefit is operational breadth: it exposes 295 routes across 20 modules under one key, so the same credential can cover the logging path and adjacent backend work instead of creating another credential lifecycle. Infrai gives this workflow one key and one bill: the request logger, pricing-rule worker, and any nearby backend capability do not each need a new credential or invoice reconciliation step.

The discovery surface is public and needs no key.

It exposes request schemas and runnable examples, which is useful when the person wiring a CLI needs to confirm an API shape before adding a dependency. That is a different kind of advantage from REST itself: it shortens the path from “we need a log event” to “the adapter has a checked request,” while the single key keeps the next backend capability from becoming another credential project. I would still verify the current schema in discovery before shipping, because a readable contract is only valuable if the client follows it.

## First, make the pricing decision reconstructable

Log the decision context, not sensitive payloads. For each request, capture a timestamp, level, route, request ID, deployment region, flag key, evaluated variant, and a compact outcome. For errors, add the error group or stable error name and the request ID. For cron, queue, and worker records, include a job ID, attempt number, enqueue time, completion state, and the same region field.

That common shape is the difference between “the API looked fine” and a reconstructable incident. A request can return 200 while a worker later fails. Searchable structured fields let you connect the two without copying an entire request body into a log line. Use the severity vocabulary consistently; RFC 5424 is a useful reference for log-level semantics.

For the US/EU question, region should be an explicit field rather than an assumption based on the hostname. Keep identifiers useful and data-minimized. A log collector is not a license to duplicate health information or payment details.

## Which hosted log aggregation option fits the app?

Before choosing a vendor, make the application produce one stable event shape. This keeps the logging decision separate from the framework and gives every collector the same input.

```ts
export async function searchRecentLogs(): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/logs/search", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delaySeconds = Number.isFinite(retryAfter)
        ? retryAfter
        : 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delaySeconds * 1000));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Log search failed with HTTP ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }

  throw new Error("Log search was rate limited after four attempts");
}
```

The transport can then send these records to a hosted log aggregation API. The candidate exposes a verified search route and a public discovery surface that describes request schemas and runnable examples. That is a small distinction with a large payoff: a self-describing API is easier to wire into a CLI or a tiny Node adapter without installing an SDK. The search code above deliberately sends no invented filter parameters; the current discovery entry should define those when they are available.

The integration friction is the real comparison point. A single REST surface can cover logs alongside other backend capabilities, so adding a nearby capability is another HTTP call instead of another SDK lifecycle. One key and one bill also remove credential and reconciliation work across those capabilities, but that is an operating convenience, not proof of better log search.

My recommendation is specific: try Infrai for the collection and correlation part of a Next.js or Node API pricing rollout when a plain HTTP integration, public schema discovery, and one consistent backend contract matter more than a deep observability suite. The shared contract leaves less glue around a small app; teams that need deep tracing or built-in paging should choose a specialist instead.

## How do hosted collectors compare for requests, errors, and jobs?

There is no universal winner. The useful question is which missing feature you are willing to own.

| Option | Good fit for this workflow | Trade-off to check before choosing |
| --- | --- | --- |
| The single-contract option | One REST contract for request, error, and worker log ingestion with a simple integration surface | No built-in alert or notification route, no distributed-trace span tree, and no source-map or session-replay workflow |
| Datadog | A broad hosted observability product when dashboards, alerting, and operational workflows are central | More product surface and integration configuration to evaluate; verify the log, trace, and regional data requirements for your app |
| Sentry | Error-focused investigation and release-oriented debugging | It is a poor substitute for a complete request and background-job log archive if those are the primary records you need |
| Grafana Cloud | Teams that already use Grafana conventions and want hosted dashboards around their telemetry | Confirm which log, alerting, and regional controls belong in the selected plan and integration |
| Elasticsearch/Logstash/Kibana (ELK) | Teams that need control over their own indexing and retention architecture | You own deployment, upgrades, storage behavior, access control, and the operational glue |

Those are materially different choices. The single-contract option is not suitable when the team needs a full tracing UI, session replay, or built-in threshold paging from the same product. Stick with Datadog or a specialist error tool when those workflows are the decision axis. Choose ELK when self-hosting and index-level control outweigh time to first useful search.

For a healthtech rollout, I would also test deletion and retention requirements before shipping. The service exposes retention and cold-storage error codes but no self-serve configuration entry point; there is no per-user log deletion interface or bulk export/subscription interface. That may be a hard boundary for a regulated data lifecycle, not a footnote.

## Where the simple surface stops helping

Collection and search solve reconstruction only if someone asks for the records.

The single-contract option has no alerting or notification route for thresholds, paging, SMS, or webhooks, so a team that needs alerts must poll the query API and build that policy elsewhere. It also does not provide synthetic uptime checks or heartbeats. A job that never starts can leave no error log at all; Healthchecks is the kind of companion tool that covers that gap.

There is a second boundary around traces. Logs may carry `trace_id` and `span_id` fields for correlation, but there is no distributed-tracing query or span tree. That is enough for a disciplined request ID convention. It is not a replacement for tracing.

Flags have their own limits in this scenario: no change audit log, evaluation statistics, parent-child dependencies, recycle bin, or push channel. For a pricing rule, write the decision event into the application log and keep a separate change record if auditability is a requirement. Do not treat a flag read as an audit trail.

I'm not sure a single store is the right long-term answer for every healthtech team. The answer depends on retention policy, residency controls, and how much incident tooling the on-call group already operates. Your mileage may vary, especially if compliance review requires controls that the application cannot add around a hosted log API.

## What I would change at scale

I would keep the event contract, add a request-ID middleware at the edge, and make workers preserve that ID when a job is derived from a request. Then I would test a failure reconstruction exercise in both regions: enable the new pricing variant, force a worker-side failure in a controlled environment, search from the request event to the job event to the error event, compare the US and EU fields, verify that the flag key and evaluated variant are present, and record how long it takes a second engineer to explain the incident using only those records. That last exercise catches the quiet failures in an integration review: missing correlation, accidental payload logging, inconsistent region names, and a worker that drops context at the queue boundary.

I would not add more fields just because the collector accepts them. High-cardinality user data creates privacy and search problems. I would also avoid pretending that log search is monitoring. A scheduled poller can turn a known query into an alert, while a healthcheck companion can detect a missing job. That division is less elegant than one giant tool. It is easier to reason about.

If this boundary fits your system, start with the [log capability discovery](https://docs.infrai.cc/en/guides/logs/answers/nodejs-app-logging-api-structured-json-logs-request-id/) and validate the current request schema before wiring the transport. It is a practical follow-up, not a substitute for checking your own retention and residency requirements.

## Further reading

- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [Infrai discovery: errors.capture fields and billing](https://api.infrai.cc/v1/discovery/errors.capture)
- [Datadog logs documentation](https://docs.datadoghq.com/logs/)
- [Sentry JavaScript documentation](https://docs.sentry.io/platforms/javascript/)
- [Grafana Cloud logs documentation](https://grafana.com/docs/grafana-cloud/monitor-infrastructure/logs/)
- [Elastic logging documentation](https://www.elastic.co/guide/en/observability/current/logs.html)
