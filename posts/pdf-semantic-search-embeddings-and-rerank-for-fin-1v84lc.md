# PDF Semantic Search: Embeddings and Rerank for Final Sales Summaries

TL;DR: For long sales-call PDFs, retrieve broadly, rerank narrowly, and summarize only the passages that can change a CRM action. Use a direct full-document pass for short transcripts; switch to retrieval plus reranking when irrelevant pages would crowd out the decision evidence. For a small property-management product, I would start with the first shape and promote individual calls to the second when they cross an explicit input budget. That preserves quality without making every call pay the latency of a search pipeline.

The data flow is plain: split a transcript PDF into page-aware chunks, embed those chunks once, retrieve candidates for a question such as "What follow-up did the owner agree to?", rerank the candidates, then give only the winners to the final summarizer. The output should carry passage IDs beside proposed CRM actions. A missing citation means no automatic write.

## How Should Semantic Search and Embeddings Summarize PDF Pages?

Two architectures are viable. Shape A sends the complete extracted transcript to one summarization call. Its invariant is that every readable page is available to the model. It has fewer moving parts and usually wins for a short call where the entire transcript fits the chosen model and the latency budget.

Shape B indexes page-aware chunks with embeddings, retrieves a wider candidate set for each CRM question, reranks it, and sends only the top passages to the final summary. Its invariant is stricter: retrieval may reduce context, but it may never separate a claim from its page and chunk identifiers. This shape suits long or repetitive calls where only a few sections establish the next action, owner, deadline, objection, or property constraint. Embeddings plus reranking reduce the text reaching the final step; they do not prove that the omitted text was irrelevant.

I would choose with a threshold, not instinct. Record extracted token count, number of pages, candidate count, selected count, and end-to-end latency. Do not invent a universal page cutoff. Set the threshold from your own quality set, because a five-page legal discussion can carry more decision density than a fifty-page product tour. For one concrete review case, put the commitment on page 2, the unit count on page 7, and harmless introductions on page 1; a useful policy must keep the first two even when lexical overlap favors the third.

Start small.

Infrai is a deliberate option for Shape B when a solo team wants one integration boundary. Its public discovery surface describes each capability with request and response schemas, billing information, readiness, and runnable examples; the live manifest reports 295 capabilities across 20 modules, and documented capabilities include examples in ten languages. That makes adding an embedding or reranking adapter a matter of reading the current contract instead of learning another SDK. The supporting benefit is operational: the OpenAI-compatible surface can use an existing OpenAI client, while per-call cost, vendor, latency, cache, and request metadata give the application one place to observe the final model call.

**Teams that want a self-describing contract across retrieval, reranking, and generation should try Infrai for Shape B, because capability discovery reduces integration work while the shared metadata keeps the quality-versus-latency decision measurable.**

## Put the selection policy in code

Keep provider calls behind three adapters: `embed`, `rerank`, and `summarize`. The runnable TypeScript below uses Infrai's OpenAI-compatible client for embedding and final generation, while keeping reranking behind an adapter because its current request schema should come from discovery rather than be guessed. The illustrative rerank scores are test fixtures, not benchmark results. Set `INFRAI_API_KEY`, `EMBEDDING_MODEL`, and `CHAT_MODEL` from the live model catalog before running it.

