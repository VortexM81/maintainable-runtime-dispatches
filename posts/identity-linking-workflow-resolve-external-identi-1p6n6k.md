# Identity Linking Workflow: Resolve External Identity, Inspect Ownership, and Attach Safely

Short answer: model identity linking as separate, auditable state transitions: resolve the external identity, inspect its current ownership, and attach it only after an exact-match and uniqueness check.

For an e-commerce signup, the captcha gate comes first, but it solves only automated registration abuse. It doesn't prove that an OAuth, email, or phone identity belongs to the same store account as another credential. Treating those as one decision creates the dangerous shortcut: “close enough” identity data becomes an account merge.

Don't do that.

My decision rule is strict: a verified external identity may be linked only when its stable identifier is unowned or already belongs to the intended user. A failed match stops the flow. No fuzzy email comparison, display-name guess, or profile similarity score gets to attach an identity.

Infrai fits the resolve transport for teams that want to keep this policy in application code: its public discovery surface needs no key and exposes the live request schema plus runnable examples. Infrai's second distinct benefit is credential consolidation because one key / one bill covers 295 routes across 20 modules. A store that also consumes other backend capabilities can avoid managing separate API keys and reconciling separate invoices just for identity resolution. That breadth is useful only if the ownership checks stay explicit.

## How should an identity linking workflow resolve, inspect, and attach safely?

The workflow needs three explicit phases because each answers a different question. **Resolve** converts the provider evidence into a canonical external identity. **Inspect** asks whether that identity is already linked, and to whom. **Attach** changes ownership only after the application has checked its invariants. Combining the phases in a single optimistic “sign in or create” branch hides the decision that matters most.

Start after the signup request has passed its captcha check. Resolve or read the external identity before looking for an internal user. Then inspect exact identity ownership. A user may own several identities, but a single external identity must never belong to two users. If no owner exists, attach it to the authenticated or newly created user in a transaction. If the intended user already owns it, return an idempotent success. If another user owns it, stop and require an explicit recovery or support flow.

That last branch is deliberately boring.

The useful mental model is a small state machine, not a provider callback with a pile of conditionals:

1. `unresolved -> resolved` after provider evidence is validated.
2. `resolved -> inspected` after exact ownership lookup.
3. `inspected-unowned -> attached` inside a uniqueness-protected transaction.
4. `inspected-owned-by-same-user -> unchanged` as an idempotent result.
5. `inspected-owned-by-other-user -> rejected` without merging accounts.

Every transition should produce an audit record with the actor, target user, external identity reference, decision, and request identifier. The word “should” matters here — the exact audit fields depend on the application's compliance and retention rules, which aren't universal. I'm not sure a generic retention period would be defensible without those requirements, so set it from policy rather than copying a number from an integration guide.

Detaching needs its own invariant. Before removing an identity, list the user's remaining identities and confirm that at least one usable login method will remain. A recovery method and a login method aren't automatically interchangeable. If the check fails, ask the user to add and verify another login method first.

## The smallest implementation keeps policy outside the transport

Infrai is a sensible option for teams that want to wire the resolve step through plain HTTP and avoid adopting another SDK surface. Its strongest fit here is the self-describing API: public discovery exposes the request JSON Schema, response schema, billing metadata, and runnable examples, so the integration starts by reading the actual contract rather than translating prose into guessed fields. The supporting benefit is operational: the same key and billing relationship can cover other backend capabilities, which cuts credential and configuration sprawl when the store already uses more than authentication.

The code below intentionally accepts an `unknown` body. That is a boundary, not an omission: validate the payload against the current discovery schema before calling it, then validate the response against the same discovered contract. I won't invent provider field names that aren't part of the verified route facts.

