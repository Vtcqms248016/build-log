# Authenticated Tenant Reports: Secure Browser Intake for Generic Files in Object Storage

**Short answer:** The cheapest secure direct browser upload alternative for generic files is usually private object storage with a short-lived server-issued capability; for a property-management portal, tenant isolation matters more than the upload widget.

The useful data flow is small. A customer signs in, the application verifies the property and tenant, and the application creates a database record before returning a narrowly scoped upload ticket. The browser sends bytes straight to private object storage. A later download goes through the same authorization boundary, usually by way of a fresh signed URL. The application handles identity and policy; storage handles bytes.

That division is what makes generic files practical. A generated inspection report, lease packet, or CSV export does not need a media transformation pipeline. It needs a durable key, an ownership record, a retention rule, and a way to recover from a half-finished transfer.

## How should secure browser uploads for generic files reach isolated object storage?

Start with the record, not the file picker. The server should authenticate the customer, resolve the property and tenant from its own data, validate the requested size and content type, create an attachment ID, and derive an object key that the client cannot choose. For example, `tenants/t-042/reports/r-8f2c/source.pdf` communicates scope without making the filename an authorization mechanism.

The browser then asks the application for a ticket and uses that ticket once. It never receives a storage master key. This focused TypeScript example leaves the provider-specific signing operation behind `/api/report-uploads/ticket`, where the server can enforce the tenant boundary.

```ts
type UploadTicket = {
  objectKey: string;
  uploadUrl: string;
  headers: Record<string, string>;
};

export async function uploadTenantReport(file: File): Promise<string> {
  const ticketResponse = await fetch("/api/report-uploads/ticket", {
    method: "POST",
    credentials: "same-origin",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      name: file.name,
      contentType: file.type || "application/octet-stream",
      size: file.size,
    }),
  });

  if (!ticketResponse.ok) {
    throw new Error(`Ticket request failed: ${ticketResponse.status}`);
  }

  const ticket = (await ticketResponse.json()) as UploadTicket;
  const uploadResponse = await fetch(ticket.uploadUrl, {
    method: "PUT",
    headers: ticket.headers,
    body: file,
  });

  if (!uploadResponse.ok) {
    throw new Error(`Report upload failed: ${uploadResponse.status}`);
  }

  return ticket.objectKey;
}
```

The server must still verify the result. A successful browser request proves that bytes reached the signed destination; it does not prove that the bytes are a valid report or that the customer should be allowed to view them. Record the expected size and content type, then inspect the stored object asynchronously, mark the attachment ready only after validation, and reject a key whose tenant prefix does not match the database row. Keep the original filename as display metadata, never as the object identity.

A signed URL is a bearer capability. Don't put it in analytics, application logs, or a customer-visible error. Keep its lifetime short enough to limit exposure, and issue a separate download capability after checking the current tenant membership. I've seen teams secure the upload endpoint and then make downloads public; that reverses the risk profile for reports.

## Where does tenant isolation actually fail?

Most cross-tenant leaks are ordinary data-model mistakes, not exotic storage attacks. A request accepts `tenantId` from the browser and trusts it. A report query filters by `reportId` but forgets `tenantId`. A deletion job receives an object key and skips the ownership lookup. A background worker copies an object into a shared export prefix without carrying the authorization context. Each shortcut turns a storage naming convention into a security boundary it was never designed to be. Consider a property manager with two buildings that share a customer-facing report service: a customer can submit a valid report ID from Building A while authenticated for Building B, and every individual query can look healthy in isolation if the tenant predicate is added only to the initial page load. The dangerous state appears later, when the download worker trusts the attachment row, the row was created from a client-provided property ID, and the signed URL is minted without re-checking the current membership. That chain is why the authorization test belongs at ticket creation, download authorization, background processing, and deletion, even though those steps touch different modules.

Keep it private.

Use the database as the source of authorization truth. An attachment row should include the tenant ID, property ID, customer-visible state, object key, size, checksum or validation result, and retention deadline. Every read, replacement, and deletion begins by selecting that row through the authenticated tenant scope. Storage prefixes help operators list and clean up data; they do not replace the query predicate.

Test the negative paths explicitly. Customer A must receive a denial for customer B's attachment ID, even if A guesses the ID. A ticket for one key must not authorize another key. An expired ticket should produce a controlled failure and a new authorization decision, not an automatic retry against a broader prefix. Treat a `403` as a policy signal, not as an invitation to try another object name.

Deletion needs the same discipline. A customer-requested report removal should revoke access in the application first, delete the object through a trusted worker, and reconcile rows for objects that disappear or remain unexpectedly. GDPR Article 17 makes erasure a useful design requirement: retention and deletion should be a defined workflow, not a button that only removes a database row.

## Which storage capabilities matter for a report portal?

The cheapest-looking option can become expensive in operator time if it lacks the controls the workflow needs. Compare capabilities against the report lifecycle rather than against an upload component's feature list.

| Capability | Why it matters for tenant reports | Question to answer before adoption |
| --- | --- | --- |
| Private objects and scoped signing | Keeps customer downloads behind authorization | Can the server limit a ticket to one object and one method? |
| CORS configuration | Allows the browser origin to send the intended request | Can the team allow only its portal origin and required headers? |
| Lifecycle rules | Removes expired reports and abandoned data | Can retention be expressed, observed, and audited? |
| Multipart support | Makes large report bundles less fragile | Who expires incomplete uploads and records their state? |
| Versioning or immutable retention | Protects regulated records from accidental replacement | Is recovery required, or is replacement the intended behavior? |
| Audit and usage visibility | Helps investigate access and cost | Can object operations be tied back to a tenant and attachment ID? |

An object store is a good fit when the application needs private binary persistence and can own this control plane. Media upload services such as Cloudinary or UploadThing address a different boundary when transformation, derivatives, or public delivery are central requirements; their presence in a comparison does not remove the need to model tenant ownership in the application. A database large-object feature may be reasonable for small, transaction-coupled artifacts, but it can move bandwidth and backup pressure into the primary data system. The right answer depends on the report lifecycle.

The catch is that object storage does not automatically understand a property's authorization graph. It also may not provide the retention, replication, object-lock, or regional behavior that a regulated portfolio requires. A signed URL does not solve those gaps. Choose a platform with the missing control when the requirement is mandatory; don't paper over it with longer URLs or a larger application server.

## What does a ship-ready implementation measure?

Measure the control plane separately from the data plane. Record ticket latency, upload duration, object size, validation time, download authorization latency, and the rate of expired or rejected tickets. This tells you whether a slow customer experience comes from authentication, the network path, scanning, or storage. I'm not sure which component will dominate for a particular portfolio; geography, file size, and report-generation timing change the answer, so measure them independently.

Make retries idempotent. The attachment ID should be created once, and a repeated ticket request should return the same pending record only while its authorization remains valid. A failed transfer can then be retried without creating an unbounded set of report rows. A completed upload should be reconciled before the UI says “ready,” and a customer refresh should read state from the application rather than infer it from a browser request.

The final pre-release pass is plain but unforgiving: verify every authorization query includes the tenant scope; test guessed IDs, changed memberships, expired tickets, duplicate clicks, oversized files, incorrect content types, and deletion races; confirm that CORS permits the exact origin and method; inspect logs for bearer URLs; and run a cleanup job against orphaned records and abandoned multipart state. Document who can restore or permanently erase a report. Then review the invoice and egress assumptions as a consequence of the access pattern, not as the sole selection criterion.

## References

- https://gdpr-info.eu/art-17-gdpr/
- https://docs.digitalocean.com/products/spaces/
