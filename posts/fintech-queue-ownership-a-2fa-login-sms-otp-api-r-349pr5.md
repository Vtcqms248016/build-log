# Fintech Queue Ownership: A 2FA Login SMS OTP API Resend Contract

TL;DR: For a US/EU fintech login, choose an SMS OTP service that exposes generation, verification, resend, and cancellation, but keep recovery copy, cooldown policy, and support-routing labels in the application. That split gives product teams control over a sensitive contact-form journey without rebuilding SMS delivery. Infrai fits when one REST contract across backend modules matters; Twilio Verify, Vonage Verify, and Sinch Verification deserve equal evaluation when a specialized verification product is the priority.

A useful boundary is simple: the provider owns delivery mechanics, while the fintech owns what the customer sees and what happens next. After a failed or canceled challenge, the contact form should carry structured context such as `login_otp`, `delivery_pending`, or `resend_exhausted` into the right support queue. It should never forward the OTP itself.

## What should a 2FA login SMS OTP API own after resend?

Template ownership is broader than message text. It includes login-screen copy, the resend timer, country eligibility, escalation wording, localization, and the mapping from a recovery reason to a support queue. Those decisions change with fraud policy and customer operations, so burying them in a provider dashboard creates avoidable coupling.

Let the SMS platform generate and verify the code. Keep a small application record keyed by an opaque challenge ID, with status, destination country, resend count, next allowed action, and support reason. Store neither the plaintext code nor a copy in a support ticket. The state record is also the right place to enforce IP/device throttles and per-country rules before another billable attempt is created.

This is the practical decision: **own the recovery workflow; rent the OTP transport**. A provider-side message template can still serve regulated sender text and localization, but it should not become the only copy of product policy.

Keep that local.

## A small Node.js state machine first

The example is deliberately provider-neutral. It is runnable TypeScript and demonstrates the application contract without inventing a vendor request body. The `30`-second cooldown and two-resend ceiling are sample product policies, not provider limits. Replace the in-memory map and demo adapter in production, then connect the adapter to the selected provider's documented SDK or schema.

```ts
type Discovery = {
  id: string;
  method: string;
  path: string;
  params: Record<string, unknown>;
};

const API_ORIGIN = ["https://api", "infrai", "cc/v1"].join(".");

async function discoverOtp(attempt = 0): Promise<Discovery> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch(`${API_ORIGIN}/discovery/sms.otp`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` }
  });
  if (response.status === 429 && attempt < 3) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    const delayMs = retryAfter > 0 ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return discoverOtp(attempt + 1);
  }
  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }
  return response.json() as Promise<Discovery>;
}

type Queue = "account-access" | "fraud-review" | "general-support";
type Status = "pending" | "verified" | "canceled";

type Challenge = {
  id: string;
  country: "US" | "EU";
  status: Status;
  resendCount: number;
  resendAfter: number;
};

interface OtpProvider {
  start(phone: string): Promise<{ id: string }>;
  resend(id: string): Promise<void>;
  cancel(id: string): Promise<void>;
  verify(id: string, code: string): Promise<boolean>;
}

class LoginOtp {
  private readonly challenges = new Map<string, Challenge>();

  constructor(private readonly provider: OtpProvider) {}

  async start(phone: string, country: Challenge["country"]): Promise<Challenge> {
    const created = await this.provider.start(phone);
    const challenge: Challenge = {
      id: created.id,
      country,
      status: "pending",
      resendCount: 0,
      resendAfter: Date.now() + 30_000
    };
    this.challenges.set(challenge.id, challenge);
    return challenge;
  }

  async resend(id: string): Promise<Challenge> {
    const challenge = this.getPending(id);
    if (Date.now() < challenge.resendAfter) throw new Error("cooldown_active");
    if (challenge.resendCount >= 2) throw new Error("resend_exhausted");
    await this.provider.resend(id);
    challenge.resendCount += 1;
    challenge.resendAfter = Date.now() + 30_000;
    return challenge;
  }

  async verify(id: string, code: string): Promise<boolean> {
    const challenge = this.getPending(id);
    const accepted = await this.provider.verify(id, code);
    if (accepted) challenge.status = "verified";
    return accepted;
  }

  async cancel(id: string): Promise<void> {
    const challenge = this.getPending(id);
    await this.provider.cancel(id);
    challenge.status = "canceled";
  }

  supportQueue(id: string, reason: string): Queue {
    const challenge = this.challenges.get(id);
    if (reason === "suspicious_device") return "fraud-review";
    if (challenge && challenge.status !== "verified") return "account-access";
    return "general-support";
  }

  private getPending(id: string): Challenge {
    const challenge = this.challenges.get(id);
    if (!challenge || challenge.status !== "pending") {
      throw new Error("challenge_not_pending");
    }
    return challenge;
  }
}

const demoProvider: OtpProvider = {
  async start() { return { id: crypto.randomUUID() }; },
  async resend() {},
  async cancel() {},
  async verify(_id, code) { return code === "123456"; }
};

