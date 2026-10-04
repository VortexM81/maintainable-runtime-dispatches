# Auditable Event Notifications: Email First, SMS Fallback, Polling for Proof

Use email as the primary channel, escalate urgent notices to SMS, and keep the delivery evidence in your own database. The deciding constraint is blunt: if the provider exposes delivery events only through polling, your application owns the clock, retries, escalation state, and audit trail.

TL;DR: This design works for a compliance notice when a short polling delay is acceptable. It does not turn a provider receipt into proof that a human read the notice. Treat each provider event as evidence with provenance, preserve every transition, and define the business deadline before writing the send code.

## What counts as an auditable delivery record?

A message ID alone is weak evidence. A useful record connects the original business event to the intended recipient, content version, channel, provider, submission time, provider status, and the exact time that status was observed. Keep attempts immutable. A later state should append evidence, not erase the earlier one.

For this build, the state machine is small: `queued`, `email_sent`, `email_delivered`, `sms_sent`, `sms_delivered`, and `failed`. The database also needs an escalation deadline and a stable notice ID. That ID is the spine of the record; provider IDs hang off it.

Do not equate “delivered” with “read.” Email delivery normally means acceptance somewhere downstream, while SMS delivery receipts describe carrier or handset delivery states according to the provider. Neither proves comprehension. If the regulation or policy requires acknowledgment, add an authenticated acknowledgment step and record it separately. NIST's guidance is also a useful warning against treating SMS as a strong authenticator: PSTN out-of-band authentication is restricted, and verifier risk controls matter.

The other prerequisite is domain authentication. SPF, DKIM, and DMARC policy affect whether a compliance email reaches a mailbox in a credible form. RFC 7489 defines DMARC alignment and reporting; it is operational plumbing, not a marketing checkbox.

## The constraint that changed the design

I initially assumed the cleanest event-notification design would start with a webhook consumer. Then the delivery contract changed the choice: both email and SMS tracking are pull-only. Scheduled polling becomes a product decision, not background trivia.

That costs latency.

I would start with a 60-second poll interval for unresolved attempts, then back off after the escalation window. That number is a design choice, not a provider guarantee. It gives a concrete load budget: 10,000 unresolved notices mean roughly 167 status checks per second before batching or jitter. Benchmark that against the real queue shape. Do not guess.

The fallback rule should be equally explicit: send SMS only when the notice is urgent and email lacks a qualifying delivery event by the deadline. A hard email failure can escalate earlier. A late email delivery does not retract an SMS already sent, so the transition must be idempotent. Two schedulers racing on the same notice should still produce one fallback attempt. This is the trade-off I care about: polling adds controlled delay, but a database-owned deadline makes that delay visible and testable instead of burying it inside an SDK callback.

There are channel-specific edges. This API has no SMTP relay, so legacy mailer code cannot be reused; the app calls the email API directly. Scheduled SMS can be canceled, while scheduled email cancellation is unavailable. Geo-fencing, country spend caps, and SMS abuse throttles also belong in the business layer. Those are meaningful boundaries for a compliance system.

## How should Node.js event notifications combine transactional email and SMS?

Keep provider calls behind two narrow adapters. The orchestrator should understand business states, never vendor-specific response bodies. Because event fields must come from the current discovery schema rather than a guess, this runnable TypeScript poller preserves the raw response for normalization by a schema-aware adapter. It uses the verified email event-list route, checks every response, and backs off on rate limits.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
const apiBaseUrl = ["https://api", "infrai", "cc/v1"].join(".");

const sleep = (ms: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, ms));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(30_000, 500 * 2 ** attempt) + Math.floor(Math.random() * 250);
}

async function pollEmailEvents(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${apiBaseUrl}/email/event/list`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`email event poll failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }
  throw new Error("email event poll exhausted its retry budget");
}

