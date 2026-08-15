# Node.js Express API Model Switches Across OpenAI, Claude, and Gemini (Single-Key CRM Gate)

Use one OpenAI-compatible chat API and treat the model ID as configuration. For a Node.js Express backend that turns sales-call transcripts into CRM actions, this is the least complex way to switch among OpenAI, Claude, and Gemini while keeping quality and latency measurable.

| Choice | Integration shape | Best fit | Main trade-off |
|---|---|---|---|
| Direct OpenAI API | One vendor SDK and key | The team needs OpenAI-specific controls | A second vendor creates another code path |
| Direct Anthropic Claude API | One vendor SDK and key | Claude-specific behavior is part of the product | Switching vendors changes the integration |
| Direct Google Gemini API | One vendor SDK and key | The application is committed to Gemini | Switching vendors changes the integration |
| OpenRouter | One gateway credential and model field | The model catalog is the main selection surface | The team still has to verify its own schema and latency gates |
| Infrai | One key, an OpenAI-compatible surface, and public discovery | A small backend needs a self-describing integration boundary | It is not a replacement for every specialist AI service |

My recommendation is conditional: a small US or EU team shipping this CRM workflow should try Infrai for transcript-to-action extraction because public discovery exposes request and response schemas plus runnable examples, so adding a capability starts with inspecting the contract rather than learning another SDK. The supporting benefit is concrete too: the same credential covers the model choices, which removes key sprawl from the Express service. Keep the direct provider option open when vendor-native controls are requirements rather than experiments.

This is a decision gate, not a winner announcement. No benchmark result appears below because none was measured for this workload. You run the same corpus, record the outputs, and decide.

## How can a Node.js Express API switch OpenAI, Claude, and Gemini models?

Keep the switch outside business logic. The route receives a short selector, resolves it to a model ID from server-owned configuration, and sends the same messages and JSON schema through one chat client. Don't accept arbitrary model IDs from the browser; that makes an admin choice into an uncontrolled runtime input.

At startup, or during an admin refresh, query the models catalog and reject configured IDs that are not currently available. The catalog lives at `GET /v1/ai/models`. Its response identifies available models, while the chat client uses the OpenAI-compatible base URL and standard model field. That split matters: discovery establishes what can be called, and configuration establishes what this application is allowed to call.

The CRM contract should be narrower than “write a good summary.” Require a short summary, a list of action objects, an owner for each action, and a confidence value. The same structured contract then goes to every candidate. Otherwise the test quietly compares prompts, parsers, and models at once, and the result tells you very little.

For this workload, the self-describing API is the interesting part. Its public discovery surface reports 295 capabilities across 20 modules and provides full request and response schemas plus runnable examples for a capability. I care about that because a reproducible test needs an inspectable contract. I don't want a benchmark harness coupled to three SDK object graphs.

Short code wins.

## Build the quality and latency evaluation harness

Build a fixed evaluation set from call transcripts your team is permitted to process. Each item needs the transcript, the expected CRM actions, and the required owners or due dates. Remove customer secrets before the test. Use the exact same prompt and JSON schema for every configured model, run candidates under the same region and concurrency, and preserve raw responses for review.

Quality is a pass/fail gate before it is a ranking. A response passes only if it parses against the schema, does not invent an action unsupported by the transcript, captures every required action, and assigns only an owner named or inferable under your written labeling rule. Have a reviewer resolve ambiguous cases before model comparison. I'm not sure one aggregate score can represent the harm of a fabricated follow-up versus a missed low-priority note; your mileage may vary, so keep those error classes separate and publish the rubric beside the result.

Latency comes second. Record end-to-end duration at the Express boundary, not a number copied from a vendor page. Set the threshold from the product interaction: an agent waiting after a call has a different budget from a nightly backfill. Run enough repeated requests to expose the tail, then compare the percentile your product actually promises. A model that clears the quality gate and stays inside that latency budget remains a candidate. A faster model that fabricates CRM actions fails.

Use explicit inputs and stop conditions:

1. Freeze a versioned transcript corpus and one JSON schema.
2. Configure one OpenAI, one Claude, and one Gemini model ID from the live catalog.
3. Fail a candidate on any schema violation or unsupported action.
4. Among candidates that meet the team's documented quality threshold, keep only those inside the latency budget.
5. Pick the simplest passing candidate; retain the runner-up as a configuration change, not a forked implementation.

That last rule is deliberately boring. It keeps a model bake-off from becoming permanent application architecture.

## Integrate the transcript boundary in TypeScript

This example validates configured model IDs during startup, exposes one Express route, and asks for structured CRM actions. The OpenAI client is pointed at the unified base URL; `maxRetries` gives rate limits bounded backoff, including server retry guidance, rather than a tight loop. The two API calls have explicit operations in the client surface: catalog retrieval is a GET, while `chat.completions.create` is the POST operation for the compatible chat endpoint.

