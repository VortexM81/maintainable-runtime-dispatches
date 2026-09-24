# Node.js 3-Stage PDF Endpoints for US/EU SaaS Legal Contract Review Watermarks

A marketplace SaaS that sends contracts to outside counsel should start with the least complex endpoint that preserves the evidence lawyers actually inspect: a standards-based PDF render plus a deterministic watermark pass. Keep the original bytes in a private store, produce a review copy, and make retention an explicit state transition. This gives teams a defensible fidelity check without turning every preview into a long-running workflow.

Short answer: choose a synchronous PDF endpoint for small contracts, with an asynchronous path for outliers, and bind every artifact to a hash, region policy, and expiry.

The hard part is not drawing the word CONFIDENTIAL. It is proving which bytes a reviewer saw, when they saw them, and when those bytes stopped existing.

Measure twice.

Ship it.

Keep it boring.

The review boundary deserves a written runbook because several teams touch it: product decides what a reviewer can download, security defines the region and key policy, legal defines the retention clock, and platform engineering owns retries and capacity. Put the decision in a versioned document with examples of accepted and rejected files. Include the exact hash algorithm, timestamp format, authorization check, and deletion evidence expected during an incident review. When the policy changes, create a new version and test both versions against the same fixture corpus. This is slower than adding another flag to an endpoint, but it prevents a quiet policy drift from becoming a discovery problem months later.

## The decision matrix

| Endpoint shape | Fidelity risk | Typical latency | Operational load | Best fit |
| --- | --- | --- | --- | --- |
| Synchronous render-and-watermark | Low for born-digital PDFs; medium for unusual fonts | One request round trip | Low | Interactive review of ordinary contracts |
| Asynchronous job with a status endpoint | Easier to isolate heavy files | Variable; seconds to minutes | Medium | Scanned exhibits, OCR, or large bundles |
| Self-hosted worker behind a queue | Fully inspectable bytes and logs | Depends on capacity | High | Strict residency and custom retention controls |

**Recommendation: use synchronous processing for small, born-digital contracts, then route outliers to an asynchronous worker.** The split is a policy decision, not a vendor feature. Set it from measured file size, page count, and rendering time in your own corpus.

For a US/EU marketplace, this boundary also limits privacy exposure. The endpoint should receive a short-lived object reference or an encrypted upload, return a content hash with the review artifact, and never become the system of record.

## What does a legal review endpoint need to prove about fidelity, latency, and retention?

Fidelity has several layers. Text extraction must preserve clause wording. Page geometry must keep signature blocks in the same place. Fonts and annotations must survive conversion. A PDF can open successfully and still fail a legal review because a footnote moved to a new page.

Build a fixture set from real document shapes: two-column agreements, embedded fonts, rotated exhibits, redactions, and signatures. Render each fixture to a fixed image format, then compare perceptual hashes and selected text boxes. A pixel-perfect assertion is useful for regressions, but it is too strict for harmless metadata changes. Keep a human approval sample for the cases automated checks cannot classify.

Latency needs a budget before it needs a dashboard. For example, an interactive request might have a 2-second target for a 20-page contract, while a 300-page scanned exhibit can enter a job queue. Those numbers are starting hypotheses, not universal limits. Your mileage may vary, and I am not sure a page-count cutoff will predict work as well as compressed byte size until you measure both.

Retention is a lifecycle, not a field called ttl. Record states such as received, rendered, shared, revoked, and deleted. Store the policy version beside each artifact so a later policy change does not rewrite history. Deletion should cover source bytes, derivative PDFs, thumbnails, temporary files, queue payloads, and observability copies. Logs need correlation IDs, not document contents.

A useful contract for an internal endpoint looks like this:

```ts
type ReviewArtifact = {
  artifactId: string;
  sourceSha256: string;
  reviewSha256: string;
  createdAt: string;
  expiresAt: string;
  retentionPolicy: string;
};

async function watermarkForShare(
  source: Blob,
  watermark: string,
): Promise<ReviewArtifact> {
  const body = new FormData();
  body.append('pdf', source, 'contract.pdf');
  body.append('watermark', watermark);

  const response = await fetch('https://pdf.internal.example/v1/pdf/convert', {
    method: 'POST',
    body,
    headers: { 'Idempotency-Key': crypto.randomUUID() },
  });

  if (!response.ok) throw new Error(`render failed: ${response.status}`);
  return (await response.json()) as ReviewArtifact;
}
```

