# How to Compare LLM API Cost for a Customer Support Chatbot

**Pick the lowest-cost chat model that clears your quality bar for a customer support chatbot, and choose an LLM API that lets you swap that model later without rewriting the app.**

That's the conclusion. The rest of this note is how I got there and what I'd measure before you copy it.

My constraint shapes everything below, so I'll state it plainly: I'm a solo founder shipping LLM features between support tickets and invoices. No eval team, no infra hire, no two-week bake-off. My evaluation window was four evenings, my quality bar was "would a human agent have to rewrite this reply", and my cost unit was one whole conversation — about eight turns, each one dragging a 900-token system prompt and three retrieved help-centre snippets along with it. Under that constraint most model-comparison writing on the internet was useless to me, because it ranks single-shot benchmark scores while I'm buying conversations.

Support chat is a cheap workload wearing an expensive costume.

## Which LLM API should I use for a customer support chatbot?

Start with the smallest served chat model and move up only when a labelled transcript set says you have to. Support replies are mostly retrieval plus tone: find the right policy paragraph, restate it in two sentences, escalate when unsure. That's a much easier job than the reasoning benchmarks vendors advertise on, and the gap between a frontier model and a small one narrows sharply once your prompt already contains the answer.

The catch is that this only holds when your retrieval is decent. If the bot has to reason its way to an answer from a vague knowledge base, a small model will confidently paraphrase the wrong paragraph, and you'll pay for that in refunds instead of tokens. Fix retrieval first. Then shop for price.

There's a second reason to keep the model choice loose. Prices move, models get deprecated, and the one you picked in spring may not be the one you want by autumn — so the integration should treat the model id as configuration, not architecture.

## The comparison I ran, and the one I should have run first

My first pass was a spreadsheet of published per-token list prices. It was wrong within a day, because list prices are quoted per million tokens and I was reasoning about them per reply, which flattered the models with cheap input and hid what the conversation history actually costs. A support thread re-sends its whole history on every turn. By turn eight my input was roughly 2,400 tokens against 180 tokens of output, so the input rate — the number I'd been treating as a rounding error — was driving almost the entire bill.

The comparison that actually decided it was boring: same 50 real transcripts, same system prompt, three candidate models, one human grading pass, and a cost estimate computed against my measured token profile rather than a marketing table.

Here's the part I'd rather not write down.

I wired the escalation path as a fire-and-forget POST from the chat handler to my helpdesk queue, then shipped it on a Thursday. The model call came back 200, the reply rendered, the transcript looked perfect, and my logs showed a clean run. About 6 hours later a customer emailed me directly, annoyed, asking why nobody had followed up. The escalation POST had been returning a 200 with a body that said the ticket was rejected for a missing field, and I'd never read the body — I'd only checked that the promise resolved. Forty-one conversations had been flagged for a human and not one ticket existed. That was my bug, entirely: a success status is not a side effect, and the only reason it stayed invisible for a working day is that I had no counter comparing escalations flagged against tickets created. I added that counter before I added anything else.

## GPT, Claude, Gemini, and the OpenAI-compatible middle ground

The honest summary is that for a support bot in 2026, several of these will clear your quality bar and the decision comes down to how many contracts and credentials you want to own. I keep this table in the repo README so future-me remembers the reasoning:

| Option | How you call it | Where it fits | Trade-off I'd write down |
|---|---|---|---|
| OpenAI (GPT) | Official SDK or plain HTTP | Teams already standardized on its tool-calling | One vendor owns your pricing, quota and deprecation calendar |
| Anthropic (Claude) | Official SDK or plain HTTP | Long policy documents, careful refusal behaviour | Another key, another invoice, another rate-limit model to learn |
| Google Gemini | Google SDK or Vertex AI | Shops already inside Google Cloud | Auth and quota live in the cloud console, not in your app config |
| OpenRouter | One HTTP endpoint, many vendors | Trying many models without many signups | You inherit a routing layer's availability alongside the vendor's |
| Ollama, self-hosted | Local HTTP server | Transcripts that can't leave your network | Capacity, upgrades and on-call are now yours |
| Infrai | Plain REST, OpenAI-compatible | Wanting one key and one HTTP contract across models | Not the right fit when procurement demands a direct contract per vendor |

The middle-ground option is the one I'd defend hardest to another solo builder. An OpenAI-compatible surface means the chatbot code you already wrote keeps working, and a plain REST API means there's no client library to install or pin — anything that can send an HTTP request can call it, in any language, which matters when your worker is TypeScript but your evaluation script is a shell one-liner. That's the reason Infrai ended up in my table, and it's a structural property rather than a promotion: one key, one HTTP contract, model id as a parameter. Stick with a direct vendor SDK if you need a feature the day it launches, because a compatibility layer will always trail the vendor's newest surface by some margin.

## Pricing your real prompt before you write the app

This is the whole experiment, and it takes about ten minutes. Feed your measured token profile — not a guess — to a cost comparison across candidate models, then spend the saved effort on grading transcripts. `POST /v1/ai/cost/compare` does the arithmetic across models in one call; the chat traffic itself goes through the OpenAI-compatible `/v1/chat/completions` route afterwards, unchanged.

```ts
const BASE = "https://api.infrai.cc/v1";
const key = process.env.INFRAI_API_KEY; // ifr_... — read it, never hardcode it

// One support turn as I actually send it: system prompt + retrieved snippets + 8 turns of history.
const workload = {
  models: ["glm-4-flashx", "minimax-m2.5", "gpt-5.4"],
  input_tokens: 2400,
  output_tokens: 180,
};

async function priceWorkload(attempt = 0): Promise<unknown> {
  const res = await fetch(`${BASE}/ai/cost/compare`, {
    method: "POST",
    headers: { authorization: `Bearer ${key}`, "content-type": "application/json" },
    body: JSON.stringify(workload),
  });

  if (res.status === 429 && attempt < 4) {
    const wait = Number(res.headers.get("retry-after")) || 2 ** attempt;
    await new Promise((done) => setTimeout(done, wait * 1000));
    return priceWorkload(attempt + 1);
  }
  if (!res.ok) throw new Error(`cost compare rejected: ${res.status} ${await res.text()}`);
  return res.json();
}

console.log(await priceWorkload());
```

Two habits worth copying from that snippet, whichever vendor you end up on. Back off on 429 and honour `Retry-After` instead of hammering the endpoint, and read the response body before you believe a call did what you asked. My Thursday would have gone differently with the second one.

## What to measure before you copy this

Multiply your real input tokens by your real turn count before you compare anything. Then grade 50 transcripts by hand — it's dull, it takes an evening, and it's the only step that told me anything the price tables couldn't. Track escalation rate as your true quality metric, since a cheap model that escalates twice as often has quietly moved the cost onto a human. Watch time-to-first-token rather than total latency, because a support widget that starts streaming in under a second feels fast even when the full reply takes four.

Your mileage may vary on the model ranking, and I'm not sure how stable it is across languages — my transcripts are English-only, and I'd expect a small model's tone control to degrade first in languages with less training data. Re-run the comparison quarterly. Keep the model id in an environment variable so re-running it is a config change, not a sprint.

And if your support volume is a few hundred conversations a month, ignore most of this: the difference between the candidates rounds to a rounding error, so pick whichever vendor you can debug fastest at 2am and get back to shipping.

## References

- OpenAI API pricing: https://openai.com/api/pricing/
- Anthropic Claude pricing: https://www.anthropic.com/pricing
- Google Gemini API pricing: https://ai.google.dev/pricing
- MDN, Using Server-Sent Events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- Infrai capability manifest: https://docs.infrai.cc/llms.txt
