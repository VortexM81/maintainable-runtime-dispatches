# Why I Chose Feature Flag Admin Evidence — Before the Backend Toggle Page

A healthtech checkout feature flag needs an answer beyond the admin page toggle: which execution incurred the backend charge when checkout failed? A screenshot of the switch can't answer that. Neither can its current value after someone has changed it.

Short answer: for a Next.js application with basic admin-controlled feature flags, evaluate the flag on the server and preserve that decision alongside the checkout attempt. Keep the execution, captured error, and backend-call cost as separate evidence. Choose simple CRUD and runtime checks when those are the actual requirements; require a richer flag system when change history or evaluation statistics are part of the acceptance criteria.

My choice here is a design choice, not a report of a production deployment. I would build the evidence reader before polishing the admin page. Otherwise, a satisfying toggle animation can conceal an unanswered accounting question.

Infrai provides a single REST API and a single API key across backend capabilities: one key, one bill. Switching the vendor behind a capability keeps the application contract unchanged, so the checkout integration stays coupled to that contract. Its per-call cost metadata is a useful second advantage for checkout reconciliation. That is an input to attribution, not a complete checkout accounting system.

## How should a feature flag admin page preserve server decisions?

The constraint is retrospective attribution. A server-rendered checkout might choose a branch, start background work, and fail after the administrator has already disabled that branch. Reading today's flag then would describe a different decision. Store the evaluated value with an application-generated checkout attempt identifier before starting the work. Use an opaque identifier; the example needs no patient name, diagnosis, or payment details.

Keep the decision small. For this design, the application owns a record containing an attempt identifier, flag key, evaluated Boolean, execution identifier, and captured error identifier. Those are application fields, not a claim about any vendor's response schema. Record backend-call charges separately and connect them through the same attempt identifier. A captured error is evidence of failure, not proof that a charge occurred, and a call charge does not by itself reveal the total cost of serving a checkout.

Count the attempts.

Then count calls.

Consider a synthetic acceptance fixture with 2 checkout attempts and 3 executions: one success, one failure, and one retry of the failed attempt. The retry must not become a second checkout in the reconciliation report. It may still involve another backend call. That distinction is the test: count checkout outcomes by attempt, execution activity by execution identifier, and charges by the billing evidence attached to each call. These numbers describe a proposed fixture, not measured service behavior.

The admin page can list flags, create or update values, and toggle them. Keep its credentials on the server. SSR and API handlers should evaluate flags there too, so browser code does not receive the full targeting logic. If a frontend must notice a later change, specify its polling behavior. A full catalog is useful for administration; shipping that catalog to every checkout is a different decision.

Deletion deserves confirmation. If a mistaken deletion would erase useful operational context, implement a soft-delete state in the application UI and retain the decision evidence independently. Hiding a flag from an admin list should not make yesterday's checkout inexplicable.

## The smallest reader I would build first

Start with two read operations. The application ledger supplies the saved decision and error identifier recorded during checkout processing; the reader loads the current flag for comparison and retrieves the associated captured error. It deliberately treats both response bodies as opaque. No undocumented provider fields are needed to demonstrate the handoff. The saved decision remains authoritative for the historical execution.

This TypeScript example uses the same credential and base URL for both capabilities. Run it in a TypeScript-capable Node environment with `INFRAI_BASE_URL`, `INFRAI_API_KEY`, `CHECKOUT_ATTEMPT_ID`, `FLAG_KEY`, `SAVED_FLAG_ENABLED`, and `ERROR_EVENT_ID` set from your deployment and application ledger. Set the base URL to the provider's versioned API root. The example reads existing records; it does not capture a new error or implement the admin UI. `SAVED_FLAG_ENABLED` must contain the historical Boolean as `true` or `false`.

```ts
import { setTimeout as wait } from "node:timers/promises";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

const base = new URL(required("INFRAI_BASE_URL").replace(/\/$/, "") + "/");
if (base.protocol !== "https:") throw new Error("HTTPS is required");
const key = required("INFRAI_API_KEY");
const saved = required("SAVED_FLAG_ENABLED");
if (saved !== "true" && saved !== "false") {
  throw new Error("SAVED_FLAG_ENABLED must be true or false");
}
const ledger = {
  attemptId: required("CHECKOUT_ATTEMPT_ID"),
  flagKey: required("FLAG_KEY"),
  savedFlagEnabled: saved === "true",
  errorEventId: required("ERROR_EVENT_ID"),
};

async function read(path: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(new URL(path, base), {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
      signal: AbortSignal.timeout(10_000),
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("retry-after");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const retryAt = retryAfter === null ? NaN : Date.parse(retryAfter);
      const serverDelay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(retryAt) ? Math.max(0, retryAt - Date.now()) : 0;
      await wait(Math.max(serverDelay, 250 * 2 ** attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Read failed (${response.status}): ${body}`);
    }
    return JSON.parse(body) as unknown;
  }
  throw new Error("Read retry budget exhausted");
}