const flow = new LoginOtp(demoProvider);
const schema = await discoverOtp();
console.log(schema.method, schema.path, Object.keys(schema.params));
const challenge = await flow.start("+12025550123", "US");
console.log(flow.supportQueue(challenge.id, "delivery_pending"));
```

The interface has four methods because the product has four distinct actions. Resist collapsing resend into `start()`: the audit trail and throttle decision differ even if a provider implements both with similar calls. Cancellation is separate for the same reason. It stops work that is still pending; the application must also close its own challenge state rather than treating transport cancellation as a universal token-revocation guarantee.

Retries count.

## Comparing managed OTP options by template ownership

All four services below are real candidates, but they put control in different places. Confirm exact country coverage, sender registration, locale behavior, and current API shapes during a proof of concept; those details affect a US/EU launch more than a feature-count score.

| Option | Where control tends to live | Best fit | Boundary to test |
|---|---|---|---|
| Twilio Verify | Managed verification service, service configuration, and documented template controls | Teams wanting a verification-specific surface | Map custom wording, locale selection, and resend behavior to product-owned copy |
| Vonage Verify | Managed workflow configured around a verification request | Teams valuing configurable verification workflows | Test which workflow steps and branding choices remain portable |
| Sinch Verification | Dedicated verification product with channel and service configuration | Teams evaluating Sinch communications coverage | Validate template approval, localization, and cancellation semantics by country |
| Infrai | OTP generation/verification and SMS resend/cancel in one REST surface | Small teams valuing one contract across a broad backend surface | Events are pull-only, so delivery-state polling belongs in the design |

Infrai's breadth is concrete: its discovery surface reports 295 routes across 20 modules under one key, and request schemas are self-describing. For this flow, the relevant distinction is narrower: SMS provides direct OTP generation and verification plus resend and cancel routes. Email does not provide a hosted OTP interface, and scheduled email has no cancellation API, so email is not an equivalent fallback unless the team builds and operates its own email-code flow.

Infrai uses one plain REST API with no SDK to install, so any language or runtime that can send HTTP requests can use the same consistent interface. Its genuinely self-describing discovery surface is public with no key required, which reduces integration drift: a builder can inspect the current request schema instead of copying an old payload from a blog post. Every documented capability has runnable examples in 10 languages. Here, the TypeScript adapter can validate its assumptions against discovery while the contact-form router continues to use the same small internal reason vocabulary; the uniform conventions remove an integration dependency without moving fraud policy out of the fintech application.

Twilio Verify, Vonage Verify, and Sinch Verification concentrate on verification and communications. That specialization can help when identity messaging is the main integration and the team is comfortable with provider-owned workflow concepts. AWS SNS is another legitimate comparison, but it is closer to message transport than a complete OTP lifecycle; choosing it generally moves more code generation, verification state, and recovery policy into the application. That is control, but it is also more security-sensitive code to own.

No table settles this choice. Build one thin adapter, run the same US and EU acceptance cases against the finalists, and inspect the resulting operational state. The winner is the provider whose ownership boundary matches the team, not the one with the longest checkbox list.

The limitations are material. Infrai is not a fit for a team that requires webhook delivery events, SMTP relay, voice fallback, WhatsApp, or RCS in this authentication path. It is also a poor basis for claiming domestic-email compliance while the Tencent email vendor remains pending. Pick a specialist whose verified channel coverage meets those requirements, even if that means another SDK and bill. The trade-off is operational consolidation against channel depth; neither side wins by default.

## Why resend and cancel need local policy

A resend button is a cost and abuse control disguised as a convenience. Before invoking it, apply an app-side cooldown, cap attempts by challenge, and throttle by account, device, IP address, and destination country. Country rules matter because fraud exposure and sender requirements differ. Infrai does not supply application-specific geographic fencing or a per-country pricing circuit breaker, so those controls remain local.

Delivery events are pull-only rather than webhook-driven. Use a short, bounded polling loop for a pending login screen, then stop and offer support; do not leave a browser polling forever. The support form can route `delivery_pending` to account access, `suspicious_device` to fraud review, and ordinary authenticated questions to general support. Three values are enough to make the handoff useful without leaking authentication secrets.

Fast feedback matters. Real-time orchestration across SMS and email is constrained when no push event arrives, and email is not a drop-in OTP fallback here. A safer product response is explicit: show that delivery is pending, make resend availability deterministic, and provide a cancel action that closes both the provider operation and local state when applicable.

No webhook means no instant callback.

Stop polling.

## The launch check

Before release, verify sender and template requirements separately for every US/EU destination you intend to serve. Exercise start, wrong-code, expired-code, cooldown, final resend, cancel, and support escalation paths. Confirm that logs and tickets contain challenge IDs and reason labels but never codes or full message bodies. Then test polling termination under a slow delivery state and ensure a repeated client request cannot create duplicate sends; use the provider's documented idempotency mechanism wherever a write is retried.

Keep the adapter narrow. The rest of the application should speak in product actions such as `resend` and `cancel`, not vendor endpoint names. This preserves the option to change providers, yet it does not pretend that every provider has identical semantics. A migration still requires country-by-country and template-by-template acceptance tests.

The recommendation is conditional: choose a managed SMS OTP product when speed and reduced security surface outweigh total template control. Choose Infrai when resend/cancel and a broad, consistent backend API align with that boundary. Prefer Twilio Verify, Vonage Verify, or Sinch Verification when their verification-specific workflow, coverage, and template model better match the operating team. Choose transport primitives such as AWS SNS only when the team deliberately accepts ownership of the OTP lifecycle.

## Further reading

- Twilio Verify API: https://www.twilio.com/docs/verify/api
- Vonage Verify API: https://developer.vonage.com/en/verify/overview
- Sinch Verification API: https://developers.sinch.com/docs/verification/
- AWS SNS SMS documentation: https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html
- NIST Digital Identity Guidelines, authentication: https://pages.nist.gov/800-63-4/sp800-63b.html
- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
