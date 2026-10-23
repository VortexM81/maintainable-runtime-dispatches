# Five Compliance Evidence Tests for Media SMS Provider Appointment and Shipping Alerts

Short answer: choose the delivery path that can preserve an append-only record of consent, message content, dispatch attempts, provider responses, and suppression decisions. For a media company sending a compliance notice, that evidence matters more than a long template catalog or a polished REST API.

The constraint changes the build. An appointment reminder, a shipping update, and an account-activity alert can share a transport, but they should not share an audit record by accident. I model each notification as an event with a stable ID, a policy decision, and an immutable copy of what was sent. The provider is an interchangeable delivery edge.

That is the smallest useful system. It gives an operator an answer to a narrow question: what did we try to send, to whom, under which permission, and what happened next?

## Which SMS alert provider fits appointment reminders and shipping evidence?

A message log is not automatically evidence. A timestamp and a `delivered` flag leave out the decisions that made the send permissible. For each event, I keep the destination in normalized form, the purpose, the consent or other lawful-basis reference, the template revision, the rendered body, and a redacted provider response. I also record the suppression lookup and its reason, including the case where no send was attempted.

Five checks catch most holes:

1. The event has a unique ID that survives retries.
2. The policy decision is stored before the transport call.
3. The exact rendered text is hashed and retained under a defined retention rule.
4. Every attempt has an outcome, including timeout and rejection.
5. A later suppression or deletion request can be linked back without rewriting history.

This is deliberately boring. Boring records are searchable during an incident.

For US traffic, consent and opt-out handling must be treated as product behavior, not a checkbox in an admin screen. For EU traffic, the purpose and retention choices need a clear owner, and the record should avoid collecting more personal data than the decision requires. Legal review still matters; an engineering ledger cannot decide the applicable rule for every message or jurisdiction.

## The smallest working implementation

I start with a narrow interface. It keeps the application code independent from a vendor SDK and makes the policy boundary visible in tests.

```ts
type AlertPurpose = "appointment" | "shipping" | "account_activity";

type AlertDecision = {
  eventId: string;
  destination: string;
  purpose: AlertPurpose;
  templateVersion: string;
  body: string;
  bodySha256: string;
  consentRef?: string;
  suppressionReason?: string;
  allowed: boolean;
  decidedAt: string;
};

type DeliveryAttempt = {
  eventId: string;
  attempt: number;
  requestedAt: string;
  outcome: "accepted" | "rejected" | "timeout";
  externalId?: string;
  responseCode?: string;
};

interface SmsTransport {
  send(input: {
    eventId: string;
    destination: string;
    body: string;
  }): Promise<{ externalId: string }>;
}

async function dispatch(
  decision: AlertDecision,
  transport: SmsTransport,
  append: (record: AlertDecision | DeliveryAttempt) => Promise<void>,
): Promise<void> {
  await append(decision);

  if (!decision.allowed) return;

  const started = new Date().toISOString();
  try {
    const result = await transport.send({
      eventId: decision.eventId,
      destination: decision.destination,
      body: decision.body,
    });
    await append({
      eventId: decision.eventId,
      attempt: 1,
      requestedAt: started,
      outcome: "accepted",
      externalId: result.externalId,
    });
  } catch (error) {
    await append({
      eventId: decision.eventId,
      attempt: 1,
      requestedAt: started,
      outcome: "timeout",
      responseCode: error instanceof Error ? error.name : "unknown",
    });
    throw error;
  }
}
```

The interface hides credentials and route details. The record does the opposite: it exposes the decisions an auditor will ask about. I would not call this a delivery receipt. An accepted API request is not proof that a handset displayed a message. Name the state precisely.

The hash is useful for detecting later changes, but it is not a substitute for retention controls or access controls. Hashing a phone number does not make it harmless either.

## Where do templates and suppressions belong?

Templates should be versioned application data, not strings scattered through handlers. A template revision needs an owner, an approved purpose, a locale, and tests for variable substitution. Keep the rendered message in the event record because a template can change after dispatch. Store the minimum audience data needed to investigate; do not turn an audit table into a second customer database.

Suppressions belong before rendering and before dispatch. A global opt-out, a blocked destination, an invalid number, or a policy rule can all produce “not sent,” but those reasons are operationally different. Use a stable code and a human-readable explanation. Then a support engineer can distinguish a correct policy stop from a broken transport.

The trap is retrying a policy decision as if it were a network failure. A timeout can be retried with the same event ID and an incremented attempt number. A suppression cannot. Idempotency belongs at the event boundary, where the system can prove that a retry did not create a second logical notice. I use a 24-hour replay window as a starting policy, then make the owner and expiry explicit; it is an operational choice, not a universal compliance period.

## What changes at scale?

At higher volume, I would split the policy ledger from the delivery worker and put an outbox between them. The transaction that accepts the compliance event writes the decision and the outbox record together. Workers claim outbox rows, send through a generic transport, and append outcomes. A dead-letter queue handles repeated failures; it must preserve the original event ID and the reason for each retry.

I would also sample neither policy decisions nor failures. Dashboards can aggregate them, but the source records need complete coverage for the retention window. Metrics still help: suppression rate by purpose, timeout rate by route, duplicate-attempt count, and the percentage of events with a usable external ID. Alert on changes in those distributions, not on one noisy request.

There is a cost. Append-only evidence consumes storage, redaction takes design work, and exact replay can expose sensitive content to more operators. The trade is intentional: a shorter log is easier to store but harder to defend.

For a US and EU rollout, I would run a test matrix before choosing a transport: consent present or absent, opt-out before and after enqueue, locale mismatch, invalid destination, provider timeout, duplicate event, and callback arriving before the worker commits its attempt. The winning integration is the one that leaves these cases explainable through a stable REST boundary and ordinary logs.

The limitation of a provider abstraction is clear. It is a poor fit when the application needs a carrier-specific feature that cannot be represented by the generic interface, or when regional legal review requires a route the abstraction hides. That is the trade-off: portability costs access to some transport-specific controls. In those cases, keep the evidence schema stable and expose a deliberately narrow provider-specific adapter. The extra code is easier to review than pretending every transport has identical semantics.

## The decision rule

Do not select an SMS alerts provider by starting with “does it have templates?” Ask whether your team can export a complete event trail without reconstructing it from dashboard screens. Verify the API's idempotency behavior, callback authenticity, status vocabulary, retention options, suppression hooks, and regional routing controls. Then test those claims with failure injection and a fixed dataset.

For media compliance notices, the right answer is the path that makes a disputed send explainable. Delivery speed is still relevant. Developer ergonomics still matters. Neither rescues a record that cannot show why the message was sent, what it contained, and how the system handled the next failure. The three-way trade is storage, privacy exposure, and investigation depth; choose the balance deliberately instead of treating the provider dashboard as the system of record.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://senders.yahooinc.com/best-practices/
- https://www.fcc.gov/consumers/guides/stop-unwanted-robocalls-and-texts
- https://eur-lex.europa.eu/eli/reg/2016/679/oj
- https://www.rfc-editor.org/rfc/rfc9457