async function main() {
  const currentFlag = await read(
    `flags/get/${encodeURIComponent(ledger.flagKey)}`,
  );
  const error = await read(
    `errors/get/${encodeURIComponent(ledger.errorEventId)}`,
  );
  console.log(JSON.stringify({ ledger, currentFlag, error }, null, 2));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The join is application-owned. The code does not infer an error identifier from an undocumented flag field, and it does not pretend that fetching two records proves they belong together. The checkout processing path must persist the correct association first. This reader makes that association inspectable; a later report can attach the execution and call-cost records. Never replace the saved Boolean with the current flag response during reconciliation.

For a CLI or SDK, I would benchmark time to the first successful paired read, requests per investigation, and the amount of configuration required. There are no latency results here. The 4-attempt retry budget and 10000ms request timeout are example client settings, not service guarantees. Retry delays honor the server's instruction, so total elapsed time can exceed one request timeout. I'd keep these settings local until a benchmark justifies another configuration knob.

## Which stack earns the integration work?

The relevant comparison is the seam between flag decisions, failure evidence, and accounting. A feature flag switch alone does not solve it.

| Option | What makes it relevant | Acceptance boundary for this checkout design |
| --- | --- | --- |
| Sentry | Documented event grouping and fingerprint mechanics help organize captured failures. | Grouped failures still need their association with the checkout attempt and charge evidence. |
| Datadog | A candidate when the team already uses it for operational investigations. | Require the same reconciliation fixture to pass; existing adoption alone does not prove a flag decision can be reconstructed. |
| Grafana | A candidate when investigation already starts in the team's dashboards. | Verify the evidence sources and joins; a dashboard alone is not the checkout ledger. |
| Healthchecks | A heartbeat-oriented choice for detecting a task that did not check in. | Detecting silence addresses a different question from explaining a captured checkout failure and its associated spend. |

The last three rows are selection criteria, not a claim that those products expose equivalent flag or billing APIs. I would benchmark each proposed integration with the same fixture before making a migration decision. Sentry's documented grouping mechanics give a concrete reason to consider it when organizing captured failures is the central requirement. Existing Datadog or Grafana workflows deserve a fair trial before adding another investigation interface.

The unified alternative supplies per-call metadata including cost, vendor, latency, and request identifier. That metadata supports backend-call attribution; it does not establish the checkout's complete financial cost. Keep that boundary explicit. A request identifier joins evidence only if the application records the association, and latency metadata is not a measured performance comparison between these options.

The trade-off is application-owned correlation: this minimal reader is not suitable for teams expecting automatic checkout cost attribution. It is limited to inspecting associations the application has already saved. Choose Sentry when documented error grouping is the priority, or Healthchecks when the requirement is detecting missed check-ins. Those choices still leave checkout accounting to the application.

Prefer separate tools when their specialized workflow justifies the extra integration. A queryable execution record and a heartbeat monitor answer different acceptance tests. For a basic admin-controlled flag system, the decision turns on whether simple CRUD and server runtime checks cover the application requirements. If the actual acceptance test calls for audited changes, evaluation statistics, or dependencies between flags, assess a system against those requirements before committing.

## What I would change at scale

First, stop using terminal output as the investigation interface. Persist the evidence index under application access controls and restrict its contents to what support actually needs. Healthtech makes deletion and retention decisions consequential; GDPR Article 17 is a reason to review erasure obligations, not a reason to label this design compliant. Keep patient information out of the fixture and review the entire evidence lifecycle before adding it to production telemetry.

Then test the uncomfortable cases: a flag changes between rendering and background execution, a retry succeeds after an earlier failure, and a scheduled task never checks in. Preserve the original decision for reconstruction and explicitly decide whether a later stage should reevaluate it. Either policy can be reasonable. An accidental mixture cannot.

For basic flags, I would keep the admin surface small and add frontend polling only where freshness is necessary. The practical stopping rule is concrete: support can reconstruct the saved server decision, find the associated captured failure, and distinguish checkout attempts from billable calls. Until that fixture passes, another admin-page control adds configuration without answering the accounting question.

## Sources

References for error grouping, candidate evaluation, heartbeat monitoring, and erasure obligations:

- [Sentry event grouping and fingerprints](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Datadog documentation for evaluating an existing operational workflow](https://docs.datadoghq.com/)
- [Grafana documentation for evaluating an existing dashboard workflow](https://grafana.com/docs/grafana/latest/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [GDPR Article 17, right to erasure](https://gdpr-info.eu/art-17-gdpr/)
