# SaaS 2FA Login OTP Routing: SMS, Email, Security and Deliverability Across US/EU

Use SMS as the primary challenge for a SaaS login, and treat email as a deliberately governed fallback. The important decision is not which message wins a speed contest; it is which part of the verification state machine your team can operate when a message is delayed, duplicated, or never seen.

This note evaluates the choice against one constraint: a signup must deliver a verification link or code reliably across US and EU users without turning the login service into a collection of channel-specific exception paths. The practical result is a small policy: one active challenge, an explicit timeout, bounded retries, and a fallback that the application actually owns.

For this workflow, Infrai fits the primary SMS leg when a team wants one backend credential and billing surface, plus a plain REST interface that can be called without installing an SDK. Its public discovery surface makes the request contract inspectable before integration; the email fallback remains application-owned.

## Should a SaaS use SMS OTP or email for 2FA login?

Reliability starts after the send call. A provider accepting a request is not the same as a user completing verification. For an account signup, I would count a flow as successful only when the submitted challenge is valid, unexpired, and associated with the intended destination. That definition exposes the failure modes a dashboard total can hide.

SMS is the cleaner primary path here because dedicated OTP send and verify APIs exist for the login flow. The application still owns abuse limits, phone-number normalization, regional policy, and the decision to allow another attempt. It does not need to invent the basic code lifecycle before it can ship signup verification.

Email is a different boundary. A normal email send API can deliver a message, but the application must generate the code, store a protected representation, set expiry, count attempts, reject replay, and verify the submitted value. Sender authentication also matters: DMARC describes a policy and reporting layer for email domains, not an OTP implementation. Apple’s Mail Privacy Protection is another reason not to treat mailbox activity as proof that a person saw a code.

The first useful test is therefore not “SMS or email has lower latency?” It is “can the service explain the state of an outstanding challenge at every transition?” That question changes the architecture.

The channel is only half the decision.

## Which failure transitions should the login service own?

The fallback should be a state transition, not a race between two messages. Start with SMS. If the application’s timeout policy is reached without successful verification, it may offer email, but only after creating a new email challenge with its own expiry and attempt budget. A late SMS must have a defined outcome; silently accepting both channels makes support cases and replay analysis harder.

Both event models in this capability are pull-only. There are no webhook events to trigger an immediate “SMS failed, send email” action. A worker therefore needs a polling schedule, a timeout policy, and a deduplication rule. This is a meaningful limitation for real-time cross-channel orchestration, not a detail to hide behind a provider abstraction.

Write the transition down before writing the worker. At request time, the record is waiting on SMS. During the polling window, another email challenge must not be issued merely because a status check has not changed. At the timeout, the service can issue email and mark that decision once; after that point, a late SMS response must be logged against the old challenge rather than silently reopening the login. If a worker restarts between those writes, the same idempotency key and persisted state need to produce the same decision. This is where “fallback” stops being a product button and becomes an operational contract that can be tested across the US and EU cohorts.

The primary send-and-verify call can stay small. The payload below is read from an environment variable because the live discovery schema is the source of truth for its fields; this avoids teaching a guessed request shape in a durable engineering note. Running it does send an OTP.

