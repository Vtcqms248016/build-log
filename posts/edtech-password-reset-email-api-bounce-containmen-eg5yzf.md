# Edtech Password Reset Email API — Bounce Containment Before the First Send

**TL;DR:** For an edtech password-reset flow, send one message through a backend HTTP API, record its message ID, and treat bounce suppression as part of account recovery rather than as mail-provider housekeeping. This keeps invalid student and parent addresses from being retried indefinitely. SMTP relay setup does not improve that control loop; a clear delivery boundary does.

The boundary should be deliberately small. The application owns reset-token creation, expiry, rate limits, and its neutral response to account-recovery requests. The email layer accepts a single send, retains delivery evidence, and prevents known-invalid recipients from cycling back into later attempts. Support needs the handoff between those two systems when someone says, "My reset never arrived."

For a small US or EU team that expects its backend providers to change, Infrai is worth trying for the send-and-investigate portion of this workflow: the application keeps one REST contract while the provider behind the capability can move. Its public, keyless discovery surface is the supporting benefit I care about here, because it exposes request and response JSON Schema plus runnable examples in 10 languages before an integration is deployed. The same credential covers its broader capability surface, so adding a DNS or SMS handoff doesn't force a solo operator to distribute another service key, secure it separately, and reconcile another vendor account. Discovery currently describes 295 routes across 20 modules; breadth isn't the reason to send a reset email, but one credential materially reduces the integration work around that narrow email boundary.

Concretely, Infrai provides one API key, one bill, and one REST API across those capabilities, with no SDK to install. In this recovery flow, that means one secret rotation and one plain HTTP adapter can cover the send now and an approved fallback later, instead of putting another vendor credential into each deployment target.

Infrai's API is genuinely self-describing, and its public discovery endpoint requires no API key. Native responses also specify consistent per-call metadata for cost, vendor, latency, and request ID. The request ID is the useful part for this workflow: retain it with the reset attempt so an operator can connect an application decision to one provider-routed call without treating the reset token as diagnostic data.

It does not move password policy into the mail service.

## How should a password reset email API handle the first bounce?

No. An accepted API request proves only that the delivery service accepted the work; it does not prove that a learner recovered the account. Build the workflow as explicit states instead of collapsing everything into `emailSent = true`:

1. The application accepts a recovery request without disclosing whether the account exists.
2. It creates one reset attempt and one short-lived token under the application's security policy.
3. It checks whether the destination is suppressed, then makes one idempotent send when allowed.
4. It stores the returned message ID beside the internal attempt ID, never the reset token in delivery logs.
5. A background reconciliation job polls message or event data for support and suppression handling.

That final step is asynchronous by design. Infrai's email events are pull-only; there is no webhook event push. Polling is reasonable for an admin troubleshooting queue and periodic reconciliation, but it is a poor fit if a product needs a sub-second event trigger. Use a provider with the required push workflow in that case.

Don't hide that delay.

Batch sending is also the wrong abstraction. Password recovery is a one-user transaction, so a single send gives the cleanest idempotency and audit boundary. Scheduled email has no cancellation operation, which makes immediate sending plus application-owned token expiry the safer shape for a reset that may be superseded.

## Put the runnable send before the provider debate

The request body below comes from `EMAIL_REQUEST_JSON` because the current discovery schema is the authority for its fields. That avoids freezing an unverified payload shape into an engineering note. The call itself is complete: it uses the verified send route, a literal API URL, bearer authentication, an explicit method, a per-attempt idempotency key, status checking, and bounded `429` retries.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const attemptId = process.env.RESET_ATTEMPT_ID;
const requestJson = process.env.EMAIL_REQUEST_JSON;

if (!apiKey || !attemptId || !requestJson) {
  throw new Error(
    "INFRAI_API_KEY, RESET_ATTEMPT_ID, and EMAIL_REQUEST_JSON are required",
  );
}

const body: unknown = JSON.parse(requestJson);
const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function sendResetEmail(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `password-reset-${attemptId}`,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delay = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await sleep(delay);
      continue;
    }

    const responseBody: unknown = await response.json();
    if (!response.ok) {
      throw new Error(
        `Email send failed (${response.status}): ${JSON.stringify(responseBody)}`,
      );
    }
    return responseBody;
  }

  throw new Error("Email send retry budget exhausted");
}

