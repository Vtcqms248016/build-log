# Marketplace TXT Records: Scheduling Service Verification Without Sacrificing Zone Hygiene

Keep an existing integration active until the replacement has confirmed domain verification. For a marketplace admin console, the useful cutover boundary is a verified service state; a successful DNS write only establishes that the requested change was accepted. Schedule cleanup separately so pressure to launch does not turn into permission to remove unrelated TXT entries.

TL;DR: manage third-party verification as an additive change with an explicit owner, preserve other services' entries, and require fresh verification evidence before cutover. A waiting deadline should produce an operator review, never automatic deletion. This trades a temporarily untidy zone for a controlled transition.

## How should service verification TXT records affect a marketplace cutover?

Put it in the admin console's change workflow. The console records a proposed TXT addition, the component responsible for DNS applies it, and the external service supplies the verification result. The console then decides whether the operator can advance the integration. Keep these observations separate: accepting a write, observing the expected DNS value, and confirming that the service accepted that value answer different questions. This is a proposed application design, rather than a claim that every service exposes the same verification interface.

Preserve the distinction.

A marketplace may have several services asking for entries under the same merchant domain. An operator adding a new verification value should not have to infer the purpose of every existing TXT value. Store the integration owner alongside the proposed change in application metadata. The record's contents alone are an inadequate deletion authorization. That matters especially when an admin console combines deterministic operations with an LLM assistant: suggested cleanup can be useful, but a model-generated guess about ownership should not authorize a DNS mutation.

The relevant mail standard makes this boundary concrete. RFC 7489 places DMARC policy in TXT records and defines discovery at a name such as `_dmarc.example.com`. Section 6.6.3 describes discarding records that do not begin with the current DMARC version and ending policy discovery if multiple records remain. A generic cleanup tool therefore needs to preserve the meaning of records it does not own. Counting TXT entries is not enough to diagnose a policy problem.

## Implement the transition before wiring the console

Start with a small decision function. The following TypeScript example is runnable without a DNS account: it produces a plan and prints it, while leaving DNS writes and service-specific verification to separate adapters. Its domain and verification string are illustrative application data, not a real service's token format. All TXT values passed into this example are already decoded logical values.

```ts
interface TxtRecord {
  name: string;
  value: string;
}

interface Change {
  integrationId: string;
  record: TxtRecord;
  deadlineMs: number;
}

interface Evidence {
  integrationId: string;
  name: string;
  value: string;
  verifiedAtMs: number;
}

type Plan =
  | { kind: 'add'; record: TxtRecord }
  | { kind: 'wait' }
  | { kind: 'review'; reason: string }
  | { kind: 'ready' };

function planVerification(
  change: Change,
  records: readonly TxtRecord[],
  evidence: Evidence | undefined,
  requestedAtMs: number,
  nowMs: number,
): Plan {
  const present = records.some(
    (record) => record.name === change.record.name
      && record.value === change.record.value,
  );
  const verified = evidence !== undefined
    && evidence.integrationId === change.integrationId
    && evidence.name === change.record.name
    && evidence.value === change.record.value
    && evidence.verifiedAtMs >= requestedAtMs
    && evidence.verifiedAtMs <= nowMs;

  if (present && verified) return { kind: 'ready' };
  if (nowMs >= change.deadlineMs) {
    return { kind: 'review', reason: 'Verification deadline reached' };
  }
  if (!present) return { kind: 'add', record: change.record };
  return { kind: 'wait' };
}

const requestedAtMs = 1_000;
const change: Change = {
  integrationId: 'merchant-42-catalog',
  record: {
    name: '_catalog-proof.merchant.example',
    value: 'illustrative-proof=merchant-42',
  },
  deadlineMs: requestedAtMs + 900_000,
};

const records: TxtRecord[] = [
  {
    name: '_dmarc.merchant.example',
    value: 'v=DMARC1; p=none',
  },
];

console.log(planVerification(
  change, records, undefined, requestedAtMs, 2_000,
));
```

The first result requests one addition. Passing the expected record on a subsequent call produces `wait` until matching evidence exists. The function has no deletion action. Cleanup requires a different authorization path because verification success does not establish that an older integration has stopped depending on its entry.

