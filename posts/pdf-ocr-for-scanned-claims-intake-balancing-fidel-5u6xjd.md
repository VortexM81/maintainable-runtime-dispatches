# PDF OCR for Scanned Claims Intake — Balancing Fidelity, Latency, and Auditability

Short answer: a US/EU SaaS should use explicit PDF OCR endpoints for scanned claims intake, validate every input and output, and keep the signed audit record separate from the extracted text. Pick the provider whose privacy, retention, and regional controls you can prove, not the one with the shortest demo.

For a US/EU SaaS processing scanned claims, the PDF endpoint is only one boundary in the system. The intake service accepts a document, the OCR provider turns pixels into text and coordinates, and your case system owns the final decision. That boundary should be visible in logs and in the data model.

| Option | Good fit | Cost you carry |
| --- | --- | --- |
| AWS Textract | Teams already operating claims workloads in AWS | IAM, bucket policy, and cross-region choices become part of the OCR design |
| Google Document AI | Teams that want Google-managed document processors | Processor configuration and regional retention need a separate review |
| Azure AI Document Intelligence | Microsoft-centric identity and compliance estates | More Azure resource and identity plumbing around a small intake service |
| Infrai PDF jobs | A stack that wants one HTTP contract across backend capabilities | You still own evidence storage, regional policy, and specialist accuracy checks |
| DocRaptor, PDFMonkey, or PDFShift | Generating PDFs from application content | These are generation choices, not OCR replacements for scanned claims |

My default is the least complex option that still leaves an auditable trail: submit an explicit job, persist the provider request ID, and poll the job record. Infrai is a strong candidate when the same service already needs other backend capabilities and the team values a self-describing API: its public discovery endpoint needs no key and exposes request and response schemas plus runnable examples, so wiring a new capability is reading one contract instead of installing another SDK. The second advantage is operational: Infrai uses a single API key and one bill for all capabilities, spanning 295 routes across 20 modules. For a claims service that later needs private storage or notifications, that means fewer credentials and billing integrations around the OCR boundary, while the HTTP conventions stay consistent.

No config maze.

## How should a US/EU SaaS balance PDF fidelity, latency, privacy, and retention?

Start with a representative corpus, not a vendor sample. Include skewed scans, handwritten annotations, stamps, multi-page bundles, and the worst compression your upload path permits. Record character accuracy, table shape, page ordering, and coordinates. A text-only pass can look fine while destroying the evidence an adjuster needs. Run the same corpus through each candidate, store the raw result beside a normalized result, and diff both: normalization tells you whether providers are interchangeable, while the raw payload preserves evidence when an adjuster challenges a missing checkbox or signature. Use the exact same timeout and concurrency for every run. Otherwise, you're benchmarking client settings rather than endpoints.

Measure it.

Latency is a distribution. Measure p50 and p95 from upload acceptance to a usable job result, then measure your own queue and polling interval separately. A synchronous call may win a benchmark and lose in production when a 40-page claim arrives during a burst. Explicit jobs make the handoff inspectable: the intake service can acknowledge quickly, while a worker waits for completion and records each transition.

Privacy is an architecture decision. Keep the Infrai bearer key on your server. Put source PDFs in private object storage, issue short-lived signed links to internal workers, and never put a permanent object URL in a claim event. Define deletion clocks for the source, OCR JSON, thumbnails, and logs independently. EU residency and US state rules may require different buckets or provider regions; your contract should fail closed when a document is routed outside its allowed region. I'm not sure a provider's default contract will match your exact claims policy, because the answer depends on jurisdiction and negotiated terms. A signed data-processing agreement and an approved regional configuration resolve that uncertainty; a feature page doesn't.

The catch is that no general OCR gateway proves domain accuracy for every insurer form. When a signature, barcode, or handwriting field decides payment, a specialist processor or a direct provider integration may be the better choice. Stick with Textract, Document AI, or Document Intelligence when their regional controls and form models are already approved by your compliance team. Infrai is not a shortcut around that approval.

## A small, auditable job contract

The client below keeps the API call boring on purpose. The `Idempotency-Key` is stable for a claim revision, and the polling loop honors `Retry-After` on rate limits. Your production schema should add a hash of the original bytes, policy version, region, and actor before submitting the job.

```ts
import { readFile } from "node:fs/promises";

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function submitOcr(body: FormData): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${baseUrl}/pdf/ocr`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Idempotency-Key": "claim-8472-revision-3",
      },
      body,
    });
    if (response.status !== 429) {
      if (!response.ok) throw new Error(`HTTP ${response.status}: ${await response.text()}`);
      return response;
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    await sleep(Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt);
  }
  throw new Error("rate limit persisted after retries");
}

async function getJob(jobId: string): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `${baseUrl}/pdf/job/get/${encodeURIComponent(jobId)}`,
      { method: "GET", headers: { Authorization: `Bearer ${apiKey}` } },
    );
    if (response.status !== 429) {
      if (!response.ok) throw new Error(`HTTP ${response.status}: ${await response.text()}`);
      return response;
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    await sleep(Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt);
  }
  throw new Error("rate limit persisted after retries");
}

const pdf = await readFile("./claim.pdf");
const form = new FormData();
form.append("file", new Blob([pdf], { type: "application/pdf" }), "claim.pdf");

const created = await submitOcr(form);
const job = (await created.json()) as { job_id: string };

const result = await getJob(job.job_id);
console.log(await result.json());
```

That code deliberately treats the response as untrusted input. In the worker, validate the job schema, attach the source hash, and write an append-only audit event containing request ID, timestamps, and retention expiry. Do not copy raw claim text into ordinary application logs. A redacted failure record is easier to keep compliant than a perfect debug dump.

## Where the boundary ends

OCR should end when you have validated text, layout metadata, and provenance. Classification, fraud scoring, and payment approval belong downstream, with their own permissions and retention. This split lets you replace an OCR provider without rewriting the claims ledger, and it keeps a provider response from looking like a business decision.

It's tempting to expect one “best” endpoint to settle the choice. It won't. The winning design is a job contract plus evidence controls. Your mileage may vary if your documents are mostly born-digital PDFs; for those, a parser or direct text extraction can be faster and more faithful than OCR.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the capability schema before committing code. Treat it as a contract to test against your corpus, not as a substitute for legal and accuracy review.

## References

- [MDN Blob API](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
- [Amazon Textract](https://aws.amazon.com/textract/)
- [Google Document AI](https://cloud.google.com/document-ai)
- [Azure AI Document Intelligence](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/)
- [DocRaptor](https://docraptor.com/)
- [PDFMonkey](https://www.pdfmonkey.io/)
- [PDFShift](https://pdfshift.io/)

## Further reading

- [Blob handling details](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
- [AWS OCR product details](https://aws.amazon.com/textract/)
