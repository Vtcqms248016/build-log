# Marketplace Media Retention: Presigned URL or Proxy Backend for File Upload Security

Short answer: use a presigned URL for the browser-to-storage byte path, but make retention and deletion a server-owned state machine; use a proxy backend when the application must inspect or transform the complete media stream before it is accepted. For a marketplace, that boundary matters more than shaving one request from the upload flow, because abandoned listings and seller deletions create a storage problem that an upload button cannot solve.

The concrete flow is small. A seller starts an upload, the backend authenticates the seller and creates a pending media record, and storage receives a short-lived upload authorization for a server-chosen key. The browser sends the large file directly to storage. A completion request then causes the backend to verify the object and move the record into the next state. The application owns meaning; storage owns bytes.

That separation keeps media out of the app process, but it does not make the media trusted.

## What should a web app check before choosing presigned URL or proxy backend upload?

Begin with the acceptance rule, not the transport. Ask one uncomfortable question: must the application see every byte before storage writes it? If the answer is yes because synchronous inspection, transcoding, or a byte-level policy is mandatory, a proxy backend is the honest design. The browser sends the file to the application, and the application forwards it to storage. That gives the server a complete inspection point, while also giving the app tier the bandwidth, memory, timeout, and retry burden.

If acceptance can be asynchronous, direct upload is usually the cleaner boundary for marketplace photos, video, and seller documents. The backend can constrain the object key, expected size, content type, owner, and expiry of the upload authorization. It can keep the object private and expose a separate application decision about when media becomes visible. A signed upload URL grants a temporary action; it is not evidence that the right file arrived, and it is not evidence that the seller is allowed to publish it.

Security moves. It does not disappear.

For this decision, compare the two paths against the same failure cases:

| Concern | Browser direct to storage | Proxy through the backend |
| --- | --- | --- |
| Large-byte pressure | Leaves payload bandwidth outside the app tier | Concentrates payload traffic in the app tier |
| Pre-write inspection | Requires a later quarantine or processing step | Can inspect the stream before forwarding |
| Authorization | Short-lived, narrow upload authority plus app checks | The app guards the upload request and its forwarding step |
| Browser setup | Requires the storage boundary to allow the production origin and required request headers | Uses the app origin as the browser-facing boundary |
| Deletion semantics | Needs a record-to-object cleanup worker | Still needs cleanup for partial or rejected objects |

The table is not a verdict. It is a prompt to find the requirement that rules out one path.

## How does a marketplace upload become a retention and deletion workflow?

Treat the upload as a state transition, not a successful `PUT`. A useful minimum is `pending`, `quarantined`, `published`, `rejected`, and `deleted`. The names can differ, but the ownership must not: only the application should decide that a listing references an object, while a cleanup process should be able to remove an object that no longer has an owner.

The object key should include an opaque media identity rather than a seller-supplied filename. Create the database record before issuing upload authority. On a lost browser response, retry the same logical upload instead of allocating an unrelated record. When the browser reports completion, verify that an object exists at the expected key and that its metadata is consistent with the pending record. Only then should a worker enqueue inspection or a moderator-visible state.

Deletion needs more than a delete button. A seller may delete a draft, a moderator may reject a listing, or an account may close while a multipart upload is still in progress. Each event should revoke application visibility immediately, mark the media for deletion, and let an idempotent worker remove the object and record the result. Keep a tombstone or audit event when the marketplace needs to explain who requested deletion and when; do not rely on a missing object as your only history.

Retention is a policy, not a single storage setting. Define separate clocks for pending uploads, rejected media, published media, and audit evidence. The exact durations belong to the marketplace's legal and product requirements. A lifecycle rule can enforce expiry for an object class, but it cannot decide whether a listing was deleted or whether another record still references the same bytes. That decision belongs in application data.

Here is the small part I would make deterministic and testable before wiring in a storage client:

