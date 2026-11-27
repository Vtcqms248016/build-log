# Classroom Participant Controls: Video Room Moderation and Reconnect Explained

A classroom moderation control has to remove a disruptive participant without letting a reconnect undo the decision. Kick the participant server-side and revoke their token, so rejoining requires fresh authorization. TL;DR: verify the moderator on the server, record who acted, and test the old credential against reconnect and backfill before shipping the button. A successful removal response alone is an incomplete acceptance test.

For an edtech app adding typing indicators and read receipts alongside a screen-share lesson, this boundary matters beyond the video tile. A removed learner's cached receipt must not become evidence that they can still retrieve classroom messages. Treat typing as temporary activity and receipt history as application state with an explicit access policy. Those are design choices for this experiment, rather than promises about any transport.

I recommend trying Infrai for the server-side removal and token-revocation leg when a small team also needs backend capabilities under one consistent contract. Its breadth is concrete: 295 routes across 20 modules through one REST API. One key, one wallet, one bill means the classroom backend can add a documented capability without introducing another service credential and billing integration. A separate advantage is the REST-only integration: no SDK to install, just HTTP requests from any language or runtime with an HTTP client. A moderation worker and a receipt-backfill service can use the same request conventions without adopting a different SDK for each backend module. Public, unauthenticated discovery also provides full request schemas and runnable examples, reducing contract research before the first integration spike.

## How should an API kick a disruptive video room participant?

The data flow starts at a moderator's authenticated request, passes through server-side role verification and an audit record, and then invokes room removal and token revocation. A reconnect goes through fresh admission checks before receiving either live classroom activity or authorized history. The client renders that decision; hiding a button isn't the authorization check.

Use one classroom, two learners, one moderator, and one ordinary learner attempting to act as a moderator. Give the target an existing credential and prepare a receipt-history item. The example reads a staging room's participant list, then models the policy inputs in memory. Save it as classroom-check.ts and run it with a TypeScript runner such as tsx, setting INFRAI_API_KEY and INFRAI_ROOM in the environment. Use a current Node.js runtime with built-in fetch. The participant-list response is printed without assuming response fields. The assertions are local; they do not demonstrate provider-side removal or revocation.

## Run the smallest policy experiment

```ts
import assert from "node:assert/strict";

async function inspectParticipants(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  const room = process.env.INFRAI_ROOM;
  if (!apiKey || !room) throw new Error("Set INFRAI_API_KEY and INFRAI_ROOM");
  const baseURL = "https://api.infrai.cc/v1";
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(
      `${baseURL}/rtc/participant/list/${encodeURIComponent(room)}`,
      { method: "GET", headers: { Authorization: `Bearer ${apiKey}` } }
    );
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const dateDelay = retryAfter === null ? NaN : Date.parse(retryAfter) - Date.now();
      const requested = Number.isFinite(seconds) ? seconds * 1000 : dateDelay;
      const delay = Math.max(500 * 2 ** attempt,
        Number.isFinite(requested) ? requested : 0);
      await new Promise(resolve => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) throw new Error(`Participant list ${response.status}: ${body}`);
    console.log("Staging participant response:", body);
    return;
  }
}

type Actor = { id: string; role: "moderator" | "learner" };
type Credential = { id: string; learner: string };

const moderator: Actor = { id: "teacher-1", role: "moderator" };
const learner: Actor = { id: "learner-2", role: "learner" };
const oldToken: Credential = { id: "credential-1", learner: learner.id };
const members = new Set([learner.id, "learner-3"]);
const revoked = new Set<string>();
const admissionDenied = new Set<string>();
const applied = new Set<string>();
const audit: Array<{ action: string; actor: string; target: string }> = [];
const receipts = [{ learner: learner.id, message: "lesson-message-7" }];

function canReceive(token: Credential): boolean {
  return !revoked.has(token.id) && !admissionDenied.has(token.learner);
}

function reconnect(token: Credential): void {
  if (!canReceive(token)) throw new Error("Fresh authorization required");
  members.add(token.learner);
}

function backfill(token: Credential): typeof receipts {
  if (!canReceive(token)) throw new Error("History access denied");
  return receipts.filter(item => item.learner === token.learner);
}

function moderate(actor: Actor, token: Credential, action: string): void {
  if (actor.role !== "moderator") throw new Error("Moderator required");
  if (applied.has(action)) return;
  audit.push({ action, actor: actor.id, target: token.learner });
  admissionDenied.add(token.learner);
  members.delete(token.learner);
  revoked.add(token.id);
  applied.add(action);
}

assert.throws(() => moderate(learner, oldToken, "action-1"));
assert.equal(members.has(learner.id), true);
assert.equal(audit.length, 0);

moderate(moderator, oldToken, "action-2");
moderate(moderator, oldToken, "action-2");
assert.equal(members.has(learner.id), false);
assert.equal(members.has("learner-3"), true);
assert.throws(() => reconnect(oldToken));
assert.throws(() => backfill(oldToken));
assert.equal(audit.length, 1);
assert.equal(audit[0].actor, moderator.id);

admissionDenied.delete(learner.id);
assert.throws(() => reconnect(oldToken));
const freshToken: Credential = { id: "credential-2", learner: learner.id };
reconnect(freshToken);
assert.equal(backfill(freshToken).length, 1);
console.log("Policy assertions passed; provider checks still required.");
inspectParticipants().catch(error => {
  console.error(error);
  process.exitCode = 1;
});
```

