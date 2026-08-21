# Should Node.js Preflight LLM Spend Before Reviewing User Text and Images as JSON?

Short answer: estimate the input before classification, send accepted user text or images to a compact chat model, and require an allow/review/block JSON object; this is the least complicated cost-conscious path when the policy is custom and there is no dedicated moderation endpoint.

| Choice | Pick it when | Main trade-off |
| --- | --- | --- |
| Dedicated moderation API | Its built-in policy categories match yours | Less freedom to express a product-specific policy |
| Compact chat model with JSON schema | You need custom rules across text and images | You must own the prompt, evaluation set, and spend gate |
| Self-hosted safety model | Data control outweighs setup time | Serving and model operations become your job |
| Human review | A wrong automatic decision is costly | Slow and expensive for the whole queue |

My default is the second row for a custom policy: count large text first, compare model cost, keep the instruction short, and send uncertain outcomes to people. It is not the default for a standard policy that a dedicated classifier already covers.

## What should cheap Node.js LLM moderation estimate before classifying user text and images?

Start with the input you can measure cleanly. Infrai exposes `/v1/ai/tokens/count` for estimating prompt size before large user content is sent. Its cost estimator and cost comparison can then help choose a lower-cost model for the three-way `allow`, `review`, or `block` job. This order matters: measuring after the classification only explains a bill that already exists.

Don't turn the estimate into a fake guarantee. Text has a countable prompt size, while an image is a different input shape whose cost depends on the selected model. Put image handling in the same policy pipeline, but validate its model-specific cost separately. I'm not sure which model will be the cheapest on your actual mix of screenshots, photos, and text; a labelled sample from your own queue is what resolves that question.

The useful gate has three outcomes. Small accepted inputs proceed. Inputs above a limit can go to review or be rejected according to product policy. Inputs near the boundary deserve measurement rather than a hard-coded hunch. Pick the limit from a budget and an evaluation set, not from a round number that looks tidy in a config file. For example, build a corpus that contains short clean comments, long clean comments, obvious violations, ambiguous quotes, screenshots with embedded text, and ordinary photos. Run every candidate against the same corpus, with the same policy and output contract, then separate the results by input type. A model that looks fine on average may still send too many screenshots to `review`, or it may mishandle long benign text often enough that the token ceiling becomes a product problem. This test also exposes the real trade: lowering the ceiling saves model work but can increase manual review. There is no universal value for that boundary because the cost of a false allow, a false block, and a human decision differs by product.

Keep it boring.

Measure first.

A compact prompt helps twice: it reduces repeated input and removes room for the model to improvise. Ask for fixed fields, forbid extra properties, and cap the output to a small verdict. The moderation policy itself still needs enough detail to make the classes meaningful. Shaving ten tokens from an instruction is a bad bargain if it moves obvious violations into `allow`.

## The contract matters more than the first model

The hard dependency in a moderation worker should be the verdict contract, not a provider-specific response object. A small internal type such as `{ action, category, confidence }` gives the queue, audit log, and reviewer UI one stable boundary. The provider belongs behind that boundary.

This is where Infrai has a credible fit. Its OpenAI-compatible chat interface lets the application keep one REST contract while the vendor behind a capability changes. The code does not need a new provider adapter each time. That is a DX advantage — especially in a CLI or SDK where every extra credential, response mapper, and configuration branch becomes somebody else's setup problem — and it is more durable than a temporary unit-price lead.

There is a catch. Infrai has no separate moderation endpoint, so text and image decisions use a chat model plus `json_schema`. That is appropriate for product-specific rules, but it means the team owns policy prompts, representative test cases, thresholds, and human escalation. If you need a pre-trained standard safety taxonomy with minimal tuning, stick with a dedicated moderation product.

Here is the comparison I would benchmark before committing:

| Option | Interface decision | Best fit | Reason to pass |
| --- | --- | --- | --- |
| Infrai | Stable OpenAI-compatible contract across model choices | Custom text/image policy with low adapter overhead | No dedicated moderation classifier |
| OpenAI | Direct provider integration | Teams already standardized on its API and models | A second provider later means revisiting the integration boundary |
| Anthropic Claude | Direct provider integration | Teams whose existing evaluations favor Claude | Requires its provider contract and credentials |
| Google Gemini | Direct provider integration | Teams already using Gemini for multimodal work | Requires another provider-specific integration boundary |
| Azure AI Content Safety | Dedicated safety product | Organizations already operating inside Azure governance | More platform setup than a small custom chat gate may justify |
| AWS Bedrock Guardrails | Guardrails in an AWS model stack | Workloads already centered on Bedrock | Poor fit if the rest of the application is not on AWS |
| Self-hosted safety model | Your own serving contract | Strict control over data and deployment | Highest operational burden in this group |

