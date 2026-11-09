# Logistics Password Reset Templates: Keeping Email and SMS Deadlines in Sync

Restore access to the logistics portal with an application-owned reset template and email delivery. Add SMS backup when a separate recovery policy justifies the extra channel. Start with a message contract small enough to review in one sitting.

| Choice | Template ownership | Operational consequence |
| --- | --- | --- |
| Email only, rendered by the application | Copy and rendering rules ship with application code | One channel to validate; no alternate channel in this design |
| Email plus explicit SMS fallback, rendered by the application | Both messages share an expiry contract | More recovery coverage to evaluate; another template and dispatch policy to maintain |
| Templates maintained outside the application | Copy changes follow a separate release process | Useful for editorial autonomy; requires coordination with the reset service |

**TL;DR:** Prefer application-owned email templates for this logistics reset flow. Keep the expiry deadline in the reset service, pass it into rendering, and make SMS an explicit, policy-controlled backup. Template ownership is the first decision; channel count comes after it. This is a practical fallback strategy for a short-lived password reset message, not a product recommendation or a declaration of US/EU compliance.

## Who owns the words when a reset expires?

The useful editorial angle is deadline semantics: who can change the promise made to someone locked out of a dispatch portal? A template that says a link is usable for ten minutes makes a promise about a different component. The reset service has to enforce that promise. Putting the sentence in a delivery dashboard doesn't move enforcement there.

Own that contract.

For an illustrative policy, choose a five-minute lifetime. That number is a design input here, not a standard or a measured optimum. Generate the deadline once, preserve it through dispatch attempts, and check it when the reset credential is redeemed. The template should receive the actual deadline. A delayed message must not claim a fresh five minutes merely because the renderer ran again. Consider a deterministic queue fixture: create the attempt at time zero with a deadline of 300,000 milliseconds, then advance the clock to 240,000 milliseconds before dispatch. The remaining lifetime is 60,000 milliseconds. Rendering “expires in five minutes” at that point would overstate the remaining time by four minutes. Rendering the stored deadline preserves the original promise. Now advance the fixture clock to 300,000 milliseconds and ask for SMS backup. Reject the expired attempt before dispatch rather than rendering a new message around the old credential. If the recovery policy permits a new attempt, issue it through that policy and test its invalidation behavior separately. None of these numbers are delivery measurements. They are test inputs that make the contract's failure visible without waiting for a real queue to become slow.

This is where template ownership earns its keep. Store the email subject, text, HTML, and SMS wording beside the code that supplies their variables. Review a change to the expiry sentence with a change to the expiry policy. If a separate team owns copy, give it the same contract and a release check; repository ownership alone cannot make a misleading sentence correct.

The Mustache manual gives a concrete rendering boundary: ordinary variables are HTML-escaped, while triple mustaches produce unescaped output. That distinction matters for a reset URL in HTML. Escaping is useful, but it doesn't validate the destination or decide whether a URL belongs to the reset service. Treat URL construction and HTML rendering as separate responsibilities. Don't apply HTML escaping rules blindly to plain text or SMS.

For the dispatch portal, a useful copy review asks whether the recipient can identify the account action and understand the deadline without opening another page. Keep shipment details out of the example message. Account recovery has no need to repeat a manifest or delivery address.

## When should password reset email allow SMS backup?

An email dispatch attempt and an authorized recovery action are different events. Design the backup policy explicitly: an eligible account can request SMS through the recovery flow, using a previously established destination and the same expiry deadline. A request must still pass the service's recovery and abuse checks. Don't let an arbitrary phone number in the reset form become the backup destination. These are proposed controls, not claims that a template engine provides authentication security.

Make the boundary visible.

In this design, a delivery receipt is operational evidence, not permission to change the recovery channel. A timeout should leave the outcome uncertain until the delivery workflow resolves it. It should not silently create another credential with another five-minute lifetime. Choose deliberately whether channels carry one credential or separate credentials tied to one recovery attempt; document how successful redemption invalidates the remaining path. The renderer should make neither decision.

US/EU questions also need their own boundary. The FTC's CAN-SPAM guide distinguishes commercial messages from transactional or relationship messages and discusses the message's primary purpose. Keep the reset message focused on recovery; don't treat a promotional footer as harmless decoration. The guide is about email and US requirements. It doesn't establish permission for SMS or settle EU obligations.

For deployment review, have the responsible legal and privacy owners determine applicable email and SMS rules for the actual audience, purpose, and jurisdictions. Record that decision separately from the template version. An application-owned template gives engineers a reviewable artifact; it cannot supply a legal basis by itself.

Email only is a reasonable starting point when the service has no approved alternate recovery path. SMS adds destination management, another message format, and another dispatch policy to test. Consider those obligations alongside delivery spend. A changing unit price is a poor substitute for deciding who owns the recovery promise.

## A small contract beats another configuration layer

