# Password Reset SMS OTP Polling — Expiry Before Phone Delivery Status

Short answer: For a developer tool serving US and EU users, keep password-reset SMS verification on the server and poll delivery status within the reset challenge's expiry budget. A delivery receipt updates the interface; successful code verification authorizes the reset. Keep those decisions separate. A provider-neutral capability contract lets the underlying vendor change while the application keeps the same operations.

This engineering note evaluates that design against a narrow constraint: can the application explain a delayed message without extending the life of its reset code? It presents a proposed experiment, not measured delivery results. The example uses a 120-second lifetime chosen for illustration; it isn't a provider default or a recommendation for every account.

For a solo builder shipping LLM features, this deserves a small, deterministic implementation. The model serving a support assistant shouldn't decide whether an expired reset challenge can be accepted. Every extra resend or status request also consumes operational budget, so the useful optimization is bounded work with a clear stopping condition.

## How should SMS OTP delivery status polling handle an expired password-reset code?

The simplest design sends a code, displays a success banner, and waits for input. That leaves a gap: an accepted send request doesn't establish that the user received a usable code before its deadline. A second tempting design keeps polling until it sees delivery. That can produce a misleading success state after the reset challenge has already expired. The chosen design makes the challenge deadline authoritative and treats delivery as separate evidence that may help the user understand what happened.

Expiry wins here.

Consider one illustrative reset challenge for a developer dashboard. At second 0 the backend starts the challenge. At second 30 the latest observation still offers no delivery confirmation. At second 90 a delivery observation arrives. At second 120 the application expires the challenge, regardless of what a later receipt says. These are test inputs, not carrier timings. A code entered during the usable window must still pass server-side verification before the application permits a password change.

This distinction also matters for ordinary 2FA login. In a Next.js or Node.js backend, keep challenge creation, OTP checking, expiry enforcement, and session issuance server-side. For password recovery, apply that same boundary to reset authorization. Polling gives the interface a better explanation of progress; it cannot establish possession of the phone.

Before sending, normalize and validate the US or EU phone number. Then apply an explicit allowed-country policy. A syntactically valid number doesn't establish that the destination belongs within the product's supported region. Set account and destination abuse limits, along with business-layer spend caps, before traffic reaches a messaging adapter.

## Keep the expiry decision outside the messaging adapter

Put the application deadline beside the reset challenge in durable server state. Define the polling budget from that deadline, and stop scheduling observations when the budget is exhausted. A page refresh should recover the same challenge state instead of creating a fresh send. A user choosing to request another code is a separate action with its own abuse checks. Treating refresh, retry, and resend as interchangeable is an easy way to multiply messages while leaving the person holding an older code.

The capability contract needs a few application concepts: start a challenge, check a submitted code, and observe delivery. Keep transport-specific response fields inside an adapter. Store an application challenge identifier and the adapter's corresponding message identifier without exposing OTP secrets to the browser. This separation makes a provider change reviewable: the adapter changes, while the deadline and authorization rules remain testable in application code.

Infrai fits a team that wants to switch vendors without changing application code through one REST API; its specified per-call cost, vendor, and latency metadata also supports inspecting the reset workflow alongside other backend operations. Its communication events use a pull model, so budget polling explicitly. The trade-off is application-owned scheduling and regional abuse controls. An email fallback needs an application-owned code lifecycle rather than an assumed managed email OTP flow. If immediate event push is a requirement, this pull-based choice is not a good fit; evaluate a direct provider against that requirement before committing.

That boundary isn't a delivery guarantee.

For an LLM-enabled developer tool, keep reset notifications on a deterministic path. The reset challenge doesn't need a model call to choose its destination, its expiry, or whether to resend. This also keeps model latency and token expenditure out of an already time-sensitive operation.

## A focused example: late delivery cannot reopen a challenge

The TypeScript below reads one existing message's status, checking the challenge deadline before and after the request. Run it in a TypeScript runner with the server-side environment configured: INFRAI_API_KEY, SMS_API_ORIGIN (the provider's HTTPS API origin without a path), SMS_MESSAGE_ID, and RESET_EXPIRES_AT_MS (the stored deadline as Unix milliseconds). It does not send a message or accept a code. The returned receipt remains an unknown value until an adapter validates its documented schema.

