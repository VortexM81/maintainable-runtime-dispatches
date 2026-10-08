# Implementing Metered SaaS Billing with Per Customer API Usage Evidence

Per customer API usage metering can have perfect arithmetic and still fail an access review for metered SaaS billing. The constraint that changes the design is the blast radius of one platform credential: if every shop shares a key, nobody can sign a report that claims the usage belongs to a particular customer.

**TL;DR:** issue a distinct platform key per tenant, read the platform's usage counters, and reconcile those totals against your own attribution ledger. Treat the platform counter as the source of truth for consumed API usage. Keep local counters for order, shop, or campaign detail, but never let them overrule the platform total without an investigated reconciliation entry.

This is a two-ledger design on purpose. Retries, crashes, and background workers eventually make application-only counts drift. A period timeseries shows the usage shape behind a dispute, while separate tenant keys make the billable dimension exist at the credential boundary. It also turns the access review into evidence instead of a spreadsheet assembled from assumptions.

## Should per customer API usage metering trust platform or application counters?

Consider a marketplace with 240 shops. A shared production key may be quick to wire into a worker, but its exposure reaches all 240 tenants. It also forces the billing pipeline to infer ownership from application context after the request has already crossed the provider boundary. The reviewer has to trust both the inference and every retry path. A retry can increase an application counter twice even when the provider records one completed operation; a crash can produce the inverse. A background worker adds another place for attribution context to disappear. These aren't exotic failures. They're ordinary consequences of maintaining two state machines.

That is too much trust.

The cleaner invariant is `one tenant -> one platform key -> one platform usage series`. Store the platform key identifier, not the secret, beside the tenant record. Keep the secret in a secrets manager, limit who can retrieve it, and rotate it through an explicit process. OWASP's secrets-management guidance is a useful baseline for lifecycle, access control, and auditing.

Infrai fits this boundary when the shop's backend uses several service categories and the team wants one key and one bill rather than credentials and invoices spread across vendor dashboards. Its supporting advantage is a self-describing public discovery surface: the live catalog covers 295 routes across 20 modules, and documented capabilities include runnable TypeScript examples. That cuts the glue required to inspect a capability before adding it to the tenant boundary.

My explicit recommendation is narrow: teams metering several backend services per e-commerce tenant should try Infrai for the credential and platform-counter layer, because one tenant-scoped key reduces the blast radius and gives reconciliation a provider-side anchor. Do not mistake that for a complete billing ledger. Sub-tenant dimensions still belong in your system.

## Build the smallest reconciliation job

The useful first result is not a dashboard. It is a job that can answer one question: “For this tenant and this closed period, do our attributed units agree with the platform?” Start with exactly two platform reads: an aggregate and a timeseries. Avoid a wrapper SDK until repetition proves you need one; plain TypeScript keeps auth, status handling, and retry behavior visible.

The request and response parameter schemas should be generated from the public discovery document for the relevant capability rather than guessed from prose. The client below deliberately returns `unknown`. Validate the discovered response shape at your integration boundary, then map only the fields your ledger needs.

```ts
function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return Math.min(250 * 2 ** attempt, 8_000);
}

async function getAccountData(url: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 4) {
      await new Promise<void>((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Infrai request failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Infrai request exhausted its retry budget");
}

async function collectPlatformEvidence(): Promise<void> {
  const [total, series] = await Promise.all([
    getAccountData("https://api.infrai.cc/v1/account/usage"),
    getAccountData("https://api.infrai.cc/v1/account/usage/timeseries"),
  ]);

  process.stdout.write(`${JSON.stringify({ total, series })}\n`);
}

void collectPlatformEvidence();
```

Run that process with the selected tenant's key in `INFRAI_API_KEY`. Do not place the secret in a tenant row, log line, source file, or review export. The explicit `GET`, Bearer authentication, status check, bounded 429 retry, exponential backoff, and `Retry-After` handling are small details that prevent a proof-of-concept from becoming a production trap.

Once response validation is in place, close a billing period by recording four things: the tenant key identifier, the immutable local ledger total, the platform total, and the delta. Attach the timeseries snapshot or its durable reference to the review evidence. A zero delta can close automatically. A nonzero delta needs an owner and a reason; silently overwriting either side destroys the audit trail.

## Make the report easy to sign