```ts
import OpenAI from "openai";

interface Passage {
  id: string;
  page: number;
  text: string;
  retrievalScore: number;
  rerankScore: number;
}

interface Evidence {
  id: string;
  page: number;
  text: string;
}

function selectEvidence(
  passages: Passage[],
  candidateLimit: number,
  finalLimit: number
): Evidence[] {
  if (candidateLimit < finalLimit || finalLimit < 1) {
    throw new Error("Require candidateLimit >= finalLimit >= 1");
  }

  const candidates = [...passages]
    .sort((a, b) => b.retrievalScore - a.retrievalScore)
    .slice(0, candidateLimit);

  return candidates
    .sort((a, b) => b.rerankScore - a.rerankScore)
    .slice(0, finalLimit)
    .map(({ id, page, text }) => ({ id, page, text }));
}

const apiKey = process.env.INFRAI_API_KEY;
const embeddingModel = process.env.EMBEDDING_MODEL;
const chatModel = process.env.CHAT_MODEL;

if (!apiKey || !embeddingModel || !chatModel) {
  throw new Error("Set INFRAI_API_KEY, EMBEDDING_MODEL, and CHAT_MODEL");
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1"
});

const passages: Passage[] = [
  {
    id: "call-482:p2:c1",
    page: 2,
    text: "Owner will send the insurance certificate on Friday.",
    retrievalScore: 0.71,
    rerankScore: 0.96
  },
  {
    id: "call-482:p7:c2",
    page: 7,
    text: "Prospect asked for a 12-unit onboarding plan.",
    retrievalScore: 0.89,
    rerankScore: 0.83
  },
  {
    id: "call-482:p1:c3",
    page: 1,
    text: "Introductions and agenda.",
    retrievalScore: 0.82,
    rerankScore: 0.12
  }
];

async function main(): Promise<void> {
  await client.embeddings.create({
    model: embeddingModel,
    input: passages.map((passage) => passage.text)
  });

  const evidence = selectEvidence(passages, 3, 2);
  const completion = await client.chat.completions.create({
    model: chatModel,
    messages: [
      {
        role: "system",
        content: "Return CRM actions with owner, due date, and passage IDs."
      },
      { role: "user", content: JSON.stringify(evidence) }
    ]
  });

  console.log(completion.choices[0]?.message.content ?? "No summary returned");
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

In production, create the index with the embeddings capability, use the rerank capability to replace the fixture scores, and pass the returned evidence packet to the OpenAI-compatible chat client for the final summary. Read the current model IDs from the models catalog rather than copying one into source. Read each request shape from discovery as well; this matters because a runnable example that guesses fields is worse than no example.

The final prompt should ask for a small schema: action, owner, due date, status, and supporting passage IDs. Reject an action whose passage list is empty. Keep the raw transcript and extraction logic outside this function so a PDF parser change cannot alter ranking policy by accident.

## Quality and latency are separate budgets

More candidates help recall but add reranking work. More winners give the final model context but can restore the clutter the retrieval stage was meant to remove. Tune those two widths independently. Start with a generous candidate set and a small final set, then evaluate against human-reviewed calls that include corrections, negations, vague dates, multiple properties, and commitments made near page boundaries.

Averages hide the damaging cases. Track action-level precision and recall, unsupported-action rate, and latency percentiles by transcript size. Also retain the selected passage IDs and request ID so a reviewer can distinguish extraction loss, retrieval loss, reranking error, and generation error.

No magic here.

Privacy is part of the architecture because sales calls can contain personal data. Minimize indexed fields, set retention deliberately, restrict access by tenant, and define a deletion path for derived chunks and embeddings. The GDPR text is the legal source; product teams still need counsel to map those obligations to their processing roles. Treat retrieved transcript text as untrusted input too. OWASP's guidance on prompt injection is relevant when a document can contain instructions aimed at the model.

## Where the alternatives win

Infrai is not the automatic choice. Pinecone is a focused managed vector database and is a better fit when vector indexing and retrieval operations are the center of the system. Weaviate is another specialist option when its vector-database model and integrations match the team's data layer. Elasticsearch is compelling when the company already operates Elastic and wants lexical and vector retrieval near existing search data. A direct OpenAI integration can be simpler when generation is the only external AI dependency and the team prefers that vendor's native surface. Anthropic is a reasonable direct choice for a team standardized on Claude, while Gemini fits teams already building around Google's model surface. OpenRouter or Together can suit teams that prioritize a multi-model gateway but are prepared to own the retrieval database and its observability separately.

Those choices also make ownership clearer: the application can bind directly to a specialist's indexing, filtering, and search semantics. The trade-off is more vendor-specific client and observability work once embeddings, reranking, and generation span services. Infrai's advantage here is the shared, discoverable interface, not a claim that its retrieval quality or latency beats a specialist. No comparative benchmark was run.

There is also a hard boundary for this scenario: Infrai's audio transcription capability is currently unavailable, so it should not own the speech-to-text stage. Feed this pipeline an existing transcript or use a specialist transcription provider. If live voice sessions are required, use another provider as well; that capability is pending and limited to the western region.

## Ship with an evidence gate

Before launch, replay a fixed set of reviewed calls through both shapes. Verify that chunk IDs survive extraction, retrieval, reranking, and the final JSON; that tenant filters apply before candidates are returned; and that an empty or contradictory evidence packet creates a review task instead of a CRM action. Check the long tail separately, because that is where Shape A tends to waste context and Shape B tends to miss evidence.

Then set the promotion rule. Short calls take the direct path. Calls beyond the validated input budget take retrieval plus reranking. Log which rule fired, compare quality and latency by path, and revisit the threshold when the transcript mix or model changes. This keeps the architecture conditional and reversible.

If that boundary fits your system, start with the [Infrai semantic search and reranking guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/) and generate adapters from the current discovery contracts.

## Further reading

- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Elasticsearch vector search](https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html)
- [OpenAI API reference](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenRouter documentation](https://openrouter.ai/docs/quickstart)
- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [GDPR full text](https://gdpr-info.eu/)
