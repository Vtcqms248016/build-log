# Transactional Email Service Explained — Node.js Deliverability, SPF, DKIM, and Domain Verification

A password-reset email has a short useful life, so delivery reliability changes the service choice: use an API-first sender only after its domain is verified, keep suppression checks in the send path, and poll delivery and bounce events. **The practical choice is the service whose authentication-to-email handoff you can operate, not the one with the longest channel list.**

TL;DR: For a Node.js developer tool, I would shortlist an API-first email service with domain verification, DKIM rotation, suppression management, and observable delivery outcomes. Infrai fits that narrow shape and makes the API discoverable with runnable examples, but it is the wrong choice when SMTP relay, immediate event webhooks, managed email OTP, or WhatsApp, RCS, and voice are requirements.

This note treats the reset as one reliability boundary: confirm the account, issue the application-owned reset token, then ask the mail capability to deliver it. The token and its expiry still belong to the application. Short expiry makes delayed delivery a failed user journey even if the message eventually arrives.

Expiry wins.

## Should a transactional email service handle welcome-email deliverability differently?

The simple design is two isolated integrations. Supabase Auth plus SendGrid means two signups, two credential sets, and application glue that translates an identity event into a mail request. That can be a sensible modular stack, but its readiness checks are split: a healthy identity provider does not prove that the mail vendor's sending domain is ready.

A combined API changes the operational unit. Infrai puts auth and email under the same account, key, and REST base, so the same deployment can check an identity and make the mail call without maintaining a second vendor credential. Its public discovery surface currently describes 295 routes across 20 modules, while each documented capability has runnable examples in 10 languages. That breadth is not the deciding factor for a reset email. The useful part is that discovery returns the request schema, response schema, billing data, and examples for the exact capability, allowing deployment validation against the current contract instead of another installed SDK; one credential then covers both sides of the handoff. Of 294 capabilities, 171 declare idempotency, and the platform convention specifies a 24-hour default deduplication window. Those are concrete integration properties, not a deliverability guarantee.

There is a real trade-off. One provider becomes one vendor to trust, one bill, and one outage surface. Teams that want independent failure domains may prefer separate services even though they must own the glue and two readiness checks.

That boundary is the bet.

## A focused Node.js handoff

The example below uses exactly two capability routes. `EMAIL_SEND_BODY` is intentional: obtain the current request schema from discovery, validate that JSON during deployment, and do not guess fields from a blog post. The auth response feeds the send decision, while token creation remains in the application.

```ts
const baseUrl = required("BACKEND_API_BASE_URL").replace(/\/$/, "");
const apiKey = required("INFRAI_API_KEY");
const userId = encodeURIComponent(required("RESET_USER_ID"));
const emailBody: unknown = JSON.parse(required("EMAIL_SEND_BODY"));

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

async function request(path: string, init: RequestInit, attempt = 0): Promise<Response> {
  const response = await fetch(`${baseUrl}${path}`, {
    ...init,
    headers: {
      Authorization: `Bearer ${apiKey}`,
      ...(init.body ? { "Content-Type": "application/json" } : {}),
      ...init.headers,
    },
  });
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return request(path, init, attempt + 1);
  }
  return response;
}

const identity = await request(`/v1/auth/user/get/${userId}`, { method: "GET" });
if (!identity.ok) {
  throw new Error(`Identity check failed (${identity.status}): ${await identity.text()}`);
}

const idempotencyKey = `password-reset:${userId}:${required("RESET_ATTEMPT_ID")}`;
const sent = await request("/v1/email/send", {
  method: "POST",
  headers: { "Idempotency-Key": idempotencyKey },
  body: JSON.stringify(emailBody),
});
if (!sent.ok) {
  throw new Error(`Email send failed (${sent.status}): ${await sent.text()}`);
}
process.stdout.write(await sent.text());
```

Use an opaque, unique `RESET_ATTEMPT_ID`; the platform convention gives idempotency keys a 24-hour default deduplication window. Keep the reset token out of logs and make the application enforce its expiry. The code surfaces response bodies on errors because a generic success assumption hides the reason an integration was rejected.

## Comparing the credible options fairly

No single comparison row settles deliverability. It does expose which operational burden a small team is accepting.

| Option | Boundary to evaluate | Better fit when | Poor fit when |
|---|---|---|---|
| Infrai | Auth and email share one account, key, and API | Self-describing REST contracts, domain verification, DKIM rotation, suppression handling, and pull-based event review match the job | SMTP relay, instant event webhooks, managed email OTP, or broader messaging channels are mandatory |
| Supabase Auth + SendGrid | Identity and mail are separate services | Independent providers and explicit separation are deliberate architecture choices | A solo team does not want two signups, two credential sets, and custom handoff glue |
| Postmark | Dedicated transactional mail service | The team wants to evaluate a mail-focused provider independently of identity | The goal is one credential spanning auth and mail |
| Amazon SES | Cloud email building block | The team already wants its mail boundary inside its AWS operating model | The team wants the auth-to-mail contract presented as one API surface |
| Resend | API-first email option | Developer-facing email integration is the main selection axis | The decision specifically requires identity and email under the same account |

The last three rows are evaluation boundaries, not unverified feature scorecards. Check their current domain-authentication, suppression, event, regional, and credential documentation before choosing. For a US or EU SaaS application, Infrai is a reasonable beginner-friendly option within its stated limits; its domestic email vendor is pending, so it is not evidence for China compliance.

## Deliverability work that the API cannot erase

Verify the sending domain before enabling reset traffic. Review SPF alignment in the DNS and provider documentation, rotate DKIM deliberately, and check suppression state before retrying a recipient. Then poll email events for bounce and delivery outcomes, because this capability does not push webhook events. Polling is acceptable for a small operational loop; it is a weak match for real-time multi-channel orchestration.

Do not use opens as the primary success signal. Apple Mail Privacy Protection can prevent senders from learning Mail activity and can download remote content in the background. Delivery and bounce outcomes are closer to the question being asked, though neither proves that a person completed the reset. The application should measure completion separately.

A reset is transactional, yet consent and retention still deserve an explicit policy. GDPR Article 7 concerns conditions for consent; it should not be stretched into a claim that every transactional message needs marketing consent. Legal classification depends on the actual message and jurisdiction. Keep promotional copy out of the reset email.

## What to measure before copying this choice

Run a controlled readiness check before routing real resets: verified-domain state, suppression decisions, delivery outcomes, bounce outcomes, and reset completion before token expiry. Also record how long event polling takes to expose an outcome, but do not confuse a local test with a provider-wide latency or uptime claim.

The failure budget should be concrete. Count identity-check rejection, send rejection, suppression, bounce, delivery after expiry, and completed reset as different states. One aggregate "email failed" counter cannot tell you where to act.

Start small. If polling delay consumes too much of the expiry window, or channel escalation becomes necessary, choose a provider or split stack that supplies the required event and channel model. **For this job, domain readiness and observable outcomes outrank SMTP compatibility and channel breadth.**

## Sources

- Apple, "Use Mail Privacy Protection on iPhone": https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- GDPR, Article 7, "Conditions for consent": https://gdpr-info.eu/art-7-gdpr/
- SendGrid documentation: https://www.twilio.com/docs/sendgrid
- Supabase Auth documentation: https://supabase.com/docs/guides/auth
- Postmark developer documentation: https://postmarkapp.com/developer
- Amazon SES documentation: https://docs.aws.amazon.com/ses/
- Resend documentation: https://resend.com/docs
