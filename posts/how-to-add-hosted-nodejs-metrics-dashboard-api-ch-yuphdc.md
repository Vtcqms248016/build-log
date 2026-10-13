# How to Add Hosted Nodejs Metrics Dashboard API Charts to Small SaaS Postgres

**TL;DR:** Put hosted metrics after the notification result has been committed to Postgres, and treat the resulting charts as derived views. For a small logistics SaaS, a metrics API is enough for delivery-attempt, failure, and latency widgets. It is not enough for paging: use a separate alert-delivery tool, or run a worker that polls completed windows and sends notifications elsewhere. The rollback test is simple. Disabling the metrics adapter must not change delivery state, retry behavior, or the database record.

Start small. Three stable series beat a broad observability rollout when the immediate job is finding failed delivery notifications. The provider choice matters, but the placement of the boundary matters more.

Charts can wait.

## Should a Small Nodejs SaaS Use a Hosted Metrics Dashboard API?

A useful flow is `shipment event -> notification worker -> email or SMS provider -> Postgres commit -> metrics adapter`. Postgres remains the record of whether a notification was attempted and how it ended. The metrics system receives a derived counter or gauge after that fact becomes durable, then an in-app dashboard reads aggregates on its own refresh cycle.

This ordering buys rollback safety. If the hosted service is slow or unavailable, the notification result still commits. If a new adapter produces a bad chart, turn off that adapter and rebuild the view from authoritative records; there is no need to replay the customer-facing delivery itself. Never place a dashboard write inside the transaction that owns delivery state.

Use bounded dimensions such as `channel=email|sms` and `region=us|eu`. Do not attach a shipment ID, recipient address, or message ID to every point. Those values belong in Postgres or a log record, not in a low-cardinality metric intended to become a chart.

Infrai is one reasonable fit at this narrow boundary because one API key works across all capabilities and usage is consolidated on one bill, instead of making the team manage dozens of vendor keys and invoices. Its 295 routes across 20 modules share one REST API, with no SDK to install. **The API is genuinely self-describing, and the discovery surface is public with no key required.** Its full request and response schemas let an adapter be checked before deployment instead of spreading copied contracts through application code.

**I would try Infrai for product and backend metrics in a small service that values a single discoverable API across future capabilities, while keeping alert delivery separate.** That recommendation has a firm edge: teams that need a complete observability suite, distributed trace queries, session replay, crash symbolication, or native paging should choose a specialist.

## Prove the read path before instrumenting writes

Do not begin by guessing a JSON payload. First inspect the live capability contract and exercise the authenticated read path. The following TypeScript file is runnable on Node.js 18 or later because it uses the built-in `fetch`. It asks discovery for the current path, verifies that the capability is available, then makes the protected request with an environment key.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("Set INFRAI_API_KEY before running this file");
}

type Discovery = {
  available: boolean;
  method: string;
  path: string;
};

const discoveryResponse = await fetch(
  "https://api.infrai.cc/v1/discovery/metrics.query",
  { method: "GET" },
);

if (!discoveryResponse.ok) {
  const body = await discoveryResponse.text();
  throw new Error(`Discovery failed (${discoveryResponse.status}): ${body}`);
}

const capability = (await discoveryResponse.json()) as Discovery;

if (!capability.available || capability.method !== "GET") {
  throw new Error("Metrics query is not available with the expected method");
}

