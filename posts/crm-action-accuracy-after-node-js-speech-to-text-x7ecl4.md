# CRM Action Accuracy After Node.js Speech-to-Text MP3/WAV Uploads in US and EU

**Short answer:** Choose the speech-to-text API that produces the most correct CRM actions from your real sales calls, not the one with the shortest MP3 upload snippet. For a fast Node.js integration, gate candidates first on US or EU processing requirements, then run the same WAV and MP3 fixtures through transcription and strict action validation.

| Decision note | Choose it when | Runner-up wins when |
|---|---|---|
| Direct file upload | Calls are short enough for one request and the API returns a complete transcript | Long recordings make durable jobs easier to recover and observe |
| Asynchronous job | The application can persist a job ID and reconcile completion | A synchronous call meets the workload without polling state |
| Transcript-only response | A separate extraction step is already tested and owned | A combined response has an explicit, enforceable action schema |

My recommendation is a two-stage contract: audio becomes a transcript, then the transcript becomes validated CRM actions. It adds one boundary, but it lets a team replace either stage and diagnose which one lost the customer name, date, owner, or next step. The fastest integration is the smallest one that stays debuggable after the demo.

## What should the decision actually optimize?

Structured output correctness is the primary score. Word accuracy matters, but a sales workflow fails in more expensive ways: “send the security packet Friday” becomes a generic note, a tentative date becomes a committed deadline, or a comment from the account executive gets attributed to the buyer. A fluent transcript can still produce the wrong task.

Define the action contract before opening any API documentation. For a small B2B SaaS team, a useful record might contain `accountId`, `summary`, `tasks`, `owner`, `dueDate`, and an evidence span copied from the transcript. Make uncertain values nullable. Don't let the model invent a date merely because the CRM field is required — route a missing date to review instead.

The benchmark should use production-shaped fixtures with expected actions reviewed by someone who understands the call. Include an MP3 exported by the meeting tool, a WAV recording, overlapping speakers, an acronym-heavy product discussion, a negated action such as “don't schedule migration yet,” and a call with no next step. Keep the gold set small enough to inspect after every adapter change but varied enough to expose destructive assumptions.

Consider one deliberately awkward fixture: the buyer says, “I can send the security questionnaire by May 8,” the account executive replies, “Great, I'll ask Priya to review it,” and later the buyer says, “Actually, don't put May 8 in the plan until legal confirms.” The expected output should preserve the questionnaire and Priya's review as context while withholding the date-bound CRM task. Now perturb the audio without changing its meaning: encode one copy as MP3, keep another as WAV, add a few seconds of cross-talk before the correction, and use the same expected action set for both. If one path emits a May 8 deadline while the other routes the ambiguous date to review, the aggregate transcript may still look excellent, but the workflow is inconsistent where it matters. Inspect the evidence span, speaker attribution, negation, and final structured object together. This one fixture tests more than a polished ten-second monologue because it forces the system to carry a correction through two contracts — transcription and extraction — without turning conversational momentum into a false commitment.

I score exact fields separately. A summary can tolerate wording variation; `owner` and `dueDate` usually can't. For each fixture, count missing required actions, invented actions, wrong owners, wrong dates, and unsupported claims. Record transcription time and extraction time independently. I'm not sure a single weighted score is defensible across every sales team, because a missed compliance promise and a misspelled company name don't carry the same risk; the team's CRM policy must supply those weights.

One result is immediate: raw transcript latency is a tiebreaker only after the action set passes its correctness threshold.

## How should a Node.js speech-to-text API handle MP3 and WAV file uploads?

Keep the upload adapter thin and explicit. The URL, authorization header, model identifier, multipart field names, and response mapping must come from the candidate's current documentation. There is no safe universal endpoint to guess, and wrapping four different contracts in a clever abstraction before any call succeeds just hides the differences that the trial needs to measure.

This TypeScript probe deliberately treats the endpoint and form fields as configuration. It accepts MP3 and WAV input, imposes a request deadline, preserves the provider response for audit, and returns an unknown payload for a provider-specific mapper to validate. The code does not claim that every service uses `file` or `model`; set those names only when its documented contract calls for them.

```ts
import { readFile } from "node:fs/promises";
import { basename, extname } from "node:path";

type UploadConfig = {
  url: string;
  apiKey: string;
  fileField: string;
  modelField?: string;
  model?: string;
};

const allowedExtensions = new Set([".mp3", ".wav"]);

async function transcribeFile(
  filePath: string,
  config: UploadConfig,
): Promise<unknown> {
  const extension = extname(filePath).toLowerCase();
  if (!allowedExtensions.has(extension)) {
    throw new Error(`Expected an MP3 or WAV file, received ${extension}`);
  }

  const bytes = await readFile(filePath);
  const form = new FormData();
  form.set(config.fileField, new Blob([bytes]), basename(filePath));

  if (config.modelField && config.model) {
    form.set(config.modelField, config.model);
  }

  const response = await fetch(config.url, {
    method: "POST",
    headers: { Authorization: `Bearer ${config.apiKey}` },
    body: form,
    signal: AbortSignal.timeout(60_000),
  });

  const responseBody = await response.text();
  if (!response.ok) {
    throw new Error(`Upload failed (${response.status}): ${responseBody}`);
  }

  return JSON.parse(responseBody) as unknown;
}
```

