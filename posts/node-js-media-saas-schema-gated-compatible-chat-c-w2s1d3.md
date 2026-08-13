# Node.js Media SaaS: Schema-Gated Compatible Chat Completions with One API Key

Short answer: for a media SaaS answering questions over a private knowledge base, treat a one-key, chat-completions-compatible layer as a replaceable transport; choose it only if every backend produces evidence-bound JSON that passes the same Node.js admission gate.

The concrete constraint is not getting text back from OpenAI, Claude, or Gemini through one credential. It is stopping a plausible answer from entering the product when its shape is wrong or its citations were never retrieved. A familiar chat envelope can simplify the first call. It can't prove that the payload is safe to use.

One key removes secret sprawl. It doesn't remove semantics.

## The constraint that changed the build

Picture a private media archive containing interview transcripts, corrections, rights notes, and published stories. Retrieval returns a small set of passages with opaque IDs. The answer layer must produce three things: an answer, the exact IDs supporting it, and a Boolean saying the supplied evidence is insufficient. If retrieval returns nothing, the answer must decline. If the model cites an ID outside that request's set, the application must reject it.

This is an authorization boundary disguised as output formatting. `story-17` is allowed because the retriever supplied it. `Story 17` is not allowed, even if a person can guess what it means. Lowercasing, punctuation stripping, and fuzzy matching would make the demo feel forgiving while silently granting the model authority to mint citation identifiers. Don't do that.

Fail closed.

The resulting architecture is deliberately boring: retrieval selects passages, a transport sends the request, and an admission function converts `unknown` into a domain object or a named failure. Provider response types end at the transport edge. The rest of the SaaS app never imports them. That boundary matters more than a shared request shape because it survives a direct integration, a common gateway, or a hybrid of both.

It also gives the comparison a hard order. First ask whether the response is admissible. Then count repair attempts and latency. Only after that should a team weigh credential custody, provider-specific controls, and maintenance load. A five-line first call is irrelevant if twenty scattered checks are needed before the answer can be stored, rendered, or exported.

## Build the smallest Node.js admission gate

The transport below returns `unknown` on purpose. It can wrap a direct client or a chat-completions-compatible endpoint without leaking either choice into the knowledge-answer contract. The parser checks exact field types and exact citation membership. No SDK types. No config maze.

```ts
type Passage = {
  id: string;
  text: string;
};

type KnowledgeAnswer = {
  answer: string;
  citations: string[];
  insufficientEvidence: boolean;
};

type InvokeChat = (prompt: string) => Promise<unknown>;

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null && !Array.isArray(value);
}

export function parseKnowledgeAnswer(
  value: unknown,
  allowedIds: ReadonlySet<string>,
): KnowledgeAnswer {
  const parsed = typeof value === "string" ? JSON.parse(value) : value;

  if (!isRecord(parsed)) throw new Error("OUTPUT_NOT_OBJECT");
  if (typeof parsed.answer !== "string") throw new Error("ANSWER_NOT_STRING");
  if (!Array.isArray(parsed.citations)) throw new Error("CITATIONS_NOT_ARRAY");
  if (typeof parsed.insufficientEvidence !== "boolean") {
    throw new Error("EVIDENCE_FLAG_NOT_BOOLEAN");
  }

  const citations = parsed.citations;
  if (!citations.every((id): id is string => typeof id === "string")) {
    throw new Error("CITATION_NOT_STRING");
  }
  if (citations.some((id) => !allowedIds.has(id))) {
    throw new Error("CITATION_OUTSIDE_CONTEXT");
  }
  if (allowedIds.size === 0 && !parsed.insufficientEvidence) {
    throw new Error("EMPTY_CONTEXT_NOT_DECLINED");
  }

  return {
    answer: parsed.answer,
    citations,
    insufficientEvidence: parsed.insufficientEvidence,
  };
}

export async function answerFromPrivateContext(
  question: string,
  passages: Passage[],
  invokeChat: InvokeChat,
): Promise<KnowledgeAnswer> {
  const allowedIds = new Set(passages.map(({ id }) => id));
  const context = passages.map(({ id, text }) => ({ id, text }));
  const prompt = JSON.stringify({
    task: "Answer only from context. Return one JSON object.",
    output: {
      answer: "string",
      citations: ["exact context id"],
      insufficientEvidence: "boolean",
    },
    question,
    context,
  });

  const raw = await invokeChat(prompt);
  return parseKnowledgeAnswer(raw, allowedIds);
}
```

This is smaller than a general JSON Schema implementation, and that is useful for a build log: the mechanism is visible. In a larger system, a standards-based validator should define extra-property policy, length limits, nested variants, and schema versioning. The invariant stays the same — untrusted model output crosses the boundary as `unknown`; validated application data leaves it.

There is another benefit. Named failures become clean observability dimensions. `CITATION_OUTSIDE_CONTEXT` points to a different contract failure than `OUTPUT_NOT_OBJECT`, while a single “bad response” counter hides the distinction. Record the case ID, model configuration, prompt version, validation code, attempt count, and latency. Avoid logging private passage text merely to make a dashboard convenient; retention and redaction must follow the application's data policy.

## Break the contract before comparing backends

A happy-path object proves almost nothing. Feed the parser hostile shapes first: an array, a Boolean encoded as `"false"`, a citation list containing a number, an unknown ID, and an empty evidence set paired with a confident answer. These cases are deterministic, cheap, and independent of any model vendor.

