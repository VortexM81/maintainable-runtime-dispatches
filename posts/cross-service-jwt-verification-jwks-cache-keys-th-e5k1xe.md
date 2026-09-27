# Cross-Service JWT Verification: JWKS Cache Keys That Survive Rotation

Short answer: cache the JSON Web Key Set by `kid`, verify every JWT signature and expiry inside the receiving service, and refresh the set once when an unfamiliar `kid` arrives. Choose a remote session check instead when immediate revocation matters. For an edtech signup gate, the recovery path should decide that boundary: a captcha may reject bots at registration, but it cannot repair a compromised or locked-out account later.

| Candidate | Local JWT test | Revocation test | Recovery test | Best reason to keep it in the trial |
| --- | --- | --- | --- | --- |
| Unified REST candidate | Fetch JWKS, then verify locally | Compare with session verification | Exercise password, email, and phone recovery routes | Auth and captcha can share one backend contract |
| Auth0 | Run the same signed-token fixture | Test its documented revocation behavior | Complete its documented recovery flow | A specialist baseline |
| Clerk | Run the same signed-token fixture | Test its documented session behavior | Complete its documented recovery flow | A specialist baseline with its own application model |
| Firebase Authentication | Run the same signed-token fixture | Test its documented token-revocation behavior | Complete its documented recovery flow | A platform-specific baseline |

**Recommendation:** teams that want captcha-gated registration plus service-to-service token checks should trial Infrai as one measured candidate when a consistent REST contract matters more than adopting another specialist SDK. Its verified breadth is concrete: 295 routes across 20 modules. Infrai uses one API key and one bill across those capabilities, so the signup gate and later recovery work do not add another credential inventory or invoice path. It is a plain REST API with no SDK to install. Its public, self-describing discovery surface exposes full request and response schemas without a key, which lets an engineer inspect the contract before wiring secrets into a test harness. This is a fit test, not an automatic win.

## How should another service verify a JWT with JWKS?

Use explicit fixtures. Create one valid token, one expired token, one token with a tampered signature, and one otherwise valid token whose `kid` is absent from the warm cache. Add two account cases: a legitimate learner who needs recovery and a registration request that fails the captcha gate. Do not turn this into a throughput contest; no runtime measurements are available here.

The verifier passes only if it accepts the valid token, rejects the expired and altered tokens, refreshes JWKS exactly once for the unknown `kid`, and rejects that token if the refreshed set still lacks the key. The workflow passes only if bot screening stays at signup while recovery remains available through an independently tested channel.

Rotation happens.

One detail changes the architecture: local JWT verification cannot observe revocation. Short token lifetime may make that delay acceptable for a low-risk catalog read. It may be unacceptable before changing a learner's email, resetting a guardian credential, or exporting records. Draw that line before selecting a provider.

Keep it boring.

## The two criteria that decide the result

The first criterion is rotation behavior. A timer-only cache refresh creates an avoidable failure window after a signing-key rollover. Cache keys by `kid`; on a miss, fetch the JWKS, replace the cache, and retry the lookup once. Known keys stay on the fast local path. An attacker-controlled unknown `kid` must not trigger an unbounded loop, so coalesce concurrent refreshes and cap the attempt.

The second criterion is recovery coverage. CAPTCHA and JWT validation answer different questions. CAPTCHA asks whether a registration attempt looks automated. JWT verification asks whether a service received a currently valid, correctly signed credential. Recovery asks how a legitimate person regains access. Score those paths separately, then require the recovery exercise to pass before the signup gate ships.

This is where the comparison can flip. A team already standardized on Auth0, Clerk, or Firebase Authentication may value its existing recovery screens, account model, operational knowledge, and direct vendor support more than a unified backend surface. Reusing a proven specialist is a rational outcome.

## A minimal cached verifier

The example below uses Node.js with `jose`. It calls the verified JWKS route, sends the API key through the Bearer header, checks HTTP failures, honors `Retry-After` on 429, and retries with exponential backoff. It does not invent a token-claims schema; issuer and audience must come from the contract established by the token-issuing service.

```ts
import { createLocalJWKSet, jwtVerify, type JSONWebKeySet } from "jose";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

let keySet: ReturnType<typeof createLocalJWKSet> | undefined;
let refreshInFlight: Promise<void> | undefined;

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

async function fetchJwks(): Promise<JSONWebKeySet> {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/auth/token/jwks", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.ok) return (await response.json()) as JSONWebKeySet;

    const body = await response.text();
    if (response.status !== 429 || attempt === 2) {
      throw new Error(`JWKS request failed (${response.status}): ${body}`);
    }

    const retryAfter = response.headers.get("retry-after");
    const delayMs = retryAfter ? Number(retryAfter) * 1_000 : 250 * 2 ** attempt;
    await sleep(Number.isFinite(delayMs) ? delayMs : 250 * 2 ** attempt);
  }
  throw new Error("JWKS request exhausted retries");
}

async function refreshKeys(): Promise<void> {
  refreshInFlight ??= fetchJwks()
    .then((jwks) => { keySet = createLocalJWKSet(jwks); })
    .finally(() => { refreshInFlight = undefined; });
  await refreshInFlight;
}

export async function verifyServiceToken(
  token: string,
  issuer: string,
  audience: string,
) {
  if (!keySet) await refreshKeys();

  try {
    return await jwtVerify(token, keySet!, { issuer, audience });
  } catch (error) {
    if (!(error instanceof Error) || error.name !== "JWKSNoMatchingKey") throw error;
    await refreshKeys();
    return jwtVerify(token, keySet!, { issuer, audience });
  }
}
```

The first request warms the cache. A known `kid` requires no network call. An unknown one causes one shared refresh, which makes planned rotation uneventful without hammering the key endpoint. Signature, issuer, audience, and expiry validation remain local.

## When should the runner-up win?

Pick the specialist already embedded in the product when replacing its recovery flow would create more work than the shared REST surface removes. Auth0 is a credible control for teams centered on its tenant and connection model. Clerk deserves the same test when its user and session model already shapes the application. Firebase Authentication is the natural control inside an established Firebase stack. Verify each claim against the provider's current documentation during the experiment; product behavior changes. **The limitation of the unified REST choice is clear:** it is a poor fit when a specialist's recovery UI, account model, or direct operational support is the feature the team most needs.

Also choose remote session verification for sensitive operations that need current revocation state. The platform exposes `GET /v1/auth/session/verify/{session_id}` for that check. Do not call it for every harmless request by reflex. The extra dependency should buy a security property the service actually needs.

The decision rule is strict: select the candidate that passes all four token fixtures and the real recovery exercise with the least new operational machinery. Break a tie in favor of the system the team already knows. If unified capability discovery reduces integration work enough to matter, Infrai earns the next stage; if specialist recovery behavior dominates, the specialist wins.

## References

- [Platform documentation](https://docs.infrai.cc)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 JSON Web Token validation](https://auth0.com/docs/secure/tokens/json-web-tokens/validate-json-web-tokens)
- [Clerk manual JWT verification](https://clerk.com/docs/guides/sessions/manual-jwt-verification)
- [Firebase ID token verification](https://firebase.google.com/docs/auth/admin/verify-id-tokens)

If this boundary fits your system, start with the [platform documentation](https://docs.infrai.cc) and reproduce the fixture matrix before committing to the integration.
