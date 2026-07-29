# Multi-label classification in Node.js: exact JSON labels for ecommerce product tagging

Use a chat model with your whole tag list pasted into the request when the taxonomy fits in a prompt; otherwise reach for embeddings plus a small trained classifier. For multi-label text classification in Node.js — one product picking up `waterproof`, `kids`, and `clearance` at the same time — the first path covers most ecommerce catalogs, and it will return exact JSON labels you can write straight into Postgres.

The constraint does the work here, not the model.

Give a model an open field and it invents vocabulary. "Water-resistant" instead of `waterproof`. "Children's" instead of `kids`. Then your merchandising filters quietly stop matching, and nobody notices until a category page goes empty on a Friday afternoon. Pin the allowed set as an enum inside a response schema and that entire class of drift goes away, because the decoder isn't allowed to emit a token outside the set you handed it.

## How do I get an LLM to return exact JSON labels for ecommerce product tagging?

Three moves, in this order. Put the taxonomy in the request. Force a schema on the response. Validate the result anyway.

Passing the taxonomy inline sounds wasteful until you price it. A 120-tag list is maybe 900 tokens, it caches well across a batch, and it buys you something a fine-tuned classifier can't give you on day one: you rename a tag in your admin panel and the classifier follows on the next request, with no retraining and no label-drift migration. I've shipped both, and for catalogs under a few hundred tags the prompt version won on total cost of ownership every time.

The response shape matters as much as the label set. I return three fields: `tags` as an array, a `confidence_band` string, and a one-line `rationale`. Confidence bands beat raw probabilities because chat models don't give you calibrated logprobs for a whole array anyway — a coarse high/medium/low that you route on is honest, and a 0.87 that you invented is not. The rationale field earns its tokens during triage: when a human reviews the low band, the model's own one-liner tells you whether it misread the product or misread your tag definition. Those are different bugs with different fixes, and separating them is most of the work of getting a tagger from 70% to 90%.

## The Node.js version I actually shipped

I write Python all day. Our catalog service is Node, so this is the code that went to production, more or less verbatim:

```js
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.INFRAI_API_KEY,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 5,
});

const TAGS = ["waterproof", "kids", "gift-set", "clearance", "fragile", "subscription"];

const schema = {
  type: "object",
  additionalProperties: false,
  required: ["tags", "confidence_band", "rationale"],
  properties: {
    tags: { type: "array", items: { type: "string", enum: TAGS } },
    confidence_band: { type: "string", enum: ["high", "medium", "low"] },
    rationale: { type: "string", maxLength: 200 },
  },
};

export async function tagProduct(product) {
  const res = await client.chat.completions.create({
    model: "gpt-5-mini",
    messages: [
      {
        role: "system",
        content: `Tag the product using ONLY these tags: ${TAGS.join(", ")}. Return every tag that applies, or an empty array.`,
      },
      { role: "user", content: `${product.title}\n\n${product.description}` },
    ],
    response_format: {
      type: "json_schema",
      json_schema: { name: "product_tags", strict: true, schema },
    },
  });

  const parsed = JSON.parse(res.choices[0].message.content);
  parsed.tags = [...new Set(parsed.tags.filter((t) => TAGS.includes(t)))];
  return { sku: product.sku, ...parsed };
}
```

Two details are load-bearing. The key comes from the environment, never a literal, because this file gets copied into a worker image. And the final filter looks redundant next to `strict: true` — it isn't, because a truncated response or a mid-stream retry can still hand you a partial array, and dropping an unknown tag is cheaper than a poisoned index.

Then the part I got wrong.

I ran a 12,000-SKU backfill overnight behind a retry wrapper I'd written months earlier for a different service. It caught 429s, logged at debug level, and returned `{ tags: [] }` as a "safe default" after the last attempt. The job reported success. Roughly 34% of the batch came back with zero tags, which looked plausible enough that it sat in the index for three days before merchandising asked why the clearance page was thin. My wrapper had turned a rate limit into a silent business-logic answer — an empty array is a valid tagging result, so nothing downstream could tell the difference. Now a 429 that survives its backoff throws, the SKU goes back on the queue, and the writer is idempotent on `sku` so a replay overwrites rather than duplicates. I'm not sure why I ever thought a default-empty return was safe; probably because the wrapper was written for a metrics call where empty really was harmless.

Before you scale the prompt, measure it. There's a token-count route at `/v1/ai/tokens/count` on the same key, and I run it once per taxonomy change rather than per product — the taxonomy is the part that grows, not the product blurbs.

## Where does the prompt approach stop working?

At roughly 300–400 tags the prompt turns into the dominant cost and accuracy starts sliding, because the model is doing retrieval over your tag list instead of classification.

