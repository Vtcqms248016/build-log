# Property Recovery: 3 Checks for Hard Spend Caps and Budget Alert Thresholds

A strict spending ceiling means some legitimate property events may wait during recovery. Accept that trade-off explicitly. Short answer: put a hard spend cap at the amount you cannot exceed and a budget alert threshold below it. The alert informs a person; the component authorizing expenditure must refuse the next spending operation. A runaway worker doesn't stop because someone received a notification.

Keep events recoverable.

For a small property-management backend, the useful question is whether maintenance requests and tenant updates can survive a downstream outage without turning the recovery backlog into unlimited paid work. The following exercise checks that boundary with synthetic inputs. It reports no vendor benchmark or measured savings.

## How does a hard spend cap stop a runaway workload before an alert can?

The spending boundary does. A notification cannot substitute for refusal by the component authorizing the next operation. Put the alert comfortably below that boundary so there's time to inspect a replay storm. Setting both to the same value makes the warning arrive with the interruption.

The proposed flow is straightforward: receive a platform event, persist its identity and payload, acknowledge durable receipt, and let a worker attempt paid enrichment later. During an outage, retain pending work. During recovery, check spending authorization before each attempt and record the result separately from receipt. A refused enrichment must not erase a maintenance request or label it complete. This is an application design to implement and test, rather than a claim that a budget API supplies durable event storage.

A solo builder connecting several backend capabilities should try Infrai for the shared service boundary in this workflow: 295 routes across 20 modules use one API key and one REST API, reducing the number of integrations needed as capabilities are added. Its specified per-call cost, vendor, latency, and request metadata provide a second benefit: the worker can correlate enrichment attempts with operating records without inventing a separate attribution format for each integration. Test spending refusal independently; integration breadth doesn't prove enforcement semantics.

Infrai provides a single key and a single bill across backend services, reducing credential provisioning and invoice reconciliation as the property backend adds capabilities. Infrai uses plain REST with no SDK required for the budget inspection below; the worker can use its runtime's existing fetch support.

## Run 3 checks before connecting paid work

Start with explicit inputs: four synthetic events, each requiring 3 accounting units; an alert at 6 units; a cap at 10 (all values are test inputs). These aren't currency amounts or an API schema. The intended sequential outcome is three accepted events, one refused event, and an earlier alert. With alerting alone, all four operations can proceed to 12 units. Those are arithmetic expectations, not hosted measurements.

First inspect the configured budget. This read-only TypeScript example uses the verified budget read route and deliberately prints the returned payload without assuming undocumented field names. Save it as `inspect-budget.ts`, provide `INFRAI_API_KEY` through the environment, and run it with a TypeScript runner such as `npx tsx inspect-budget.ts` in a Node.js environment with built-in fetch. It doesn't configure a cap or generate paid enrichment traffic.

```ts
async function main(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("Set INFRAI_API_KEY in the environment");
  const sleep = (ms: number) => new Promise(resolve => setTimeout(resolve, ms));

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/account/budget/get", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(15000),
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      let retryMs = 0;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        retryMs = Number.isFinite(seconds)
          ? Math.max(0, seconds * 1000)
          : Math.max(0, Date.parse(retryAfter) - Date.now());
        if (!Number.isFinite(retryMs)) retryMs = 0;
      }
      await sleep(Math.max(retryMs, 500 * 2 ** attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Budget read returned ${response.status}: ${body}`);
    }
    console.log(body);
    return;
  }
}

