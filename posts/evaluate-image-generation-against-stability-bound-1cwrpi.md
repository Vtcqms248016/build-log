# Evaluate Image Generation Against Stability Boundaries for Simple Support Chatbot Applications

**Short answer:** To evaluate OpenAI DALL-E against Stability AI for a simple support chatbot, test whether the integration preserves a valid, grounded response when generation is slow, unavailable, or unsuitable; the visual result is secondary to the response contract.

Evaluate OpenAI DALL-E against Stability AI and Midjourney for a simple support chatbot by testing that contract first, not by judging a sample image. For a bot answering questions over a private knowledge base, structured output correctness is the deciding constraint: an image can be useful, but it cannot replace the grounded answer, source identifiers, or escalation state.

The simple implementation is to send the user's message to an image service and return whatever comes back. That makes a quick demo. It also lets a failed or misleading illustration damage an otherwise correct support answer. I would measure schema-valid response rate, p95 latency, timeout behavior, and post-processing failures before copying the choice. A small fixed smoke set can expose missing fields before a larger evaluation does.

## How Should Teams Evaluate Image Generation Against Boundaries for a Simple Chatbot Application?

Treat the image as an optional attachment to a support response, not as the response itself. A question about replacing a filter may need a knowledge-base answer, a source identifier, and an illustration. The illustration cannot silently invent a model number or turn an uncertain answer into a confident one.

Start with the integration boundary. DALL-E, Stability AI, and Midjourney do not represent the same operational contract. Public material for DALL-E and Stability AI describes programmatic image-generation interfaces. Midjourney's public documentation describes its creation workflow and account features, so a team should verify the current official automation path before treating it as interchangeable with an API-first option. That difference matters more than a visually impressive prompt result for a support bot.

The evaluation should use questions from the actual private knowledge base: a replacement procedure, an ambiguous product name, a request for an illustration, and a question that must escalate. Keep the text answer and the image result as separate scores. A response passes only when its required fields validate, its evidence references are present, and the media status accurately reports what happened.

```ts
type SupportResponse = {
  answer: string;
  sourceIds: string[];
  needsHuman: boolean;
  image?: {
    status: "ready" | "not_requested" | "rejected" | "failed";
    url?: string;
    alt: string;
  };
};

function attachImage(
  response: Omit<SupportResponse, "image">,
  image: SupportResponse["image"]
): SupportResponse {
  return { ...response, image };
}
```

`sourceIds` remain required even when no image is produced, and `needsHuman` is independent of generation success. Validate the complete object after the model call and before serialization; validate the provider response before its URL enters the object.

## Compare the Integration Boundary, Not the Sample Image

Ask four practical questions:

- Can the application submit a prompt and receive a result through an officially documented interface?
- Can the result be associated with a request ID, timeout policy, and retry budget?
- Are moderation, dimensions, output formats, and retention behavior explicit?
- Can the application return a valid text answer when image generation is unavailable?

Do not collapse those answers into one quality score. A generator can produce a better-looking illustration and still be the worse runtime dependency if its operational contract is unclear. A clean request path does not make generated labels or product details accurate.

I keep retrieved product identifiers out of free-form image instructions unless the application has a verified asset or deterministic rendering route. An image model may draw a plausible control panel with the wrong button label. For procedural support, a screenshot or approved diagram is safer than a newly generated depiction. Generated media is better suited to low-risk orientation, such as a generic illustration of where a component sits.

```ts
type ImageRequest = {
  prompt: string;
  alt: string;
  purpose: "orientation" | "approved_asset";
};

function imageRequestForSupport(topic: string): ImageRequest | null {
  if (!topic.trim()) return null;
  return {
    prompt: `Create a generic orientation illustration for: ${topic}. Do not add labels, model numbers, controls, or safety instructions.`,
    alt: `Generic orientation illustration for ${topic}`,
    purpose: "orientation"
  };
}
```

That restriction is a product decision, not prompt decoration. Safety-critical steps stay in text sourced from the knowledge base and route the visual request to review or an approved asset library.

## The Failure Modes Hide in the Handoff

The brittle points are the handoffs: retrieved facts become prompt text, prompt text becomes an image request, an external result becomes a URL, and that URL becomes a field in a structured response. Each handoff needs validation.

Use a provider-neutral interface with explicit timeouts and an outcome enum. The adapter should return a normalized result, never a provider-shaped object that leaks into the rest of the application. Record latency, status, request correlation, and output validation separately.

```ts
type ImageOutcome =
  | { kind: "ready"; url: string; width: number; height: number }
  | { kind: "rejected"; reason: string }
  | { kind: "failed"; retryable: boolean };

interface ImageProvider {
  generate(request: ImageRequest, signal: AbortSignal): Promise<ImageOutcome>;
}
```

The fallback is intentional. A timed-out illustration should not turn a useful support answer into a failed chat turn. Retries need a cap, and duplicate user-visible attachments need correlation with the original request. Normalize dimensions and formats at the boundary; a small image-processing step can also remove unexpected metadata before storage or delivery.

Ship the contract first.

The main limitation is that generated imagery is a poor fit for safety-critical diagrams, exact product labels, or workflows that require a guaranteed public automation contract. In those cases, use an approved asset or a text-only answer, even when a provider can return an attractive image. That trade-off favors correctness over visual coverage, and it should be recorded as a product requirement rather than hidden in a prompt.

## What Should Be Measured Before Choosing?

Run a fixed evaluation set from the actual knowledge base: ordinary questions, ambiguous product names, diagram requests, and questions that must escalate. Score the text envelope separately from the image. A response passes only when the answer is grounded, the required fields validate, and the media state reports what happened.

Track four groups: correctness of `sourceIds` and escalation state; latency from request to complete envelope; failure and timeout behavior under concurrency; and visual usefulness for the limited orientation cases that are allowed. Add a human check for misleading labels or invented controls.

The decision rule is straightforward. Select the integration that preserves a stable, documented runtime contract and keeps the text answer correct when generation is slow, unavailable, or unsuitable. Keep the adapter replaceable. The support workflow, schema, and evaluation set are durable; the image backend is an implementation detail.

## References

- OpenAI, “Image generation guide”: https://platform.openai.com/docs/guides/images
- Stability AI developer documentation: https://platform.stability.ai/docs/getting-started
- Midjourney documentation: https://docs.midjourney.com/
- sharp documentation: https://sharp.pixelplumbing.com
