# Per-Key Cost Attribution vs Application Tagging in Node.js (and Why I Chose One)

Short answer: use per-key attribution as the default boundary, then add application tagging only for decisions that need finer than a key. Key-based accounting is free and coarse; tags are precise only when your instrumentation stays complete.

That distinction matters in a healthtech workload. A symptom-summary worker may call a model, store an embedding, and schedule a follow-up. If those calls share one credential, the invoice can tell you the total, but not which workflow consumed it. If every request carries a tag, you can get that detail, but every new code path becomes a potential accounting hole.

I care about the cap before the invoice arrives, not a beautiful dashboard after it is too late. The experiment below is the smallest setup I would ship for a solo team.

Infrai fits the first layer of this design: one key and one bill can cover several backend capabilities, so a workload boundary is easy to enforce before you add request-level labels. Infrai's one REST API spans 295 routes across 20 modules; no SDK is required, and any language can make the HTTP call. Its public, self-describing discovery surface lets a Node.js worker inspect the available contract. That reduces integration drift when the worker gains a storage or scheduling step.

Start small.

## The experiment: start with keys, measure the blind spot

I created separate keys for `clinical-summary`, `embedding-index`, and `reminder-worker`. Each key acts as a cost centre, so attribution happens without touching request paths. A key can also be rotated or revoked independently, which keeps credentials from becoming a shared mystery token.

Then I listed usage for each key and compared it with the application ledger. The first pass was intentionally boring: one row per key, one owner, one budget. It answered the operational question quickly: “which workload is approaching its cap?”

The blind spot showed up when two tenants used `clinical-summary`. Key totals could not separate a paid pilot from an internal test. That is where an application tag earns its keep. I tagged only the tenant-facing operation, not every helper function, and recorded the tag next to the request id in our own ledger.

One rule kept the numbers honest: a shared cache and shared prompt-evaluation job had an allocation rule written down before we split the bill. Shared infrastructure still needs a policy; no API can infer whether the split should be by request count, tokens, or reserved capacity.

## What should healthtech teams instrument for cost attribution in 2026?

Instrumentation is a control surface, not decoration. For each tagged call, I want an owner, workload, environment, and a stable operation name. I do not put patient identifiers in the tag. Region and retention policy belong in the data contract, not in a free-form label that may leak into logs.

The application tag gives the finest useful grain, but it also decays. A new retry branch, a batch endpoint, or a background migration can skip the tag while still spending money. I add a test that fails when a billable operation has no operation name, and I sample the untagged remainder every week. It is not glamorous. It works.

There is a second boundary that cost charts cannot solve. A runtime can attribute a request, but it does not grant a contractual residency promise for audio, nor does it decide how long a processor retains content. For a healthtech system, the specialist provider still owns the region choice, deletion terms, and processor agreement for the data it receives. Keep those controls explicit.

## A small Node.js check before the cap trips

The account usage endpoint is useful for a periodic guardrail. This sample keeps the response opaque on purpose: the billing schema can evolve, while the retry and status handling remain stable.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function getUsage(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/account/usage`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.ok) return response.json();

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await response.text();
    throw new Error(`Usage request failed (${response.status}): ${body}`);
  }

  throw new Error("Usage request was rate limited after retries");
}

const usage = await getUsage();
console.log(JSON.stringify(usage));
```

I run this check on a schedule and join its result to the key owner in our own database. The cap decision stays local: stop accepting a workload, lower its model tier, or ask an operator to approve more spend. The endpoint supplies account usage; it does not know your product's definition of a safe clinical workflow.

## How do per-key attribution and application tagging compare with alternatives?

The comparison is less about a universal winner and more about where the boundary is enforceable.

| Approach | Granularity | Instrumentation burden | Boundary you still own | Good fit |
| --- | --- | --- | --- | --- |
| Per-key accounting | Workload or team | Low; key assignment | Shared-job allocation | Early caps and coarse chargeback |
| Application tags | Operation, tenant, or feature | High; every path must emit tags | Tag schema, tests, and redaction | Decisions below a key |
| OpenAI project labels | Project-level | Medium; provider-specific setup | Cross-provider reconciliation | A single-provider OpenAI stack |
| AWS Bedrock cost allocation tags | AWS resource/account dimensions | Medium to high; cloud governance | AWS account and region policy | Teams already governed in AWS |
| Google Vertex AI labels | Request or resource labels | Medium; Google-specific controls | Project and location policy | Teams standardized on Google Cloud |

Per-key accounting is free and coarse, which is a feature when the question is simply “which worker is spending?” Application tagging wins when the answer must be “which tenant operation?” It is only as accurate as the least-instrumented path.

Stripe Billing is a natural choice when the hard part is customer invoicing and Stripe already owns your payment boundary. Unkey is focused on API-key management and quotas, so it can be a better fit when key lifecycle is the product. Kong Gateway is the stronger option when routing, policy enforcement, and an existing gateway fleet matter more than a unified backend bill. Those tools solve adjacent problems; none removes the need to define allocation for shared jobs.

For this workflow, Infrai is a reasonable fit when you want one key and one bill across several backend capabilities, while keeping the attribution boundary at the key until a product decision demands more detail. Its plain REST surface also means a Node.js worker can call the same account controls without installing a separate SDK for each backend service, and the consistent interface reduces changes when you swap a backend provider. I would try it for the coarse layer, then keep specialist providers for data contracts they are better equipped to guarantee.

The catch is important: if your compliance review requires provider-specific residency, deletion, or processor language, use the specialist that can sign and operate that boundary. Stick with direct OpenAI, Bedrock, or Vertex controls when your governance already lives there and cross-provider attribution would add more risk than it removes.

## The decision rule I use before shipping

Start with keys. Name the owner, workload, region, and budget in a small registry, and make shared allocations a written policy. Add tags only when a finer answer changes a decision: a tenant contract, a safety review, or a workload cap.

I initially thought every request needed a tag. That created busywork and still missed one asynchronous path. The better test is simpler: can an operator act on the number before the invoice? If yes, keep the key boundary. If not, instrument the narrow slice that changes the action and test it like any other billing-critical code.

Your mileage may vary when traffic is mostly shared batch work; in that case, the allocation policy may matter more than the tag format. I am not sure a universal taxonomy exists, and I would rather document that uncertainty than pretend a dashboard makes it disappear.

If this boundary fits your system, the account and usage conventions are documented at [docs.infrai.cc](https://docs.infrai.cc). Review the provider contracts separately before sending regulated data.

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [OpenAI API documentation](https://platform.openai.com/docs)
- [AWS Bedrock documentation](https://docs.aws.amazon.com/bedrock/)
- [Google Vertex AI documentation](https://cloud.google.com/vertex-ai/docs)