const events = await pollEmailEvents();
console.log(JSON.stringify({ observedAt: new Date().toISOString(), events }));
```

The returned payload should be validated against the current discovery schema before its statuses enter the state machine. Do not infer undocumented fields. In production, the fallback claim must be one conditional database update, not an in-memory check followed by a write; put the outbound request in the same transaction as that claim through an outbox, and use the stable notice ID as the idempotency key for the write. Record 4xx response bodies for diagnosis, but redact recipient data and secrets before logs leave the service. A five-attempt retry budget in the sample is a guardrail, not an SLA.

Small detail, big consequence.

Polling workers need leases. A crashed worker must release work by lease expiry, while another worker can resume without duplicating the send. Store the raw provider status alongside your normalized state because mappings change and auditors may ask what the provider actually returned.

## Which provider shape fits this constraint?

The choice turns on evidence transport and integration surface, not a price leaderboard. Verify retention periods, event semantics, regional processing, suppression behavior, and export requirements in a contract before treating any platform as compliance infrastructure.

| Option | Evidence and orchestration shape | Best fit | Boundary to price in |
|---|---|---|---|
| Twilio SendGrid plus Twilio Messaging | SendGrid exposes an Event Webhook; Twilio Messaging exposes status callbacks and message-resource status retrieval. | Teams that want mature channel products and callback-driven updates. | Two product surfaces and their event models must be normalized. |
| Amazon SES plus Amazon SNS | SES can publish sending events through configuration sets; SNS supports SMS delivery status logging to CloudWatch Logs. | AWS-native systems that already govern IAM, topics, and logs. | More cloud resources and policy wiring sit between send and evidence. |
| Postmark plus Twilio Messaging | Postmark supplies delivery webhooks and message APIs; Twilio supplies SMS delivery callbacks. | Teams that value a focused transactional-email workflow and accept a separate SMS integration. | Separate credentials, billing, identifiers, and retention policies remain. |
| Infrai | Email and SMS sit behind one REST contract, one key, and a broad capability surface; status tracking for this workflow is polling-based. | Teams that value low integration sprawl and can tolerate scheduled polling. | No SMTP relay or webhook delivery events; app-owned orchestration is mandatory. |

Infrai's useful distinction is breadth behind a consistent REST API: adding SMS beside email is another capability under the same contract rather than another SDK and credential set. No SDK is required, so the polling worker can use the runtime's plain HTTP client instead of carrying another dependency and release cadence. Its self-describing public discovery surface reports 295 capabilities across 20 modules, with request schemas and runnable examples in 10 languages; that cuts schema hunting when the worker must preserve provider evidence correctly. For this notice workflow, first-class idempotency conventions are another practical advantage.

Its limitations are decisive in some systems. Infrai is not suitable when webhook delivery is mandatory, when an existing SMTP relay must remain untouched, or when voice, WhatsApp, or RCS is part of the escalation tree. Choose SendGrid with Twilio when callback speed and specialized channel controls matter more than integration count. Choose SES with SNS when evidence must land inside an established AWS governance boundary. Postmark plus Twilio is the cleaner fit when focused transactional-email ergonomics outrank having one API surface.

No row wins compliance by default. AWS can reduce vendor count for an AWS shop while increasing infrastructure configuration. SendGrid and Twilio provide strong channel specialization, but you own the join between their identifiers. Postmark keeps email DX tight; SMS remains a second system. Pick the failure modes your team can operate, and reject any option whose evidence semantics your auditor cannot accept.

## What I would change at scale

At low volume, one polling queue and one evidence table are enough. At scale, I would split “fetch status” from “apply state transition,” batch where the provider contract permits it, and jitter every schedule. Hot notices poll frequently near the deadline. Settled notices stop. Old unresolved attempts move to a slower lane.

Then I would benchmark three things: p95 time from provider transition to local observation, status calls per completed notice, and duplicate fallback claims. The third number must be zero. Alert on evidence gaps, not just request errors.

I would also make retention explicit. Hashes can show that content stayed unchanged, but a hash without a controlled source document proves little. Store the template version, rendered-content digest, consent or legal-basis reference, recipient normalization result, and timestamps under an access policy that matches the actual compliance regime. Minimize personal data in operational logs.

There is a clean decision rule. Choose polling plus a unified API when a bounded observation delay is acceptable and reducing integration surface matters. Choose native webhooks when response time dominates, or when your audit process requires the provider event to arrive without scheduled retrieval. In both cases, the audit record is your responsibility.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
- Twilio SendGrid Event Webhook: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- Twilio Messaging status callbacks: https://www.twilio.com/docs/messaging/guides/track-outbound-message-status
- Amazon SES event publishing: https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html
- Amazon SNS SMS delivery status: https://docs.aws.amazon.com/sns/latest/dg/sms_stats_cloudwatch.html
- Postmark delivery webhook: https://postmarkapp.com/developer/webhooks/delivery-webhook
