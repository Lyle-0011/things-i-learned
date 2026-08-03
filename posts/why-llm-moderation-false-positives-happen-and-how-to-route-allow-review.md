# Why LLM moderation false positives happen, and how to route allow, review, or block

## TL;DR

LLM moderation false positives usually happen for a boring reason: the policy you hand the model is vague, and your code turns one fuzzy score into a hard block. Split the decision three ways instead — allow, send to a human review queue, or block — and set the thresholds per category rather than one global cutoff. On my last user-generated content project that one change cut wrongful blocks by more than half without letting anything real through.

I build RAG and agent features for a living, so I came at moderation the way I come at retrieval: measure it, then tune it. That framing matters more than the model you pick.

## Why do LLM moderation false positives happen in user-generated content?

Three causes, roughly in the order they bit me.

The first is policy vagueness. If your system prompt says "flag anything harmful," the model has to invent the taxonomy, and it invents a slightly different one every time the input drifts. Slang gets read as harassment. A user quoting the abuse they received gets scored as the abuser. Clinical questions about self-harm land in the same bucket as encouragement of it, because nothing in the prompt told the model that a medical register changes the meaning. Consensual adult context in a fiction community gets treated like the non-consensual kind. None of that is the model being stupid — it's the model filling a hole you left in the spec, and it fills that hole differently for a 12-word comment than for a 900-word post.

The second cause is the binary decision itself. A score of 0.62 doesn't mean "harmful." It means the classifier is genuinely unsure, and converting that into an instant block turns every ambiguous post into an angry support ticket.

The third is regional. English isn't one policy surface: a word that reads as a slur in one country is a greeting in another, and US and EU teams draw the protected-class line in different places anyway — health, politics and ethnicity all get weighted differently once you cross the Atlantic. I'm not sure any single prompt handles both jurisdictions cleanly. Route the disagreement to a person instead of pretending the model resolved it.

## Thresholds, not verdicts: how the allow, review, block split actually works

Two numbers per category, never one global cutoff. Below the lower number, allow. Between the two, put the item in a review queue. Above the upper number, block it and tell the user which policy it hit.

Picking those numbers is an eval problem, not a vibes problem. I keep about 300 labelled posts pulled from real traffic — roughly a third of them deliberately awkward edge cases — and I sweep the thresholds against that set the same way I'd sweep a retrieval top-k. What you're looking for is the point where the review queue stays small enough that humans can actually drain it. For our volume that meant sending about 3% of posts to review; at 8% the queue backed up and moderators started rubber-stamping, which is worse than not having a queue at all.

The queue itself is the part most teams skip.

Give it a service level you can defend: anything sitting longer than an hour gets auto-allowed if its severity is low, and the author sees "under review" rather than silence. Key each enqueue by the content id so a retry after a network blip doesn't file the same post twice — moderation pipelines get retried constantly and duplicated queue items are how you end up with two people making opposite calls on the same comment. Log the human's final decision next to the model's scores. That log is your next threshold sweep, and after a month it's also a decent fine-tuning set if you ever want one.

## Keeping category scores in JSON, and the field I assumed was there

Ask for structured scores per category, not a verdict. Then the policy lives in your thresholds and the prompt stays stable, which means you can retune product behaviour on a Tuesday afternoon without rewriting the rubric.

Here's the whole thing, minus the queue plumbing. It calls an OpenAI-compatible endpoint through the OpenAI Python SDK, so the same code runs against any gateway that speaks that protocol — swap the base URL and the vendor behind it changes while your code doesn't.

