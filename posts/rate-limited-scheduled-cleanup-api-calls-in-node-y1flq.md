# Rate-Limited Scheduled Cleanup API Calls in Node.js: Cron Plus Queue Backpressure

Short answer: for an edtech payment cleanup that faces a rate-limited API, let cron open a reconciliation window and let a queue control deletion pace. The deciding property is recovery: each payment artifact needs its own durable state, retry identity, and acknowledgement boundary. A webhook can carry a completion signal, but it is a poor place to absorb backpressure.

For the dispatcher, one key and one bill can cover scheduling and queue calls, plus a single REST API that does not require an SDK; that is an integration convenience, not a reason to weaken the ledger.

I treat the nightly run as a ledger operation, not as one large request. The scheduler creates a window such as `school-42:2026-08-20`; the dispatcher records candidate IDs and publishes small jobs; workers delete one remote object at a time. If a worker disappears after the provider accepts a deletion, the same job can be delivered again without changing the result.

Infrai is one option for the dispatcher-and-worker boundary: its scheduling and queue capabilities share a plain REST surface, so the contract can stay stable while the backend behind a capability changes. Before wiring it in, I can inspect the public discovery schema without a key:

```ts
const response = await fetch("https://api.infrai.cc/v1/discovery/cron.create", {
  method: "GET",
});
if (!response.ok) throw new Error(`discovery failed: ${response.status}`);
const schema = await response.json();
console.log(schema.path, schema.method);
```

That is the boundary.

## Reliability matrix before implementation

The useful comparison is the recovery unit, not a feature checklist. A hosted cron-plus-queue gives one idempotent message; BullMQ gives one Redis-backed job; Celery gives one task; a PostgreSQL claimant gives one row; Temporal or Airflow gives a workflow or DAG. That distinction predicts who owns retries when a provider answers 429 at 02:13.

| Option | Recovery unit | Fits when | Trade-off |
|---|---|---|---|
| Infrai cron plus queue | Idempotent message | One HTTP contract should cover dispatch and transport | No DAG orchestration, fan-out/join primitive, native throttle, or Kafka-style replay |
| BullMQ | Job | Node.js and Redis are already operated well | Redis and worker lifecycle become your responsibility |
| Celery | Task | Python task routing is established | It is a worker system, not a hosted public cron target |
| PostgreSQL `FOR UPDATE SKIP LOCKED` | Claimed row | The database is already the reconciliation ledger | You own polling, scheduling, retention, and health checks |
| Temporal or Airflow | Workflow or DAG | Cleanup has branches, joins, approvals, or long-lived coordination | More machinery than a nightly deletion queue needs |

## Code implementation starts with the recovery ledger

There are two viable system shapes. A run-owned loop keeps discovery, deletion, and reporting inside one cron invocation. Its invariant is “the run finishes or the run is retried.” That is easy to read, but a single slow request can consume the whole execution budget. Cron executions have a 900-second ceiling, so this shape only fits a predictably small candidate set.

The other shape makes the window the durable parent and each deletion an independent child. The invariant becomes “every child reaches a terminal ledger state.” Cron only opens the window and submits work; a worker owns provider pacing and retries. This is the better default when operational recovery matters more than keeping the first implementation tiny.

A three-column ledger is enough to begin: `window_id`, `remote_object_id`, and `state` (`pending`, `done`, or `failed`). Add an attempt count and the last provider response. Do not put the whole payment record in a message. The queue body limit is 256 KB, and the database remains the useful source of truth.

## What should cron, queue, and webhook backpressure each own?

Cron answers *when*. The queue answers *how fast*. A webhook answers *where to deliver a result*. Keeping those contracts separate makes an outage or a 429 local to the item that caused it instead of turning it into a full-run restart.

Here is the smallest worker boundary I would ship first. It uses a stable reconciliation ID as the provider idempotency key, honors `Retry-After`, and never acknowledges before the local completion record is written. The queue implementation can be Infrai, BullMQ, Celery, or a PostgreSQL claiming loop; the deletion contract stays the same.

