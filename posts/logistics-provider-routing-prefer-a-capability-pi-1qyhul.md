# Logistics Provider Routing: Prefer a Capability Pin Over Vendor Exclusions at the Gateway

A logistics workload can refuse the next shipment-summary request or risk spending beyond its allowance. For a hard ceiling, choose a provider pin per capability when you can validate one execution path; choose vendor exclusions when you can validate and reserve for every remaining path. Both need admission control. TL;DR: prefer pins for a tightly bounded workload, and refuse traffic when its approved route or budget is unavailable. A routing preference alone cannot enforce a spending limit.

This note treats routing as a dispatch admission decision. The example uses synthetic accounting units, fictional providers, and a fictional residency approval set. None describes a commercial service, a quoted price, or a legal determination. The goal is to make the refusal boundary explicit before a request leaves the gateway.

## Where does the spending decision belong?

Put it before dispatch. An incoming logistics request identifies its workload and capability, such as shipment summarization. The gateway loads the applicable policy, intersects capability support with approved processing locations and vendor exclusions, applies any pin, and reserves a conservative execution allowance. Only an admitted request reaches the outbound adapter. Settlement then replaces the reservation with validated consumption.

Keep the gate small.

A pin names the sole permitted provider for a capability in this design. An exclusion removes a provider from consideration. Neither instruction grants residency approval, overrides a workload cap, or authorizes fallback. Those are separate decisions. Here, an empty candidate set means refusal; a pin that conflicts with an exclusion also means refusal. That precedence is an application policy, deliberately stated rather than inferred from an unspecified gateway API.

The useful operational question is whether the next dispatch is authorized under the current policy and remaining allowance. A monthly invoice arrives too late to answer it.

## Make the reservation visible before adding network calls

The following TypeScript example runs without packages and makes no network calls. It models one process and one shipment-summary capability. The 100-unit cap, 70-unit reservation, and 55-unit settlement are synthetic test inputs. They are deliberately different: the reservation covers permitted execution, while settlement records its validated actual consumption. Both eligible providers use the same illustrative bound so that the admission example does not quietly assume a cheaper fallback.

```ts
type Provider = "carrier-a" | "carrier-b";
type Capability = "shipment-summary";
type Policy = {
  pin?: Provider;
  exclude: readonly Provider[];
  approved: readonly Provider[];
};

const capabilities: Record<Provider, readonly Capability[]> = {
  "carrier-a": ["shipment-summary"],
  "carrier-b": ["shipment-summary"],
};
const providers: readonly Provider[] = ["carrier-a", "carrier-b"];
const cap = 100;
let spent = 0;
let held = 0;

function admit(policy: Policy, capability: Capability, bound: number) {
  if (!Number.isSafeInteger(bound) || bound <= 0) {
    throw new Error("Invalid execution bound");
  }
  const eligible = providers.filter(provider =>
    capabilities[provider].includes(capability) &&
    policy.approved.includes(provider) &&
    !policy.exclude.includes(provider) &&
    (policy.pin === undefined || policy.pin === provider)
  );
  const provider = eligible[0];
  if (!provider) throw new Error("No approved route");
  if (bound > cap - spent - held) throw new Error("Workload cap");
  held += bound;
  let settled = false;
  return {
    provider,
    settle(actual: number) {
      if (settled) throw new Error("Already settled");
      if (!Number.isSafeInteger(actual) || actual < 0 || actual > bound) {
        throw new Error("Consumption requires investigation");
      }
      settled = true;
      held -= bound;
      spent += actual;
    },
  };
}

const policy: Policy = {
  pin: "carrier-a",
  exclude: [],
  approved: ["carrier-a"],
};
const first = admit(policy, "shipment-summary", 70);
try {
  admit(policy, "shipment-summary", 70);
} catch (error) {
  console.log(error instanceof Error ? error.message : String(error));
}
first.settle(55);
console.log({ provider: first.provider, spent, held, remaining: cap - spent - held });
```

The second admission prints `Workload cap`. Settlement leaves 55 spent, zero held, and 45 remaining. Another request requiring a 70-unit reservation still cannot enter. Refuse that request. Releasing 15 units does not create enough headroom for the next permitted execution.