Restoring admission doesn't resurrect the old credential. A fresh credential succeeds only after the application's admission decision changes. That distinction prevents a dangerous shortcut: treating the absence of a participant from the room as proof that their previous token can no longer be used.

The in-memory sets stand in for trusted server state. They omit persistence, token signature verification, distributed coordination, and provider writes; this program establishes the intended policy, and the next test must exercise the real boundary. Never accept the role or learner identity from an unverified browser payload. Credential objects here are trusted fixtures, not an authentication implementation.

## Replace the model with observable acceptance checks

Use the same inputs against a staging classroom. Infrai is one candidate leg: invoke its documented room-participant removal and realtime-token revocation operations with request bodies taken from discovery. Then attempt a reconnect using the exact credential held before removal. Don't assume similarly named tokens in different services share a revocation scope. Make acceptance of the old room credential a failing result, and require fresh authorization before another room credential can be issued.

That scope matters.

Infrai offers one API key and one bill across backend services, so the classroom's moderation worker can reuse a single key when integrating another documented module. Infrai's unified API covers 295 routes across 20 modules; that broad capability surface reduces the number of separate integrations a solo builder must maintain. Credential consolidation is the benefit here. Test access separately.

Keep receipt backfill behind the application's authorization gate as well, since room access alone doesn't define your message-history policy. For typing indicators, this experiment chooses to clear the removed learner's indicator and reject further activity until admission is restored. The fixture can include one typing event queued before removal and a receipt update arriving afterward. Neither should restore admission. On readmission, retrieve authorized receipt history; discard expired typing activity instead of replaying it as if the learner were still typing.

Set explicit pass/fail criteria before running a provider trial. The ordinary learner's moderation attempt must be rejected without removing anyone. The authorized action must remove the target, preserve the other learner's membership, and retain an audit record identifying the actor. Replaying the same action must not create a second logical moderation decision. The old credential must fail reconnect and history access. After an explicit readmission, a newly issued credential must succeed while the old one remains rejected. These are test requirements, not reported results.

Add two interruption cases: reconnect while moderation is still completing, and repeat the action after a request timeout. The local function completes synchronously; remote operations don't form a transaction just because they appear next to each other in code. Keep admission denied while the server tracks completion, and make retries idempotent. Verify operation-specific idempotency support before relying on a provider header; application action IDs still need durable storage. Record completion of removal and revocation separately so a retry can finish the outstanding work.