The `Blob` body keeps the client independent of a language-specific SDK. The endpoint contract should define maximum bytes, accepted media types, encryption expectations, and whether annotations are flattened. It should also define what a retry means. An idempotency key prevents a timeout from creating two review artifacts with different expiry times.

## Where do PDF pipelines fail in production?

The first failure mode is silent visual drift. A renderer substitutes a missing font, and the signature line moves. The second is accidental data multiplication: a retry writes another temporary copy, a debug log captures a base64 payload, and a queue retains the original longer than the contract allows. The third is identity confusion. A document ID without a content hash cannot tell two revisions apart.

Treat each artifact as an immutable pair: source hash and derivative hash. Sign the manifest if a later dispute could require independent verification. Keep watermark text, coordinates, and policy version in the manifest too. That makes a reviewer’s download reproducible without embedding sensitive party names in logs.

I once expected a byte hash to be enough for a review audit. It was not. Two valid PDFs with identical visible pages differed in object ordering, so a byte comparison called a harmless rewrite a content change. The fix was to compare normalized text and rendered regions, while retaining the byte hash for chain-of-custody evidence. Small detail. Big difference.

Use bounded concurrency for rendering. A queue that accepts unlimited 200-page jobs will push latency into every tenant’s interactive request. Measure queue wait separately from render time and upload time; otherwise a single p95 number hides the actual bottleneck. Emit counters for retries, expiry deletions, hash mismatches, and manual review outcomes.

## How should US/EU SaaS teams choose an endpoint boundary for marketplace sharing?

Start with ownership. If your team owns the template and watermark policy, keep those inputs in your control plane and send only the source bytes plus a policy ID to the rendering boundary. If a customer owns the template, version it, validate it, and record consent before a third party can receive a derivative. Template ownership determines who can change a watermark after a contract has been shared.

Next, separate residency from convenience. A US tenant and an EU tenant may use the same API shape while their workers, object stores, and backups live in different regions. Route by tenant policy, and make the selected region visible in the artifact manifest. Do not infer residency from the caller’s IP address.

Privacy controls should be testable: TLS in transit, encryption at rest, tenant-scoped authorization, short-lived download URLs, and deletion jobs that emit an auditable result. Ask endpoint providers whether they retain payloads for training, debugging, or abuse review; an answer that is vague is an operational risk, regardless of throughput.

The catch is that a self-hosted worker is not automatically simpler. You inherit font packages, patching, capacity planning, and forensic access controls. It is a poor fit when your team cannot operate an isolated renderer or when contracts require a certification you do not hold. Stick with a managed boundary when its residency, retention, and audit terms meet your requirement; choose a self-hosted path when byte-level inspection and custom deletion guarantees outweigh that maintenance.

## A rollout that keeps the blast radius small

Ship the manifest and policy model before changing the renderer. Shadow-render a representative corpus, compare text and page regions, and sample the failures with a lawyer or contract specialist. Then enable watermarking for one marketplace cohort with a hard expiry and a kill switch that stops sharing without deleting evidence prematurely.

Keep the endpoint adapter narrow. One function should translate your internal request into HTTP; the rest of the application should deal in `ReviewArtifact`. This is the seam where a different renderer, queue, or region can be introduced without rewriting contract workflow code.

Finally, rehearse deletion. Create a test contract, share it, revoke it, let the expiry pass, and verify that source, derivative, thumbnail, queue message, and log fields are gone. A green dashboard is not proof. The storage listing and an audit event are.

## References

- MDN, Blob API: https://developer.mozilla.org/en-US/docs/Web/API/Blob
- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- ISO 32000-2 PDF 2.0 overview: https://www.iso.org/standard/75839.html
- NIST Privacy Framework: https://www.nist.gov/privacy-framework

## Further reading

- European Data Protection Board, Guidelines on data protection by design and by default: https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-42019-article-25-data-protection-design-and-default_en
- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