```ts
type Fixture = {
  name: string;
  value: unknown;
  allowedIds: string[];
  expectedCode?: string;
};

const fixtures: Fixture[] = [
  {
    name: "accepts exact retrieved citation",
    value: {
      answer: "The correction changed the publication date.",
      citations: ["correction-17"],
      insufficientEvidence: false,
    },
    allowedIds: ["story-17", "correction-17"],
  },
  {
    name: "rejects human-readable citation alias",
    value: {
      answer: "The correction changed the publication date.",
      citations: ["Correction 17"],
      insufficientEvidence: false,
    },
    allowedIds: ["story-17", "correction-17"],
    expectedCode: "CITATION_OUTSIDE_CONTEXT",
  },
  {
    name: "requires decline without retrieved evidence",
    value: {
      answer: "A date was changed.",
      citations: [],
      insufficientEvidence: false,
    },
    allowedIds: [],
    expectedCode: "EMPTY_CONTEXT_NOT_DECLINED",
  },
];

for (const fixture of fixtures) {
  try {
    parseKnowledgeAnswer(fixture.value, new Set(fixture.allowedIds));
    if (fixture.expectedCode) throw new Error(`MISSED_${fixture.expectedCode}`);
  } catch (error) {
    const code = error instanceof Error ? error.message : "UNKNOWN_ERROR";
    if (code !== fixture.expectedCode) throw error;
  }
}
```

Now use the same fixture corpus around each injected transport. Add ordinary archive questions, conflicting passages, malicious instructions embedded inside a passage, questions answerable only from outside knowledge, and passages whose correction supersedes an earlier story. Keep retrieval inputs, prompt version, schema, and acceptance function fixed while changing the backend. Otherwise the comparison measures several moving parts at once.

Repair attempts need separate accounting. A failed object may be retried with the validator code and the same allowed IDs, but cap that loop. A configuration accepted on its third attempt is operationally different from one accepted immediately, even if their final objects match. Preserve the original failure in the result rather than letting a later success erase it.

Measure that.

No invented winner belongs here. The useful result is an acceptance rate by fixture class, plus retry count and latency for the team's actual archive. I'm not sure a public benchmark can predict behavior on a private editorial corpus; retrieval quality, document length, correction patterns, prompt wording, and refusal policy all affect the outcome. A replayable internal corpus is what resolves that uncertainty.

## How should a SaaS app compare one-key compatible chat completions?

Compare ownership boundaries after the schema gate is in place. OpenAI, Claude, and Gemini may be reached through separate adapters or through a shared chat-shaped layer, but neither topology guarantees structured-output correctness. The application validator remains authoritative.

| Integration shape | Credential and transport boundary | Suitable when | Limitation |
| --- | --- | --- | --- |
| Direct adapters | Separate credentials and provider transports | Direct custody and provider-specific controls are required | More adapter code and secret configuration |
| Common gateway | One credential and one shared transport | Central policy and interchangeable model configurations matter | A common shape may omit provider-specific controls |
| Hybrid | Shared default plus selected direct adapters | Most calls fit a common contract but explicit exceptions remain | Two operational paths must be tested and observed |

A one-key layer is suitable when centralized credential handling is valuable, the shared interface exposes enough information to attribute results to each configuration, and the required controls fit its common request shape. **It is not suitable** when policy demands direct vendor custody, the application relies on controls outside that shape, or independent failure domains are mandatory. Stick with direct adapters in those cases. The extra configuration buys explicit control.

The catch is that compatibility can become a lowest-common-denominator contract. Passthrough flags accumulate, model names leak into business logic, and the tidy first call grows a second configuration system. Keep a transport interface narrow, but don't hide a backend capability the application genuinely needs. A hybrid is honest when exceptions are few and deliberate; it is config bloat when every call is an exception.

Exceptions compound.

Structured output is still the deciding axis for this media job. Shortlist only integration shapes that preserve the `unknown` transport boundary and the exact citation gate. Then run identical fixtures per backend configuration. Correctness comes first, retries and latency second, and operational ownership third. A single API key is an operational convenience.

It is not evidence quality.

## What I would change at scale

At higher volume, split offline admission testing from live monitoring. The offline suite should block changes to prompts, schemas, retrieval configuration, adapters, or model selection when known fixtures regress. Live metrics should reveal distribution shifts without turning user traffic into an uncontrolled experiment. Version those five inputs together so a changed acceptance rate has a traceable cause.

Non-interactive work also deserves a separate execution path. Indexing, archive enrichment, and nightly replay do not need the latency profile of interactive questions. The OpenAI Batch API guide documents one primary example of a batch path. Its results should still pass through the same domain validator. Likewise, speech recognition can be an independently operated ingestion component; the open-source Whisper repository is a primary reference, but transcript production and evidence-bound answering are separate contracts.

Add property-based tests around the parser for wrong scalar types, duplicated IDs, unknown IDs, extra keys, oversized strings, and deeply nested input. Give transports contract tests with synthetic fixtures, not private articles. Keep a thin end-to-end test proving that retrieval IDs survive unchanged from selection through rendering. At scale, routing may use task class and observed validation history, but it must not silently change the answer schema. Keep the domain contract stable. Route below it.

This design has real costs. Strict admission produces more visible declines, schema migrations require coordination, and exact citation matching rejects aliases a human might accept. For exploratory chat where no answer is stored or acted upon, that machinery may be excessive. For private media knowledge answers that appear authoritative, the friction is the point: it turns a model response into a typed, evidence-authorized application result.

## References

- https://platform.openai.com/docs/guides/batch
- https://github.com/openai/whisper
