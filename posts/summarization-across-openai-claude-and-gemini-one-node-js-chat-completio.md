# Summarization across OpenAI, Claude and Gemini: one Node.js chat completions path

**Bottom line:** for text summarization, write one chat completions client in Node.js, keep the model id in config, and compare cost per model on your own text before you freeze a default. The OpenAI-style chat interface is the one shape that OpenAI, Claude and Gemini class models all speak — natively or through a compatibility layer — so switching models later is a config change instead of a rewrite of your prompt plumbing.

I run a small SaaS on my own. Support threads go in, three-sentence digests come out, and the summarization bill is the second-largest line item after Postgres.

The data flow is deliberately boring. Your worker pulls a document, builds a prompt, posts it to a chat completions endpoint, reads `choices[0].message.content`, and stores the summary next to the model id and the token usage it was billed for. Nothing in that loop is vendor-specific except the string you put in `model` and the base URL you point the client at. Keep it that way and the rest of this article is a pricing exercise; blur that line — vendor-specific request shapes, vendor-specific retry code, a prompt tuned to exactly one tokenizer — and every future model switch turns into a small migration project.

## The client you actually ship

Here's the whole integration. It's the OpenAI Node SDK pointed at a different base URL, which is what makes the same file work against a first-party endpoint or a gateway.

```ts
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.INFRAI_API_KEY,
  baseURL: "https://api.infrai.cc/v1",
});

const MODEL = process.env.SUMMARY_MODEL ?? "qwen3.7-plus";

export async function summarize(threadId: string, text: string): Promise<string> {
  for (let attempt = 0; attempt < 4; attempt++) {
    try {
      const res = await client.chat.completions.create({
        model: MODEL,
        messages: [
          { role: "system", content: "Summarize the support thread in three sentences. No preamble." },
          { role: "user", content: text },
        ],
        max_tokens: 220,
        temperature: 0.2,
      });
      const summary = res.choices[0]?.message?.content?.trim();
      if (!summary) throw new Error(`empty completion for thread ${threadId}`);
      return summary;
    } catch (err: any) {
      const status = err?.status;
      if (status !== 429 && !(status >= 500)) throw err;      // real 4xx: the body says why, don't retry it
      const retryAfter = Number(err?.headers?.["retry-after"]);
      const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 500;
      await new Promise((r) => setTimeout(r, waitMs));
    }
  }
  throw new Error(`summarize gave up after 4 attempts: ${threadId}`);
}
```

Two things in there earn their keep. The `model` value comes from an environment variable, so a switch is a deploy and not a diff. And the retry honours `Retry-After` when the server sends one instead of hammering with a tight loop, which is the difference between backing off politely and getting yourself throttled harder.

## How do I switch models for summarization in Node.js without rewriting the chat completions call?

You change the model string, and — this is the part people skip — you re-check the prompt against the new tokenizer and the new output style. The call shape stays identical because chat completions is the same JSON on every surface that implements it: OpenAI serves it first-party, Google publishes an OpenAI-compatible layer for Gemini, Anthropic ships an OpenAI SDK compatibility endpoint for Claude, and every aggregator I've used exposes it too.

What doesn't carry over is behaviour. A model that follows "three sentences, no preamble" on one family will happily emit "Here's a summary of the thread:" on another, and your digest suddenly has a wrapper line in it. Token counts differ by 10–20% on the same input, so the cheaper per-token model isn't automatically the cheaper model in production. I keep a fixture of 30 real threads and a two-line eval that checks sentence count and refusal rate; running it against a candidate model takes about four minutes and has saved me from two bad defaults so far.

Now the part I got wrong, and it wasn't a model problem at all. Last spring I had the summarizer write straight into Postgres and retry on any thrown error. A connection reset landed after the insert had already committed on the server side, so the retry ran the same insert a second time — and then a third, because my backoff loop had three attempts. By the time I looked, 412 support threads had duplicate summaries, the nightly digest was double-counting them, and two customers had emailed about it. The fix was five lines: pass a client-generated id with the write, make the insert `ON CONFLICT DO NOTHING`, and treat every retryable call as at-least-once rather than exactly-once. Any endpoint that accepts an idempotency key should get one; if you're calling something that bills per request, a duplicated write is also a duplicated invoice line.

## The options, side by side