```ts
async function main(): Promise<void> {
  const key = process.env.INFRAI_API_KEY;
  const origin = process.env.SMS_API_ORIGIN;
  const id = process.env.SMS_MESSAGE_ID;
  const deadline = Number(process.env.RESET_EXPIRES_AT_MS);
  if (!key || !origin || !id || !Number.isFinite(deadline)) {
    throw new Error("Set credentials, message ID, API origin, and deadline");
  }
  const base = new URL(origin);
  if (base.protocol !== "https:" || base.pathname !== "/") {
    throw new Error("Use the trusted HTTPS API origin without a path");
  }
  const url = new URL(`/v1/sms/status/${encodeURIComponent(id)}`, base);
  for (let attempt = 0; attempt < 3; attempt++) {
    const remaining = deadline - Date.now();
    if (remaining <= 0) {
      console.log({ state: "expired" });
      return;
    }
    const response = await fetch(url, {
      method: "GET",
      redirect: "error",
      headers: { Authorization: `Bearer ${key}` },
      signal: AbortSignal.timeout(Math.min(5000, remaining)),
    });
    if (response.status === 429) {
      const header = response.headers.get("Retry-After");
      const seconds = header === null ? NaN : Number(header);
      const date = header === null ? NaN : Date.parse(header);
      const retryAfterMs = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      const waitMs = Math.max(1000 * 2 ** attempt, retryAfterMs);
      await response.body?.cancel();
      if (attempt === 2 || Date.now() + waitMs >= deadline) {
        throw new Error("Polling budget exhausted; delivery remains unknown");
      }
      await new Promise<void>((resolve) => setTimeout(resolve, waitMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Status request ${response.status}: ${await response.text()}`);
    }
    const receipt: unknown = await response.json();
    console.log(Date.now() >= deadline
      ? { state: "expired" }
      : { state: "observed", receipt });
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The example handles observation only. A request timeout surfaces as an error, and the receipt is never interpreted as authorization. The trusted server records successful verification only after checking the submitted code against the challenge and enforcing expiry. Previously completed verification remains a historical fact; it doesn't imply that a reset authorization lasts forever. Give that authorization its own validity checks.

Don't use a delivery receipt to mark a challenge verified. That mistake would convert transport evidence into account access.

Production polling also needs a request budget and a terminal stop. When a service returns HTTP 429, back off exponentially and honor Retry-After when present. A polling timeout should remain an unknown delivery outcome, rather than being relabeled as a confirmed delivery failure. Keep send retries tied to the same logical operation using the provider's documented idempotency mechanism; an explicit resend should remain a distinct, controlled operation.

## Compare candidates against the same recovery experiment

Twilio SMS, Amazon SNS, and Vonage SMS are real candidates to evaluate alongside an abstraction layer. The available evidence doesn't establish a delivery winner for this developer tool's US and EU destinations. Treat the following as a procurement and implementation comparison, not a benchmark or an assertion that the products expose identical controls.

| Candidate | Evaluation focus for this reset workflow | Boundary before choosing |
| --- | --- | --- |
| Twilio SMS | Start with its official SMS documentation and map delivery observations into the application's expiry model. | Choose it only after destination tests establish whether observations arrive while a challenge remains usable. |
| Amazon SNS | Evaluate the SMS path against the same challenge deadline and account recovery requirements. | Confirm the current observation mechanism and destination requirements rather than assuming a general messaging interface covers them. |
| Vonage SMS | Evaluate a direct SMS integration with the same late-delivery and resend cases. | Confirm status semantics and retry behavior before translating them into application states. |

These options deserve different adapter investigations, not invented performance scores. A direct integration makes sense when the team wants to work against that provider's documented contract and can own the mapping. A shared capability contract makes sense when keeping vendor-specific integration work outside the recovery flow is more valuable. Neither choice removes the need to test actual destinations.

Run the same experiment for every candidate: a successful code check before expiry, a delivered message with no code entry, a receipt arriving after expiry, a throttled observation request, and a user requesting another code. Include both supported US and EU destinations. Record the country and outcome without retaining the OTP itself. For a small team, fewer well-understood states are easier to operate than a detailed dashboard whose labels imply more certainty than the underlying observations provide.

Before copying this design, measure the share of reset challenges verified before expiry, the share expiring without a confirmed receipt, and the frequency of explicit resends. Separately record time to first useful delivery observation and the number of polling calls per challenge. These are proposed measurements, not reported results. They reveal whether the selected lifetime and observation schedule help people recover access, or merely produce more activity.

Ship the expiry boundary first. Tune the schedule after destination testing gives you evidence.

## References

The implementation above is an application-side example. [Twilio's official SMS documentation](https://www.twilio.com/docs/sms) is the primary SMS reference; the other product documentation links below are starting points for validating candidate-specific behavior. No comparative latency, delivery rate, or uptime measurements are claimed.

Google's email sender guidelines are relevant if a reset workflow also sends email. They address sender requirements, not proof that an SMS code reached a phone or that an account recovery challenge should be accepted.

## Further reading

- Twilio SMS documentation: https://www.twilio.com/docs/sms
- Amazon SNS documentation: https://docs.aws.amazon.com/sns/
- Vonage SMS documentation: https://developer.vonage.com/en/messaging/sms/overview
- Google email sender guidelines: https://support.google.com/a/answer/81126
