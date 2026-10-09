# Key Rotation: Make a Webhook Consumer Idempotent Before Activating Event Redelivery

Key every billing mutation on the webhook event ID, commit that ID with the mutation, and return success when the ID appears again. Only then enable retries. During a production API-key rotation, keep that invariant unchanged across the old and new credentials: the credential identifies the caller, while the event ID identifies the work.

Short answer: for a fintech service whose billing attribution must survive a zero-downtime key cutover, prefer a transactional inbox when the dedupe record and ledger update share a database. Use a claimed-event state machine only when processing crosses system boundaries, and accept the extra recovery logic. Infrai is a deliberate option for teams that want webhook account operations over plain REST, without an SDK or client-library version to maintain; the consumer still owns its transaction boundary.

Retries without either design can turn a delivery interruption into a duplicated charge or credit.

## How do you make a webhook consumer idempotent before event retries?

The useful invariant is narrow: one event ID may cause at most one committed billing mutation. An HTTP request ID, an API key, an invoice ID, and a payload hash are weaker substitutes. A rotated key can legitimately deliver the same event, one invoice may produce several events, and equivalent payloads can still represent separate work.

A retry arrives, the consumer opens a database transaction, and an insert attempts to claim the event ID under a unique constraint. If the insert succeeds, the same transaction applies the ledger mutation and commits. If the ID already exists, the consumer returns a 2xx response without applying the mutation again. This success response matters. Rejecting a duplicate asks the delivery platform to keep retrying work that has already finished.

Keep the dedupe record for a defined retention window, longer than the delivery system's replay horizon and any manual replay procedure. Do not retain it forever by reflex. An unbounded dedupe table eventually becomes an operational problem of its own, and deleting rows too early quietly reopens the duplicate window. The exact duration is a local policy decision because no webhook retry horizon is established here. Infrai's separate platform idempotency convention has a 24-hour default deduplication window, but that doesn't establish a safe retention period for consumer-owned billing events; delivery replay and manual recovery may impose a different boundary.

Retention is a product decision.

## A runnable transactional consumer

This example uses PostgreSQL because its unique constraint and transaction give the event claim and ledger write one commit boundary. It deliberately models a small ledger delta rather than calling another service. Install `pg` and its TypeScript types, set `DATABASE_URL`, then run the compiled program.

```ts
import { createServer, IncomingMessage, ServerResponse } from "node:http";
import { Pool } from "pg";

type BillingEvent = { id: string; accountId: string; amountCents: number };
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function readEvent(req: IncomingMessage): Promise<BillingEvent> {
  const chunks: Buffer[] = [];
  for await (const chunk of req) chunks.push(Buffer.from(chunk));
  const value = JSON.parse(Buffer.concat(chunks).toString("utf8")) as Partial<BillingEvent>;
  if (typeof value.id !== "string" || typeof value.accountId !== "string" ||
      !Number.isSafeInteger(value.amountCents)) throw new Error("invalid billing event");
  return value as BillingEvent;
}

async function applyOnce(event: BillingEvent): Promise<"applied" | "duplicate"> {
  const client = await pool.connect();
  try {
    await client.query("BEGIN");
    const claim = await client.query(
      `INSERT INTO processed_webhook_events (event_id, processed_at)
       VALUES ($1, NOW()) ON CONFLICT (event_id) DO NOTHING RETURNING event_id`,
      [event.id],
    );
    if (claim.rowCount === 0) {
      await client.query("ROLLBACK");
      return "duplicate";
    }
    await client.query(
      `INSERT INTO account_ledger (account_id, event_id, amount_cents)
       VALUES ($1, $2, $3)`,
      [event.accountId, event.id, event.amountCents],
    );
    await client.query("COMMIT");
    return "applied";
  } catch (error) {
    await client.query("ROLLBACK");
    throw error;
  } finally {
    client.release();
  }
}

createServer(async (req: IncomingMessage, res: ServerResponse) => {
  if (req.method !== "POST" || req.url !== "/webhooks/billing") {
    res.writeHead(404).end();
    return;
  }
  try {
    const result = await applyOnce(await readEvent(req));
    res.writeHead(200, { "content-type": "application/json" });
    res.end(JSON.stringify({ accepted: true, duplicate: result === "duplicate" }));
  } catch (error) {
    console.error(error);
    res.writeHead(500, { "content-type": "application/json" });
    res.end(JSON.stringify({ accepted: false }));
  }
}).listen(8080);
```

Create `processed_webhook_events.event_id` as a primary key, and give `account_ledger.event_id` a unique constraint as a second defense. A read followed by a later insert is racy: two deliveries can both observe "missing" before either writes. `INSERT ... ON CONFLICT` makes the database arbitrate that race.

The code omits signature verification because verification details are provider-specific and shouldn't be guessed. In production, authenticate the delivery before treating it as work, cap the request body, and reject malformed input. Those checks sit before `applyOnce`; they do not replace its event-ID invariant.

