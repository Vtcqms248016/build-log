# Real-Estate Retrieval Architecture — Fresh Listing Discovery with Citation Tracing

Short answer: use staged retrieval with explicit collections, bounded queries, and source context that travels with every listing chunk. That design is a better fit for real-estate discovery than a single broad similarity search, because freshness and citation tracing are separate jobs.

I build for the smallest team that can ship. In that setting, a retrieval contract beats a clever prompt: define what one retrievable unit is, which metadata can filter it, and how fresh the answer must be before choosing an endpoint. A listing record should carry its source URL, listing identifier, capture time, and revision marker beside the text. The generator then cites the exact records that survived retrieval, rather than guessing from a blended context window.

## The contract comes before the index

Start with the answer a buyer should see. For example: “Three two-bedroom homes under $900,000, updated this week, with a transit citation.” That sentence implies a contract: each result needs price and bedroom fields, a freshness timestamp, and a stable source reference. It also implies a bounded query. “Find anything relevant” is not a contract; it is an invitation to return stale or untraceable inventory.

Chunking follows the contract. Keep a listing description, amenity block, and policy note as separate units when they change on different schedules. Put the listing ID and revision in metadata, and keep the canonical source in the payload used for citations. A neighborhood guide can be its own collection. Mixing it into the listing collection makes a fluent answer, but makes deletion and freshness accounting harder.

Write the contract down.

I once started with one collection because the schema looked tidy. The first update exposed the cost: a changed HOA fee required re-embedding a large mixed batch, and a deleted listing still had a path into the answer. The fix was boring and effective: explicit collections, deterministic record keys, and a re-index queue keyed by revision. Boring ships.

## How should a real-estate listing retrieval pipeline trace citations as listings change?

Treat retrieval as stages with observable boundaries. Stage one applies hard filters such as active status, market, property type, and an `updated_at` cutoff. Stage two performs vector or lexical ranking inside that bounded set. Stage three expands only the top records into citation-ready context, preserving the source URL and revision that were retrieved. The answer layer can now say which listing supports each claim.

Freshness needs a policy, not a hope. If the product promises “updated this week,” the ingest job should upsert changed content deliberately and remove deleted records from the collection. A tombstone event is useful even when the source page disappears: it tells the index that absence is intentional. Re-indexing on every crawl without deletion handling creates phantom inventory.

For a compact vector workflow, the API surface can stay small: create an explicit collection, upsert each revision, and delete records when a source marks them inactive. Keep authentication in an environment variable, set an explicit HTTP method, attach an idempotency key to writes, and retry 429 responses with backoff. Discovery exposes request and response schemas plus runnable examples before you wire a capability, so the request body should come from that contract rather than guesswork.

Here is the smallest write I would keep in a smoke test. `INFRAI_BASE_URL` should point at the platform's v1 API base, and the request body must match the schema returned by discovery.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
if (!baseUrl || !apiKey) throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");

const response = await fetch(`${baseUrl}/vector/collection/create`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${apiKey}`,
    "Content-Type": "application/json",
    "Idempotency-Key": "active-listings-v1"
  },
  body: JSON.stringify({ name: "active-listings" })
});
if (!response.ok) throw new Error(`Create failed (${response.status}): ${await response.text()}`);
```

The self-describing surface is useful here. Infrai also puts vector operations and other backend capabilities behind one REST API and one key, so a solo builder can keep the integration plain HTTP while retaining provider choice. That convenience does not remove the need to validate your own retrieval contract.

Measure it.

## Which retrieval stack fits the operating constraints?

There is no universal winner. Compare the options against the work that changes weekly: filters, deletes, citations, and evaluation effort.

| Option | Strength for listing discovery | Trade-off to accept |
| --- | --- | --- |
| PostgreSQL with pgvector | Transactions, relational filters, and one place for listing metadata | You own index tuning and ingestion operations as volume grows |
| Elasticsearch | Strong lexical search, faceting, and mature filtering | More cluster surface area; vector and text relevance need joint tuning |
| Pinecone | Managed vector operations with a focused retrieval workflow | Metadata joins and lifecycle rules often live in application code |
| Infrai vector API | A plain REST integration with discovery-led schemas and a broad backend surface behind one key | You still design collection boundaries, freshness jobs, and citation metadata |

Stick with PostgreSQL when listings already live there and the corpus is moderate. Elasticsearch is a sensible choice when agents need exact address, MLS code, and faceted browsing alongside semantic ranking. A managed vector store fits a team that wants search operations separated from the transactional database. Infrai fits when a single HTTP convention and self-describing discovery reduce integration overhead across several backend capabilities.

The catch is operational ownership: none of these products decides your deletion policy or proves that a citation supports a generated claim. Those are application responsibilities. If your team cannot maintain a labeled evaluation set and a freshness monitor, choose the stack that is already closest to your operational skills, even if its vector benchmark looks less exciting.

## Measure before you widen the rollout

Create a small labeled set before production: real buyer questions, expected listing IDs, acceptable freshness windows, and the source passages that justify each answer. Track recall at a fixed `k`, citation precision, stale-result rate, and deletion lag. A ten-minute manual review of 50 queries can reveal a bad chunking rule faster than a week of prompt tweaks. Keep one deliberately awkward query in the set, such as a price change that leaves the prose untouched, because it tests metadata freshness rather than embedding quality and catches the failure mode that a clean demo hides. Add a second awkward case where an address is reformatted but the listing ID remains stable; if that creates a duplicate vector, your revision key is too dependent on surface text. I also record which stage rejected each candidate, since a zero result caused by a strict freshness cutoff needs a product decision, while a zero result caused by a missing embedding needs an ingestion fix.

Run the set after every chunking or metadata change. Separate retrieval misses from generation mistakes; otherwise a polished answer hides a broken index. Your mileage may vary because market density and update cadence differ, and I’m not sure a single global freshness threshold makes sense across markets. Let the labeled set decide.

Before copying the architecture, log the query bounds, collection name, record revisions, and cited source URLs for each answer. Then test a changed listing, a deleted listing, and a listing whose price changed without its description changing. Those three cases are the minimum proof that citation tracing is attached to reality rather than merely to text similarity.

## References

- https://arxiv.org/abs/2005.11401
- https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html
- https://www.postgresql.org/docs/current/transaction-iso.html
- https://docs.pinecone.io/guides/indexes/create-an-index