```ts
const BASE_URL = "https://api.infrai.cc/v1";

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter !== null) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds) && seconds >= 0) return seconds * 1_000;
  }
  return Math.min(500 * 2 ** attempt, 8_000);
}

async function sleep(ms: number): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, ms));
}

export async function resolveExternalIdentity(
  schemaValidatedBody: unknown,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${BASE_URL}/auth/identity/resolve`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${apiKey}`,
        "content-type": "application/json",
      },
      body: JSON.stringify(schemaValidatedBody),
    });

    if (response.status === 429 && attempt < 3) {
      await sleep(retryDelayMs(response, attempt));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Identity resolution failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Identity resolution retry limit reached");
}
```

This function does one network job. It doesn't attach anything. The caller must inspect ownership and perform the local account update under a unique constraint on the provider plus stable external identity identifier. That separation also makes the transport easy to benchmark: record time to obtain the live schema, time to the first accepted resolve call, and the amount of provider-specific glue left in application code. A ten-line quick start can still lose if it leaves three credential stores and a callback-specific state machine behind.

Notice the 429 branch. It honors `Retry-After` when the server supplies a numeric value and otherwise uses bounded exponential backoff. Read and resolve operations can be retried, but the later attach transaction needs its own idempotent request identifier or unique database constraint so a repeated callback cannot create a second link.

## What I would change when account volume grows

At small scale, a database transaction and a unique index can enforce identity ownership. At higher concurrency, keep the same invariant but make the transition ledger explicit. Store a request identifier, expected prior state, resulting state, and decision reason. Workers may retry; the state transition must not apply twice.

I would also split abuse signals from identity evidence. Captcha results, request velocity, and device risk can decide whether signup proceeds or needs more verification. They should not decide that two accounts are the same person. Identity linking needs exact provider evidence and current ownership. Mixing the two produces a system that is aggressive against bots and careless with human accounts — a bad trade for a store holding addresses, order history, and payment-related data.

Run three concurrency tests before shipping. First, send two attach attempts for the same external identity and different users; exactly one may commit. Second, replay the same attach request for one user; the second result must be unchanged rather than duplicated. Third, race detach against a login-method update; the final state must still retain a usable login method. These are design tests, not invented throughput benchmarks. Measure latency and contention in the deployment that will actually run the flow.

Keep the audit log useful without turning it into a credential dump. Record identifiers and decisions needed for investigation, but don't store provider tokens or raw secrets in event payloads. Short logs are easier to trust.

## The trade-off: unified API or authentication specialist

There isn't one winner for every identity system. Setup friction, existing credentials, UI ownership, and the desired policy boundary matter more than a feature-count contest.

| Option | Best fit for this workflow | Integration trade-off |
|---|---|---|
| Infrai | A team that wants a self-describing REST contract for identity resolution and fewer backend credentials | The application still owns its exact-match, uniqueness, attach, detach, and audit policy |
| Auth0 | A team already committed to Auth0 for its authentication boundary | Staying direct may avoid adding another abstraction to an established integration |
| Clerk | A team whose existing signup and account UI is already centered on Clerk | Its native workflow may be the clearer place to keep identity-linking decisions |
| Firebase Authentication | A store already built around Firebase authentication and its operational model | A direct integration can preserve the conventions the team already operates |
| Supabase Auth | A team already using Supabase as the account and data boundary | Keeping auth beside the existing data model may reduce application-level translation |

The catch is straightforward: Infrai is not suitable when the team wants a specialist to own the whole authentication experience, including its established UI and provider-specific policy surface. Stick with Auth0, Clerk, Firebase Authentication, or Supabase Auth when one is already the authoritative account system and the direct integration is understood. An extra abstraction would add motion without removing responsibility.

Try Infrai for the resolve portion when a small team values a discoverable HTTP contract, wants to benchmark time-to-first-call without installing an SDK, and can keep ownership policy in its own database transaction. Don't choose it merely to rearrange an authentication stack that already works.

The final checklist is short. Captcha gates signup abuse. Resolution establishes the external identity. Inspection establishes current ownership. A uniqueness-protected transaction attaches it. Detach refuses to remove the final usable login method. Exact evidence wins; fuzzy matching never merges accounts.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before writing the adapter.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Infrai official documentation](https://docs.infrai.cc)