The following TypeScript example implements a narrow boundary suitable for a Node.js service. It receives an already issued reset attempt, renders channel-specific text, and hands both the deadline and a stable attempt identifier to a generic delivery adapter. The reserved example domain represents the logistics portal. The five-minute value is fixture data.

```ts
type Channel = "email" | "sms";

type ResetAttempt = Readonly<{
  id: string;
  resetUrl: string;
  expiresAtMs: number;
}>;

type Delivery = Readonly<{
  attemptId: string;
  channel: Channel;
  destination: string;
  text: string;
  expiresAtMs: number;
}>;

type Sender = (delivery: Delivery) => Promise<void>;

async function dispatchReset(
  attempt: ResetAttempt,
  channel: Channel,
  destination: string,
  send: Sender,
  now: () => number = Date.now,
): Promise<void> {
  if (!Number.isFinite(attempt.expiresAtMs)) {
    throw new Error("Invalid reset deadline");
  }
  if (now() >= attempt.expiresAtMs) {
    throw new Error("Reset attempt expired");
  }

  const url = new URL(attempt.resetUrl);
  if (url.origin !== "https://dispatch.example.com") {
    throw new Error("Unexpected reset origin");
  }
  if (url.username || url.password) {
    throw new Error("Reset URL must not contain user information");
  }

  const deadline = new Date(attempt.expiresAtMs).toISOString();
  const text = channel === "email"
    ? `Reset your dispatch portal password: ${url.href}\nExpires at ${deadline}.`
    : `Dispatch portal password reset: ${url.href} Expires at ${deadline}.`;

  await send({
    attemptId: attempt.id,
    channel,
    destination,
    text,
    expiresAtMs: attempt.expiresAtMs,
  });
}

const fixtureStartMs = 0;
const fixtureAttempt: ResetAttempt = {
  id: "reset-fixture-001",
  resetUrl: "https://dispatch.example.com/reset?token=fixture-only",
  expiresAtMs: fixtureStartMs + 5 * 60 * 1000,
};

const captured: Delivery[] = [];
const capture: Sender = async (delivery) => {
  captured.push(delivery);
};

void dispatchReset(
  fixtureAttempt,
  "email",
  "dispatcher@example.com",
  capture,
  () => fixtureStartMs,
).catch(() => {
  throw new Error("Reset fixture failed");
});
```

This is a dispatch example, not a complete reset service. Credential generation, secure storage, destination eligibility, rate limits, and redemption belong outside it. The adapter must honor the deadline before a deferred send, since a queue can outlive the synchronous check. Redemption must enforce expiry independently. Passing a timestamp is a contract, not enforcement by magic.

Keep the adapter small. Require a documented retry policy and duplicate-dispatch handling keyed to the recovery attempt and channel. Preserve the attempt's deadline across retries. If an adapter rejects a submission, distinguish a definitive rejection from an uncertain outcome before retrying. Record a sanitized reason; don't put the credential, full reset URL, phone number, or email address into routine logs.

Benchmark the whole path before adding configuration. Measure request-to-queue time, queue age at dispatch, and time remaining before expiry. Report the test load, sample size, and percentile with the result. There are no measured results here. The useful acceptance question is whether the chosen policy leaves enough usable time after the queue delay, including the slow tail.

Test the deadline boundary with an injected clock: one instant before expiry, exactly at expiry, and after expiry. Add a deferred-send case and a retry case. Snapshot each channel's rendered text so a copy change cannot quietly replace the absolute deadline with a newly calculated duration. During deployment, keep old and new queue consumers compatible with the attempt contract until pending messages drain. That compatibility work matters more than another template setting.

## When external template ownership is the better choice

The runner-up is an externally managed template with an application-owned data contract. Choose it when copy owners need an independent publishing workflow and can accept a review rule for security-sensitive wording. Require preview fixtures for both channels, a version recorded with each dispatch, and a way to restore a previously approved version. Treat these as acceptance criteria to verify, not capabilities to assume.

There is a real trade-off: an independent copy release can shorten editorial turnaround, while a shared application release makes contract changes easier to review together. Measure both the time to the first correct test message and the work required to change expiry wording safely. A fast first send says little about the second task.

For this logistics portal, keep the decision attached to a verifiable result: the delivered message and the redemption service agree on the deadline. Start with application-owned email; add SMS only after its recovery policy and ownership are explicit. More channels cannot repair a false promise in the template.

## References

- Mustache template syntax manual: HTML escaping and unescaped variable syntax. https://mustache.github.io/mustache.5.html
- FTC, CAN-SPAM Act compliance guide for business: commercial versus transactional or relationship email and primary-purpose considerations. https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business

## Further reading

Read the template manual's variable examples alongside the renderer review. Use the FTC guide's primary-purpose discussion for the US email review, and obtain applicable jurisdiction-specific guidance before approving the SMS and EU portions of the recovery policy.

- https://mustache.github.io/mustache.5.html
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
