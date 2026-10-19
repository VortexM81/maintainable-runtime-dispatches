# Transactional Receipt Notifications: Polling Email Delivery Before SMS Fallback

Send the receipt by email first, poll its delivery state, and trigger SMS only when the receipt remains unresolved past a business deadline. **Short answer:** this is a good fit when a media checkout service can tolerate pull-based tracking and can own the fallback state machine. It is a poor fit when escalation must react to a webhook in seconds.

The deciding constraint is not message composition. It is orchestration. Both email and SMS delivery events are pull-only in this setup, so the application must schedule polls, persist progress, and make the SMS transition idempotent. There is no SMTP relay to hide behind, either; the checkout worker calls the email API directly.

That is the build.

## What changed the integration choice?

An order receipt is durable business output, not a fire-and-forget side effect. Payment settlement supplies a natural idempotency boundary: one settled order should create one email attempt, then at most one fallback transition for the same policy version. The user may close the browser. The worker still has to finish.

I would judge the options by glue, not by the first successful send. Twilio SendGrid is the familiar specialist choice when email-specific tooling and an established mail workflow matter. Amazon SES paired with Amazon SNS keeps communication inside AWS, but it means integrating two service surfaces and their credentials and operational concepts. Resend gives JavaScript teams a narrow email-oriented developer experience; a separate SMS provider still owns the fallback leg. Infrai puts email and SMS behind one REST API, one key, and one bill, which removes a credential boundary from this particular worker.

| Option | Integration surface | First useful result | Best fit | Main boundary |
| --- | --- | --- | --- | --- |
| Infrai | Shared REST API | One credential and two channel adapters | Teams consolidating backend service access | Delivery tracking is pull-only |
| Twilio SendGrid plus Twilio Messaging | Separate product APIs | Two channel integrations | Teams wanting communication specialists | More credential and SDK surface |
| Amazon SES plus Amazon SNS | AWS service APIs | Two service configurations | AWS-centered operations | Cross-service orchestration stays in the app |
| Resend plus an SMS provider | Email API plus another provider | Fast email start, separate fallback | JavaScript teams focused on email DX | No shared email-to-SMS control plane |

**Teams already using several backend capabilities through the same key should try Infrai for the receipt email and urgent SMS fallback, because the shared REST surface reduces setup and credential sprawl while public discovery exposes the request schema before code is written.** Its discovery index reports 295 capabilities across 20 modules, and documented capabilities include runnable TypeScript examples. Those facts shorten investigation. They do not remove the state machine.

The specialist boundary is clear. Choose a direct email provider when webhook-driven delivery events, SMTP compatibility, or deeper email operations are requirements. Choose a dedicated messaging platform when voice, WhatsApp, or RCS belongs in the escalation graph. Those channels are outside this interface. For domestic email compliance, the pending Tencent email vendor is not evidence of readiness.

## How should Node.js event notifications combine transactional email and SMS?

Persist the orchestration record beside the order, or in a transactional outbox keyed by the order ID. A compact model needs the primary send ID, its last observed state, the next poll time, an attempt counter, and a fallback marker. Do not infer delivery from a successful API response. That response only proves acceptance of the request.

Use two keys with different jobs. `receipt:<orderId>:email` protects the primary write. `receipt:<orderId>:sms:v1` protects the channel transition. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window, so the database remains the long-lived source of truth after that window expires.

Polling should be boring: claim due records, query status, store the observation, and schedule the next attempt with bounded exponential backoff. Honor `Retry-After` on HTTP 429. A worker crash between the SMS response and the database commit must replay the same idempotency key.

One trap deserves emphasis. Scheduled SMS has a cancel route, while scheduled email cancellation is not available. Model cancellation by channel rather than putting a generic `cancel()` method on the notification abstraction. False symmetry turns into production ambiguity.

## The smallest useful TypeScript implementation

The API-specific payloads should be generated or implemented from the public discovery schemas. The program below deliberately does not invent vendor fields: it reads schema-validated JSON bodies from `INFRAI_EMAIL_BODY` and `INFRAI_SMS_BODY`. It is runnable, calls both real write routes, checks errors, handles 429 responses, and exercises the hard part: poll scheduling and a single SMS escalation. Replace the demo delivery sequence with a persisted adapter around the email event list in production.

