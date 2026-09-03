# Transactional SMS Alerts Explained — Reliable Delivery Tracking and Resend Control

Short answer: choose an SMS alerts service by the delivery evidence it can preserve, then test reliability, suppression, and resend behavior before comparing cost. For a US/EU media startup sending compliance notices, a direct API with pollable status can be enough, but only if a background job reconciles every message into an audit record.

The hard constraint isn't sending a text. Almost any candidate can make a phone buzz. The hard constraint is answering a compliance review weeks later: what notice was approved, which recipient was targeted, when did the provider accept it, what delivery state followed, and was a resend deliberate?

Evidence wins.

## How should a startup choose a reliable transactional SMS alerts service?

Start with a small evidence contract that belongs to the application, not the provider. I would record an internal notice ID, a content or template revision, recipient, jurisdiction, scheduled time, provider message ID, each observed status payload, and the actor or rule that authorized a resend. Picture the awkward review: notice `legal-2026-041` was scheduled for 09:00 UTC, the first attempt has a provider ID but no reconciled status, and a support agent clicked resend at 09:07. A dashboard screenshot cannot tell the reviewer whether the second text reused approved copy or bypassed suppression. An application ledger can, provided it links both provider IDs to the same immutable notice revision and records the resend authorization separately. Encrypt or tokenize recipient data according to the startup's retention policy; an audit trail is not permission to keep personal data forever.

Then benchmark time-to-first-call and the amount of glue required to keep that contract current. The same test should run against Twilio, Vonage, Amazon SNS, and Infrai: submit a controlled notice, retain the returned identifier, reconcile delivery state, suppress a blocked recipient, and exercise one intentional resend. Don't score a polished dashboard as evidence unless the data can be exported into the system of record.

I use this decision matrix before touching pricing:

| Candidate | Sensible reason to test it | Evidence question that decides the result |
|---|---|---|
| Twilio | It is the incumbent named in the migration question | Can the existing integration produce the required audit record without a rewrite? |
| Vonage | It gives the team a second dedicated communications provider to benchmark | Can its status data map cleanly to the same internal evidence contract? |
| Amazon SNS | It belongs in the trial when the application already operates inside AWS | Does the resulting operational model keep reconciliation and audit ownership clear? |
| Infrai | Its one REST API works over plain HTTP without an SDK, and the contract lets a team swap vendors without changing application code | Is polling-based delivery tracking timely enough for the compliance workflow? |

This table is deliberately not a universal ranking. The available evidence here does not establish comparative uptime or latency, and I'm not sure which candidate will win for a particular US/EU traffic mix until the team runs controlled calls and reviews current regional terms. Your mileage may vary. The useful result is a repeatable acceptance test, not a logo preference.

## Why does webhook-free reliability change the architecture?

Delivery updates for this capability are pulled, not pushed. There are no webhook event notifications, so a scheduled background job must revisit pending message IDs. That adds latency between a provider-side change and the application's record of it. For a compliance notice that can tolerate periodic reconciliation, this is ordinary queue work. It is not suitable when downstream action must fire immediately from a delivery webhook; stick with a provider whose verified webhook contract meets that requirement.

The same boundary applies to channel scope. The one-REST-contract option is credible when the team wants plain HTTP, one key, and application code that stays unchanged as the underlying vendor moves. Its broader surface covers 295 routes across 20 modules, which can also reduce credential and SDK sprawl. The catch is that it has no voice, WhatsApp, or RCS channel, and no SMTP relay. A startup planning rich conversational messaging should test Twilio or Vonage directly instead. A team centered on AWS operations may prefer to keep Amazon SNS in its established control plane.

Polling is a product decision.

Scheduling does not erase application responsibility. Persist the notice before dispatch, give the scheduling job a unique internal ID, and make the transition from scheduled to submitted idempotent. Resend support can recover a failed or missed alert without reconstructing the original payload flow, but it should be a recorded state transition with an authorization reason. Suppression also needs an application rule: check the recipient against policy before every attempt, including resends.