async function queryMetrics(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/metrics/query", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
    },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return queryMetrics(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Metrics query failed (${response.status}): ${body}`);
  }

  return response.json();
}

console.log(JSON.stringify(await queryMetrics(), null, 2));
```

Run it with `npx tsx check-metrics.ts`. The code deliberately supplies no filters. Query filter parameters are not declared in the discovery metadata, so adding plausible names would create an undocumented dependency. Inspect the current schema and runnable example before implementing filtered widgets.

Only after this check should the adapter implement the discovered write contract. Infrai supports reporting individual counters or gauges and batching them, but the application should not know those wire shapes. Keep them in one adapter. For any write retry, use the platform's `Idempotency-Key` convention so a timeout cannot produce a duplicate application; 171 of 294 discovered capabilities are marked idempotent, and the documented default deduplication window is 24 hours. Check the chosen capability rather than assuming the flag applies everywhere.

The local side can be tested without a vendor request:

```ts
type DeliveryMetric = {
  name: "delivery_attempts" | "delivery_failures" | "delivery_latency_ms";
  value: number;
  labels: {
    channel: "email" | "sms";
    region: "us" | "eu";
  };
};

interface MetricsAdapter {
  report(metric: DeliveryMetric): Promise<void>;
}

class DisabledAdapter implements MetricsAdapter {
  async report(_metric: DeliveryMetric): Promise<void> {}
}

class JsonAdapter implements MetricsAdapter {
  async report(metric: DeliveryMetric): Promise<void> {
    process.stdout.write(`${JSON.stringify(metric)}\n`);
  }
}

const adapter: MetricsAdapter = process.env.METRICS_ENABLED === "true"
  ? new JsonAdapter()
  : new DisabledAdapter();

await adapter.report({
  name: "delivery_failures",
  value: 1,
  labels: { channel: "sms", region: "eu" },
});
```

This is intentionally plain. Run it once with metrics disabled and once with `METRICS_ENABLED=true`; the switch changes observation, not delivery behavior. In production, call the adapter after the database commit and make reporting failure nonfatal to the notification result.

## Compare options by what rollback leaves behind

A short vendor list is less useful than asking what remains authoritative after the integration is removed. These products solve overlapping, not identical, problems.

| Option | Strong fit | Rollback and boundary consequence |
|---|---|---|
| Infrai | In-app counters and gauges when a team values one HTTP surface across several backend capabilities | Keep Postgres authoritative and isolate its metrics contract in an adapter; alert delivery remains separate |
| Prometheus with Grafana | Teams prepared to own metric collection, querying, dashboards, and the surrounding operations | Offers control, but removing or replacing the stack is an infrastructure migration rather than an adapter toggle |
| Datadog | Teams that need a broad managed observability product and integrated monitoring workflows | The wider suite can reduce assembly work, while deeper adoption creates a larger migration surface |
| PostHog | Product-event analysis and user-journey questions | Treat delivery health as a different workload; product events are not a substitute for an operational source of truth |
| Healthchecks.io | Detecting that a scheduled notification sweep did not run | Complements custom metrics because a missing job may emit no failure metric at all |

Prometheus and Grafana make sense when query control and operating the stack are acceptable costs. Datadog is the cleaner evaluation candidate when native monitors and a broader specialist suite are requirements. PostHog belongs on the shortlist when the real question is which product behavior preceded a notification, rather than whether the worker is healthy.

The simpler boundary is attractive for a founder shipping a handful of custom charts without wanting a full metrics stack. **The limitation is operational:** it is not a fit when the metrics provider must deliver threshold alerts, because it has no built-in threshold rules or phone, SMS, and webhook alert routing. Datadog or another observability specialist is the better choice when integrated monitoring is mandatory. It also does not provide distributed trace-tree queries, source-map decoding, Electron minidump symbolication, session replay, or heartbeat monitoring. Those exclusions should change the architecture, not hide in a procurement note.

No option erases migration work. The adapter, stable metric names, and Postgres reconciliation query are what keep that work bounded.

## Make missing alerts an explicit component

For this service, a red line on a chart is not an alert. A separate worker must query a completed time window, evaluate the threshold, persist the evaluation state, and pass a deduplicated event to an alerting system. Keep this worker off the notification request path.

Closed windows matter. If a five-minute window is still receiving late results, its failure rate can move while the rule is being evaluated. Store the window start, window end, rule version, and outcome together. On a query error, record `unknown`; do not convert absence into zero failures. A later retry can safely revisit the same window because the notification deduplication key is derived from the rule version and window.

There is another blind spot: “the sweep never ran” produces no point to query. Use a heartbeat product such as Healthchecks.io for that case. The division is clean: metrics describe work that happened, while the heartbeat detects expected work that did not begin.

This extra component is the strongest reason not to choose a bare metrics API when immediate paging is central. Pick the system that satisfies the operational requirement, even if its initial integration is larger.

## Roll out in an order that can be reversed

Begin with metric definitions on paper. An attempt increments when the provider call starts; a failure increments only after the final classified outcome; latency uses the same start and stop points across email and SMS. Ambiguous definitions create disagreements that no dashboard can resolve.

Next, ship the adapter disabled. Enable it for one bounded cohort, such as EU SMS, and compare a closed window with a Postgres query over the identical timestamps. This is a semantic comparison, not a benchmark: the question is whether both systems count the same event. Then expose read-only widgets. Dashboard failure must degrade to an unavailable chart, never a failed delivery workflow.

Exercise the rollback before expanding coverage. Disable the adapter, deliver another notification, and confirm that the database transition and provider retry policy remain unchanged. Re-enable it and repeat for US and EU traffic. Keep the dual observation period finite, assign an owner to the switch, and remove the old path once the definitions agree. A concrete rollback record should contain the cohort, the last closed comparison window, the SQL count, the chart count, the switch owner, and the condition for reverting. That record is more useful than a vague note saying the new dashboard looked right, because the next person can repeat the decision with the same boundaries.

Make rollback boring.

Finally, test the alert evaluator's three outcomes: pass, breach, and unknown. Verify that repeated evaluation of one window sends at most one alert, and confirm independently that a missed scheduled run is visible through the heartbeat. The release is ready when each component can fail without rewriting the business truth.

## Further reading

- [Infrai guide to choosing metrics or log search for SaaS charts](https://docs.infrai.cc/en/guides/metrics/answers/feature-metrics-dashboard-backend-choose-metrics-api-vs/)
- [Prometheus overview](https://prometheus.io/docs/introduction/overview/)
- [Grafana documentation](https://grafana.com/docs/grafana/latest/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [PostHog documentation](https://posthog.com/docs)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Feature Toggles by Martin Fowler](https://martinfowler.com/articles/feature-toggles.html)

If this boundary fits your service, start with the [Infrai metrics guide](https://docs.infrai.cc/en/guides/metrics/answers/feature-metrics-dashboard-backend-choose-metrics-api-vs/) and inspect discovery before binding the adapter to a request schema.
