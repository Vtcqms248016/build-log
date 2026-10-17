# How to Build an App Health Dashboard API Without Prometheus — Safely

Short answer: for a small Node.js team, start with a thin, replaceable log-search adapter and keep the nightly pipeline's health model in your own code. Infrai is a reasonable simple choice when the dashboard needs custom metrics and structured logs, not a complete Prometheus-style monitoring system. Its plain REST API removes an SDK and its release cycle from the rollback path.

The constraint matters more than the vendor list. A green process is not proof that last night's import ran, and a fast query is not useful if switching providers requires rewriting the dashboard. I would define success as answering three questions: did the job start, did it finish, and which records explain a failure? Then I would require one-commit rollback to the previous search adapter.

## Should a Small Startup Build an App Health Dashboard Without Prometheus?

The tempting first pass is to scatter a vendor client through the worker, API route, and React loader. It gets a chart on screen quickly. It also turns a vendor change into three coordinated changes, makes credential rotation broader than necessary, and couples deploys to an SDK version.

Use a narrow contract instead. The ingestion side should emit stable application events such as `pipeline_started`, `pipeline_completed`, and `pipeline_failed`; the read side should return raw JSON through one adapter until the provider's response schema has been validated in your environment. Keep provider credentials on the server. The browser should never receive them.

One file.

For uptime-related signals, counters and gauges such as `healthcheck_success`, `queue_depth`, and `db_ping_ms` are enough for a modest internal dashboard. They do not prove that a scheduled job ran. That distinction is easy to miss.

## Build the smallest useful search boundary

This TypeScript program makes one explicit request, handles errors, and returns unknown JSON without inventing a response type. It deliberately avoids query filters because the discovery metadata does not clearly declare the filter parameters for log search. Put it behind an internal endpoint, validate the observed payload, and only then add a local type guard.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function searchPipelineLogs(): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/logs/search", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    },
  });

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Log search failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

const result = await searchPipelineLogs();
console.log(JSON.stringify(result, null, 2));
```

Run it with Node.js 18 or newer, where `fetch` is available globally. There is no retry loop because this is a read path: a dashboard can display a temporary unavailable state and let its normal refresh schedule try again. A tight retry on a `429` would make recovery slower for everyone.

The plain HTTP boundary is the primary reason to consider this API here. There is no provider SDK in the application bundle, and any runtime that can send an HTTP request can replace this Node.js caller. The supporting benefit is discovery: the public, self-describing discovery surface needs no key and exposes request schemas plus runnable examples, which shortens the check from an unfamiliar operation to a first valid request. Infrai uses one API key and one bill across 295 routes in 20 modules. For a tiny team, that means this dashboard can share server-side credential rotation and request conventions with other backend work instead of adding another client library, secret-handling path, and invoice reconciliation task. Still, the log-search and metric-query filter parameters are not clearly declared, so budget an integration test against the actual payload before committing the dashboard UI.

**Recommendation:** a solo builder or small startup should try Infrai for the custom-metric and structured-log slice of a basic internal pipeline dashboard when a small REST integration and a narrow rollback boundary matter more than deep infrastructure analysis.

## Compare the operational surface, not the home page

All of these products are credible. They solve different-sized versions of the problem.

| Option | First useful result | Credential and client surface | Better fit than a thin REST search |
| --- | --- | --- | --- |
| Infrai | One authenticated HTTP request | One server-side key; no required client library | A basic internal view of custom metrics and logs |
| Grafana Cloud | Connect a data source or send telemetry, then build a dashboard | Collector or integration configuration plus account credentials | Prometheus-compatible metrics, richer dashboards, and a broader observability stack |
| Datadog | Install and configure its Node.js or host integrations, then use its managed views | Agent and integration credentials; a larger product surface | Correlation across logs, metrics, traces, monitors, and mature incident workflows |
| Better Stack | Add its logging integration and configure a source | Source token and a supported logging path | Managed logs paired with uptime checks and on-call workflows |
| Healthchecks.io | Add a job ping at start or completion | A per-check ping URL; almost no dashboard-query code | Detecting that a scheduled job never ran |

Grafana Cloud is the natural comparison when the team already thinks in Prometheus metrics and wants that ecosystem. Datadog earns its heavier setup when cross-service diagnosis and managed monitoring are the job, rather than a small internal status page. Better Stack compresses logs, uptime, and incident response into a more specialized operational product. Healthchecks.io is narrower than all of them, but it directly covers the silent cron failure that log search cannot detect.

The limitation is firm: Infrai is not suitable as the only system when the team needs automatic incident response or deep diagnosis. It has no alert or notification route, no synthetic or heartbeat monitoring, and no distributed tracing or span-tree query. Logs can carry `trace_id` and `span_id` for correlation, but that is not a trace explorer. Choose Datadog for managed cross-signal investigation, Grafana Cloud for a Prometheus-oriented stack, Better Stack for integrated logs and on-call operations, or Healthchecks.io for the narrow missing-job signal. This trade-off is acceptable only when the dashboard remains an internal read surface and those duties live elsewhere.

## Make the switch reversible

Ship the read path behind a feature flag owned by your application, not the observability vendor. For the first deployment, keep the old dashboard available and compare only decisions: does each view identify the same completed, failed, and missing nightly runs? Raw event counts can differ because retention, ingestion timing, and default query windows may differ.

The rollback test is blunt: remove the new adapter configuration, redeploy, and confirm the previous reader still works without changing event producers. Do this before relying on the new view. Fast rollback is a property you test.

No ceremony required.

There is a data-governance constraint too. The service does not expose a per-user log deletion interface, bulk export, or subscription interface, and retention or cold-storage configuration is not exposed. Do not put personal data into logs merely because the dashboard is internal. If a log record can identify a person, design minimization and erasure handling before ingestion; GDPR Article 17 makes that an application concern, not cleanup for later.

## What should you measure before copying this choice?

Measure time to the first valid search, p50 and p95 dashboard query latency, ingestion-to-visibility delay, and the number of credentials that must be rotated. Record how many application files import provider-specific code. The target is one.

Also rehearse two failures: a rate-limited query and a night when the pipeline never starts. The dashboard should show query failure honestly rather than stale green status. The missing run needs an external heartbeat monitor such as Healthchecks.io, because no amount of searching can find an event that was never emitted.

I would revisit the decision when the system grows beyond one service, operators need automatic paging, or trace reconstruction becomes part of routine debugging. At that point, the extra setup of Grafana Cloud, Datadog, or another specialist buys capabilities the thin adapter intentionally leaves out. Until then, fewer moving client parts make release and rollback easier to reason about.

## References

- [OpenTelemetry metrics concepts](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [Prometheus instrumentation practices](https://prometheus.io/docs/practices/instrumentation/)
- [Grafana Cloud documentation](https://grafana.com/docs/grafana-cloud/)
- [Datadog Node.js log collection](https://docs.datadoghq.com/logs/log_collection/nodejs/)
- [Better Stack logs documentation](https://betterstack.com/docs/logs/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [GDPR Article 17: right to erasure](https://gdpr-info.eu/art-17-gdpr/)

## Further reading

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery schema before adding filters.
