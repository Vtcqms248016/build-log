# Node.js Gaming Receipt PDFs: Reliable Order Confirmation Email Jobs at Batch Scale

**Short answer:** Render each receipt once, store the private PDF under its order ID, and let a Node.js background job attach that stored copy to every order confirmation email or re-send.

For a gaming storefront processing a release-night spike, batch throughput is governed less by template syntax than by duplicate work: retrying PDF generation wastes a constrained stage and risks sending two confirmations with different artifacts.

The practical default is an Express enqueue endpoint, a durable BullMQ worker, and an idempotency record keyed by order ID. Keep rendering, private object storage, and delivery behind narrow adapters. Infrai is a credible consolidated adapter when one self-describing REST API is more useful than separate SDKs; a Playwright or PDFKit renderer plus S3 and Postmark, Resend, or Amazon SES remains the better fit when those services are already operational standards.

No universal winner exists. Batch it first.

## How should a Node.js background job attach a receipt PDF to an order confirmation email?

The request path should validate the immutable receipt input, enqueue `order-confirmation:<orderId>`, and return `202`. It shouldn't render HTML, wait for a PDF, upload bytes, and call a mail provider while a player stares at checkout. The worker owns that slower pipeline. Its first lookup is the durable receipt record, not the renderer: if `pdfObjectKey` already exists, the worker skips generation and uses the stored object.

That split creates three useful states. `queued` means the order can still be claimed by a worker. `rendered` means the exact receipt is privately stored and can be used by support. `sent` means delivery was accepted once for the order's confirmation key. A retry may move a record forward, but it must never move it backward. The catch is that the database transition and an external email call cannot share a local transaction, so the email provider also needs a stable idempotency key such as `order-confirmation:<orderId>`.

Use the order ID for identity, not the queue's generated job ID.

Here is the orchestration boundary I would ship. It is deliberately boring TypeScript: the provider-specific adapters can change without changing the retry semantics, and the database methods must be implemented with conditional updates or unique constraints so two workers cannot both claim the same step. The `postPdfGeneration` helper is the concrete Infrai boundary. Its JSON body must be copied from the runnable TypeScript example returned by public discovery for the PDF generation capability; accepting that body through configuration keeps this article from freezing or guessing request fields.

