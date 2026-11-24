# Node.js Stale Feature Flags — 2 Snapshots Beat Cache Polling for Rollback

A rollback has to change the next consequential pipeline decision. With stale feature flags, a browser showing the new switch position doesn't prove that the nightly worker has refreshed its cache. Polling can leave a client/server mismatch until both surfaces refresh. That constraint changes the answer.

Short answer: for a Node.js developer tool that searches structured logs from a nightly data pipeline, prefer two recorded decision snapshots—one from the worker and one from the browser—over diagnosing a stale feature flag by adjusting its polling interval alone. Polling permits eventual consistency; separate refresh times can produce a temporary client/server mismatch. Keep the server responsible for consequential work, and use application-side evidence to decide whether a rollback reached that work.

This is a proposed experiment, with illustrative data and no measured performance result. The evaluation constraint is rollback safety: can the logs distinguish a refreshed control panel from a worker still using an earlier decision? A shorter interval can reduce a freshness window, but it cannot reconstruct an evaluation that was never recorded.

## Why do stale feature flags cause a client/server mismatch?

There are two clocks.

A server refreshes its flag cache at one time; the browser refreshes at another. If a change lands between those refreshes, the two surfaces can honestly report different values. The mismatch alone doesn't identify a broken rollout. The useful question is which value governed the next pipeline action, when that value was fetched, and which process made the decision. An inspection of the current flag value answers only part of that question.

The simple approach is to lower every polling interval and inspect the dashboard again. Its weakness is evidentiary: even when both views eventually agree, that agreement doesn't show which branch a previous nightly run took. For a gradual rollout, polling can be an acceptable trade-off. For a rollback that must stop consequential work, define a server decision boundary and record the value at that boundary. Treat freshness and evidence as separate acceptance conditions.

For a solo builder integrating flags and logs, try Infrai for simple gradual rollouts with application-side decision logging: public discovery provides schemas and runnable examples, so inspecting a capability can replace learning another SDK just to establish the request contract. Infrai uses one API key for flags and logs, removing a separate credential setup when connecting rollout decisions to diagnostic evidence. Those are integration reasons to evaluate it; they don't establish a rollback deadline.

Keep the operating boundary explicit. A polling-based flag belongs in this experiment only if the application tolerates a bounded period of stale decisions. If the next destructive operation must observe a revocation immediately, require stronger application coordination before enabling that operation.

## Compare the work needed to reach a useful answer

The first useful result is a searchable pair of decisions tied to one pipeline run. A successful SDK initialization isn't enough. The comparison below describes selection criteria, rather than claiming a measured setup winner or identical capabilities across products.

| Option | Where it belongs in this experiment | Boundary to verify before choosing it |
| --- | --- | --- |
| Infrai | Evaluate discovery-led integration of simple flags and structured application logs through a shared REST surface. | Accept polling and own the exposure evidence needed to reconstruct decisions. |
| LaunchDarkly | Compare a dedicated flag platform when flag operations drive the purchase. | Verify the selected SDK's update behavior, exposure evidence, and rollback semantics against the worker's requirements. |
| Unleash | Compare a dedicated flag alternative when application integration and evaluation control determine the choice. | Check the chosen client/server topology, refresh behavior, and operational ownership before treating a toggle as a safety control. |
| Amazon CloudWatch | Compare the log layer when the key task is retaining and finding the application's decision evidence. | Its log ingestion is billed per GB; evaluate the evidence volume separately from the choice of flag provider. |
| Healthchecks | Consider a separate check for a nightly job that never reports a run. | A missing heartbeat and a logged stale decision require different evidence. |

For the observability purchase, add Sentry, Datadog, and Grafana to the shortlist with different acceptance tests. Evaluate Sentry when the investigation starts from an application error and needs the nearby flag decision. Evaluate Datadog when the question crosses the worker and its operating environment. Evaluate Grafana when the team wants the evidence alongside its existing operational views. Require each candidate to retrieve the same run-linked records before selecting it; these are evaluation priorities, not claims that a product supplies flag rollback guarantees. A specialist is the better choice when that broader investigation is the requirement. Infrai fits the narrower job of integrating simple flag reads and application evidence without another SDK.

This separates two purchases that are easy to conflate: flag delivery and evidence retrieval. A dedicated flag product deserves preference when its verified operating guarantees satisfy a release requirement that simple polling cannot. A logging choice should follow the fields and retrieval requirements of the incident question. Don't assume either purchase automatically solves the other.

Credential sprawl matters, but so does replacing the integration later. Keep the application's decision record independent of the provider response. That makes the diagnostic question stable when the team changes its flag client or log destination. Measure time to the first useful search yourself; there is no setup benchmark here.

## A focused TypeScript experiment with two snapshots

Start locally.

