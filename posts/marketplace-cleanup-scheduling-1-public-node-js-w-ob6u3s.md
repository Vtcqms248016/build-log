# Marketplace Cleanup Scheduling — 1 Public Node.js Webhook Producing Daily Email Reports

Short answer: use a hosted cron service to `POST` one authenticated public Express endpoint, have that endpoint create a uniquely keyed cleanup job and return `202`, then let a worker perform the marketplace cleanup and send the daily report email. Choose the scheduler for retry controls and delivery visibility, but put idempotency in your application. A scheduler can repeat a request; it cannot decide whether yesterday's report was already sent.

The data flow is short: scheduler to webhook, webhook to durable job store, job store to worker, and worker to cleanup plus email. This keeps a slow cleanup out of the web request, gives each retry a stable identity, and leaves one place to inspect every daily run. It also avoids tying correctness to the scheduler's dashboard or to a particular Node.js process staying alive overnight.

That is the decision rule.

## How can a cron service reach a public Node.js Express webhook endpoint?

Schedule one daily `POST`, use UTC in the schedule, and make the intended business date explicit in the JSON body. The endpoint should authenticate before parsing work, reject a malformed date, derive the same idempotency key for every attempt, and enqueue rather than execute. Don't use the request arrival timestamp as the key: a retry at `00:01` and the original attempt at `23:59` could otherwise become different jobs even though they represent the same marketplace reporting period.

The easiest setup is not the service with the fewest form fields. It is the setup whose failure behavior can be explained in one sentence: “the scheduler may deliver more than once, and duplicate deliveries converge on one stored job.” A public endpoint still needs a long random secret, TLS termination, narrow request limits, and logs that omit credentials. Rotate the secret as an operational credential, not as a value embedded in a scheduler URL.

Follow one failed delivery through the system before comparing scheduler controls. The first request creates `marketplace-cleanup:YYYY-MM-DD`; a repeated request observes that row and still receives `202`. A worker leases the row, applies the cleanup, and submits the corresponding report. If the worker stops, the lease expires and another worker takes the same identity. This timeline exposes the state the application must own before a polished scheduling screen can distract from it.

## Code the acceptance API

This focused TypeScript example keeps the storage and email adapters generic. `enqueueUnique` must be backed by a database uniqueness constraint on `id`; an in-memory `Set` is not enough because process restarts and multiple replicas are normal deployment events. The worker contract includes a lease so a claimed job can become eligible again if its worker disappears.

```ts
import express from "express";

type CleanupJob = {
  id: string;
  runDate: string;
  attempt: number;
};

interface JobStore {
  enqueueUnique(job: { id: string; runDate: string }): Promise<"created" | "exists">;
  claim(now: Date, leaseMs: number): Promise<CleanupJob | null>;
  complete(id: string): Promise<void>;
  retry(id: string, nextAttemptAt: Date, reason: string): Promise<void>;
  deadLetter(id: string, reason: string): Promise<void>;
}

interface Marketplace {
  removeExpiredListings(runDate: string): Promise<{ removed: number }>;
}

interface Mailer {
  sendDailyReport(input: {
    runDate: string;
    removed: number;
    idempotencyKey: string;
  }): Promise<void>;
}

declare const jobs: JobStore;
declare const marketplace: Marketplace;
declare const mailer: Mailer;

const app = express();
app.use(express.json({ limit: "4kb" }));

app.post("/hooks/marketplace-cleanup", async (req, res) => {
  const expected = `Bearer ${process.env.CRON_WEBHOOK_SECRET ?? ""}`;
  if (!process.env.CRON_WEBHOOK_SECRET || req.header("authorization") !== expected) {
    res.sendStatus(401);
    return;
  }

  const runDate = req.body?.runDate;
  if (typeof runDate !== "string" || !/^\d{4}-\d{2}-\d{2}$/.test(runDate)) {
    res.status(400).json({ error: "runDate must use YYYY-MM-DD" });
    return;
  }

  const id = `marketplace-cleanup:${runDate}`;
  const result = await jobs.enqueueUnique({ id, runDate });
  res.status(202).json({ id, accepted: true, duplicate: result === "exists" });
});

const maxAttempts = 6;

async function workOnce(): Promise<void> {
  const job = await jobs.claim(new Date(), 60_000);
  if (!job) return;

  try {
    const summary = await marketplace.removeExpiredListings(job.runDate);
    await mailer.sendDailyReport({
      runDate: job.runDate,
      removed: summary.removed,
      idempotencyKey: `${job.id}:email`,
    });
    await jobs.complete(job.id);
  } catch (error) {
    const reason = error instanceof Error ? error.message : "unknown failure";
    const nextAttempt = job.attempt + 1;
    if (nextAttempt >= maxAttempts) {
      await jobs.deadLetter(job.id, reason);
      return;
    }

    const delayMs = Math.min(30_000 * 2 ** job.attempt, 30 * 60_000);
    await jobs.retry(job.id, new Date(Date.now() + delayMs), reason);
  }
}

setInterval(() => void workOnce(), 1_000);
app.listen(3000);
```

The HTTP handler is intentionally boring. It doesn't delete listings, render a template, or call the mail gateway. Those operations may take seconds or minutes, and holding the scheduler request open couples its timeout to every downstream dependency. Returning `202` means “accepted for processing” in this application; it does not mean the cleanup or email succeeded. I don't treat that response as proof of completion.

