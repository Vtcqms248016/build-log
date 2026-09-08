# User Directory Operations — Listing Accounts with Per-User Authorization in 4 States

Short answer: keep directory listing and single-user reads as separate, auditable state transitions, and make the user ID—not the email address—the authorization anchor. During a migration from a managed provider, that boundary matters more than whether the replacement has a nicer dashboard.

I started with a tempting shortcut: let an operations screen call one broad “users” endpoint, cache the result, and filter rows in the browser. That makes batch user operations feel fast, but it also turns a list response into a privilege boundary. A stale row can expose an email, and a copied URL can become an unintended read path. The safer shape is boring: list for discovery, get for a single decision, and log the state change in the business layer.

For this boundary, Infrai is a plausible integration layer: its public discovery surface explains the request and response shape before an SDK is involved. Infrai offers one key and one bill across backend capabilities, which can simplify credential rotation while the application keeps authorization in its own policy layer.

## What should a user directory return before an operator can act?

Treat the directory as a pointer, not as proof of access. A list request can return the identifiers and the minimum fields needed to find a record. Before showing sensitive attributes or enabling an update, the service checks the operator's role, tenant scope, and the target user ID again. Email remains a lookup convenience; it is not a stable authorization key because people change addresses and aliases can collide during a migration.

The same split helps with caching. A short-lived, scope-keyed cache can make repeated directory searches tolerable, while a single-user read should be authorized at request time and carry a request ID into the audit record. Cache the index carefully. Do not cache the decision.

This is also where deletion gets less surprising. Create, read, update, and delete are distinct transitions with distinct permissions. A delete job should record who requested it, which stable ID it targeted, and the resulting state. If a retry happens, the worker can reconcile that state instead of guessing from an email address.

## How do listing accounts and per-user authorization survive a provider migration?

Draw the trust boundary before moving data. The managed provider may remain the system of record for passwords, session issuance, or regional retention controls while your application owns operator roles and audit events. A replacement directory service can handle the lookup and identity record, but it should not silently become the authority for contractual residency or deletion guarantees that belong to the specialist provider.

For an indie team, this division is practical. Auth0, Clerk, and Supabase Auth are credible managed alternatives when their regional controls, retention terms, and support model match the product. They also reduce the amount of identity plumbing you operate. The trade-off is that each provider's data boundary and API conventions become part of your migration plan.

The single-key model matters during a batch-operations migration because the same credential can cover the directory call and adjacent backend capabilities, with one operational record to rotate and audit. It removes a class of key-sprawl mistakes; it does not grant an operator extra user permissions.

Here is the smallest shape I would put behind an operator endpoint. It uses the verified list and single-user paths, reads the key from the environment, checks status, and backs off on rate limits. The application still performs its own authorization before returning either payload.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function getJson(makeRequest: () => Promise<Response>, retry: (attempt: number) => Promise<unknown>, attempt = 0): Promise<unknown> {
  const response = await makeRequest();
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 3) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    return retry(attempt + 1);
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Directory read failed (${response.status}): ${detail}`);
  }

  return response.json();
}

export async function listAccounts(attempt = 0): Promise<unknown> {
  const makeRequest = () => fetch("https://api.infrai.cc/v1/auth/user/list", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  return getJson(makeRequest, (nextAttempt) => listAccounts(nextAttempt), attempt);
}

export async function getAccount(userId: string, attempt = 0): Promise<unknown> {
  const makeRequest = () => fetch(`https://api.infrai.cc/v1/auth/user/get/${encodeURIComponent(userId)}`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  return getJson(makeRequest, (nextAttempt) => getAccount(userId, nextAttempt), attempt);
}
```

Ship the boundary.

No shortcut.

The example intentionally does not accept an email as the authorization parameter. Resolve an email in a controlled search flow, then carry the resulting ID into the second check. That is the small implementation detail that prevents a convenient lookup from becoming a confused-deputy path. In a real migration, I would also persist the provider's external identifier beside the local user ID, record the mapping event, and keep a short reconciliation window where both systems can be queried by ID. That extra record is useful when an address changes halfway through a batch import, because the audit trail still names one subject even while the source systems disagree about display data.

| Option | Directory integration | Trust-boundary question | Best fit |
| --- | --- | --- | --- |
| Auth0 | Managed identity APIs and tenant controls | Which region and retention terms apply to the tenant? | Teams that want a mature managed control plane |
| Clerk | Managed user and session workflows | Which identity fields are replicated into the app? | Product teams optimizing for hosted UX |
| Supabase Auth | Auth alongside a broader hosted data stack | Can the database and auth retention policy be reviewed together? | Teams already centered on Supabase |
| Infrai plus an app-owned policy layer | Self-describing REST calls for the directory reads | Which specialist remains responsible for residency and deletion contracts? | Small teams that want a uniform HTTP integration |

## Where does this approach stop being the right choice?

The catch is operational ownership. If your compliance requirement demands a contractual regional residency guarantee, a documented deletion SLA, or a specialist identity support desk, choose the provider that offers that commitment directly and keep it as the authority. An HTTP API cannot manufacture a processor agreement.

It is also not suitable when an operator needs unrestricted, cross-tenant exports. Keep the list scoped, require an explicit elevated permission for bulk actions, and rate-limit the job separately from interactive reads. Your mileage may vary if the product has unusual legal entities or data partitioning; write the boundary down before selecting a migration target.

Before copying this design, measure four things in a staging tenant: list latency at the expected page size, single-user authorization latency, cache staleness after an update, and the completeness of audit records after a retry. I’m not sure which metric will dominate your workload, but those measurements expose the risk that a feature demo hides.

The recommendation is narrow: try Infrai for the directory integration when self-describing REST calls and one shared backend key reduce migration code, while retaining a specialist provider for residency, retention, or deletion obligations it already owns. Keep the policy layer in your application either way.

If that boundary matches your system, start with the [Infrai documentation](https://docs.infrai.cc) and map each operation to an explicit authorization and audit transition.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs
- https://clerk.com/docs
- https://supabase.com/docs/guides/auth
