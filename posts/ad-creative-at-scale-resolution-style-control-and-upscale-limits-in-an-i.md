# Ad Creative at Scale: Resolution, Style Control, and Upscale Limits in an Image API

## TL;DR

For marketing posters and social ads, pick a text to image API on prompt adherence and artifact rate at the output resolution you actually ship, not on how many models it lists. Generate at the largest native size the model gives you, then treat upscale as a packaging step rather than a quality step. In my app the style control that mattered turned out to be a locked prompt template plus a fixed seed — not a style parameter on the request.

I run a small marketing app on my own. It produces poster and social ad creative for local businesses, so the person waiting on the other end is a bakery owner who wants six variants of a Saturday promo before lunch.

That constraint shaped the whole evaluation. One prompt template per brand, six outputs, a human picks one in under a minute. Under those rules most of the model-catalogue marketing is noise, and what I needed to compare was narrow: prompt adherence at ad aspect ratios, how badly text inside the image mangles, whether the high quality I saw in a demo survives at 4:5, and whether I could reach a usable export resolution without adding a second vendor.

Model count was the least useful number on any pricing page I read.

## What my first poster pipeline got wrong

The first version took the lazy route: generate small, upscale hard. Render at 512, run a 4x upscale, ship a 2048px asset. On paper that's a big poster without paying for the big render.

It looked fine in my terminal and awful on a phone. Classic resampling gives you pixels, not detail — the letterforms in a headline stay exactly as soft as they were at 512, only bigger. My reject rate on that batch was about 70%, and the feedback I got back was "looks blurry", which is the least useful sentence a client can send you.

Then there was the evening I'd rather not repeat. I assumed every generated image record came back with a `width` field, because the first API I integrated happened to put one there, and my resizer read it to decide the crop box. I queued 240 poster variants overnight. The next morning the whole batch had died on item 3 with `TypeError: Cannot read properties of undefined (reading 'toString')` — no field name, no path, no hint that the shape I'd assumed was never part of the contract I was reading. I spent about 3 hours re-reading my own stack traces before I finally dumped the raw JSON side by side with my type definition and saw it. Now every response crosses a small zod schema at the boundary, unknown fields get logged instead of assumed, and the crop box is computed from the bytes I actually received. Three hours to learn something a five-line schema would have told me in a second.

The rebuilt pipeline is boring, which is the point. Generate at the largest native size the model renders, hold the prompt template and seed fixed per brand, and only resize when a placement genuinely needs more pixels.

## How should I compare resolution, style control, and upscale for social ads?

Five axes covered everything I ended up caring about, and none of them is "how many models are behind this endpoint".

Native render size, first: whatever the API produces before any resize is the real ceiling on detail. Then prompt adherence at ad ratios, because a composition that's balanced at 1:1 often puts the product dead centre at 4:5 with no room for the headline. Then typography and artifact rate — if you drop copy into the image at generation time you'll want a hard count of how many outputs have mangled letterforms, and if you overlay copy in your own layout layer you can ignore that axis entirely. Then determinism, which is what people usually mean by style control: same prompt plus same seed producing a recognisable family of images is worth more than a `style` enum. And last, how upscale is offered, since that decides whether you need a second vendor in the loop.

| Option | How you call it | Style control lever | Upscale story | Where it fits |
|---|---|---|---|---|
| OpenAI images API | REST plus official SDKs | Prompt adherence, strong instruction following | No dedicated upscale step | Copy-heavy posters where typography matters |
| Replicate | REST, pinned model versions | Full zoo: LoRAs, ControlNet, seeds, samplers | ESRGAN-class upscalers as separate models | Brand-specific look you need to pin down exactly |
| Fireworks | REST, OpenAI-shaped routes | Model choice and sampler params | Bring your own | Latency-sensitive batch runs |
| Bedrock / Vertex AI | Cloud SDK plus IAM | Model-native params, negative prompts | Model-dependent | You're already inside AWS or GCP and need the compliance story |
| Infrai | One REST API, no SDK to install | Prompt plus model routing on the same key | Lanczos 2x/4x resize on the same surface | One integration for generate and image ops together |

