# Logistics Document Summarization: Batch Boundaries That Preserve Tenant Attribution

For a logistics knowledge base, group documents by tenant and revision before sending them for batch summarization. TL;DR: use an asynchronous summarization API for background ingestion, reconcile each result against an application-owned manifest, and publish only summaries whose source and tenant are known. Keep interactive answers on the published evidence.

A shipment exception note and a warehouse handling guide can both become retrieval material. They should not become interchangeable evidence. The useful boundary is the tenant, followed by the source revision; a provider job identifier alone cannot tell the application which private knowledge base should receive a summary or which customer should carry its attributed expense.

## How should a summarization API batch multiple documents by tenant?

The data flow starts with private source documents, groups their text into a tenant-specific submission, and records a stable reference before remote work begins. A background worker submits the group, checks completion, joins returned summaries to the manifest, and prepares private storage records for retrieval. The question-answering path reads only published records. It does not wait for ingestion to finish.

Keep attribution explicit. Don't guess.

The asynchronous batch workflow supports submission, completion checks, and result retrieval; its stated completion range is typically 5–30 minutes. Treat that range as a scheduling expectation, not a deadline or measured latency guarantee. Background preparation is a plausible fit. A dispatcher asking about a delayed shipment needs whatever evidence has already been published, so a newer revision can remain pending without silently replacing a usable source.

Start with one tenant per batch. That choice creates more jobs than mixing every customer's documents together, but it makes cancellation decisions, reconciliation, and access checks easier to reason about. Change the packing strategy once the ledger demonstrates why the extra complexity is necessary.

## A runnable Node.js example for the application boundary

First retrieve an existing batch's results with this read-only TypeScript example. Set `INFRAI_BASE_URL` to the service's documented v1 base URL, `INFRAI_API_KEY` to your secret, and `BATCH_ID` to an existing job identifier. Save it as `read-results.ts` and run it with `node read-results.ts` on Node.js 24. Run it only after the job reports completion. It prints the returned JSON without assuming undocumented result fields.

```ts
import { setTimeout as delay } from "node:timers/promises";

const baseURL = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
const batchId = process.env.BATCH_ID;
if (!baseURL || !apiKey || !batchId) {
  throw new Error("Set INFRAI_BASE_URL, INFRAI_API_KEY, and BATCH_ID");
}
const base = new URL(baseURL);
if (base.protocol !== "https:") throw new Error("HTTPS is required");
const url = new URL(
  `ai/batch/results/${encodeURIComponent(batchId)}`,
  `${base.toString().replace(/\/$/, "")}/`,
);
async function readResults(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(30_000),
      redirect: "error",
    });
    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requested = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      await response.body?.cancel();
      await delay(Math.max(requested, 1000 * 2 ** attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Results HTTP ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Retry budget exhausted");
}
readResults().then((result) => console.log(JSON.stringify(result, null, 2)))
  .catch((error: unknown) => { console.error(error); process.exitCode = 1; });
```

The next example exercises the result-to-storage handoff locally. Save it as `tenant-batch.ts` and run it with `node tenant-batch.ts` on the same runtime. The three documents and usage amounts are synthetic fixtures, not provider responses or measured charges. Its output contains private storage intents; an authenticated storage adapter must persist them before publication.

These are application records, deliberately separate from provider payloads. Use the discovery schema to map remote results into those records. This example makes that reconciliation reproducible; it does not submit documents or pretend that its fixture fields are an API request schema.

```ts
import { strict as assert } from "node:assert";

type Source = {
  tenant: string;
  document: string;
  revision: number;
  text: string;
};
type Result = { reference: string; summary: string };
type Usage = { reference: string; costMicros: number };

function reference(source: Source): string {
  return JSON.stringify([source.tenant, source.document, source.revision]);
}

function reconcile(sources: Source[], results: Result[], usage: Usage[]) {
  if (sources.length === 0) throw new Error("Empty batch");
  const tenant = sources[0].tenant;
  if (sources.some((source) => source.tenant !== tenant)) {
    throw new Error("A batch must belong to one tenant");
  }
  const manifest = new Map(sources.map((source) => [reference(source), source]));
  if (manifest.size !== sources.length) throw new Error("Duplicate source");
  const summaries = new Map<string, string>();
  for (const result of results) {
    if (!manifest.has(result.reference)) throw new Error("Unknown result");
    if (summaries.has(result.reference)) throw new Error("Duplicate result");
    if (!result.summary.trim()) throw new Error("Empty summary");
    summaries.set(result.reference, result.summary);
  }
  const costs = new Map<string, number>();
  for (const item of usage) {
    if (!manifest.has(item.reference)) throw new Error("Unknown usage");
    if (!Number.isSafeInteger(item.costMicros) || item.costMicros < 0) {
      throw new Error("Invalid cost");
    }
    const total = (costs.get(item.reference) ?? 0) + item.costMicros;
    if (!Number.isSafeInteger(total)) throw new Error("Cost overflow");
    costs.set(item.reference, total);
  }
  return sources.map((source) => {
    const id = reference(source);
    const summary = summaries.get(id);
    const costMicros = costs.get(id);
    if (summary === undefined || costMicros === undefined) {
      throw new Error("Incomplete reconciliation");
    }
    return {
      tenant,
      key: [tenant, source.document, String(source.revision)]
        .map(encodeURIComponent).join("/"),
      acl: "private" as const,
      sourceReference: id,
      summary,
      costMicros,
    };
  });
}

const sources: Source[] = [
  { tenant: "harbor", document: "receiving", revision: 2,
    text: "Record the seal number before unloading." },
  { tenant: "harbor", document: "exceptions", revision: 4,
    text: "Escalate damaged pallets to the shift supervisor." },
  { tenant: "harbor", document: "cold-chain", revision: 1,
    text: "Keep the temperature log with the shipment record." },
];
const fixtureSummaries = [
  "Record seal numbers before unloading.",
  "Escalate pallet damage to the shift supervisor.",
  "Retain the temperature log with the shipment record.",
];
const results: Result[] = sources.map((source, index) => ({
  reference: reference(source), summary: fixtureSummaries[index],
})).reverse();
const usage: Usage[] = sources.map((source) => ({
  reference: reference(source), costMicros: 100,
}));
const records = reconcile(sources, results, usage);
assert.equal(records.length, 3);
assert.equal(records[0].summary, fixtureSummaries[0]);
assert.equal(records.reduce((sum, record) => sum + record.costMicros, 0), 300);
assert.throws(() => reconcile(sources, results.slice(1), usage));
console.log(JSON.stringify(records, null, 2));
```