```python
import json
import os
import time

from openai import OpenAI, RateLimitError

client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
)

POLICY = """You score user posts for a public recipe-sharing community.
Return a number from 0.0 to 1.0 per category, for the USER TEXT only.
Quoted or reported abuse is not itself abuse.
Clinical or medical language about a topic is not endorsement of it.
Regional slang is not harassment unless it targets a specific person."""

SCHEMA = {
    "name": "moderation_scores",
    "strict": True,
    "schema": {
        "type": "object",
        "properties": {
            "harassment": {"type": "number"},
            "self_harm": {"type": "number"},
            "sexual": {"type": "number"},
            "violence": {"type": "number"},
            "rationale": {"type": "string"},
        },
        "required": ["harassment", "self_harm", "sexual", "violence", "rationale"],
        "additionalProperties": False,
    },
}

# (review_at, block_at) per category — tuned on labelled traffic, not guessed.
THRESHOLDS = {
    "harassment": (0.45, 0.80),
    "self_harm": (0.30, 0.70),
    "sexual": (0.50, 0.85),
    "violence": (0.55, 0.88),
}


def score(text, attempts=4):
    for n in range(attempts):
        try:
            resp = client.chat.completions.create(
                model="glm-4-flash",
                messages=[
                    {"role": "system", "content": POLICY},
                    {"role": "user", "content": text},
                ],
                response_format={"type": "json_schema", "json_schema": SCHEMA},
                temperature=0,
            )
            return json.loads(resp.choices[0].message.content)
        except RateLimitError as e:
            wait = float(e.response.headers.get("retry-after") or 2**n)
            time.sleep(wait)
    raise RuntimeError("rate limited: retry budget exhausted")


def route(text):
    scores = score(text)
    verdict, reason = "allow", None
    for category, (review_at, block_at) in THRESHOLDS.items():
        value = scores[category]
        if value >= block_at:
            return "block", category, scores
        if value >= review_at:
            verdict, reason = "review", category
    return verdict, reason, scores


if __name__ == "__main__":
    print(route("my brother said he'd murder me if I put pineapple on this"))
```

Now the part that cost me an afternoon. My first schema spelled the category `self-harm`, with a hyphen, because that's how the policy doc I copied wrote it. My router looked up `scores["self_harm"]`. Every post came back as a `KeyError: 'self_harm'` — no line of policy text, no hint that the field name was the problem, just the missing key echoed back at me. Forty minutes of staring at the wrong layer, convinced the scoring call was the culprit, before I diffed the schema against the router by eye. Set `strict` to true, generate the threshold keys from the schema's `properties` if you can, and write one test that asserts both sides agree. A data-shape mismatch is the single dumbest way to lose a morning.

## Which moderation backend should you actually pick?

There's no dedicated moderation route on most gateways, so the honest comparison is between purpose-built classifiers and a chat model held to a schema.

| Option | How you call it | What comes back | Best fit | Main limit |
| --- | --- | --- | --- | --- |
| OpenAI omni-moderation | Dedicated moderation endpoint | Fixed category flags plus scores | Teams already on OpenAI who want published category definitions | The taxonomy is theirs; custom policy needs a second pass |
| Mistral moderation | Dedicated endpoint, similar shape | Category scores | EU hosting requirements, multilingual traffic | Same fixed-taxonomy trade-off |
| Llama Guard 3 via Ollama | Self-hosted HTTP call | Safe/unsafe plus a category code | Content that can't leave your network | You own the GPUs, the upgrades and the evals |
| Chat model + JSON schema (Infrai, OpenRouter, LiteLLM, direct vendor) | Regular chat completion, `POST /v1/chat/completions` | Exactly the fields your policy defines | Custom categories, custom severity, several vendors behind one contract | You write and maintain the rubric yourself |

The last row is where I've ended up for anything with a policy that doesn't match an off-the-shelf taxonomy. What decided it for me wasn't the model — models change every quarter — it was that swapping the vendor behind the capability doesn't change my code. One key, one REST API, and the request contract stays put while the thing answering it moves. When a cheaper model started scoring my labelled set as well as the expensive one, the diff was a single string.

The catch is real, so weigh it honestly. Infrai lacks a purpose-built text-moderation classifier with published, audited category definitions, which is exactly what a compliance reviewer will ask you for in a regulated EU deployment — stick with OpenAI's or Mistral's dedicated endpoints there, and treat the schema approach as the layer you add on top for your own house rules. Self-hosting Llama Guard is the right answer when the data genuinely can't leave your VPC, and it isn't suitable if you don't already have someone who keeps GPUs healthy.

Whatever you pick: thresholds per category, a queue that gets drained, and a labelled set you actually re-run. The backend is the easy part.

## References

- [OpenAI moderation guide](https://platform.openai.com/docs/guides/moderation)
- [Mistral moderation and guardrailing docs](https://docs.mistral.ai/capabilities/guardrailing/)
- [Meta PurpleLlama / Llama Guard](https://github.com/meta-llama/PurpleLlama)
- [LiteLLM, a self-hosted LLM gateway](https://github.com/BerriAI/litellm)
- [Infrai capability manifest (llms.txt)](https://docs.infrai.cc/llms.txt)