Abuse controls stay local. Country restrictions and pricing-based throttles are not supplied as an automatic guardrail here, so the application must reject disallowed destinations and trip its own per-country spend or volume circuit breaker. There is also no cost-reporting API aggregated by tag. Keep campaign, notice, and jurisdiction dimensions in the internal ledger if finance or compliance will need that slice later.

## Implementation: the smallest auditable polling loop

The following TypeScript program polls one existing message ID and appends every successful response to a JSON Lines audit file. It uses the verified status route, sends an explicit method, honors `Retry-After` on a `429`, applies exponential backoff otherwise, and surfaces every non-success response. No response fields are assumed; storing the raw JSON avoids silently discarding evidence when a provider adds detail.

```ts
import { appendFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.SMS_API_BASE_URL;
const messageId = process.env.SMS_ID;

if (!apiKey || !baseUrl || !messageId) {
  throw new Error(
    "Set INFRAI_API_KEY, SMS_API_BASE_URL, and SMS_ID before running this script",
  );
}

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function fetchStatus(attempt = 0): Promise<unknown> {
  const response = await fetch(
    `${baseUrl}/v1/sms/status/${encodeURIComponent(messageId)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
    return fetchStatus(attempt + 1);
  }

  const body = await response.text();
  if (!response.ok) {
    throw new Error(`Status request failed (${response.status}): ${body}`);
  }

  return JSON.parse(body) as unknown;
}

const providerResponse = await fetchStatus();
const auditEntry = {
  observedAt: new Date().toISOString(),
  messageId,
  providerResponse,
};

await appendFile("sms-delivery-audit.jsonl", `${JSON.stringify(auditEntry)}\n`);
console.log(JSON.stringify(auditEntry, null, 2));
```

Run it from a scheduler at the interval your compliance SLA permits, passing the documented API base URL and a message ID already returned by the send flow. Stop polling according to an application policy after a terminal state or a defined retention window. Those terminal values are intentionally not hardcoded above because they are not established here; inspect the live discovery schema and a controlled response before encoding them.

One trap deserves a concrete number. Five workers retrying a `429` in a tight loop can multiply pressure at exactly the wrong moment. The bounded retry path above honors the server hint when present and otherwise waits 500, 1,000, 2,000, 4,000, then 8,000 milliseconds.

Short code. Measurable behavior.

## What governance should change at scale?

The local JSONL file is a minimum proof, not the final compliance store. At scale I would put pending message IDs on a durable queue, use a database append-only event table, restrict access to recipient data, and define deletion rules with counsel. The poller must tolerate duplicate work: use the internal notice ID plus provider message ID and observation identity to prevent duplicate records from becoming duplicate business actions.

I would also separate delivery evidence from delivery interpretation. Preserve the provider response, then map it into a small internal state machine in a versioned projection. That lets a migration from Twilio, Vonage, Amazon SNS, or an Infrai-backed route change the adapter without changing the compliance record consumed by the newsroom and legal team — exactly the kind of boring portability that pays off during an audit.

Templates and signatures can standardize recurring notices, but template discovery needs implementation planning even though the live route set includes template retrieval and listing. Email is not a complete fallback here: there is no managed email OTP interface, scheduled email has no cancel route, and a pending domestic email vendor cannot establish China compliance. Keep the initial design narrow. SMS compliance notices in the US/EU are the stated job.

The final choice is conditional. Pick the one-contract option when a plain REST integration, vendor portability, and polling-based delivery evidence fit the workflow. Keep Twilio when the existing implementation already satisfies the audit contract with less migration risk. Trial Vonage when a dedicated communications alternative is valuable, and evaluate Amazon SNS when AWS operational alignment matters. If real-time webhooks or broader messaging channels are mandatory, the polling option loses regardless of its integration simplicity.

## References

- https://resend.com/docs/introduction
- https://developer.apple.com/documentation/security/password_autofill