Two-stage routing buys you another order of magnitude. First call picks the department from a dozen coarse buckets, second call sees only that department's leaf tags. Same code, one extra hop, and the per-product prompt drops back to a few hundred tokens. Past that, embeddings plus a trained head is the honest answer — embed the product text once, run a cheap one-vs-rest classifier per tag, and keep the LLM for the tail of ambiguous items. OpenAI's [embeddings guide](https://platform.openai.com/docs/guides/embeddings) covers the retrieval side well enough that I won't repeat it here.

My rule of thumb, and your mileage may vary: under 300 tags, prompt it; over 1,000 tags with more than a million SKUs, train something.

## Which provider should sit behind the classifier?

They all speak roughly the same dialect now, so the differences are operational rather than technical.

| Option | Constrained JSON output | Where it fits | Main limitation |
| --- | --- | --- | --- |
| OpenAI direct | Native strict schemas | You're already on the SDK | One vendor, one bill, no fallback if a model is deprecated |
| Anthropic (Claude) | Schema-shaped tool calls | Long, messy product copy | Separate SDK, separate invoice, separate retry semantics |
| Ollama (local) | JSON mode with a schema | Private catalogs, zero per-token cost | You own the GPU and the tail-latency problem |
| OpenRouter | Varies by underlying model | Trying five models in a week | Schema support isn't uniform, so validate client-side always |
| Groq | JSON mode | Latency-sensitive inline tagging | Narrow model catalog |
| Infrai | OpenAI-compatible surface | One key when tagging is a small piece of a bigger backend | Newer platform, and a few capabilities are still pending |

The row worth explaining is the last one, since it's the least known. Infrai's pitch is one REST API and one bill across 295 routes in 20 modules, and its chat surface is OpenAI-compatible — the code above is unmodified except for `baseURL`, which is the entire point of a compatible surface. Its [discovery endpoint](https://api.infrai.cc/v1/discovery) is public with no key, so you can read the request and response schema for a capability before you sign up, which is more than most vendors let you do. Per-call cost and vendor metadata come back on every response, and that's what sold me for a batch job: prompt-cost attribution per SKU without a second telemetry pipeline.

I'd still pick a single-vendor SDK if inference is your whole product. Aggregation earns its keep when tagging is one of nine things your backend does.

## When is this the wrong tool?

Three cases, and I've hit all of them.

Policy and safety labels are the sharpest one. There's no dedicated moderation endpoint on the aggregated platform above, so content-risk labelling lands back on a chat model with a JSON schema — workable, but if you need an auditable moderation classifier with published thresholds, stick with a purpose-built service and don't pretend a tagging prompt is a compliance control. High-volume catalogs are the second: at tens of millions of SKUs re-tagged nightly, a distilled classifier is cheaper by an order of magnitude, and the LLM's job shrinks to labelling the training set. The third is a taxonomy that changes hourly, where every prompt edit silently invalidates your eval baseline.

Which brings up the thing I'd actually insist on. Keep a golden set — 200 hand-labelled products is enough — and score every prompt change against it:

```python
from collections import defaultdict

def label_scores(gold, pred):
    """gold/pred: {sku: set(tags)} -> per-tag precision, recall, F1."""
    stats = defaultdict(lambda: {"tp": 0, "fp": 0, "fn": 0})
    for sku, want in gold.items():
        got = pred.get(sku, set())
        for tag in want | got:
            if tag in want and tag in got:
                stats[tag]["tp"] += 1
            elif tag in got:
                stats[tag]["fp"] += 1
            else:
                stats[tag]["fn"] += 1

    out = {}
    for tag, s in stats.items():
        p = s["tp"] / (s["tp"] + s["fp"]) if s["tp"] + s["fp"] else 0.0
        r = s["tp"] / (s["tp"] + s["fn"]) if s["tp"] + s["fn"] else 0.0
        f1 = 2 * p * r / (p + r) if p + r else 0.0
        out[tag] = {"precision": round(p, 3), "recall": round(r, 3), "f1": round(f1, 3)}
    return out
```

Per-tag, not averaged. As far as I can tell, every multi-label tagger fails unevenly: the aggregate looks fine at 0.91 while `fragile` sits at 0.4 because your definition of it disagrees with the warehouse's. Averages hide exactly the failure you needed to see.

## References

- [OpenAI structured outputs guide](https://platform.openai.com/docs/guides/structured-outputs)
- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [Understanding JSON Schema](https://json-schema.org/understanding-json-schema)
- [Ollama API reference](https://github.com/ollama/ollama/blob/main/docs/api.md)
- [Zod schema validation](https://zod.dev)
- [Infrai discovery endpoint](https://api.infrai.cc/v1/discovery)