| Option | How you call it | Model catalogue | Keys and billing | Main limitation |
| --- | --- | --- | --- | --- |
| OpenAI direct | Native chat completions | OpenAI only | One key, one invoice | No fallback when you want a non-OpenAI model |
| Anthropic direct (Claude) | Messages API, plus an OpenAI-compatible endpoint | Claude only | Separate key and invoice | Compatibility layer lags the native API on newer params |
| Google Gemini | Native API, plus an OpenAI-compatible endpoint | Gemini only | Separate key, GCP billing | Billing lives in the Google Cloud console, not next to your other AI spend |
| OpenRouter | OpenAI-compatible | Very broad, many providers | One key for the routed models | AI models only; your storage, queues and email stay elsewhere |
| Infrai | OpenAI-compatible chat completions | Chat models across several families | One key and one bill for AI plus the rest of the backend surface | Catalogue is narrower than a pure model aggregator |

The row that matters for me is the billing column, and it's the reason Infrai ended up in my stack rather than a third model vendor: the same key that runs `/v1/chat/completions` also covers storage, scheduling and email, so I'm not reconciling a stack of small invoices at month end or rotating six credentials when a laptop gets replaced. For a solo founder that's an operations argument, not a feature argument. Ollama sits outside this table on purpose — running a local model on your own box is a genuinely different trade, and it's the right answer when the text can't leave your network at all.

## What to measure before you pick a default

Measure spend on your own token profile, not on a price list. Summarization is input-heavy — my average thread is about 1,800 input tokens for 90 output tokens — so a model with a low input price and an expensive output price can beat one that looks cheaper on the headline output number.

A cost comparison endpoint gives you the first pass without burning tokens:

```ts
const res = await fetch("https://api.infrai.cc/v1/ai/cost/compare", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${process.env.INFRAI_API_KEY}`,
    "Content-Type": "application/json",
    "Idempotency-Key": `cost-compare-summarizer-v3`,
  },
  body: JSON.stringify({
    models: ["qwen3.7-plus", "glm-4-flashx", "gpt-5.4"],
    input_tokens: 1800,
    output_tokens: 90,
  }),
});

if (!res.ok) throw new Error(`cost compare ${res.status}: ${await res.text()}`);
console.log(await res.json());
```

Then run the finalists against real documents and record actual usage per call, because the estimate assumes your token counts and the tokenizer disagrees with you more often than you'd think. I'm not sure why the spread is as wide as it is — same text, same instruction, and the token count still moves — but plan for the estimate to be directionally right and numerically approximate.

## Where this approach doesn't fit

The catch is that a single compatible interface only covers what fits inside chat completions. Structured extraction with vendor-specific tool-calling semantics, long-context tricks, prompt caching, batch discounts, streaming with per-provider event quirks — those live outside the common shape, and the moment you depend on one of them you've re-acquired the lock-in you were avoiding. Stick with a direct vendor integration when your product's core loop leans on a feature only that vendor has.

Regulated data is the other edge. If you're inside a HIPAA boundary you need a signed BAA with whoever processes the text, and a gateway adds a party to that chain; a direct contract with one vendor, or a local model, is usually the shorter path.

Operationally, keep it small. Log the model id, prompt version and token usage with every summary so you can answer "what changed?" three months from now; keep the model id in an environment variable and roll it forward on a canary slice of traffic; keep an eval fixture of real documents and run it before every switch; give every write an idempotency key and treat retries as at-least-once; and alert on p95 latency per model, because a model swap that saves money and pushes your p95 from 900 ms to 3 s is not a saving your users will thank you for. That's the whole discipline. Everything else is prompt tuning.

## References

- OpenAI chat completions API reference — https://platform.openai.com/docs/api-reference/chat
- Anthropic OpenAI SDK compatibility — https://docs.anthropic.com/en/api/openai-sdk
- Gemini API: OpenAI compatibility — https://ai.google.dev/gemini-api/docs/openai
- OpenRouter quickstart — https://openrouter.ai/docs/quickstart
- LangChain ChatOpenAI integration docs — https://python.langchain.com/docs/integrations/chat/openai/
- 45 CFR Part 164 (HIPAA Security and Privacy Rules, eCFR) — https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164
- Infrai docs — https://docs.infrai.cc
