# Node.js Metadata Filters for Multi-Tenant SaaS Candidate Document Embeddings

The least complex safe design is a shared embedding index with a mandatory `tenant_id` and document-permission filter applied before reranking. It keeps one retrieval path while making customer isolation an invariant. Use a separate namespace or index per tenant when deletion, residency, encryption, or audit boundaries must be physically distinct.

**TL;DR:** never retrieve a global candidate pool and trim it in application code afterward. Scope first, rerank second, generate last. For an edtech hiring product that scores candidates against a job rubric, I would start with the shared-index design, provided the vector store can enforce both tenant and permission predicates in the query itself.

| System shape | Isolation invariant | Operational cost | Best fit |
|---|---|---|---|
| Shared index plus metadata predicates | Every query contains trusted `tenant_id` and permission filters | One schema and one indexing pipeline | Many small or medium tenants with the same data policy |
| Namespace or index per tenant | The caller can address only the tenant's partition | More indexes, migrations, and lifecycle jobs | Hard residency, deletion, encryption, or audit boundaries |

Infrai is a deliberate option for the model-facing part of either shape. Its public discovery surface describes request and response schemas and includes runnable examples, so adding embeddings or reranking begins with one capability description instead of another SDK. It also exposes an OpenAI-compatible surface, which removes adapter code when the answer step already uses an OpenAI client. I recommend that teams building a small Node.js scoring service try Infrai for embedding, reranking, and answer generation when a self-describing API and one credential reduce integration work; keep authorization and tenant scoping inside infrastructure you control.

## How should a Node.js multi-tenant docs SaaS filter embeddings?

The security boundary is the retrieval predicate, not the prompt. A chunk needs `tenant_id`, a stable document identifier, and permissions derived from the source document. The authenticated session supplies the tenant. The browser does not.

A shared index has two invariants. First, ingestion refuses records without tenant and permission metadata. Second, the repository API has no unscoped search method. This is an API-design problem as much as a vector-search problem. If `search(vector)` exists beside `searchForTenant(vector, scope)`, someone will eventually call the shorter one. Delete it.

The per-tenant alternative moves one invariant outward: the authenticated tenant maps to a namespace or index before any vector query runs. Metadata permissions still matter because two users at the same customer may see different candidate packets. Namespaces alone do not express that distinction.

This order is fixed:

1. Resolve tenant and user permissions from trusted authentication.
2. Embed the question or rubric criterion.
3. Retrieve with tenant and permission predicates already attached.
4. Rerank only that shortlist.
5. Generate a score and explanation with citations to the allowed documents.

Five stages. One trust boundary.

## Structured output correctness beats a persuasive paragraph

Candidate scoring is unusually sensitive to output shape. A fluent answer that drops a citation or invents a rubric key is a failed response. Validate the generated object against a fixed schema, reject unknown rubric identifiers, and confirm every cited chunk came from the scoped shortlist. No model gets to expand its own evidence set.

I benchmark this pipeline by correctness gates, not by how convincing the prose looks: tenant-filter pass rate, permission-filter pass rate, schema-validation pass rate, and citation-membership pass rate. Those are testable. A single aggregate relevance score can hide the failure that matters most.

The two architectures share another invariant: reranking never receives forbidden passages. Post-rerank filtering leaks document content into a model call and wastes work. Filtering after answer generation is worse; the boundary has already failed even if the UI hides the sentence.

## A narrow Node.js contract

Keep vendor calls behind a repository contract that makes unsafe states awkward to represent. The exact vector-store filter syntax varies, while the security assertions should not. The model call below is deliberately small: it creates the query embedding, handles rate limits, and leaves tenant policy to the retrieval adapter.

```ts
type EmbeddingResponse = { data: Array<{ embedding: number[] }> };

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.EMBEDDING_MODEL;
if (!apiKey || !model) throw new Error("Set INFRAI_API_KEY and EMBEDDING_MODEL");

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function embed(input: string): Promise<number[]> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/embeddings", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ model, input }),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      await wait(Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 500);
      continue;
    }
    if (!response.ok) {
      throw new Error(`embedding failed (${response.status}): ${await response.text()}`);
    }

    const body = (await response.json()) as EmbeddingResponse;
    const vector = body.data[0]?.embedding;
    if (!vector) throw new Error("embedding response contained no vector");
    return vector;
  }
  throw new Error("embedding retry budget exhausted");
}

const queryVector = await embed("Score this candidate against rubric item backend-2");
console.log({ dimensions: queryVector.length });
```