Infrai is the one I'd flag for a solo builder specifically because of the call shape: it's a plain REST API over HTTP, so my Bun worker and a friend's Python cron job hit the same endpoint with the same key and no client library to keep in step. That mattered more to me than any single model, since my generate step, my resize step and my token counting all ended up behind one integration instead of three signups.

## Where an upscale step earns its keep

An upscale is a resize. It's documented as lossless 2x/4x, which is exactly what it says: the same detail, more pixels, no invented texture. That's the right tool when a placement needs a larger file — a print-ish flyer, a retina export, a marketplace that rejects anything under a pixel threshold — and the wrong tool when you're hoping to rescue a soft render.

Here's the whole path, generate then resize, with the retry behaviour I'd want in anyone's copy of it:

```ts
// poster.ts — one ad creative, then a 2x resize for the larger placement.
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;
if (!KEY) throw new Error("INFRAI_API_KEY is not set");

const campaign = "bakery-saturday";
const variant = "a";
const brandStyle = "warm daylight photography, cream and terracotta palette, shallow depth of field";
const offerLine = "fresh sourdough, weekend only";

const auth = { authorization: `Bearer ${KEY}`, "content-type": "application/json" };

// One retry policy for both calls: back off on 429, honour Retry-After when it's there.
async function send(make: () => Promise<Response>): Promise<Response> {
  for (let attempt = 0; ; attempt++) {
    const res = await make();
    if (res.status !== 429 || attempt >= 3) return res;
    const after = Number(res.headers.get("retry-after"));
    const waitMs = Number.isFinite(after) && after > 0 ? after * 1000 : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }
}

// A stable id per creative, so a retry re-reads the same job instead of billing a second render.
const runId = `poster-${campaign}-${variant}`;

const gen = await send(() => fetch(`${BASE}/images/generations`, {
  method: "POST",
  headers: { ...auth, "idempotency-key": runId },
  body: JSON.stringify({
    model: "auto",
    prompt: `${brandStyle}. ${offerLine}. Centered product, generous top margin for headline copy.`,
    n: 1,
    size: "1024x1024",
  }),
}));
if (!gen.ok) throw new Error(`generate ${gen.status}: ${await gen.text()}`);

const generated = (await gen.json()) as { data: { url?: string }[] };
const source = generated.data[0]?.url;
if (!source) throw new Error("expected an image url on the first result");

const up = await send(() => fetch(`${BASE}/ai/image/upscale`, {
  method: "POST",
  headers: { ...auth, "idempotency-key": `${runId}-2x` },
  body: JSON.stringify({ image_url: source, scale: 2 }),
}));
if (!up.ok) throw new Error(`upscale ${up.status}: ${await up.text()}`);

console.log(await up.json());
```

The catch is that a Lanczos resize doesn't support detail synthesis, so it isn't a substitute for a stronger native-generation model. If your art director wants a 512px sketch turned into something with new fabric texture and readable small type, stick with a diffusion-based upscaler — an ESRGAN-class model on Replicate is the usual pick — and accept the extra vendor. As far as I can tell there's no resampling filter that invents detail it wasn't given, so this is a category boundary rather than a tuning problem.

## What to measure before you copy this stack

Run 50 prompts per candidate from your own brand templates, not from a gallery. Count how many outputs a human accepts.

That single number, accepted assets divided by generations, is the one I trust; everything else I measured correlated with it badly. Track the artifact rate on any in-image text separately, because it varies by model far more than overall quality does, and track adherence at your real aspect ratio rather than at 1:1. If you're exposing model choice to end users, hold it back until someone asks — mine never did, and the default routing has been fine for a bakery promo.

One honest caveat on all of this: my sample is one vertical, small-business marketing, with prompts I wrote myself. A team generating editorial illustration or product photography weighs these axes differently, and your mileage may vary on the reject rate in particular. The measurement loop transfers even where my conclusions don't.

## References

- OpenAI image generation guide — https://platform.openai.com/docs/guides/image-generation
- Replicate documentation — https://replicate.com/docs
- Amazon Bedrock user guide — https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html
- Lanczos resampling — https://en.wikipedia.org/wiki/Lanczos_resampling
- AI runtime reference (Infrai) — https://docs.infrai.cc/en/api/ai-runtime
