# Structured LLM Extraction Retries: Idempotency for Duplicate Webhook Records

Short answer: treat every webhook and model response as replayable input, assign identity before extraction, and make the database commit the one durable decision about a record. A retry should revisit the same work item, never create a new identity.

The useful mental model is a small state machine: receive, claim, extract, validate, commit, acknowledge. The sender may repeat receive, the queue may repeat claim, and a timeout may repeat extract. Only commit needs an idempotency guarantee. A Node.js worker can therefore stay simple if its boundary is explicit: persist a deterministic job key, keep raw input, validate JSON, and acknowledge the webhook only after the durable write succeeds. It's a little dull. That's the point.

## How should a Node.js webhook worker handle LLM JSON retries?

Start with the event, not the generated object. A webhook delivery id is a good candidate when the sender promises it is stable; otherwise combine a source id with a hash of the exact body bytes. Add a schema or prompt revision when reprocessing under a new contract must create a separate result. Do not hash the model output: field order, whitespace, truncation, and model revisions can change it while the source event remains the same.

The worker should record a `received` row before expensive work. A second delivery then finds the same key. It can safely return the already committed result, resume a pending row, or leave the row for a queue retry. This is an application-level interpretation of idempotency; RFC 9110 defines the network property, while the database constraint supplies the stronger local invariant.

That is the invariant.

```ts
import { createHash } from "node:crypto";
import type { PoolClient } from "pg";

type Extracted = { customerId: string; amountCents: number };

function jobKey(eventId: string, rawBody: Buffer, schemaVersion: string): string {
  const bodyDigest = createHash("sha256").update(rawBody).digest("hex");
  return `${eventId}:${schemaVersion}:${bodyDigest.slice(0, 24)}`;
}

export async function processWebhook(
  db: PoolClient,
  eventId: string,
  rawBody: Buffer,
  schemaVersion: string,
): Promise<{ key: string; status: "done" | "claimed" }> {
  const key = jobKey(eventId, rawBody, schemaVersion);

  await db.query(
    `insert into extraction_jobs (job_key, state, raw_body)
     values ($1, 'received', $2)
     on conflict (job_key) do nothing`,
    [key, rawBody],
  );

  const current = await db.query<{ state: string }>(
    "select state from extraction_jobs where job_key = $1",
    [key],
  );
  if (current.rows[0]?.state === "done") return { key, status: "done" };

  const extracted: Extracted = await extractAndValidate(rawBody, schemaVersion);

  await db.query("begin");
  try {
    await db.query(
      `insert into customer_amounts (job_key, customer_id, amount_cents)
       values ($1, $2, $3)
       on conflict (job_key) do nothing`,
      [key, extracted.customerId, extracted.amountCents],
    );
    await db.query(
      "update extraction_jobs set state = 'done', finished_at = now() where job_key = $1",
      [key],
    );
    await db.query("commit");
  } catch (error) {
    await db.query("rollback");
    throw error;
  }

  return { key, status: "claimed" };
}

declare function extractAndValidate(body: Buffer, schemaVersion: string): Promise<Extracted>;
```

The pre-check is an optimization. The unique constraint and the transaction are the safety mechanism. Keep the raw bytes long enough to replay, but redact or encrypt sensitive fields before logs and backups. Parsing is not validation: reject unknown shapes, range violations, and missing required fields before any downstream side effect. OWASP calls this improper output handling, and structured extraction is still untrusted input.

## Where does idempotency stop in a JSON pipeline?

One database can make the record write and the job state atomic. It cannot atomically update a search index, publish a webhook, and charge a card in three other systems. That boundary needs an outbox: insert an event in the same transaction as the record, then let a separate dispatcher deliver it with its own idempotency key. Consumers must deduplicate too; an outbox changes the failure mode from lost intent to repeatable delivery. In a larger pipeline, that means tracing one key across the worker, the outbox row, and each consumer, retaining delivery attempts and response codes, and deciding which side owns a compensating action when a consumer accepts the event but fails before its own commit. The extra bookkeeping is real, yet it gives operators a replayable trail instead of a mystery gap between two green dashboards.

There is a second boundary around acknowledgement. A 2xx response means the receiver accepted the request, not that every consumer has finished. Acknowledge after the durable `received` insert, then let the worker process asynchronously only when the sender's contract permits it. If the sender requires synchronous completion, hold the response open only within its timeout and still make the write replay-safe.

## Which retry policy fits malformed model output?

Separate transport failures from semantic failures. Timeouts, connection resets, and rate limits are candidates for bounded exponential backoff with jitter. Invalid JSON or a schema violation should be captured as a terminal attempt with the raw response, prompt revision, and parser error; blindly repeating the same request burns tokens without changing the input. A small repair attempt can be useful, but it must write under the same job key and pass the same validator.

Do not let a retry loop own the whole pipeline. Set an attempt limit, a lease or visibility timeout longer than the model deadline, and a dead-letter path that preserves enough context for inspection. Metrics should count deliveries, unique job keys, validation failures, committed rows, and outbox lag separately. One ratio is not enough to explain where work disappeared.

I'm not sure there is a clean answer when a retry changes the meaning of a result, such as a partial extraction becoming complete. Version the prompt and schema, then choose explicitly between a new record and an update; your mileage may vary if consumers cannot tolerate two versions of one fact.

## What should be tested before shipping duplicate-safe extraction?

Test the crash windows deliberately: die after the model call, after the record insert, and before the acknowledgement. Send the same webhook twice concurrently and assert one committed row and one completed job. Replay a corrected source body and assert a new key. Replay the old body with a new schema revision and assert the intended versioning behavior. These tests exercise the invariant instead of asserting that a happy-path response looks plausible.

Keep the operational checklist in the code review: deterministic identity from source bytes, a database uniqueness rule, one transaction for result plus state, schema validation before side effects, bounded retries, and an outbox for external delivery. The catch is that this design is not suitable when a downstream system cannot accept an idempotency key or when legal requirements demand destructive, irreversible processing on every delivery; in those cases, use a domain-specific reconciliation process and an explicit human review queue.

Small rules. Big payoff.

## References

- OWASP Top 10 for Large Language Model Applications — https://owasp.org/www-project-top-10-for-large-language-model-applications/
- Prompt Engineering Guide — https://www.promptingguide.ai
- RFC 9110: HTTP Semantics, Idempotent Methods — https://www.rfc-editor.org/rfc/rfc9110#name-idempotent-methods
- PostgreSQL: INSERT ... ON CONFLICT — https://www.postgresql.org/docs/current/sql-insert.html
- MDN Web Docs: Idempotent — https://developer.mozilla.org/en-US/docs/Glossary/Idempotent