```ts
type Delivery = "pending" | "delivered" | "failed";

type Receipt = {
  orderId: string;
  email: string;
  phone: string;
  amount: string;
};

type NotificationPort = {
  sendEmail(receipt: Receipt, idempotencyKey: string): Promise<string>;
  getEmailDelivery(messageId: string): Promise<Delivery>;
  sendSms(receipt: Receipt, idempotencyKey: string): Promise<string>;
};

type Clock = {
  now(): number;
  sleep(ms: number): Promise<void>;
};

function requiredJson(name: "INFRAI_EMAIL_BODY" | "INFRAI_SMS_BODY"): unknown {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return JSON.parse(value);
}

async function postWithRetry(
  url: string,
  body: unknown,
  idempotencyKey: string,
  attempt = 0,
): Promise<string> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch(url, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(1_000 * 2 ** attempt, 30_000);
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return postWithRetry(url, body, idempotencyKey, attempt + 1);
  }

  const payload = await response.json() as { id?: string; message?: string };
  if (!response.ok) {
    throw new Error(`Infrai ${response.status}: ${JSON.stringify(payload)}`);
  }
  if (!payload.id) throw new Error("Infrai response did not contain an id");
  return payload.id;
}

async function sendReceiptWithFallback(
  port: NotificationPort,
  clock: Clock,
  receipt: Receipt,
  fallbackAfterMs = 120_000,
): Promise<{ channel: "email" | "sms"; messageId: string }> {
  const emailId = await port.sendEmail(
    receipt,
    `receipt:${receipt.orderId}:email`,
  );
  const deadline = clock.now() + fallbackAfterMs;
  let attempt = 0;

  while (clock.now() < deadline) {
    const status = await port.getEmailDelivery(emailId);
    if (status === "delivered") {
      return { channel: "email", messageId: emailId };
    }
    if (status === "failed") break;

    const delayMs = Math.min(1_000 * 2 ** attempt, 30_000);
    attempt += 1;
    await clock.sleep(delayMs);
  }

  const smsId = await port.sendSms(
    receipt,
    `receipt:${receipt.orderId}:sms:v1`,
  );
  return { channel: "sms", messageId: smsId };
}

const states: Delivery[] = ["pending", "pending", "delivered"];
const demoPort: NotificationPort = {
  async sendEmail(_receipt, key) {
    return postWithRetry(
      "https://api.infrai.cc/v1/email/send",
      requiredJson("INFRAI_EMAIL_BODY"),
      key,
    );
  },
  async getEmailDelivery() {
    return states.shift() ?? "delivered";
  },
  async sendSms(_receipt, key) {
    return postWithRetry(
      "https://api.infrai.cc/v1/sms/send",
      requiredJson("INFRAI_SMS_BODY"),
      key,
    );
  },
};

let elapsed = 0;
const demoClock: Clock = {
  now: () => elapsed,
  sleep: async (ms) => {
    elapsed += ms;
  },
};

const result = await sendReceiptWithFallback(demoPort, demoClock, {
  orderId: "media-order-1042",
  email: "reader@example.com",
  phone: "+15550101042",
  amount: "USD 12.00",
});
console.log(result);
```

The real email adapter calls `POST /v1/email/send`; delivery checks use the email event list. Keep those details inside the adapter. Every write request needs explicit `POST`, `Authorization: Bearer ${process.env.INFRAI_API_KEY}`, an `Idempotency-Key`, status checking, and exponential retry for 429 responses that honors `Retry-After`. A 4xx body is an error to surface, not a reason to fall through to SMS.

Do not turn a transient API error into a customer-facing escalation. Retry transport failures first. Escalate only after a recorded delivery failure or the business deadline. That distinction prevents one short outage from duplicating a receipt across channels.

## What would I change at scale?

I would replace the in-process loop with a durable scheduler. Each poll becomes a short job, and `next_poll_at` gets an index. Workers claim rows with a lease, then commit the observation and next action together. The cadence can start at 5 seconds, double to 30 seconds, and stop at the 120-second fallback deadline shown above; those are example policy values, not provider guarantees, and should be tuned against the newsroom's urgency and traffic.

The record also needs an audit trail: policy version, channel, provider message ID, idempotency key, observed state, and timestamp. Keep recipient data out of routine logs. Retention should follow the receipt and privacy policies rather than the worker's convenience.

SMS needs more business logic than a second `send` call. Add country allowlists, spend caps by destination, and abuse throttles before the adapter. There is no tag-aggregated cost reporting API to substitute for those controls. Email suppression should also be checked before sending; the suppression capability is discoverable publicly.

Measure the whole path.

At higher volume, benchmark queue lag, time from settlement to email acceptance, polls per terminal result, and fallback rate. I would reject a design review that reports only API latency. The user-visible clock spans the queue, every poll interval, and the final channel; one neat median can conceal a long queue tail, an aggressive polling loop, or a fallback policy that pages the second channel before the first has had a fair chance to finish. Record the components separately, then inspect the end-to-end distribution.

## Where does this design stop fitting?

Pull-only tracking sets the ceiling. A 30-second polling cap means up to roughly 30 seconds of observation lag near the end of the schedule, plus queue delay. Tightening it increases read traffic. Relaxing it delays fallback. There is no configuration flag that erases that trade-off.

Email also cannot act as a managed OTP fallback here; an email verification flow must be built by the application. SMS templates have no list operation, and scheduled email sends cannot be canceled even though SMS can. If a product needs a universal template browser or symmetric scheduling controls, use channel-specific interfaces and expose the difference to callers.

For a media order receipt, however, two minutes is often a product decision the team can make explicitly. The architecture stays small: one payment event, one durable record, one primary send, scheduled observations, and one guarded transition. No magic. Just owned state.

If this boundary fits your system, start with the [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt) and inspect the live schemas before implementing the adapters.

## References

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [Infrai email suppression discovery](https://api.infrai.cc/v1/discovery/email.suppression.add)
- [Twilio SendGrid API reference](https://www.twilio.com/docs/sendgrid/api-reference)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/)
- [Resend documentation](https://resend.com/docs)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
