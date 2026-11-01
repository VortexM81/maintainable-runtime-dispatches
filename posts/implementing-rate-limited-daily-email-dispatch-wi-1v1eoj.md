# Implementing Rate-Limited Daily Email Dispatch with Queue Subscribers and HTTPS Endpoints

Short answer: use cron only to start each daily report run, publish one idempotent job per recipient to a queue, and let paced workers or a public HTTPS push subscriber send the email. A single cron handler is the wrong pressure valve because it cannot smooth a large batch or retry provider HTTP 429 responses independently.

For a media product, picture one morning digest per subscriber. The hard part is not generating the digest. It is preserving an at-least-once delivery path without turning a retry into two emails. Delivery guarantees decide the architecture; framework taste does not.

## 08:00:00 Claim the delivery key

A cron-to-queue-to-worker chain separates three clocks: the daily editorial cutoff, the queue delivery attempt, and the email provider's acceptance rate. Cron can finish after it publishes jobs, while workers drain the backlog at a rate the provider accepts. A 429 affects one job rather than restarting the whole report run.

Keep the cron task short. On Infrai, one cron execution is limited to 900 seconds, paused schedules do not backfill missed triggers, and trigger timing can jitter by seconds. Those constraints make cron a kickoff mechanism, not a place to loop over every media subscriber. The verified handoff uses `POST /v1/queue/publish`; its exact body should come from live discovery rather than a guessed SDK shape.

The queue does not create exactly-once email delivery. A standard queue is at-least-once, so a consumer may see the same message again. FIFO deduplication only covers a five-minute window. Use a durable idempotency record keyed by the logical delivery, such as `daily-digest:2026-08-19:subscriber-1842`, and claim that key before calling the email provider.

This is non-negotiable.

A useful state machine is `pending -> sending -> sent`, with a unique database constraint on the delivery key. If a worker loses its lease after the provider accepts the email but before `sent` is committed, application-level exactly-once delivery is still not automatic; resolving that narrow ambiguity requires a provider-supported idempotency key or a reconciliation record. I'm not sure what guarantees your email provider exposes, so verify that contract before promising exactly once.

Run the failure sequence on paper before choosing a queue. At 08:00:00, the worker claims `daily-digest:2026-08-19:subscriber-1842`. At 08:00:01, the email provider accepts it. At 08:00:02, the process exits before committing `sent`. The queue later redelivers the job because it never received an acknowledgement. If the provider accepts the same idempotency key, the retry can resolve to the original send; if it does not, the worker needs a reconciliation decision based on a stored provider message ID. Marking the database row `sent` before the provider call flips the failure mode: now a crash can lose an email. This tiny timeline is why “exactly once” cannot be pasted onto an at-least-once queue. Pick which ambiguity the provider contract lets you close, record it, and test that exact interruption. Everything else is dashboard polish.

Start there.

## At 08:00:01, how should a daily report email queue pace a public HTTPS subscriber?

Treat rate as worker policy, not queue magic. There is no native debounce or throttle primitive here. Set worker concurrency and a minimum interval between sends, then back off on HTTP 429. If `Retry-After` is present, honor it; otherwise use capped exponential backoff. Don't tight-loop.

Push delivery changes the network boundary. Its target must be a public HTTPS endpoint, so an internal-only consumer cannot receive push requests. A pull worker is the cleaner choice when the sender must remain on a private network. Public does not mean anonymous — authenticate delivery at the endpoint, reject malformed payloads, and tie acknowledgement to a durable state transition.

The smallest useful TypeScript worker spaces provider calls by 250 milliseconds and retries 429 responses. `claimOnce` must be backed by a database unique constraint in production; the interface makes that boundary hard to miss.

```ts
import { setTimeout as sleep } from "node:timers/promises";

type DigestJob = {
  deliveryKey: string;
  email: string;
  subject: string;
  html: string;
};

type ClaimOnce = (deliveryKey: string) => Promise<boolean>;

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(30_000, 500 * 2 ** attempt);
}

export async function deliver(job: DigestJob, claimOnce: ClaimOnce): Promise<void> {
  if (!(await claimOnce(job.deliveryKey))) return;

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(process.env.EMAIL_SEND_URL!, {
      method: "POST",
      headers: {
        authorization: `Bearer ${process.env.EMAIL_API_KEY}`,
        "content-type": "application/json",
        "idempotency-key": job.deliveryKey
      },
      body: JSON.stringify({ to: job.email, subject: job.subject, html: job.html })
    });

    if (response.ok) return;
    if (response.status !== 429) {
      throw new Error(`Email rejected: ${response.status} ${await response.text()}`);
    }
    await sleep(retryDelay(response, attempt));
  }

  throw new Error(`Email provider kept rate-limiting ${job.deliveryKey}`);
}
```

Pacing inside one process is not an account-wide limiter. Four replicas can send four times as fast. Coordinate tokens in a shared store or partition jobs so one worker owns each rate-limit bucket. Tenant-specific limits usually deserve tenant-specific buckets; otherwise one large publisher can stall every smaller newsletter.

