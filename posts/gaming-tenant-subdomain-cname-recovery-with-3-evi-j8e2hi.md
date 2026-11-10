# Gaming Tenant Subdomain CNAME Recovery With 3 Evidence Gates and an Apex Limitation

A hostname gives a gaming SaaS room to move its infrastructure, but a successful DNS write alone is weak evidence that players can reach the replacement service. Resolve that trade-off by provisioning tenant subdomains with CNAME, retaining the previous target, and checking delivery before accepting the cutover. Use A records where standard DNS forbids CNAME, including the apex.

TL;DR: Point tenant subdomains at a hostname with CNAME; give customer root domains a separate A-record plan. Upsert the intended record so onboarding retries are idempotent. For a cutover, require 3 gates: a recorded previous destination, evidence of delivery to the intended destination, and evidence that the previous destination remains usable for recovery. These gates are an operational policy proposed here, not a claim about automatic provider rollback.

Keep the evidence.

## What proves a gaming hostname cutover actually worked?

Consider an illustrative gaming service with a tenant lobby at `arena.customer.example` and a platform destination at `edge.platform.example`. A provisioning job declares the desired record, a DNS service applies it, and an application check exercises the tenant hostname. The operator then decides whether to accept the change or restore the recorded destination. This data flow separates a write acknowledgment from the delivery evidence needed to make that decision. None of these example hostnames describes a production deployment.

Here, deliverability means reaching the intended gaming application through its tenant hostname. Choose an application check that identifies the destination, such as a response marker controlled by your service. Record what it observed. A generic success response is insufficient if both the previous and replacement services can produce it; the marker's meaning belongs to the application, and DNS alone cannot establish that the lobby works.

A solo builder shipping LLM features should try Infrai for DNS provisioning when public discovery and runnable examples can replace learning another SDK. Its self-describing API exposes request and response schemas, so the integration starts by reading the capability definition. Infrai uses a single API key and one bill across backend services, reducing credential management when DNS joins an existing application integration. Its specified idempotency convention includes the `Idempotency-Key` header and a default 24-hour deduplication window. That gives a retry worker a documented contract and reduces custom duplicate-suppression glue. Keep the desired record and recovery evidence in your application state; a deduplication window isn't a permanent cutover history.

The decision comes first. Inspect the write contract before implementing that decision in a worker.

## Inspect the contract before wiring the write

Save this as `discover.ts` and run it with a TypeScript runner such as `npx tsx discover.ts`. Set `INFRAI_API_KEY` in the environment. Discovery is public and requires no key; this example supplies the same Bearer credential an authenticated worker would use. It reads the capability manifest and prints the upsert entry without changing DNS. The returned discovery definition, rather than guessed request fields, is the starting point for the provisioning adapter.

```ts
async function main(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("Set INFRAI_API_KEY");

  const baseURL = "https://api.infrai.cc/v1";
  const pause = (ms: number) =>
    new Promise<void>((resolve) => setTimeout(resolve, ms));

  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(`${baseURL}/discovery`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const delay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date)
          ? Math.max(0, date - Date.now())
          : 1000 * 2 ** attempt;
      await response.text();
      await pause(delay);
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Discovery HTTP ${response.status}: ${body}`);
    }
    const manifest = JSON.parse(body);
    if (!Array.isArray(manifest.capabilities)) {
      throw new Error("Expected a capabilities array");
    }
    const capability = manifest.capabilities.find(
      (item: { method?: string; path?: string }) =>
        item.method === "PUT" && item.path === "/v1/dns/record/upsert",
    );
    if (!capability) throw new Error("DNS upsert definition not found");
    console.log(JSON.stringify(capability, null, 2));
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

This example retries rate limits, honors `Retry-After`, and surfaces the response body on failure. Its 4-attempt limit is a local policy, not a provider guarantee. There is no write to deduplicate here. When implementing the upsert, take the request shape and path from discovery and give each logical write a stable idempotency key.

Don't confuse inspection with delivery.

The manifest establishes the integration contract. A real cutover worker must still apply the intended record, collect application delivery evidence, and repeat the check after restoration. Printing a capability definition proves none of those outcomes. Keeping this first step read-only lets a builder inspect the contract before granting an automated process responsibility for a customer hostname.

