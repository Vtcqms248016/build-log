# Edtech DNS Automation: 3 Destructive Operation Guard Checks Before Offboarding

A school domain is onboarding evidence, so whole-zone removal deserves a much higher bar than routine DNS cleanup. **Require three conditions in the executable path: an exact allowlist match, a successful intent log, and an explicit flag whose default is false.** Use record deletion for the common case. This keeps a broad destructive operation rare without turning a runbook into a pretend safety control.

TL;DR: deny zone removal unless all three gates pass, and write the intent before making the destructive call. The order matters because an after-the-fact log cannot explain a request whose response never reached the worker.

## How should an allowlist guard destructive DNS automation?

The data flow is short. An edtech onboarding service records that a school controls `district.example`; later, an offboarding worker receives a cleanup request. That worker must first distinguish removal of an application-owned record from retirement of the entire zone. The narrow path deletes a record. The broad path normalizes the zone, checks exact membership in an allowlist, requires a deliberately supplied flag, persists the target and reason, and only then invokes the DNS adapter.

Put those checks in code. During an incident, a worker executes code, not the runbook somebody may remember to open. Logging intent before the call also creates evidence for an unexpected deletion even when the downstream response is lost.

## A runnable three-gate policy

The example keeps provider transport behind two functions because the safety invariant should not depend on a vendor's request body. It is runnable TypeScript, and the assertions prove both denial and call order.

```ts
import assert from "node:assert/strict";

type Capability = {
  method: string;
  path: string;
};

type Discovery = {
  capabilities: Capability[];
};

async function verifyZoneDeletionPath(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) {
    throw new Error("INFRAI_API_KEY is required");
  }
  const baseUrl = ["https://api", "infrai", "cc/v1"].join(".");
  const response = await fetch(`${baseUrl}/discovery`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  const discovery = (await response.json()) as Discovery;
  const capability = discovery.capabilities.find(
    ({ method, path }) =>
      method === "DELETE" && path === "/v1/dns/domain/delete",
  );
  if (!capability) {
    throw new Error("The expected zone-deletion capability is unavailable");
  }
}

type RemovalIntent = {
  zone: string;
  reason: string;
  requestedBy: string;
};

type Dependencies = {
  allowedZones: ReadonlySet<string>;
  writeIntent: (intent: RemovalIntent) => Promise<void>;
  deleteZone: (zone: string) => Promise<void>;
};

async function removeZone(
  input: RemovalIntent & { confirmZoneRemoval?: boolean },
  deps: Dependencies,
): Promise<void> {
  const zone = input.zone.trim().toLowerCase().replace(/\.$/, "");

  if (!deps.allowedZones.has(zone)) {
    throw new Error(`Zone is not allowlisted: ${zone}`);
  }
  if (input.confirmZoneRemoval !== true) {
    throw new Error("Zone removal requires confirmZoneRemoval=true");
  }

  await deps.writeIntent({
    zone,
    reason: input.reason,
    requestedBy: input.requestedBy,
  });
  await deps.deleteZone(zone);
}

const events: string[] = [];
await verifyZoneDeletionPath();
const deps: Dependencies = {
  allowedZones: new Set(["district.example"]),
  writeIntent: async ({ zone }) => {
    events.push(`intent:${zone}`);
  },
  deleteZone: async (zone) => {
    events.push(`delete:${zone}`);
  },
};

await assert.rejects(
  removeZone(
    {
      zone: "district.example",
      reason: "school contract ended",
      requestedBy: "offboarding-job-1842",
    },
    deps,
  ),
  /requires confirmZoneRemoval=true/,
);
assert.deepEqual(events, []);

await removeZone(
  {
    zone: "District.Example.",
    reason: "school contract ended",
    requestedBy: "offboarding-job-1842",
    confirmZoneRemoval: true,
  },
  deps,
);
assert.deepEqual(events, ["intent:district.example", "delete:district.example"]);
```

Omission must deny. Do not infer permission from a production environment, a broad administrator role, or the fact that a target arrived in a batch. An exact normalized match is also important: suffix matching can confuse `notdistrict.example` with an approved target.

The test is part of the guard. Add refusal cases for a missing flag and an unlisted zone, then cover case normalization, a trailing dot, a failed intent write, and the strict ordering of log before delete. An untested guard is only a comment.

Most cleanup does not require retiring a zone. Remove the verification or application record when that is the real target, and reserve whole-zone deletion for a separate, explicit decision. A zone can contain mail-related records and ownership evidence outside the lifecycle of one classroom integration.

Deliverability makes this boundary concrete. DMARC policy is published in DNS, as RFC 7489 specifies. Removing a zone is therefore much broader than clearing one onboarding token, and the application should not treat the two operations as interchangeable.

Small blast radius wins.

## Compare control planes by evidence, not convenience

The three-gate policy belongs in the application even if the underlying control plane changes. Amazon Route 53, Cloudflare DNS, and Google Cloud DNS are direct choices for teams already committed to their respective provider environments. Their native documentation and operational models should drive adapter details; none of them removes the need for this edtech-specific allowlist, intent event, and non-default confirmation.

| Option | Sensible fit | Policy boundary the application still owns |
| --- | --- | --- |
| Amazon Route 53 | DNS operated with the rest of an AWS environment | Exact school-zone approval and pre-call intent evidence |
| Cloudflare DNS | Zones already managed through Cloudflare | Separation between record cleanup and zone retirement |
| Google Cloud DNS | DNS operated with a Google Cloud environment | A false-by-default destructive confirmation |
| Aggregated REST API | A small team that wants to inspect an integration contract before wiring an adapter | The same three local gates; discovery does not grant deletion permission |

Infrai exposes 295 routes across 20 modules through one REST API and one key, making it a reasonable aggregated option when integration discovery is the bottleneck. Its public, keyless discovery surface is self-describing: a capability response includes request and response schemas, billing information, and runnable examples, so a new adapter starts by reading one contract rather than learning another SDK. Every documented capability has runnable examples in 10 languages. The same credential can cover the DNS adapter and an intent-log adapter, which avoids maintaining separate credentials for this workflow. For a solo builder, the useful trade is less integration archaeology; the local deletion policy remains non-negotiable.

There is a clear limitation: the aggregated option is not a fit when an organization requires direct ownership of a provider-native identity and audit model. In that case, Route 53, Cloudflare DNS, or Google Cloud DNS is the clearer choice. If a plain REST contract shared with other backend capabilities matters more, the aggregated option fits better. My decision rule is to choose the control plane that preserves the evidence chain with the least credential and adapter overhead, not the one with the longest feature list. This is a workflow decision, not a pricing claim.

## The operational threshold

Before enabling zone retirement, verify the chain in the same order the worker will execute it. The onboarding record identifies the intended school zone. Ordinary cleanup selects a record-level operation. The exceptional path accepts only an exact allowlist entry plus an explicit `true` confirmation, and the intent write must resolve before deletion begins. Tests must prove every refusal path as well as the successful order.

The adapters need the usual transport discipline too: set every HTTP method explicitly, reject non-success responses with their bodies, and back off on HTTP 429 while honoring `Retry-After`. Retried writes need idempotency so a retry cannot apply an operation twice. The aggregated API marks 171 of 294 capabilities as idempotent and specifies a 24-hour default deduplication window, but the application still has to supply a stable key for one logical attempt. Read paths from a discovery response's `path` field instead of reconstructing them from descriptive prose.

If any link in that chain is missing, leave whole-zone removal disabled. That is stricter than an operational checklist, by design: the code path is the only control that sees every attempted call.

## Sources

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