```ts
type MediaState =
  | "pending"
  | "quarantined"
  | "published"
  | "rejected"
  | "deleted";

type MediaRecord = {
  mediaId: string;
  listingId: string;
  objectKey: string;
  state: MediaState;
  deleteRequestedAt?: string;
};

export function requestDeletion(
  record: MediaRecord,
  requestedAt: string,
): MediaRecord {
  if (record.state === "deleted") return record;

  return {
    ...record,
    state: "deleted",
    deleteRequestedAt: requestedAt,
  };
}
```

The storage deletion can be retried after this transition. That order is deliberate: once the application stops serving the media, a delayed storage operation is an operational cleanup concern rather than a visibility leak. Your mileage may vary if the product must provide immediate physical erasure, so document that distinction instead of promising a stronger guarantee than the system has.

## Where do cost, latency, and security trade places?

A direct upload generally removes the large payload from the backend's data path. The browser still makes a control-plane request to obtain authorization and another application request to report completion, while storage handles the bytes. That can reduce application bandwidth demand and upload-related timeout exposure. It does not eliminate storage requests, transfer costs, browser retries, or the need to clean up abandoned data.

A proxy has a different shape. It can make one browser-facing origin and one central admission point easier to operate, especially when the storage boundary cannot be configured for the production web origin. The price is payload traffic through the application, where concurrency limits and request timeouts become part of the media design. For a small image in an internal tool, that simplicity may be worth it. For seller videos, it can become the bottleneck.

Latency is not just “direct is faster.” Region placement, payload size, TLS connections, upload retries, and the extra authorization round trip all affect the result. Measure time to first byte, time to complete, and time from completion to publish separately. A fast upload that leaves media stuck in quarantine is not a fast marketplace workflow.

The same care applies to security. A proxy can inspect earlier, but it also becomes a high-value byte relay. A direct flow narrows the backend's data path, but it requires strict object-key ownership, private storage, short authorization lifetimes, origin configuration, post-upload verification, and publication checks. Never use a permanent public object URL as a substitute for an authorization decision.

## What does the TypeScript implementation need to make deletion reliable?

Keep storage-provider calls behind an interface so the application tests policy without making network calls. The interface should express the operations the marketplace needs, rather than leaking every feature of a particular storage product into listing code.

```ts
type UploadGrant = {
  objectKey: string;
  method: "PUT";
  expiresAt: string;
};

interface MediaStore {
  createUploadGrant(input: {
    objectKey: string;
    contentType: string;
    maxBytes: number;
  }): Promise<UploadGrant>;
  inspect(objectKey: string): Promise<{ size: number; contentType: string }>;
  deleteObject(objectKey: string): Promise<void>;
}

export async function beginMediaUpload(
  store: MediaStore,
  input: { mediaId: string; contentType: string; maxBytes: number },
): Promise<UploadGrant> {
  const objectKey = `marketplace/quarantine/${input.mediaId}`;

  return store.createUploadGrant({
    objectKey,
    contentType: input.contentType,
    maxBytes: input.maxBytes,
  });
}
```

The example intentionally stops at granting authority. The application still has to persist the pending record, authenticate the seller, bind `mediaId` to the listing, and reject a completion whose observed metadata does not match policy. Those checks should have tests for duplicate completion, deletion during quarantine, a second seller attempting another listing's key, and a worker retry after a successful object delete.

I would also emit an event for each state change and measure the age and count of pending, rejected, and delete-requested media. A growing delete queue is a storage-cost signal and a product-integrity signal. It deserves an alert before the bill or a seller complaint finds it.

## The decision rule I would ship

Choose browser direct-to-storage when the marketplace can quarantine media, verify it after upload, and configure the storage boundary for the web origin. Make deletion a durable application transition followed by idempotent cleanup. This is the right default for large media when the server does not need to transform every byte synchronously.

Choose a proxy when pre-write inspection is non-negotiable, the browser cannot be granted the required narrow authority, or the files are small enough that centralizing the path is an intentional operational choice. The catch is that the proxy owns the payload pressure; budget for that in concurrency, timeout, retry, and observability design.

Do not use either transport as a retention policy. Keep the listing-to-media reference authoritative, keep rejected and abandoned objects out of the public path, and make deletion repeatable. That is the part that survives a seller retry, a moderation decision, and the first unexpected cleanup backlog.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://cloud.google.com/storage/docs