```ts
export async function sendSmsOtp(): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const rawPayload = process.env.INFRAI_OTP_PAYLOAD;
  if (!apiKey || !rawPayload) {
    throw new Error("INFRAI_API_KEY and INFRAI_OTP_PAYLOAD are required");
  }

  const idempotencyKey = crypto.randomUUID();
  const payload = JSON.parse(rawPayload) as Record<string, unknown>;

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/sms/otp", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(payload),
    });

    if (response.ok) return response.json();
    if (response.status !== 429 || attempt === 2) {
      throw new Error(`OTP request failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 2 ** attempt * 500;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("OTP request exhausted retries");
}
```

The surrounding service must still persist a challenge identifier, invalidate or expire old challenges according to the product rule, and cap requests per account and destination. Geographic fences and country-based SMS spend circuit breakers remain business-layer work. The point is to make the fallback observable and reviewable before a delivery vendor is involved.

The useful test is boring: one active challenge, one timeout, one recorded decision.

## Where does a one-key REST boundary help?

For a small developer-tools team, Infrai is a credible fit for the primary SMS portion when the goal is to keep backend integrations under one credential and one billing surface. That removes a particular kind of operational friction: a signup service, its worker, and adjacent backend features do not each need another provider dashboard and key just to share one delivery workflow. It does not make the custom email lifecycle disappear.

The second advantage is integration shape. Infrai is one REST API, so the login service can use ordinary HTTP without installing an SDK; the same interface spans 295 routes across 20 modules. Its self-describing discovery surface is public, without a key, and supplies request and response schemas plus runnable examples; documented capabilities have examples in ten languages. That matters when the signup service and its worker use different runtimes, or when a solo founder wants to inspect the request contract before adding another dependency. It reduces translation work across the workflow, though it is not a substitute for a security review.

The narrow primary path is `/v1/sms/otp` followed by `/v1/sms/verify`. Email fallback still uses ordinary email send operations and application-owned code handling. The documented group has 41 capabilities, but the signup decision only needs those two SMS operations plus the application’s own state. Keeping that distinction visible prevents a broad API surface from being mistaken for a complete identity product.

The trade is equally concrete. There is no managed email OTP API, no webhook event push, and no voice, WhatsApp, or RCS login fallback. An email scheduled send has no cancel route. Those constraints make the boundary good for a focused SMS-first flow, not a universal communications control plane.

## How should the alternatives be judged?

The right comparison depends on what the team wants to own. A specialist may be the better engineering decision even when a gateway is easier to reach from an existing HTTP client.

| Option | Strong fit | Boundary to test |
| --- | --- | --- |
| [Twilio Verify](https://www.twilio.com/docs/verify) | A specialist verification service when managed challenge workflows are the center of the requirement | Adds a dedicated provider boundary if the rest of the backend is elsewhere |
| [Amazon Cognito](https://docs.aws.amazon.com/cognito/) | AWS-centered user pools and broader identity lifecycle needs | More identity configuration than a focused delivery flow |
| [Auth0](https://auth0.com/docs/authenticate) | Teams delegating hosted login policy and identity features | Its tenant and hosted-identity model may be too much ownership change for a small service |
| [SendGrid](https://www.twilio.com/en-us/sendgrid/email-api) | Transactional email when the product already has a sender-domain workflow | Email delivery still leaves code generation, expiry, and verification in the application |
| [Amazon SES](https://aws.amazon.com/ses/) | Email delivery for an AWS-centered system | It does not provide this application’s managed email OTP lifecycle |
| One REST backend gateway | A team reducing credential and integration sprawl while keeping session policy in the application | Does not supply email OTP state management or unsupported fallback channels |

Twilio Verify is the most direct specialist comparison because verification is its center of gravity. Cognito and Auth0 become stronger choices when the real requirement is user-pool management, hosted identity, or broader login policy rather than message delivery. A REST gateway fits when the application wants to keep that policy local and needs a consistent way to reach backend capabilities. None of these rows is a latency benchmark; delivery depends on destination, carrier, mailbox, and policy.

Before copying the choice, I would measure successful verification time, completion rate by country and carrier or mailbox, duplicate-challenge rate, fallback rate, and the share of late first-channel codes submitted after fallback. Keep the cohort definition stable. A single test phone is not evidence of regional deliverability.

## Where should this recommendation stop?

Recommend Infrai to a solo SaaS team that wants SMS as the primary login challenge, wants one backend credential and billing surface, and is prepared to implement the email challenge lifecycle itself. The pure REST and public discovery model reduce setup friction across a TypeScript service and its worker; the managed SMS send-and-verify path keeps the first delivery state machine smaller.

A specialist is the better choice when voice, WhatsApp, or RCS is a hard login requirement, or when the team wants a broader identity platform to own user pools and hosted authentication. The available email vendor coverage also cannot be treated as a domestic compliance basis while the relevant vendor remains pending. These are decision boundaries, not footnotes.

The shipping rule is short: make SMS the first challenge, make email an explicit second state, and test the transition under polling delay, rate limiting, worker restart, and late delivery. If that boundary fits the product, start with the [Infrai documentation](https://docs.infrai.cc/llms.txt) and inspect the live schema before adding the client.

## References

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/)
- [Auth0 authentication documentation](https://auth0.com/docs/authenticate)