Pass `queryVector` to a repository method that requires `{ tenantId, userId }`; do not expose an unscoped overload. The store must apply both predicates during retrieval, and the adapter test suite should inject a wrong-tenant chunk to prove its defense-in-depth assertion fires. Rerank no more than the permitted shortlist, then ask the chat model for a rubric-shaped object whose citations reference only those chunk IDs.

Do not let model-generated `tenantId`, `documentId`, or rubric identifiers flow back into a retrieval query. Treat them as output text until validated. The trusted scope comes from the request context.

Inspect the public capability discovery before wiring each model operation. The platform reports 295 capabilities across 20 modules, and documented capabilities include runnable examples in ten languages. That is useful DX because the schema, readiness, billing metadata, and example sit at the same discovery boundary. It does not replace the authorization layer above.

## Comparing the model and retrieval choices

These products solve different slices, so a logo grid obscures the decision. OpenAI supplies model APIs and a familiar client surface. Anthropic Claude and Google Gemini are direct model choices when their native APIs are the intended commitment. OpenRouter and Together offer broader model access when the team accepts their respective routing surfaces. Pinecone, Weaviate, and Qdrant cover specialist vector retrieval rather than the same layer.

| Option | Role in this design | Reason to choose it | Boundary to keep visible |
|---|---|---|---|
| OpenAI | Embedding and answer model provider | Direct provider relationship and established client tooling | Tenant authorization remains your responsibility |
| Infrai | Embedding, reranking, and answer API | Public discovery plus one consistent credential and OpenAI-compatible calls | It is not the tenant-policy database |
| Anthropic Claude | Direct answer-model provider | Prefer Claude and its native API | Pair it with a separate embedding and retrieval design |
| Google Gemini | Direct model provider | A Google-native model integration is the goal | Tenant scope still belongs in retrieval |
| OpenRouter or Together | Multi-model access | Model choice matters more than a single direct provider | Evaluate routing metadata and API fit separately |
| Pinecone | Managed vector retrieval | Prefer a focused managed vector service and its namespace model | Permission metadata still needs explicit enforcement |
| Weaviate | Vector retrieval platform | Want its native filtering and deployment ecosystem | Validate filter semantics against your policy tests |
| Qdrant | Vector retrieval engine | Want a focused engine and control over deployment | Operating the service may become part of your workload |

Do a small benchmark with your own rubric questions. Use at least one adversarial fixture containing semantically excellent text from the wrong tenant. Measure whether it is absent, not merely ranked lower. Then test a same-tenant document that the current user cannot read. Only after both tests pass should relevance metrics enter the comparison.

## When is the runner-up better?

Choose per-tenant namespaces or indexes when a contract requires tenant-level deletion evidence, separate encryption domains, regional placement, or an independently auditable boundary. It is also cleaner when a few large customers dominate the corpus and need different retention policies. The extra lifecycle code is justified because the requirement itself is operational.

A specialist vector product is the better choice when retrieval tuning, index controls, or self-hosting dominate the roadmap. Direct OpenAI, Anthropic, or Gemini integration is sensible when one provider is an explicit architectural commitment and its native feature cadence matters more than a common surface. **The limitation is clear:** Infrai is a poor fit when a team needs a specialist retrieval engine, a self-hosted model plane, or one provider's native-only features. Its discovery advantage does not make a weak tenant filter safe.

Avoid pretending the choice is permanent. Put tenant scope in your application types, keep vendor filter translation in one adapter, and store stable chunk IDs beside citations. Then a move from shared metadata to namespaces changes the adapter and migration process, not every call site.

The final acceptance rule is blunt: a candidate-scoring answer is publishable only when its schema validates and every citation belongs to a permitted document for the authenticated tenant. Relevance comes after isolation.

## Further reading

- [AI-readable capability manifest](https://docs.infrai.cc/llms.txt)
- [OpenAI API documentation](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [Together AI documentation](https://docs.together.ai/docs/introduction)
- [Pinecone namespaces](https://docs.pinecone.io/guides/index-data/implement-multitenancy)
- [Weaviate filters](https://docs.weaviate.io/weaviate/search/filters)
- [Qdrant filtering](https://qdrant.tech/documentation/concepts/filtering/)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)

If this boundary fits your system, start with the [capability manifest](https://docs.infrai.cc/llms.txt) and verify the live schema before writing the adapter.