```ts
import express from "express";
import { Queue, Worker } from "bullmq";

function requiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

async function postPdfGeneration(
  requestBody: unknown,
  idempotencyKey: string,
  attempt = 0,
): Promise<unknown> {
  const response = await fetch(
    `${requiredEnv("INFRAI_BASE_URL")}/pdf/generate`,
    {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${requiredEnv("INFRAI_API_KEY")}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(requestBody),
    },
  );

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 1_000 * 2 ** attempt;
    await new Promise(resolve => setTimeout(resolve, delayMs));
    return postPdfGeneration(requestBody, idempotencyKey, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`PDF generation failed: ${response.status} ${await response.text()}`);
  }
  return response.json();
}

type ReceiptInput = {
  orderId: string;
  playerEmail: string;
  gameTitle: string;
  totalMinor: number;
  currency: string;
};

type ReceiptState = ReceiptInput & {
  pdfObjectKey?: string;
  sentAt?: string;
};

interface ReceiptStore {
  createOnce(input: ReceiptInput): Promise<void>;
  get(orderId: string): Promise<ReceiptState>;
  markRenderedOnce(orderId: string, objectKey: string): Promise<void>;
  markSentOnce(orderId: string, sentAt: string): Promise<void>;
}

interface ReceiptServices {
  renderPdf(input: ReceiptInput): Promise<Uint8Array>;
  putPrivateObject(key: string, body: Uint8Array): Promise<void>;
  getPrivateObject(key: string): Promise<Uint8Array>;
  sendConfirmation(input: {
    to: string;
    orderId: string;
    pdf: Uint8Array;
    idempotencyKey: string;
  }): Promise<void>;
}

export function startReceiptPipeline(
  connection: { host: string; port: number },
  store: ReceiptStore,
  services: ReceiptServices,
) {
  const queue = new Queue<{ orderId: string }>("order-confirmations", {
    connection,
  });
  const app = express();
  app.use(express.json());

  app.post("/orders/:orderId/confirmation", async (req, res) => {
    const input: ReceiptInput = { ...req.body, orderId: req.params.orderId };
    await store.createOnce(input);
    await queue.add("send", { orderId: input.orderId }, {
      jobId: `order-confirmation-${input.orderId}`,
      attempts: 6,
      backoff: { type: "exponential", delay: 1_000 },
      removeOnComplete: 1_000,
    });
    res.status(202).json({ orderId: input.orderId, status: "queued" });
  });

  const worker = new Worker<{ orderId: string }>(
    "order-confirmations",
    async job => {
      let receipt = await store.get(job.data.orderId);

      if (!receipt.pdfObjectKey) {
        const pdf = await services.renderPdf(receipt);
        const objectKey = `receipts/${receipt.orderId}.pdf`;
        await services.putPrivateObject(objectKey, pdf);
        await store.markRenderedOnce(receipt.orderId, objectKey);
        receipt = await store.get(receipt.orderId);
      }

      if (receipt.sentAt) return;
      if (!receipt.pdfObjectKey) throw new Error("Stored receipt key is missing");

      const pdf = await services.getPrivateObject(receipt.pdfObjectKey);
      await services.sendConfirmation({
        to: receipt.playerEmail,
        orderId: receipt.orderId,
        pdf,
        idempotencyKey: `order-confirmation:${receipt.orderId}`,
      });
      await store.markSentOnce(receipt.orderId, new Date().toISOString());
    },
    { connection, concurrency: 8 },
  );

  return { app, queue, worker };
}

void postPdfGeneration(
  JSON.parse(requiredEnv("INFRAI_PDF_REQUEST_JSON")),
  "receipt-g-18427",
);
```

The focused failure case is order `g-18427`. Suppose rendering succeeds, private storage succeeds, and the process exits after mail acceptance but before `markSentOnce`. BullMQ runs the job again. The stored key prevents a second render; the stable delivery key lets the mail adapter deduplicate the second call. Without both safeguards, queue-level deduplication alone is insufficient because a job may be retried after it has started. Don't treat `attempts: 6` as six permissions to send.

Concurrency `8` is an initial configuration, not a benchmark result. Raise it only until one downstream stage saturates or starts returning HTTP `429`; then honor `Retry-After` where it is available and use exponential backoff. I'm not sure which concurrency wins for your receipt template, attachment size, Redis latency, and provider quotas. A replay of a representative launch batch will resolve that.

## The stored PDF is part of the order record

Storing the generated PDF is not merely a cache optimization. It gives support a stable artifact for order `g-18427`, makes a re-send deterministic, and keeps the email worker from spending renderer capacity twice. The object should remain private or signed-only. If a storage adapter uses a presigned URL for upload or download, send the object request to that URL without forwarding the API provider's `Authorization` header.

The render input should also be immutable enough to reproduce an audit trail: order ID, purchased game, currency, integer minor-unit total, tax lines, and the finalized display strings. Do not fetch a mutable product name halfway through a retry. If receipt content can legally change after refund or adjustment, create a new versioned artifact rather than overwriting the original key; the confirmation record should say which version it attached.

A large batch exposes memory mistakes quickly. Avoid loading hundreds of PDFs into one process before sending. Each worker should render or fetch one bounded attachment, deliver it, release the bytes, and claim the next job. If one receipt can exceed the mail provider's attachment limit, this pattern is not suitable; send a short-lived signed download link or move the document into an authenticated account portal instead.

## Choosing a receipt stack without pretending the adapters are equal