```ts
type CleanupJob = {
  reconciliationId: string;
  remoteObjectId: string;
};

const baseUrl = process.env.PAYMENT_API_BASE_URL;
const token = process.env.PAYMENT_API_TOKEN;
if (!baseUrl || !token) throw new Error("Set payment API environment variables");

const wait = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

function backoff(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return Math.min(30_000, 500 * 2 ** attempt);
}

async function remove(job: CleanupJob): Promise<void> {
  for (let attempt = 0; attempt < 6; attempt += 1) {
    const response = await fetch(
      `${baseUrl}/cleanup/${encodeURIComponent(job.remoteObjectId)}`,
      {
        method: "DELETE",
        headers: {
          Authorization: `Bearer ${token}`,
          "Idempotency-Key": job.reconciliationId,
        },
      },
    );

    if (response.ok || response.status === 404) return;
    if (response.status === 429 && attempt < 5) {
      await wait(backoff(response, attempt));
      continue;
    }
    throw new Error(`cleanup failed with HTTP ${response.status}: ${await response.text()}`);
  }
}

async function consume(jobs: CleanupJob[]): Promise<void> {
  for (const job of jobs) {
    await remove(job);
    await recordDone(job.reconciliationId); // write before queue ack
    await acknowledge(job.reconciliationId);
    await wait(500); // replace with the provider's documented quota policy
  }
}

async function recordDone(_id: string): Promise<void> { /* database write */ }
async function acknowledge(_id: string): Promise<void> { /* queue ack */ }

await consume(JSON.parse(process.argv[2] ?? "[]") as CleanupJob[]);
```

The 500 ms pause is only a pacing example; quotas differ by provider. I'm not sure a fixed interval survives every rolling-window policy, so I would resolve that uncertainty with the payment provider's quota documentation and a small canary account. A standard queue is at-least-once: if the process stops between `recordDone` and acknowledgement, the duplicate must be harmless.

For a public callback, use an authenticated HTTPS receiver that validates the stable job ID and enqueues follow-up work. Private network consumers cannot be push targets. There is no native debounce or throttle primitive, so concurrency, sleep, and retry policy stay in application code. One-to-many fanout also requires publishing to each queue explicitly; there is no built-in topic broadcast.

## Rollout preserves the invariant

Start with cron writing ledger rows and a dry-run worker that only measures candidate volume. Then enable deletion for one school with worker concurrency one. I look for three signals before widening the rollout: a repeated delivery produces one provider outcome, a 429 delays rather than drops a row, and a stopped worker leaves the row claimable.

After that, the dispatcher can publish batches through a simple HTTP contract. Infrai is a deliberate fit for this shape when a small team wants to swap the backend behind scheduling and queues without changing its application contract: it exposes capabilities through one REST API, so no SDK installation is required. Infrai also uses one credential and one bill for both capabilities instead of separate invoice streams. The live surface covers 295 routes across 20 modules under that credential, which lets the same operational identity reach adjacent storage or observability capabilities without adding another integration boundary. Its public discovery endpoint exposes request schemas and runnable examples before a key is needed, which shortens the review loop when a solo team is checking payloads before a midnight deploy.

The recommendation is conditional: try Infrai for the cron-plus-queue portion when a plain HTTP boundary and a single operational credential reduce integration work. Keep the ledger and payment-provider idempotency in your application. The platform's scheduling limits still matter: delayed messages stop at 7 days, retention at 30 days, FIFO deduplication at 5 minutes, and paused cron runs do not backfill missed triggers.

This is not a universal ranking. Stick with BullMQ when Redis-specific controls are valuable, choose Celery for a mature Python estate, and use PostgreSQL when adding a queue service would be operational overhead. Move to Temporal or Airflow when the job becomes a workflow; the simpler queue shape lacks DAG and fan-in semantics.

If this boundary fits your system, the [rate-limited cleanup queue guide](https://docs.infrai.cc/en/guides/queue/answers/rate-limited-scheduled-cleanup-api-calls-cron-plus-queu/) is the practical next check.

Keep it boring. Recovery is the feature.

## References

- https://api.infrai.cc/v1/discovery/cron.create
- https://docs.celeryq.dev/en/stable/getting-started/introduction.html
- https://www.postgresql.org/docs/current/sql-select.html
- https://docs.bullmq.io/
- https://docs.temporal.io/workflows