## Retry budgets for cleanup and mail

There are three distinct retry loops: the cron service can repeat the webhook call, the worker can retry a claimed cleanup job, and the email provider may throttle or retry delivery. Combining them into one counter hides where progress stopped. Keep separate timestamps and attempt counts for webhook receipt, job execution, and email submission. When an email API answers with `429 Too Many Requests`, its `Retry-After` response may tell the client how long to wait; honor that signal rather than immediately adding traffic. The MDN reference describes both the status and the optional header.

The database key handles duplicate webhook delivery. Cleanup operations need their own idempotent shape as well: delete or archive records selected by a stable cutoff, and make repeating that transition harmless. The email boundary is harder. If the gateway accepts an idempotency key, pass `${job.id}:email` on every attempt. If it does not, a crash after the provider accepts the message but before `complete` commits can produce a duplicate report. No local transaction can atomically cover an unrelated email system. The honest choices are to accept rare duplicate mail, select a gateway with deduplication semantics, or add a reconciliation step that checks an attributable provider receipt before resending.

This edge matters.

Dead-lettering belongs after a finite retry budget, not after an arbitrary wall-clock timeout hidden in the route. AWS documents a dead-letter queue as a place for messages that were not processed successfully and warns that retention settings affect how long failed messages remain available. The same idea applies even if the backing store is a relational table: preserve the payload, final error class, attempt count, and first and last attempt times so an operator can decide whether replay is safe.

## Measure the run ledger before choosing a scheduler

Use the scheduling layer for time calculation, authenticated HTTP delivery, bounded retry policy, and delivery history. Keep business-day selection, deduplication, cleanup state, email state, and replay authorization in the application. That division makes the public webhook replaceable. It also prevents a dashboard's green “delivered” marker from being confused with a sent marketplace report.

| Concern | Scheduler | Application |
| --- | --- | --- |
| Daily trigger time | Owns | Records expected UTC date |
| HTTP delivery retry | Owns | Deduplicates every attempt |
| Cleanup retry | Observes only | Owns attempts, lease, and cutoff |
| Email submission | Does not own | Owns idempotency key and receipt |
| Final failure | Exposes delivery log | Owns dead-letter state and replay |

Evaluate a cron service with a disposable endpoint before wiring production. Verify what counts as a successful delivery, which status codes cause retry, whether `Retry-After` is honored, how many attempts occur, how long logs remain visible, how secrets are stored, and whether a missed schedule can be replayed with the original body. I'm not sure any vendor's defaults match a particular recovery objective until those cases are exercised; the answer is in observed request timestamps and documented policy, not a “reliable” label.

Cost still matters for a solo operation, but count the whole path: scheduler executions, database writes, worker runtime, email calls, log retention, and the time needed to inspect a failed run. A tiny scheduler bill does not rescue a design that wakes a full application continuously or makes replay unsafe. Conversely, a queue with elaborate orchestration can be excessive for one daily job. Start with one durable jobs table and one worker when that meets the recovery target.

## Rollout across private and public ingress

The catch is that a public webhook plus a polling worker creates two deployable paths and an internet-facing authentication boundary. It is not suitable when policy forbids inbound public traffic, when cleanup must run inside an isolated network, or when the platform already provides a durable internal scheduler connected to the same queue. In those cases, keep the trigger private and preserve the same job key, lease, retry, and dead-letter rules.

Stick with an operating-system cron entry only when one machine is deliberately the scheduler, its persistence and clock behavior are owned, and missed executions have an explicit recovery procedure. An in-process timer can be reasonable for disposable local development, but it becomes ambiguous across replicas: each process may believe it owns the daily tick. A workflow engine is the better category when cleanup has many dependent stages, long waits, compensation steps, or operator approvals. More machinery earns its place when it removes real recovery code.

The scheduler is replaceable. The run ledger isn't.

## Retention and replay governance after failure tests

Before deployment, create a synthetic run date and send the identical webhook body twice; the ledger should show one job. Force a worker exception before email submission, allow the lease to expire, and confirm the same job is reclaimed with a higher attempt count. Then simulate a `429` response with a `Retry-After` value and check that the next attempt waits. Finally, exhaust all six attempts, inspect the dead-letter record, and replay only after confirming the cleanup transition and email send are safe to repeat.

In production, log the job ID on every transition and track four times: scheduled, accepted, cleanup completed, and email accepted. Alert on overdue completion rather than on one failed attempt because retries are expected behavior. Keep the webhook response small, never log its authorization header, and expose job status through an authenticated internal view instead of adding detail to the public route. Review dead-letter retention and the replay procedure during ordinary maintenance; discovering that failed payloads expired during an incident is too late.

The operational checklist is therefore a story of one date moving through explicit states. At the expected UTC time, a signed request arrives. One durable row appears. A leased worker applies a repeatable cleanup, submits one attributable report, and records completion. Temporary pressure backs off; terminal failures remain inspectable. Once that sequence is visible in logs and reproducible in a test environment, the cron service itself becomes the least interesting component—which is exactly where scheduling infrastructure should land.

## References

- AWS SQS dead-letter queues documentation: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- MDN, HTTP 429 Too Many Requests: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429