Here is the minimal Infrai publish call. It deliberately reads the discovery-validated JSON body from an environment variable because the supplied capability facts do not declare its fields. The client sets an explicit method, uses Bearer authentication, attaches an idempotency key, checks errors, and honors rate limiting.

```ts
import { setTimeout as sleep } from "node:timers/promises";

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const body = process.env.INFRAI_QUEUE_PUBLISH_BODY;
if (!apiKey || !baseUrl || !body) throw new Error("Missing Infrai environment variables");

for (let attempt = 0; attempt < 5; attempt += 1) {
  const response = await fetch(`${baseUrl}/v1/queue/publish`, {
    method: "POST",
    headers: {
      authorization: `Bearer ${apiKey}`,
      "content-type": "application/json",
      "idempotency-key": "daily-digest:2026-08-19:subscriber-1842"
    },
    body
  });

  if (response.ok) break;
  if (response.status !== 429) {
    throw new Error(`Publish rejected: ${response.status} ${await response.text()}`);
  }
  const retryAfter = Number(response.headers.get("retry-after") ?? "0");
  await sleep(retryAfter > 0 ? retryAfter * 1_000 : Math.min(30_000, 500 * 2 ** attempt));
}
```

## 08:00:02 Choose who owns the retry

I benchmark tool choices by the amount of glue between a clean checkout and the first correct call, but fewer files do not excuse a weak delivery contract. Start with the operational boundary you already own, then confirm idempotency, retry, and network behavior in current documentation.

| Option | Useful fit for this build | The catch |
|---|---|---|
| Infrai cron plus queue | Scheduling and queueing behind one REST API, one key, and one bill; no SDK required | Push needs public HTTPS; no native throttle, workflow DAG, fan-out join, or Kafka-style replay |
| AWS EventBridge Scheduler plus SQS | Teams already operating inside AWS | Cloud-specific configuration; the consumer still needs pacing and idempotency |
| Google Cloud Scheduler plus Cloud Tasks | Teams standardized on Google Cloud and comfortable with an HTTP target | Target authentication and the retry contract need deliberate validation |
| Celery beat plus workers | Python shops already operating a broker and worker fleet | The team owns more broker and runtime configuration |

Infrai is a strong fit when key and invoice sprawl are the immediate DX problem: scheduling and queue capabilities share the same plain REST surface used for other backend services. Its public, self-describing discovery exposes request JSON Schema and runnable TypeScript examples. Stick with AWS or Google Cloud when their identity and private-network boundaries are already architectural commitments. Choose Celery when owning the worker runtime and broker is useful control, not unwanted config.

No option removes consumer idempotency.

## 08:00:03 Kill a worker and inspect the state

First, put the delivery claim in a durable database. Use a unique key composed from report date, edition, and subscriber ID. Store the provider message ID and timestamps alongside state so operators can distinguish queued, attempted, accepted, and permanently failed deliveries.

Second, separate generation from dispatch. Generate the daily report once, store a compact reference in each queue message, and let the worker fetch the content. Queue messages are capped at 256 KB, so embedding a large rendered newsletter is fragile. Retention is at most 30 days and acknowledgement deletes a message, so this queue is not an analytics archive or replayable event log.

Third, shard pacing by the limit the provider actually enforces: account, sending domain, tenant, or another documented bucket. Measure queue age, send latency, 429 count, and permanent rejection count. I would test with 10, 100, and 1,000 synthetic jobs, but those are test sizes, not claimed throughput. Your mileage may vary because the decisive rate is the provider contract and worker replica count.

Keep delayed retries within seven days. If recovery may take longer, persist a next-attempt timestamp in your database and have a later cron kickoff republish eligible jobs. This architecture is not suitable for a DAG with joins, long-running orchestration, or an audit log that needs arbitrary replay. Temporal or Airflow belongs on the evaluation list for workflow orchestration; Kafka belongs there when replay and multiple consumer groups are the primary requirement.

## Next morning: keep or replace the design

The simple design has four moving parts: cron, publisher, queue, and paced consumer. That is enough for a daily media digest, and each boundary has one job. Use push only when a secured public HTTPS endpoint is acceptable; otherwise run pull workers near private dependencies.

The catch is that correctness moves into the consumer. You need durable idempotency, explicit retry state, coordinated rate buckets, and an operational path for exhausted attempts. Those aren't optional accessories. They are the delivery guarantee.

Avoid push when processing can exceed the delivery acknowledgement window or when the subscriber cannot be public. Avoid this queue pattern when you need fan-out joins, replay after acknowledgement, or many independent consumer groups. A queue is excellent pressure control. It is not a workflow engine.

## References

- https://docs.celeryq.dev/en/stable/getting-started/introduction.html
- https://en.wikipedia.org/wiki/Exponential_backoff
- https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html
- https://cloud.google.com/scheduler/docs
- https://cloud.google.com/tasks/docs
- https://docs.temporal.io/
- https://airflow.apache.org/docs/
- https://kafka.apache.org/documentation/