Reversing the fixture results matters: joining by array position would attach the cold-chain summary to the receiving guide. The reference join survives that change. Revisions also prevent a later summary from overwriting a different source version just because the document name stayed the same. Consider receiving revision 2 and revision 3 completing in reverse order: a last-write-wins record keyed only by document name would publish whichever result arrived last, even if that result described the older instruction. Keeping revision identity in the manifest makes the publication decision explicit instead of confusing arrival order with source freshness.

Arrival order isn't freshness.

The integer amounts keep the arithmetic inspectable. They do not solve billing attribution by themselves. If a remote system supplies only an aggregate job charge, record that charge at job level and label any document-level allocation as an estimate. Dividing a total cannot produce observed per-document usage. Persist raw evidence separately from allocations so that a later accounting correction does not alter the summary's source record.

## Choose the service at the generation-to-storage seam

Infrai combines one API key, one bill, and a self-describing REST API for multiple backend services. Public discovery needs no key and provides request and response schemas plus runnable examples, so wiring a capability starts with reading its contract instead of learning another SDK. The same API key covers generation and private storage with unified billing. That shared credential reduces the account handoff between the summarization worker and its storage integration. The platform covers 295 routes across 20 modules, bringing multiple backend services under consistent REST conventions. Its specified per-call cost, vendor, and latency metadata also provides useful ledger inputs. A tenant ledger must still distinguish a call charge from an allocation to individual documents.

Tenant ownership remains the application's responsibility.

The table compares integration boundaries rather than ranking summary quality. The alternative service guides in References are the contract checks to make before implementing an adapter. Keep the manifest and publication rule independent of that adapter so replacing generation does not require redesigning tenant ownership.

| Option | Integration approach | Setup work | Suitable boundary | Main limitation for this workflow |
| --- | --- | --- | --- | --- |
| Unified REST option above | Public discovery, REST, runnable examples | Read the capability schema and map results to the manifest | Generation and private storage under a shared credential and bill | A shared bill still needs application-owned tenant attribution |
| OpenAI API | Batch API through REST or a client SDK | Prepare batch input and reconcile output with source identifiers | An application choosing a generation-focused service and its own private store | Generation output still needs a separate publication and storage boundary |
| Anthropic API | Message Batches through REST or a client SDK | Map message requests and batch results into the ingestion worker | An application choosing its message-generation contract | Its batch contract is a separate adapter; it does not define the application's tenant ledger |
| Amazon Bedrock | AWS API or SDK for batch inference | Configure batch input, output, and AWS access permissions | A system whose storage and access ownership already lives in AWS | AWS permissions and job setup become part of the ingestion boundary |

For an OpenAI API integration, decide whether a generation-focused account plus a separate private store suits the application. For Anthropic API, evaluate source fidelity before adapting results into the same ledger; an attractive summary is insufficient if its document association gets lost. For Amazon Bedrock, assess the AWS account boundary alongside the model interface. Existing permissions and operational familiarity can matter more than reducing the number of integrations. None of these choices removes the need for tenant reconciliation, and this comparison claims no measured quality or latency winner.

The combined approach concentrates trust and billing in one vendor. Accept that boundary deliberately.

## Publish evidence only after reconciliation

This approach is not suitable for ingestion that must finish before a dispatcher gets an answer. Its limitation is the asynchronous completion boundary: choose a real-time generation workflow for that requirement. Smaller batches make attribution easier, but require more job tracking. Mixing tenants may improve packing, yet demands a stronger reconciliation layer before either expense or evidence can be assigned safely.

Before enabling background ingestion, check that the authenticated tenant determines the manifest owner, every document revision has a stable reference, and submission retries reuse a durable idempotency key. Record the returned job identifier with that submission. Keep completion checks separate from result publication, and reject unknown, duplicate, empty, or missing results rather than treating a finished remote job as proof that the knowledge base is ready. The local fixture demonstrates these checks at the handoff; the worker still needs durable state across restarts.

Then inspect the ledger. Preserve the original charge evidence and the rule used for any allocation. A shared service credential does not enforce tenant access, so the application must authorize reads of both the source and its summary. Persist private objects before updating the published revision pointer; use presigned URLs for any authorized object download, keeping the service credential out of that download request. Retain a source reference with each published summary so the answering path can return to the underlying logistics instruction when a compressed passage is ambiguous.

Ship that boundary first. More aggressive packing can wait until the records show which tenant owns the work, which revision supplied the answer, and which charges are observed rather than allocated.

## References

- OpenAI, Batch API guide: https://platform.openai.com/docs/guides/batch
- Anthropic, Message Batches guide: https://docs.anthropic.com/en/docs/build-with-claude/batch-processing
- Amazon Bedrock, Batch inference: https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html
- Node.js, TypeScript execution: https://nodejs.org/api/typescript.html
- AWS, Sharing objects with presigned URLs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html