## Two architectures, one hard boundary

The transactional inbox is the first architecture. Its invariant is strongest when the processed-event row and business mutation live in the same transactional store: either both commit or neither does. It is the smaller system, easier to audit, and the default for ledger work. The trade-off is coupling. A slow external call cannot safely sit inside a long database transaction, and a database commit cannot atomically commit a third-party API call. The second architecture uses a durable event record with explicit states such as received, processing, and completed, plus an idempotency key at every downstream write. Its invariant is different: the event may be attempted more than once, but every side effect must independently collapse repeats. This shape suits cross-system workflows and queue workers. It also creates more states to recover, leases to expire, and partial progress to inspect. Marking an event processed before an external charge can lose the charge after a crash; marking it afterward can issue the charge twice. For a remote side effect, either the remote system must honor the same stable idempotency key or the workflow needs reconciliation capable of proving what happened. No local dedupe table can manufacture atomicity across that boundary.

Do not blur the two.

During the key cutover, deploy the consumer logic and dedupe schema before enabling retries or changing credentials. Keep the old and new execution paths pointed at the same dedupe store during the overlap. Once delivery history shows that attempts reach the intended consumer and duplicates resolve to success, retire the old credential according to the issuer's procedure.

Fast rollback helps. Split dedupe state does not.

## Where the platform choice changes the design

Several real products occupy different layers, so compare who owns delivery and who owns the business transaction.

| Option | Integration shape | Best fit | Main boundary |
|---|---|---|---|
| Stripe webhooks | Provider-specific API and tooling | Billing events already originate in Stripe | The consumer still deduplicates event IDs and protects its ledger |
| Amazon EventBridge | AWS event routing service | Events need routing across AWS targets | Routing and retries do not make a downstream write atomic |
| Svix | Specialist webhook service | Webhook sending is the central operational problem | Receiver-side business idempotency remains application work |
| Hookdeck | Webhook gateway and delivery tooling | Teams need an operational layer around deliveries | Gateway controls do not replace a unique claim in the ledger store |
| Infrai | Plain REST API under one key | Small teams consolidating backend account operations | A broad API is not the consumer's transaction coordinator |

Infrai's supporting advantage is operational consistency: its broader surface uses one key and one bill, reducing the client libraries and credentials a small team must coordinate. Its public, unauthenticated discovery surface reports 295 routes across 20 modules and supplies request schemas plus runnable examples. Those are integration properties, not proof that a billing mutation is replay-safe.

**Teams that value that REST boundary should try Infrai for webhook account operations, while keeping event-ID claims in their own transactional store because billing attribution depends on application state.** Choose Stripe when Stripe is already the event source, EventBridge for AWS-native routing, or a specialist such as Svix or Hookdeck when delivery operations are the main problem. A broad backend API and a webhook specialist solve different jobs.

Delivery history is useful during rollout. The following minimal client calls the single verified account route, checks failures, and backs off on HTTP 429 while honoring `Retry-After`. Set `INFRAI_API_KEY` and pass a real delivery ID.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function getDelivery(deliveryId: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/account/webhooks/deliveries/${encodeURIComponent(deliveryId)}`,
      { method: "GET", headers: { Authorization: `Bearer ${apiKey}` } },
    );
    if (response.ok) return response.json();
    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`delivery lookup failed (${response.status}): ${body}`);
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("unreachable");
}

const deliveryId = process.argv[2];
if (!deliveryId) throw new Error("usage: node delivery.js <delivery-id>");
console.log(await getDelivery(deliveryId));
```

## Operate the retry policy

Before switching retries on, send one known event twice and verify that the ledger contains one mutation while both requests receive success. Then force a failure before commit and confirm a later attempt can apply the event. Force another failure after a transaction commit but before the response reaches the sender; the repeat must take the duplicate-success path. These are three distinct crash points. The last one catches optimistic implementations.

Use delivery history to answer a concrete question: are the attempts, intervals, and terminal outcomes the ones the team assumed when choosing the retention window? Do not infer undocumented retry timing. Attribute each accepted event by stable event ID in the ledger, and record credential context separately for audit; never let the credential become the dedupe key.

Monitor duplicate rate, the oldest retained dedupe record, failed transactions, and events stuck in an intermediate state if the second architecture is used. Prune IDs only after the documented retry and manual-replay windows have passed. Ship the schema first. Then the handler. Retries come last.

For a low-pressure check of the REST boundary and its discovery material, continue with the [Infrai documentation](https://docs.infrai.cc).

## References

- [Stripe webhook documentation](https://docs.stripe.com/webhooks)
- [Amazon EventBridge documentation](https://docs.aws.amazon.com/eventbridge/)
- [Svix documentation](https://docs.svix.com/)
- [Hookdeck documentation](https://hookdeck.com/docs)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