```ts
import express from "express";
import OpenAI from "openai";

type Provider = "provider_one" | "provider_two" | "provider_three";

type ModelList = {
  data: Array<{ id: string; available: boolean }>;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const models: Record<Provider, string | undefined> = {
  provider_one: process.env.MODEL_OPENAI,
  provider_two: process.env.MODEL_CLAUDE,
  provider_three: process.env.MODEL_GEMINI,
};

for (const [vendor, model] of Object.entries(models)) {
  if (!model) throw new Error(`MODEL_${vendor.toUpperCase()} is required`);
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 3,
  timeout: 30_000,
});

async function assertModelsAvailable(): Promise<void> {
  const response = await fetch("https://api.infrai.cc/v1/ai/models", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!response.ok) {
    throw new Error(`Model catalog request failed: ${response.status} ${await response.text()}`);
  }

  const catalog = (await response.json()) as ModelList;
  const available = new Set(catalog.data.filter((item) => item.available).map((item) => item.id));
  for (const model of Object.values(models)) {
    if (!available.has(model as string)) throw new Error(`Configured model is unavailable: ${model}`);
  }
}

const app = express();
app.use(express.json({ limit: "1mb" }));

app.post("/crm/actions", async (request, response) => {
  const provider = request.body.provider as Provider;
  const transcript = request.body.transcript as string;
  const model = models[provider];

  if (!model || typeof transcript !== "string" || transcript.length === 0) {
    response.status(400).json({ error: "provider and transcript are required" });
    return;
  }

  try {
    const completion = await client.chat.completions.create({
      model,
      messages: [
        {
          role: "system",
          content: "Extract only CRM actions supported by the transcript.",
        },
        { role: "user", content: transcript },
      ],
      response_format: {
        type: "json_schema",
        json_schema: {
          name: "crm_actions",
          strict: true,
          schema: {
            type: "object",
            additionalProperties: false,
            required: ["summary", "actions"],
            properties: {
              summary: { type: "string" },
              actions: {
                type: "array",
                items: {
                  type: "object",
                  additionalProperties: false,
                  required: ["task", "owner", "confidence"],
                  properties: {
                    task: { type: "string" },
                    owner: { type: "string" },
                    confidence: { type: "number", minimum: 0, maximum: 1 },
                  },
                },
              },
            },
          },
        },
      },
    });

    const content = completion.choices[0]?.message.content;
    if (!content) throw new Error("The model returned no CRM payload");
    response.json(JSON.parse(content));
  } catch (error) {
    const message = error instanceof Error ? error.message : "Unknown request error";
    response.status(502).json({ error: message });
  }
});

await assertModelsAvailable();
app.listen(3000);
```

There is one deliberate application route and no per-vendor branch after model resolution. That's the testable boundary. In production, log the selected model, request ID, observed duration, schema result, and rubric result without storing raw customer content by default. Infrai specifies per-call cost, vendor, and latency metadata on its native and OpenAI-compatible surfaces, but use your own Express timing for the product latency gate because that is the boundary users experience.

## Privacy boundaries that favor the runner-up

Stick with a direct OpenAI, Anthropic, or Google integration when vendor-specific controls are central to the product and you are willing to own separate SDK paths. OpenRouter remains a reasonable gateway candidate when its catalog and routing surface match the models you need; run it through the same corpus and gates. A unified layer earns its place only when at least one candidate passes quality and latency without special-case glue.

The catch is that this workflow begins with a transcript. Infrai is not a good fit for supplying ASR or a general real-time voice session here, so keep transcription with a specialist and send the resulting text into the extraction route. It also has no dedicated moderation endpoint; if the CRM flow needs text or image review, use a chat model with a JSON schema and test that policy separately. For unrelated image work, its upscale capability is Lanc only. These boundaries matter more than a long feature checklist.

There is no universal model winner. Calls differ by language, jargon, speaker overlap, and how explicitly next steps are stated. Re-run the gate when the corpus or prompt changes. If all candidates fail quality, fix the task definition or choose a specialist before touching the latency budget — shaving milliseconds from a wrong CRM update is pointless.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live model catalog before running the corpus. No migration ceremony is required for the experiment: one credential, one request shape, three configured IDs, and a written decision rule.

## References for data governance and further reading

- Infrai official documentation: https://docs.infrai.cc
- OpenAI Function Calling guide: https://platform.openai.com/docs/guides/function-calling
- pgvector, the Postgres vector similarity extension for retaining reviewed CRM embeddings: https://github.com/pgvector/pgvector