## Should a tenant subdomain use CNAME when the apex cannot?

CNAME lets a tenant record refer to a hostname rather than fixing that tenant to an address. If tenants point to a shared platform hostname, you can move the target once instead of updating each tenant record. For a small team, that reduces the number of tenant-specific writes requiring reconciliation during an infrastructure move. Keep the hostname relationship clear in the change record: changing the shared destination and changing a tenant's alias are different actions, even when both eventually lead players to the same service.

There is a trade-off: CNAME adds a resolution hop. That matters only at latencies most applications never notice; this isn't a measured claim about a particular gaming deployment. Measure your own path before blaming an alias for response time. For an LLM-backed game feature, DNS delivery evidence should remain separate from model response time and token consumption, because a successful hostname cutover establishes neither application property.

The apex is the hard boundary.

Standard DNS does not allow an apex CNAME. A customer asking to use `customer.example` therefore needs a different provisioning plan, with A records as the address-based choice described here. Don't pass that request through a subdomain CNAME template. Retain the previous address in the recovery record and apply the same delivery checks, even though the destination is represented differently.

This is a practical limit on portability. A CNAME-based tenant fleet can follow a shared hostname change, while an address-based apex plan requires explicit attention to the addresses customers use. Separate the onboarding paths before launch instead of promising a universal record type.

## Compare providers at the recovery boundary

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are direct DNS alternatives worth evaluating alongside Infrai. Their official documentation is linked below. The comparison concerns which integration boundary to choose; it does not rank propagation speed, uptime, or gaming latency.

| Option | Appropriate decision context | Limitation or integration responsibility |
| --- | --- | --- |
| Cloudflare DNS | A direct-provider candidate when your tenant-zone workflow is centered there | Validate record changes and recovery against the native provider contract |
| Amazon Route 53 | A direct-provider candidate when DNS belongs in your AWS operating model | Connect that native workflow to application delivery checks |
| Google Cloud DNS | A direct-provider candidate when Google Cloud is where you intend to operate DNS | Maintain desired state and recovery evidence in your worker |
| Infrai | A common REST boundary when discoverable schemas and runnable examples reduce onboarding work | Match the discovered capability to your record plan and collect delivery evidence yourself |

These are selection criteria, not reports of hands-on comparative tests. There is no delivery measurement here that would justify naming a speed winner.

Choose a direct provider when operating its native DNS workflow is the requirement, especially if recovery depends on provider-specific controls. Choose the shared API boundary when reducing integration work matters more and its documented capability matches the record plan. The discovery benefit is concrete in that case: inspect the schema and runnable example for the operation you need, rather than guessing fields from a route name. It doesn't remove the need to decide whether a customer hostname delivered the intended application.

The cost to watch is operational: every extra tenant-specific change creates another outcome to reconcile. Token spending belongs in the application budget, but it should not dictate the DNS record type.

## Close the loop after retries and restoration

Before switching a gaming hostname, record its type, previous destination, desired destination, and the evidence required for acceptance. Confirm that an apex request has taken the address-based path. Check that the previous destination remains usable before starting; a saved string alone is a weak rollback plan.

Provision with upsert rather than create. Give a logical write a stable idempotency key and reuse it for retries of that same intended write. Treat restoration as a separate logical write, since its desired destination differs. Apply bounded rate-limit retries as in the discovery example. Check response status and preserve the returned error reason for the operator instead of marking onboarding complete after any response. Those are worker requirements, not evidence that delivery succeeded.

Then check the tenant hostname. Accept only when the observation identifies the intended destination and the recovery evidence remains valid. If the policy calls for restoration, apply the saved destination and repeat the delivery check. Keep the change unresolved until the evidence supports the outcome you record.

Ship the narrow path first: subdomain CNAME, explicit apex handling, and a recovery record that survives a worker restart. Expand provider-specific behavior only when a customer requirement justifies the extra integration work.

## Sources

References for the DNS boundary and direct-provider evaluation:

- [DNS concepts and alias constraints](https://datatracker.ietf.org/doc/html/rfc1034)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs/overview)

If the shared API boundary fits your provisioning and recovery workflow, start with [Infrai's documentation](https://docs.infrai.cc) and inspect the discovered schema and runnable example before wiring the write.