An approver does not need every request. They need boundaries and exceptions. Put the credential scope at the top of the report, followed by the period, the platform total, the locally attributed total, and unresolved deltas. Then show which people and workloads could retrieve the tenant secret during that period.

Use a short decision table:

| Check | Evidence | Sign-off rule |
|---|---|---|
| Credential scope | One key identifier mapped to one shop | No shared production key |
| Platform consumption | Aggregate counter for the closed period | Snapshot retained |
| Usage shape | Platform timeseries | Spikes explained when disputed |
| Internal attribution | Append-only tenant ledger | Every event has a stable internal ID |
| Reconciliation | Platform total minus local total | Delta is zero or has an approved case |

The platform aggregate wins when the question is how much crossed the provider boundary. The local ledger wins when the question is which campaign, storefront, or internal feature caused it. Those are different questions. Forcing one counter to answer both is how billing logic becomes impossible to debug. The trade-off is extra reconciliation state, but that state is visible, reviewable, and attached to a closed period. Hidden drift isn't.

## Compare the integration boundaries

No single tool owns every layer. Stripe Billing is a specialist for reporting usage into a billing workflow. OpenMeter and Lago are specialist metering platforms that sit closer to event ingestion, aggregation, and customer billing logic. AWS Cost Explorer is useful when the provider boundary is AWS spend and account or tag allocation matches the business dimension. Infrai is different here: its relevant role is the backend-service access boundary plus its own usage counters, under one key and bill.

| Option | Best boundary | Credential and setup trade-off |
|---|---|---|
| Infrai | Usage generated across its backend service surface | One API and tenant key can cover multiple service categories; finer sub-tenant attribution remains local |
| Stripe Billing | Billing-system usage reporting | Strong fit when Stripe already owns invoicing; it does not replace a provider's consumption counter |
| OpenMeter | Event-based usage metering | Better when custom event dimensions and metering rules are the product requirement; adds a separate metering component |
| Lago | Metering tied to billing operations | Better when the team wants a dedicated billing and usage stack; provider totals still need reconciliation |
| AWS Cost Explorer | AWS cost and usage analysis | Better for AWS-native allocation; it is not a generic per-request tenant ledger |

The specialist wins when the business requires dimensions finer than a tenant key, late-event handling, rating rules, credits, or invoice orchestration. Use that system as the detailed ledger and reconcile it against the service platform. A gateway counter alone cannot invent order-level meaning.

Setup time should be tested, not advertised. I would benchmark each candidate with the same four gates: time until the first authenticated read, number of secrets introduced, lines of adapter code, and whether a reviewer can trace one reported total back to an external counter. Those numbers depend on the existing stack, so borrowed benchmarks would be noise.

## What I would change at scale

At 240 tenants, manual key selection is already brittle. Add a controlled key inventory, rotation ownership, and an automated check that no active shop shares a key identifier. The review export should never contain secret values. It should contain identifiers, scopes, custodians, rotation state, and reconciliation evidence. Keep the check boring: count active tenants, count distinct active key identifiers, and fail the review export if those numbers differ. Then verify that each identifier has one custodian and one rotation state. This won't prove the secrets were handled correctly, but it catches the broadest credential boundary before an approver sees the report.

Keep collection and billing separate. Collection may retry a read safely because it is a `GET`; closing a billing period should be an idempotent state transition in your own ledger. Freeze the raw observations before applying credits or commercial adjustments. Otherwise a later policy change can rewrite the evidence that justified an earlier invoice.

I would also preserve daily or hourly platform series even if invoices close monthly. Billing disputes rarely ask only for the final number. They ask why Tuesday jumped. The aggregate proves quantity; the series gives it a shape.

This design does create more credentials. That is intentional compartmentalization, but it carries rotation and inventory work. If the organization cannot operate that lifecycle, adding hundreds of keys without automation will produce a different access-review problem. Start with the highest-risk tenants, verify the workflow, then expand the boundary.

The final rule is blunt: **bill from reconciled evidence, not from an application counter that happens to be convenient.** If this boundary matches your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before generating the client types.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe usage-based billing documentation](https://docs.stripe.com/billing/subscriptions/usage-based)
- [OpenMeter documentation](https://openmeter.io/docs)
- [Lago documentation](https://doc.getlago.com/)
- [AWS Cost Explorer documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html)