That's enough glue.

Do not silently retry every failure. A `429` backoff retry should follow the service's documented rate-limit behavior, but a `400` or `422` usually signals that the request contract or input needs attention. For asynchronous APIs, persist the provider job ID beside an internal call ID before polling. Use a documented idempotency key for submission retries when the service supports one; polling retries do not create the same duplication risk. Those details belong in the evaluation because they determine time-to-first-*reliable*-call.

The mapper after `transcribeFile` should reject missing transcript text rather than converting it to an empty string. Save the raw response, normalized transcript, extraction input, structured output, schema-validation result, and correlation ID. Redact or encrypt those artifacts according to the company's audio and transcript policy. Logs should expose timing and state transitions without copying an entire customer conversation into routine application telemetry.

## Regional fit is a gate, not a checkbox

“Works for EU users” and “processes audio in an approved EU region” are different statements. Before benchmarking speed, verify the current service terms, selectable processing location, storage behavior, retention controls, and any subprocessors against the team's requirements. Run the probe from the deployment region that will make the production request. Repeat the same check for a US workload rather than assuming one global result.

Reject a candidate that misses the required processing boundary even if its example runs in five minutes. Also reject an integration whose region selection lives only in an untracked dashboard setting. Config that changes data handling should be reviewable and testable alongside the application — config bloat is bad, invisible config is worse.

Keep the regional test factual. Capture the selected region, documented control, request origin, timestamp, and reviewer. Don't infer residency from latency, a hostname, or the location of the customer. Your mileage may vary as contracts and deployment options change, so repeat this gate during procurement review and before a material architecture change.

## The second criterion is recoverable completion

A direct upload is attractive because the control flow fits on one screen. For short recordings, that can be the right trade. The catch is that HTTP request lifetime, reverse-proxy limits, and client disconnects become part of the job model. If the application cannot tell “still processing” from “lost response,” a minimal example has produced ambiguous state.

An asynchronous design costs more code: submit, persist, poll or receive a callback, validate, and reconcile. It is the better runner-up when calls are long, processing duration varies, or workers must resume after a deploy. The team should measure that extra state honestly. Count provider-specific configuration, adapter lines, required secrets, webhook verification, retry branches, and dashboards touched before the first validated CRM action lands.

No benchmark invented here.

Use a state machine such as `received`, `uploaded`, `transcribed`, `extracted`, `validated`, and `review_required`. Each transition should be idempotent and carry one correlation ID. Failed schema validation must retain the transcript and validation errors for controlled review; it must not write a half-valid task into the CRM. For duplicate callbacks or worker retries, the CRM write needs a stable action key so one promise to call Friday doesn't become three tasks.

Batch processing is worth considering only when immediate CRM updates are unnecessary. It changes the product promise and the recovery loop, so evaluate it as a distinct architecture rather than an optimization toggle. Likewise, vector search may help retrieve earlier account context, but it should not become a prerequisite for the first correct action. Every extra store expands the test surface.

## When should the runner-up become the default?

Stick with the synchronous runner-up when real recordings finish comfortably inside the application's request budget, the response contract is stable, and the smaller operational surface makes failures easier to understand. Choose the asynchronous runner-up when durable recovery and long-call behavior outweigh the extra polling or callback machinery. If strict regional controls are the hard requirement, the eligible design with explicit, reviewable location settings wins before either convenience argument.

The two-stage transcript-then-extract recommendation is not suitable when the selected API can enforce the exact CRM schema in one documented operation and the team can independently audit transcript evidence. It is also a poor fit for a live agent-assist interface where partial results and turn latency define the experience; that workload needs a streaming evaluation, not this file-upload gate.

For the sales-call workflow, ship only after every gold fixture produces schema-valid actions, all high-risk fields meet the team's threshold, unsupported actions go to review, MP3 and WAV behavior is understood, and the approved US or EU processing setup is documented. Then rerun the fixture set whenever the upload adapter, transcription configuration, extraction prompt, schema, or model changes.

That is the selection rule: choose the least-glue architecture that preserves evidence, passes the regional gate, and writes the right CRM actions. A quick upload is useful. A reproducible decision is better.

## Sources

- https://platform.openai.com/docs/guides/batch
- https://github.com/pgvector/pgvector