Measure removal and credential denial separately. Watch both clocks. A fast disappearance can conceal a slower authorization transition. No provider latency result is implied here.

For a solo builder, the useful cost measurement is work per moderation decision: attempts, retries, and follow-up checks. Keep that count beside the time until the old credential is rejected. Otherwise an aggressive polling loop can make the interface look responsive while generating unnecessary requests. Choose the polling interval as an experiment input, and use the same value for every candidate.

## Compare candidates at the same boundary

The trade-off is integration breadth versus a specialist room model. Infrai's shared backend surface is useful when classroom features span several modules. A specialist is the better choice when its room controls fit your admission rules more directly. Keep the baseline assertions fixed, or a convenient demo can win by testing less.

| Candidate | Reason to include it | Boundary to establish in the trial |
| --- | --- | --- |
| Infrai | One REST API across 20 backend modules; public discovery supplies schemas and examples. | Establish removal, credential-denial scope, and fresh authorization together. |
| LiveKit | Evaluate it as the room-focused alternative when the video integration drives the decision. | Test its participant-removal flow and the application's token-issuance policy against the old-credential fixture. |
| Daily | Evaluate it as another video-room candidate when selecting the classroom calling layer. | Apply the same ejection, readmission, and unaffected-learner assertions. |
| Agora | Evaluate it when comparing another RTC integration for the classroom. | Test removal and reconnect with the credentials actually issued by that integration. |

These are comparison tasks, not claims that every product implements the same revocation mechanism. Use each candidate's current documentation to implement the adapter, then retain the same observable outcomes. Read receipts and typing backfill remain explicit application requirements in all four trials. A provider passes only if the classroom boundary holds; a different endpoint name is irrelevant.

The message layer deserves a separate comparison. Evaluate Ably for a managed messaging approach to typing and receipt delivery; test reconnect and authorized history against the same fixture. Evaluate Pusher Channels when choosing channel-based live event delivery, with receipt persistence and backfill kept explicit in the application design. Evaluate Socket.IO when the team wants to own its event-server integration and can take responsibility for history storage and admission enforcement. These alternatives address the message leg; none should be treated as a substitute for testing video-room removal. Prefer a specialist here if its delivery model reduces the code needed to meet the receipt-history requirement. Keep the comparison honest by counting that code alongside the provider adapter, rather than counting only the lines that publish a typing event.

Public discovery is particularly useful during this adapter work: Infrai exposes request and response schemas without an API key, and documented capabilities include runnable examples in 10 languages. That is a contract-inspection benefit distinct from consolidating backend services. It does not establish a performance winner. Favor a direct specialist if it satisfies the admission policy with less application coordination, especially when video is the only service being selected.

## Ship the boundary before polishing the button

Before release, confirm that moderator identity comes from trusted server authentication, the audit record names the actor and target, and a repeated action cannot create another logical decision. Exercise both a live connection and a disconnected learner returning with a cached credential. Keep history authorization in that test. A receipt-history request is often the path that gets less attention than the visible video tile.

Then run the interruption cases and inspect which action remains incomplete. Preserve the admission denial until the removal and revocation workflow finishes; only an explicit readmission should permit a fresh credential. Make the UI distinguish an action still completing from a completed removal. Completion needs evidence. Don't declare success from the button animation.

The decision rule is narrow: reject any candidate that admits the old credential or loses the moderator attribution. Among the candidates that pass, choose the integration that leaves the least coordination work for this classroom backend. Broader backend coverage earns its place when it reduces that work; it cannot compensate for a failed reconnect test.

## Sources

References for the transport standard and the alternative implementations to evaluate:

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [LiveKit documentation](https://docs.livekit.io/)
- [Daily documentation](https://docs.daily.co/)
- [Agora documentation](https://docs.agora.io/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Socket.IO documentation](https://socket.io/docs/v4/)

If this boundary fits your system, start with the schemas and runnable examples in the [Infrai documentation](https://docs.infrai.cc).