The example's 900,000 milliseconds is an illustrative 15-minute review window, not a propagation estimate or service guarantee. Give the UI that meaning explicitly. Otherwise an operator will read a countdown as a promise and interpret the deadline as evidence that a record is wrong.

Before connecting this function to an actual zone, define the adapter contract. Names must use one canonical representation, timestamps must come from a trusted clock, and the evidence must be bound to this particular change request in persistent storage. The compact example matches integration, name, value, and observation time; a production workflow should also use a change identifier to distinguish repeated attempts with the same value. Treat the supplied record snapshot as input to a plan, not a lock on the zone.

## Propagation delay changes what the console can promise

Separate an observation schedule from a cutover deadline. The former controls when the application checks again; the latter tells an operator when to investigate. Neither should manufacture a successful verification result. If the authoritative data appears correct while the external service still reports pending, the console should display those two observations with their timestamps. An operator then has evidence to investigate rather than an ambiguous green status.

Wait is valid.

The limitation is explicit: this workflow cannot guarantee immediate cutover. Its trade-off is additional coordination while two integrations coexist. For an emergency that requires an immediate traffic change, use a separately authorized incident procedure; a pending verification check is not enough evidence to approve it.

Choose the scheduling policy around the launch requirement. If the existing integration can keep serving the marketplace, leave it active while verification proceeds. That gives up immediate cleanup and may require tracking two integrations temporarily. If launch depends on a fixed cutover time, perform the addition and obtain the required verification evidence before that time; reaching the deadline without evidence should keep the cutover gated. These are application decision rules, not guarantees about how quickly every resolver or service will observe a change.

Polling also consumes an operating budget. Bound retries, retain the latest observation, and avoid sending unchanged pending states through an LLM on every check. A deterministic state transition is sufficient here. Reserve model calls for explanations an operator actually needs, such as summarizing the history of a stalled change. This keeps token consumption tied to useful work without turning price into the selection criterion.

There is a subtle failure mode in the example's `add` result: two workers can compute it from the same snapshot. The writer adapter must reconcile against current state before applying the addition, or use the DNS system's concurrency controls where available. Retrying a plan must not replace the whole collection of TXT values with the single requested value. Also distinguish a failed observation from an observed absence; a read error supplies no evidence that an entry is missing.

## Treat zone hygiene as a separate operation

A short zone is not necessarily a healthy zone. For each managed verification entry, retain its owner, change identifier, verification evidence, and the reason it may eventually be removed. An entry that the console did not create should remain outside automated cleanup until its ownership and dependencies are established. This allows onboarding to ship before a complete inventory exists, while making that incomplete inventory visible.

DMARC is a useful test case because its policy discovery has specific semantics. RFC 7489 section 6.1 also describes concatenating the character strings of a TXT resource record to construct its value. Keep that decoding in the DNS adapter: the planner should compare logical values rather than presentation fragments. Multiple strings inside one TXT record and multiple TXT records are different structures. Collapsing that distinction can make an apparently harmless normalization change alter what the application believes is present.

Removal deserves its own evidence.

A replacement's successful verification establishes something about the replacement; it does not establish the retention requirements of the old service. Record those requirements when onboarding each integration. If they are unknown, require review before removal. Do not infer that a token is disposable just because its name resembles another token.

## Release with evidence an operator can inspect

Before enabling writes, exercise the planner against a zone snapshot containing the requested proof, unrelated TXT values, and a DMARC policy. Check that the proposed change preserves the unrelated values. Then exercise pending verification, stale evidence from a previous request, concurrent additions, and an observation failure. The useful assertion is what remains intact when the workflow cannot advance.

For deployment, record the proposed addition and its change identifier before dispatching work. Capture the write acknowledgment separately from DNS observations and external verification. Surface the oldest pending change, its next scheduled check, and the last successful observation in the console. Store enough evidence to explain a decision without copying verification payloads into every log message. Keep a human review path for unknown ownership and missed deadlines.

Ship the additive workflow first.

Enable cutover when the required evidence is present, and schedule removal only after the old integration's dependencies have been resolved. The marketplace gets a clear launch decision while zone hygiene remains an accountable maintenance operation.

## References

- RFC 7489 — Domain-based Message Authentication, Reporting, and Conformance (DMARC), particularly sections 6.1 and 6.6.3: https://datatracker.ietf.org/doc/html/rfc7489