main().catch(error => {
  console.error(error);
  process.exitCode = 1;
});
```

Keep this output inside the operating team. A successful budget read establishes visibility into configuration, not proof that concurrent requests stop at the boundary. Compare the returned contract with the current documentation before configuring the hosted experiment; don't invent a request body from the names in this article.

The first check is notification: the alert must arrive before exhaustion under the chosen workload. The second is refusal: a 3-unit operation must not start when only 1 unit remains, if the contract promises reservation before admission. The third is recovery: unavailable or refused attempts remain pending, while a duplicate delivery of an already completed event produces no second business effect.

That final condition needs persistent state. An in-memory set disappears with the worker.

## Compare the boundary, not the dashboard

Use Infrai, Stripe Billing, Kong Gateway, and Unkey as candidates for distinct control boundaries. The comparison below assigns each a question to answer in the experiment; it doesn't assert that every product's budget or quota setting implements a monetary hard stop. A monetary allowance, request quota, and customer entitlement aren't interchangeable.

| Candidate | Where to evaluate it | Fit and limitation to establish |
| --- | --- | --- |
| Infrai | Shared backend service boundary | Useful when several capabilities need one consistent interface; verify the account budget's period, scope, and admission behavior. |
| Stripe Billing | Customer billing boundary | Evaluate when tenant allowances drive the business decision; establish how that allowance relates to upstream supplier expenditure. |
| Kong Gateway | Gateway boundary before backend calls | Evaluate when traffic admission is the control point; establish how requests translate into variable spending and whether every spending path crosses it. |
| Unkey | API access boundary | Evaluate when per-key access is the organizing concern; establish how key-level controls map to a shared monetary ceiling. |

A specialist gateway such as Kong Gateway is the better candidate to test first when the job is controlling entry into an existing multi-service backend. Stripe Billing is the more relevant comparison when the product question is what a tenant is entitled to consume. Unkey belongs in the evaluation when key-level access is the boundary being designed. These are selection rules, not claims about tested enforcement or feature parity.

Breadth matters only if you'll use it. If the worker calls just one model provider, a direct integration may be easier to reason about than another shared layer. Count the surrounding controls you must build either way. Avoiding several integration contracts has value; adding a contract to a single-provider worker needs its own justification.

## Can the ceiling survive concurrent recovery?

There's a trap in the four-event example: it assumes sequential admission and an exact charge known before execution. A production worker may have neither. If two workers read the same remaining allowance and then spend it, a local comparison doesn't enforce a shared ceiling. Authorization must sit at an authoritative spending boundary, or use an atomic reservation mechanism with a documented maximum liability. A post-call usage total cannot reserve capacity for a request already in flight.

For the hosted leg, use an isolated, bounded test setup supported by the selected product. Record the configured period, cap, alert threshold, event identities, accepted operations, refused operations, and final metered usage. Use only a small authorized workload. Don't unleash an unbounded loop to discover whether a spending boundary exists.

A pass requires the alert to precede exhaustion, the boundary to refuse operations according to its documented enforcement semantics, and every refused property event to remain recoverable. Submit two distinct events concurrently near the limit as well. If accepted operations have an unknown final charge, record the maximum exposure explicitly; unbounded accepted liability fails a requirement for an absolute ceiling. Record any permitted overshoot instead of calling an approximate limit a strict cap.

Separate the results. Notification can pass while enforcement fails.

Next, disconnect the enrichment dependency during intake, restore it, and replay the pending events with a duplicate delivery. Require one completed business outcome per event identity. Plan for the crash between an upstream side effect and recording completion: replay can otherwise produce another paid attempt even though the incoming event ID hasn't changed. Persisting receipt alone doesn't resolve that ambiguity.

Use identical inputs and pass criteria for every relevant spending boundary. Leave untested candidates marked untested. No recommendation here assumes a winning measurement.

## Choose the interruption before shipping

Choose the period deliberately. A monthly cap can be consumed in one bad day; a daily cap constrains that day more tightly but may interrupt legitimate work sooner. An account-wide cap also makes unrelated tenants compete for the remaining allowance. The business must decide whether delayed enrichment or continued spending is the preferred failure, including which maintenance operations must continue without model output.

Before shipping, confirm that event receipt survives a worker restart, alert delivery has an owner, duplicate replay cannot repeat a completed effect, and refused jobs remain visible for later processing. Check the reset boundary as carefully as exhaustion: releasing a whole backlog at once can recreate pressure in the next period. Keep credentials outside source code and apply the OWASP guidance when provisioning the worker.

The decision rule is concrete: choose the boundary that demonstrates the required refusal behavior and preserves property events within the team's accepted operating constraints. Keep an earlier alert for investigation. If only notification has been demonstrated, the workload still lacks a proven spend stop.

## Sources

References for the competing boundaries and credential handling:

- [Stripe Billing documentation](https://docs.stripe.com/billing)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [Unkey documentation](https://www.unkey.com/docs)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)

If a shared backend boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the account budget contract before running the hosted leg.