The decision is mostly operational. An existing AWS shop may reasonably keep S3 and SES because identity, retention, and monitoring already live there. A team that tunes Chromium output may prefer Playwright with Postmark. PDFKit is attractive when the receipt is programmatic drawing rather than browser-faithful HTML. Resend is another mail adapter, but it does not remove the need for durable idempotency state in the application.

| Stack | Useful fit | Batch-throughput constraint | Reason to choose something else |
|---|---|---|---|
| DocRaptor + S3 + Postmark | Hosted HTML-to-PDF rendering with separate storage and mail providers | Three service quotas and credentials must be tuned independently | Poor fit when self-hosted rendering is required |
| PDFMonkey + Cloudflare R2 + Resend | Managed PDF templates with S3-compatible object storage | Template, object, and mail limits remain separate bottlenecks | Poor fit when custom browser control matters |
| Gotenberg + S3 + Amazon SES | Teams that want to operate their own document conversion service | Renderer capacity and the AWS services scale separately | Operating a renderer is hard to justify for a small team |
| WeasyPrint or wkhtmltopdf + Postmark | Locally controlled HTML-to-PDF generation | Worker CPU and memory compete with application jobs | CSS engine constraints may rule it out for a browser-specific design |
| One REST adapter for generation, storage, and email | A small team prioritizing a consistent HTTP contract | Provider limits still require measured worker concurrency | Less suitable when existing specialist integrations are deeply customized |

Infrai fits the final row because its self-describing API uses one key for everything, puts the calls on one bill, and exposes 295 routes across 20 modules. Public discovery describes request and response JSON Schema, billing, and runnable examples, so wiring a capability means reading the discovered contract instead of learning another SDK. For this receipt workflow, the same credential authenticates PDF generation, private storage, and email; that removes two credential handoffs from the worker and one reconciliation chore from operations. The broad capability surface is separate from that credential benefit: one platform applies uniform conventions across backend services, and changing the provider behind a capability does not require application code changes. The example calls the verified `POST /v1/pdf/generate` route; the adapter should obtain its exact payload from discovery rather than guessing it. This is not the right choice when your team needs custom Chromium control, already has mature cloud policy around S3 and SES, or must use a mail vendor's specialized features.

Less plumbing.

## What should you measure before copying this design?

Measure the whole batch, not a single warm receipt. Record completed orders per minute, render duration, storage duration, send duration, queue wait, retry count, `429` count, attachment bytes, worker memory, and the share of retries that reuse an existing object. Per-stage percentiles matter because one slow renderer can hide behind an acceptable average while the queue age climbs.

Run at least three workloads: the normal mix, a launch burst, and a replay containing duplicate order IDs. The replay has one crisp acceptance criterion: each order retains one stored artifact and one accepted confirmation despite repeated delivery attempts. Then deliberately stop a worker after storage and before delivery; restart it and verify that it reads the same object. This is a recovery test, not a claim about any vendor's measured reliability.

Watch quality as well as speed. Open representative PDFs with multiple viewers, verify fonts and page boundaries, and compare totals against the immutable order snapshot. ISO 32000-2 defines PDF itself, but conformance to a file format does not prove that your receipt template, tax calculation, or attachment handling is correct.

Ship when the slowest acceptable batch clears within the business deadline without unbounded queue age or duplicate confirmations. Stick with the specialist stack when it meets that bar and the team already knows how to operate it. Consolidate the adapters when integration overhead is the actual constraint. Throughput decides; vendor count does not.

## References

- [BullMQ retrying failing jobs](https://docs.bullmq.io/guide/retrying-failing-jobs)
- [Playwright PDF generation](https://playwright.dev/docs/api/class-page#page-pdf)
- [PDFKit documentation](https://pdfkit.org/docs/getting_started.html)
- [Amazon S3 presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [wkhtmltopdf documentation](https://wkhtmltopdf.org/docs.html)
- [ISO 32000-2:2020, Portable Document Format](https://www.iso.org/standard/75839.html)