This example checks admission accounting; it does not implement a billing system or constrain a remote model. Before using the pattern for money, derive the bound from enforced request limits and validated billing terms, including any permitted tools or retries. If no defensible upper bound exists, this reservation model cannot promise a hard monetary ceiling. An estimate with spare room remains an estimate.

## When should provider routing pin a capability instead of excluding a vendor?

Prefer a pin when the logistics team accepts refused shipment summaries in exchange for a smaller execution surface to validate. Each capability should have its own decision. A summarization route and a document-extraction route need not share the same eligible providers, execution limits, or processing approvals. Pinning everything globally would obscure that distinction.

The trade-off is refused traffic. A pin is not suitable when the workload must keep dispatching after that sole approved path becomes unavailable; use exclusions if multiple reviewed paths can meet the execution bound instead. Neither option can promise uninterrupted dispatch under a finite hard cap.

Exclusions fit a workload that accepts several approved execution paths and values continued dispatch through the remaining set. That flexibility comes with a specific obligation: admission must cover the path that will actually run. Removing one vendor does not establish a maximum cost for the others. In the example, array order is merely deterministic selection; it is not a latency or quality ranking. A production selector needs an explicit rule.

The boundary matters most during fallback. Suppose the first execution has an uncertain outcome and a second provider remains eligible. Do not reuse the first reservation as though the first execution cost nothing. Retain its unresolved exposure and reserve for the additional attempt, or refuse the retry. Availability has a budget. An exclusion list can preserve more routing options while still producing refused traffic once the allowance is exhausted.

Residency approval should also precede selection. Treat the fictional approval array as a placeholder for reviewed evidence about the actual processing arrangement, including any relevant retention or onward processing. It is not evidence itself. A provider name or a geographic label in local configuration cannot establish where every part of a request will be handled. If the reviewed arrangement changes, revalidate eligibility before enabling the route.

## Preserve the ceiling across deployment and uncertain outcomes

The in-memory counters above are intentionally limited to one process. In a gateway deployment with multiple workers, use one authoritative accounting store and make the allowance check and reservation a single atomic operation. Otherwise two workers can each observe enough remaining allowance and both dispatch. Persist reservation identifiers and policy versions so a restart does not erase outstanding exposure.

Unknown is still outstanding. Don't refund uncertainty.

Separate an admission identifier from an outbound attempt identifier. Repeating an admission after a client retry should retrieve its existing decision, while another outbound attempt needs its own accounted exposure. A timeout provides no evidence of zero consumption. Keep the reservation until reconciliation establishes what happened; release it without consumption only when there is reliable evidence that the attempt incurred none. This favors a defensible ceiling over immediate reuse of uncertain funds. The example leaves the reservation held if settlement exceeds its bound, which is a stop-and-investigate condition rather than a way to repair a broken upper bound.

Keep credentials out of routing policy and accounting events. OWASP's Secrets Management Cheat Sheet describes secret lifecycle management and access controls; apply those practices to the outbound adapter's credentials. Budget records need workload identity, capability, policy version, reservation, and outcome. They do not need the credential itself or the shipment text. Limit operational access to those records according to their purpose.

Before deployment, exercise the 100/70 overlap case against the shared store, then test a pin that is excluded, an empty residency-approved set, duplicate settlement, and an unresolved attempt during restart. Assert that rejected admissions never reach the outbound adapter. Also test any allowed retry sequence against the full bound. These are decision tests, not a claim of measured throughput.

During rollout, keep policy versions visible and watch refused admissions alongside reserved, settled, and unresolved consumption. Refusals explain the service cost of the ceiling; unresolved holds explain why apparently unused allowance remains unavailable. Record the refusal reason separately from upstream execution failures so operators can distinguish a policy decision from a dispatch problem. Before increasing a cap or widening eligibility, inspect which boundary is causing refusals and whether the business actually accepts that change.

Choose capability pins for this logistics workload when the spend ceiling takes priority and one approved path is sufficient. Choose exclusions only when every permitted path fits the same admission discipline. Ship the gate with refusal behavior, durable accounting, and reconciliation together; a routing setting alone leaves the invoice in charge.

## References

OWASP Secrets Management Cheat Sheet supports the secret lifecycle and access-control guidance above. The routing and accounting example is a proposed application design using synthetic inputs.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