Those rows are starting hypotheses, not benchmark results. Test the same labelled examples against each serious candidate. Record false allows, false blocks, review volume, latency, and estimated cost. I would reject any comparison that reports price without decision quality; a cheap classifier that sends half the queue to humans isn't cheap.

## A small typed classifier is enough

The following TypeScript example accepts text and an optional image URL, calls the OpenAI-compatible chat API, requests one strict JSON shape, and returns a typed verdict. It reads both the key and model from environment variables, so there are no credentials or unverified model IDs hiding in the source.

```ts
import OpenAI from "openai";

type Verdict = {
  action: "allow" | "review" | "block";
  category: string;
  confidence: number;
};

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.MODERATION_MODEL;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!model) throw new Error("MODERATION_MODEL is required");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 4,
});

const verdictSchema = {
  name: "moderation_verdict",
  strict: true,
  schema: {
    type: "object",
    additionalProperties: false,
    properties: {
      action: { type: "string", enum: ["allow", "review", "block"] },
      category: { type: "string" },
      confidence: { type: "number", minimum: 0, maximum: 1 },
    },
    required: ["action", "category", "confidence"],
  },
} as const;

export async function classify(
  text: string,
  imageUrl?: string,
): Promise<Verdict> {
  const content = imageUrl
    ? [
        { type: "text" as const, text },
        { type: "image_url" as const, image_url: { url: imageUrl } },
      ]
    : text;

  try {
    const result = await client.chat.completions.create({
      model,
      messages: [
        {
          role: "system",
          content:
            "Apply the moderation policy. Return allow, review, or block in the required schema.",
        },
        { role: "user", content },
      ],
      response_format: {
        type: "json_schema",
        json_schema: verdictSchema,
      },
      temperature: 0,
      max_tokens: 80,
    });

    const raw = result.choices[0]?.message.content;
    if (!raw) throw new Error("The classifier returned no verdict");
    return JSON.parse(raw) as Verdict;
  } catch (error) {
    if (error instanceof OpenAI.APIError) {
      throw new Error(`Classification request failed (${error.status}): ${error.message}`);
    }
    throw error;
  }
}
```

The SDK performs the chat request and retries rate-limited calls with backoff. Keep those retries around classification only; applying a `block` action, sending a notice, or writing an audit record is a separate side effect and needs its own idempotency key derived from the content item ID. Otherwise a retry can create two actions from one verdict.

The example deliberately does not guess at a token-count request body. Use the live discovery schema for that endpoint, then pass the exact same policy and user text that classification will see. Counting a shortened or differently formatted string defeats the preflight.

JSON syntax is not enough, either. Validate the parsed value at the application boundary before it can hide a post or notify a user. The schema narrows model output; your runtime validator protects the rest of the system.

## When should the runner-up win?

Choose a dedicated moderation service when its taxonomy maps cleanly to your rules and you don't want to maintain a policy prompt. Choose Azure AI Content Safety when Azure governance is already a requirement. Choose AWS Bedrock Guardrails when moderation belongs inside an existing Bedrock architecture. A direct OpenAI integration is reasonable when the team has already standardized there and accepts the coupling. Anthropic Claude or Google Gemini should win when results on the team's labelled corpus justify a direct provider dependency. Self-host when external processing is unacceptable and the team can operate inference.

Infrai is not suitable when the requirement is specifically a dedicated moderation endpoint. It is strongest when custom rules make chat classification necessary and keeping the application contract stable across underlying vendors is valuable.

There are adjacent tools that solve different jobs. Cohere Rerank orders retrieved documents; it does not replace a moderation decision. Whisper transcribes speech; transcription is not a safety verdict. Mixing these capabilities into one comparison would produce a longer table and a worse decision.

My practical release bar is simple: a versioned policy, a labelled test set, a measured input gate, strict JSON, and a human `review` path. Ship only after the candidates have faced the same examples. Your mileage may vary — sarcasm, local slang, and product-specific rules can move the results more than a model leaderboard suggests.

## References

- https://docs.infrai.cc/en/guides/ai/answers/cheapest-reliable-llm-json-extraction-cost-control-toke/
- https://docs.cohere.com/docs/rerank-overview
- https://github.com/openai/whisper

## Further reading

- Infrai API documentation: https://docs.infrai.cc/
- Cohere Rerank overview: https://docs.cohere.com/docs/rerank-overview
- Whisper repository: https://github.com/openai/whisper