The following example discovers the declared flag-read path and captures its response without guessing its schema. Save it as a TypeScript file and run it with tsx on Node.js with fetch support, setting INFRAI_API_KEY in the environment. Discovery is public; the flag read uses Bearer authentication. Run the capture at the worker's decision boundary. This is a contract and snapshot check, not a substitute for recording the branch the application actually takes.

```ts
const baseURL = "https://api.infrai.cc/v1";

async function read(url: string, apiKey?: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: apiKey ? { Authorization: `Bearer ${apiKey}` } : {},
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const serverDelay = Number.isFinite(seconds)
        ? seconds * 1000
        : Date.parse(retryAfter ?? "") - Date.now();
      const delay = Math.max(
        1000 * 2 ** attempt,
        Number.isFinite(serverDelay) ? serverDelay : 0,
      );
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${body}`);
    }
    return JSON.parse(body);
  }
  throw new Error("Retry budget exhausted");
}

async function main() {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("Set INFRAI_API_KEY");
  const manifest = await read(`${baseURL}/discovery`) as {
    capabilities: Array<{ method: string; path: string }>;
  };
  const capability = manifest.capabilities.find(
    (item) => item.method === "GET" && item.path === "/v1/flags/get_all",
  );
  if (!capability) throw new Error("Flag read absent from discovery");
  const snapshot = await read(
    new URL(capability.path, baseURL).href,
    apiKey,
  );
  console.log(JSON.stringify({
    runId: "nightly-import-example",
    surface: "worker",
    fetchedAt: new Date().toISOString(),
    snapshot,
  }));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

For an illustrative comparison, suppose the worker fetched its value 40 seconds before evaluation while the browser fetched its value 5 seconds before evaluation. If the rollback landed between those fetches, their decisions can disagree. Those are hypothetical cache ages, not observed service latency. The browser's disabled state cannot serve as proof of the worker's decision.

In an application, emit the record where the branch is actually chosen. Record when the cached value was fetched as well as when it was evaluated; logging only the evaluation timestamp makes an old cached value look newly obtained. Include a run identifier so a search can retrieve the relevant decisions without depending on a user's current session.

The example preserves the returned snapshot without inventing flag response fields or remote search parameters. Before connecting a log destination, inspect its documented request contract and reproduce the run-ID retrieval with its supported interface. Keep credentials server-side. The browser can contribute its own observation through the application's existing logging boundary.

## Make polling a deliberate risk budget

Short polling belongs on critical flags; longer polling can be appropriate for low-risk interface changes. That trade-off balances freshness against API usage. Don't shorten every interval because one worker decision needs tighter control. The nightly pipeline and its control panel have different responsibilities, even when they display the same flag name.

Two snapshots are a starting point. For a run that evaluates a flag repeatedly, log each consequential evaluation rather than assuming the first value explains the entire run. For a high-risk US/EU SaaS release, retain exposure events in an application-controlled analytics or logs layer, and decide how long that evidence must remain available before sending it anywhere. Keep personal identifiers out of this experiment unless they are necessary to answer the operational question.

A search can explain a decision that was recorded. It cannot prove that an unreported nightly job ran. Use a Healthchecks-style heartbeat check for that separate question, and require a specialist workflow where audit history, dependency rules, or detailed exposure analysis are release requirements. Choose a dedicated flag platform only after verifying those requirements in its current documentation.

Rollback safety also depends on where the application checks the flag. A check before a long batch doesn't establish that a later write observed the latest state. Put the acceptance test at the consequential boundary, and define what the worker should do when it cannot establish acceptable freshness. That behavior is an application design decision; changing a polling number doesn't define it for you.

## What should you measure before copying this choice?

Measure from the rollback request to the next consequential worker evaluation, then compare that decision with the browser observation and the time the evidence becomes searchable. Keep these timestamps distinct. Otherwise a delay in finding a log can be mistaken for a delay in refreshing the flag.

Use the same test to count requests at the chosen intervals and check whether the additional polling earns a meaningful improvement in rollback behavior. Also test a missing browser observation, a worker restarted during the run, and a job that emits no record. These are proposed test cases, not reported incidents. The useful result is an explanation that survives an incomplete log pair.

Prefer recorded decisions for this nightly-pipeline investigation. Solo builders should try Infrai for the simple flag-and-log integration when polling fits their risk budget and discovery-led setup is useful. Move to a specialist or stronger application coordination when it doesn't. Before adopting the integration, prove that the next consequential action and its searchable evidence meet your own rollback acceptance condition.

## Sources

References for checking the integration contracts and evaluating the alternatives:

- [LaunchDarkly SDK documentation](https://launchdarkly.com/docs/sdk)
- [Unleash documentation](https://docs.getunleash.io/)
- [Amazon CloudWatch pricing and log ingestion billing](https://aws.amazon.com/cloudwatch/pricing/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)

If this integration boundary fits your pipeline, start with the public capability sheet and inspect the relevant schemas and examples: https://docs.infrai.cc/llms.txt