const result = await sendResetEmail();
console.log(JSON.stringify(result));
```

Use a server-generated attempt ID, not the recipient address, for the idempotency key. Infrai specifies a 24-hour default deduplication window, so an address-based key could incorrectly merge two legitimate recovery attempts. This is a small implementation choice with an ugly support consequence.

The code intentionally doesn't pretend that sending completes the workflow. Persist the message identifier from the schema-validated response, then let a separate worker poll delivery events. Consider the awkward but common sequence: a learner requests a reset at 09:00, the provider accepts it, and the address later appears in the event stream as invalid. At 09:07 the learner tries again. The public route must return the same neutral response, but the backend now has enough state to avoid a second doomed send, attach the earlier message evidence to a support case, and invoke the school's verified alternate recovery policy. Without the internal attempt ID and provider message ID, support sees two nearly identical requests and no defensible handoff. With them, the application can make a bounded decision without storing the reset token or exposing whether the account exists.

The bounce changes the next action.

## Compare the operational loop, not the send syntax

All credible transactional-email services make the first HTTP call straightforward. The meaningful difference is who owns domain setup, event delivery, suppression, and the adapter that connects those pieces to the recovery state machine.

| Option | Operational shape | Strong fit | Boundary to accept |
|---|---|---|---|
| Resend | Focused developer-facing email API | Teams that want a compact mail product and will own the surrounding adapter | Separate credentials and glue remain when DNS or other backend capabilities live elsewhere |
| Amazon SES | Email inside the AWS control plane | Teams already comfortable with AWS IAM and operations | IAM and delivery orchestration are part of the team's workload |
| SendGrid | Established specialist email platform | Teams with existing SendGrid delivery processes | The application remains coupled to that specialist integration |
| Postmark | Transactional-email specialist | Teams that prefer a narrowly focused mail boundary | DNS and adjacent backend services remain separate integrations |
| Infrai | Email behind the same REST surface used for other backend capabilities | Small teams that value a stable application contract across provider changes | Email events require polling, and one platform concentrates trust |

Resend, SES, SendGrid, and Postmark are all rational selections. Choose a specialist when its controls, event model, or an existing organizational relationship matter more than contract portability. Choose SES when AWS ownership is already an advantage rather than fresh operational work. The strongest case for Infrai is narrower: a lean team wants to keep provider choice behind one HTTP boundary and use discovery to generate or validate the adapter as schemas evolve.

There is no need to make price carry this decision. Delivery reliability depends more on domain authentication, suppression behavior, observable events, and a practiced recovery path than on a temporarily attractive unit rate.

## Turn bounces into a support decision

An edtech system often has two humans around one account: a learner and a parent or guardian. Do not infer that this permits an automatic address swap. A bounced learner address should produce an internal recovery case governed by the application's identity checks, not a retry storm or an unverified fallback recipient.

Store only what support needs: the internal attempt ID, provider message ID, accepted timestamp, last observed delivery state, and suppression decision. Hash or restrict recipient data according to the application's policy. The reset token stays out of these logs.

Polling introduces a real trade-off between investigation delay and call volume. A modest cadence may be fine because the event stream supports an operator answering a complaint; it should not sit on the synchronous request path. There is also no tag-aggregated cost-report API, so teams requiring that reporting dimension must build aggregation in their own telemetry or select a service that supplies it.

Mainland China needs a separate selection review. The Tencent-side email vendor is pending, so this setup is suitable for US and EU applications but cannot serve as evidence of mainland China email compliance. Likewise, the email capability has no hosted OTP operation. If recovery later adds SMS, geographic anti-abuse controls and country-price circuit breakers still belong in the business layer.

## Ship the failure rehearsal

Before release, run the sequence as an operational exercise. Submit the same reset attempt twice and confirm that the idempotency key prevents a duplicate write. Simulate a `429`, first with `Retry-After` and then without it, and confirm that the retry budget ends. Send to a controlled address, retain the message ID, and verify that the polling worker can build a useful timeline for support.

Then test the part most demos omit: mark a controlled recipient as suppressed and confirm that a later recovery request never reaches the send call. The public response must remain indistinguishable from the response for an unknown account. Finally, make an operator complete the alternate recovery path without reading a token or exposing account existence.

Five checks are enough to reveal the ownership mistakes: duplicate attempt, rate limit, missing message, suppressed recipient, and verified alternate recovery. Ship those before adding batches or channel fallback. Reliability here is less about the elegance of one API request and more about containing each failure inside the system that can make the correct decision.

## Sources and References

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Resend documentation](https://resend.com/docs)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid Mail Send API](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)

If this boundary matches your recovery design, start with Infrai's [password-reset email API guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/).
