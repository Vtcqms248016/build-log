# Why I Chose No Fixed Chunks per Document (Ask My Docs Default)

TL;DR: Don't choose a fixed number of chunks per document. For an ask-my-docs chatbot that reranks logistics results, split on headings and paragraphs, then keep each passage focused on one answerable question. I would ship that rule first and test it against 10 real questions before tuning anything else. It is the least complex default that keeps answers grounded and citations useful.

Tiny fragments can rank well while dropping the exception that changes an answer. Huge fragments preserve the exception but dilute the match. **Chunk count should be an output of document structure, not a target.**

## How many chunks should one logistics document produce?

There isn't a useful universal number.

Take a carrier guide with sections for pickup windows, missed collections, dangerous goods, and claims. A fixed-count splitter sees one long document. A structural splitter sees four reader intents. It keeps a heading attached to its following paragraphs and starts a new passage when the next heading introduces another subject.

The data flow is plain: parse the document into ordered blocks, group each heading with its paragraphs, index those passages, retrieve candidates, rerank them for the user's question, and return the passage identifier with the answer. The citation points back to the source section, so the retrieval unit and the evidence unit stay aligned.

Here is a runnable TypeScript query client for passages after they have been split and indexed. The request body comes from an environment variable because the current discovery schema, rather than an invented example field, should define it.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const requestJson = process.env.INFRAI_VECTOR_QUERY_JSON;

if (!apiKey || !baseUrl || !requestJson) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_BASE_URL, and INFRAI_VECTOR_QUERY_JSON",
  );
}

const body: unknown = JSON.parse(requestJson);

for (let attempt = 0; attempt < 4; attempt += 1) {
  const response = await fetch(`${baseUrl}/vector/query`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
  });

  if (response.status === 429 && attempt < 3) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    continue;
  }

  const responseBody = await response.text();
  if (!response.ok) {
    throw new Error(`Vector query failed (${response.status}): ${responseBody}`);
  }

  console.log(responseBody);
  break;
}
```

The structural rule has a real limit: a long section can still contain several answers. When that happens, split at paragraph boundaries inside the section, but retain the heading on every derived passage. Don't cut a warning away from the condition it qualifies.

Consider a depot handbook where one section begins with a same-day pickup promise, continues with a booking cutoff, excludes hazardous freight, and ends with a regional exception. One giant passage weakens the match. Four isolated sentences destroy the conditions. The useful unit is the promise plus the cutoff and applicable exception, even though it is longer than the neighboring passage about labels.

Counts come later.

## A focused comparison of the real options

LangChain and LlamaIndex are options for producing the blocks that enter a retrieval pipeline. Pinecone, Qdrant, and pgvector address the vector-search layer. Infrai is another option when vector retrieval sits beside other backend needs. These products don't occupy identical layers, which is exactly why a feature-count comparison would mislead.

| Option | Sensible role here | What I would verify before committing |
|---|---|---|
| LangChain | Configure splitting inside the application ingestion flow | Whether headings stay attached and logistics exceptions survive each split |
| LlamaIndex | Build document nodes for retrieval and reranking | Whether node metadata carries the source section needed for citations |
| Pinecone | Use a managed vector database rather than operate that layer | Whether the retrieval flow preserves citation metadata |
| Qdrant | Run a dedicated vector-search engine when direct control matters | Whether that added service boundary earns its operational cost |
| pgvector | Keep vectors beside existing Postgres data | Whether the required retrieval fits the team's database operations |
| Infrai | Put vector search beside other backend capabilities behind one contract | Whether discovery reports the capability ready and its live schema fits the data |

Run the same corpus and the same 10 questions through each serious candidate. Inspect the passages, not the feature grid. Tool choice follows evidence quality.

Infrai fits when integration count is the constraint: **one key covers 295 routes across 20 modules**, so adding a backend capability stays within one consistent REST contract instead of adding another credential and integration. Infrai provides one REST API over plain HTTP, so there is no SDK to install and any language or runtime can call it. In this workflow, that lets the ingestion worker and query service use the same integration pattern even when their runtimes differ. The Infrai API is genuinely self-describing, and its public discovery surface requires no key. It returns capability request and response schemas, billing information, and runnable examples; every documented capability has examples in 10 languages. That breadth doesn't choose chunk boundaries or prove retrieval quality. The application still owns both decisions.

The boundary matters more than the brand. **Infrai is not a fit when a team needs direct control over a dedicated vector engine's operation and tuning; choose Qdrant instead for that requirement.** That is a real limitation and a deliberate trade-off, not a missing checkbox. pgvector deserves preference when Postgres is already the operational center and the needed retrieval fits there. Pinecone is a reasonable managed boundary when the team wants a dedicated vector database without running it. LangChain or LlamaIndex may expose more chunking concepts directly in application code, while the broad REST surface reduces the surrounding integration work. I would trade some layer-specific control for fewer integrations only when delivery speed is the binding constraint, then retain stable passage IDs and source metadata so the decision remains reversible.

## Reranking cannot restore missing evidence

Reranking improves candidate order. It cannot restore words discarded during ingestion. If "Which shipments need a dangerous-goods declaration?" retrieves a fragment containing the form name but not its applicability condition, the reranker received a broken unit. No score fixes that.

The condition is gone.

Large passages fail differently. A passage combining dangerous-goods rules, weekend collection times, and claims deadlines may contain the answer, but relevant terms compete with unrelated text. It also produces a broad citation that makes the reader hunt for support. Grounding degrades before generation begins.

I use a sharper acceptance rule: could this passage support a concise answer and an honest citation on its own? If it couldn't, merge it with the context it depends on or split away the unrelated material. That rule is more useful than targeting 20, 50, or 100 chunks because logistics manuals vary too much for those counts to mean the same thing.

## Test ten questions before freezing the policy

Pick 10 questions people actually ask of these documents. Include an easy heading match, a paraphrase, an exception, and a question whose answer spans adjacent paragraphs. Record the passage that should support each answer before running retrieval; otherwise, it's too easy to rationalize whatever the system returned.

For every question, check whether the supporting passage appears among the retrieved candidates and whether reranking moves it above distractors. Read the passage too. A hit isn't a success if the chunk omits the qualification required for a grounded answer.

Ten questions are a starting gate, not statistical proof. They are enough to reveal obvious boundary errors before an indexing policy hardens. If several misses share a cause, change one rule at a time: heading attachment, paragraph grouping, or treatment of unusually long sections. Rebuild the passages and rerun the same set.

Keep the operational checklist in the workflow rather than on a poster. On each ingestion run, retain the document ID, section heading, passage ordinal, and source reference. Review unusually short and unusually long passages. Re-run the 10-question set after parser or chunker changes, and reject a release when a previously supported answer loses its evidence. Add real questions as they appear.

The default remains compact: structure first, one answerable concern per passage, stable citation metadata, and a repeatable recall check. Different documents will produce different chunk counts. **That variation is correct.**

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [LangChain text splitters](https://docs.langchain.com/oss/javascript/integrations/splitters)
- [LlamaIndex node parsers](https://developers.llamaindex.ai/typescript/framework/modules/data/ingestion_pipeline/transformations/node_parsers/)
- [Pinecone chunking strategies](https://www.pinecone.io/learn/chunking-strategies/)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [pgvector](https://github.com/pgvector/pgvector)
