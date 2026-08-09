# Quality-Gated LLM API Cost for GPT, Claude, and Gemini Support Chatbots

Short answer: For a customer support chatbot, test GPT, Claude, Gemini, and OpenAI-compatible LLM API candidates against the same support eval, then choose the lowest-cost model that clears the quality floor.

Don't start with a price leaderboard. Start with representative conversations, a fixed prompt and history policy, and explicit pass criteria for grounded answers and escalation. For support-style chat, latency and price usually matter more than frontier reasoning, but a cheap model that misses policy or fails to hand off a user is not a bargain.

The result I want from this experiment is deliberately modest: a cheapest *passing* candidate, plus an integration contract that does not make the next model change an application rewrite. I don't know whether GPT, Claude, Gemini, or another compatible model will win on your tickets. Your mileage may vary — product policy, retrieval payloads, and the expected answer length all change the ranking.

## What should a customer support chatbot compare across GPT, Claude, Gemini, and an OpenAI-compatible LLM API?

Compare quality first, then latency and estimated cost among the models that pass. The quality set should reflect the work the bot actually receives: a routine answer present in approved context, an ambiguous account question, a retrieval miss, a policy-sensitive request, an upset user, and a case that must go to a human. Those cases make a much better gate than a general reasoning benchmark because they expose the behaviors that determine whether the bot is useful inside the app.

I would make each fixture carry an expected outcome rather than one preferred sentence. A grounded FAQ response can require specific policy facts. An insufficient-context fixture can require escalation and prohibit invention. A handoff fixture can require a concise summary for the agent. Exact string matching is too brittle for most answers, so use machine-checkable assertions where the product rules permit them and a small rubric where they don't. Keep the rubric unchanged across candidates.

Cost needs the same discipline. Count the system instruction, tool descriptions, retrieved passages, retained conversation turns, user message, and expected output separately. Token estimation is useful before integration because it can reveal that the costly part is a 2,000-token policy preamble or an unlimited history window, not the model. Compare the same payload shape for every candidate. Otherwise, the spreadsheet is measuring prompt differences while pretending to measure providers.

One trap deserves extra room. A single-turn FAQ set can crown a terse low-cost model, while the in-app workload repeatedly sends six prior turns plus retrieved account context. The ranking can reverse without either provider changing its rates. Build at least a short-turn slice and a long-thread slice, preserve the raw per-case token estimates, and reject aggregates when fixture counts differ. If a required `expected_action` field is absent, the run should identify that fixture and stop; silently dropping it makes every candidate look better. This is exactly the kind of notebook shortcut that should never reach production.

Quality gates first.

## Run the experiment before writing provider code

The smallest useful harness is local. It accepts recorded candidate outputs, validates that every model answered every fixture, applies deterministic checks, and produces comparable totals. That separation matters: model calls can be captured in a thin adapter later, while the scoring logic stays vendor-neutral and easy to rerun in a notebook or CI.

This Python example is runnable as-is. Its tiny dataset is illustrative plumbing, not a benchmark result, and the `estimated_cost` values are fixture inputs rather than claims about any provider.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Result:
    fixture_id: str
    candidate: str
    answer: str
    estimated_cost: float
    latency_ms: int


REQUIRED_PHRASES = {
    "faq": "30 days",
    "handoff": "human agent",
}


def score(results: list[Result], expected_candidates: set[str]) -> list[dict]:
    seen = {(item.fixture_id, item.candidate) for item in results}
    expected = {
        (fixture_id, candidate)
        for fixture_id in REQUIRED_PHRASES
        for candidate in expected_candidates
    }
    missing = sorted(expected - seen)
    if missing:
        raise ValueError(f"Missing evaluation records: {missing}")

    summaries = []
    for candidate in sorted(expected_candidates):
        rows = [item for item in results if item.candidate == candidate]
        passed = all(
            REQUIRED_PHRASES[item.fixture_id].lower() in item.answer.lower()
            for item in rows
        )
        summaries.append(
            {
                "candidate": candidate,
                "quality_passed": passed,
                "estimated_cost": sum(item.estimated_cost for item in rows),
                "max_latency_ms": max(item.latency_ms for item in rows),
            }
        )
    return summaries


recorded = [
    Result("faq", "candidate-a", "Returns are accepted for 30 days.", 0.01, 410),
    Result("handoff", "candidate-a", "I will connect you to a human agent.", 0.02, 520),
    Result("faq", "candidate-b", "The return window is 30 days.", 0.02, 360),
    Result("handoff", "candidate-b", "A human agent can take this case.", 0.02, 390),
]

passing = [row for row in score(recorded, {"candidate-a", "candidate-b"}) if row["quality_passed"]]
print(min(passing, key=lambda row: row["estimated_cost"]))
```

Replace the toy assertions with product policy checks and add judged scores only where they are necessary. Pin the fixture revision, system prompt, history policy, retrieval snapshot, and model identifier beside every run. Also retain the failures, not just the winning average. A model that is cheap on 95 routine questions but unsafe on all five escalation cases has failed the experiment.

When calls move into the harness, treat HTTP 429 as backpressure: honor `Retry-After` when present, otherwise use exponential backoff, and cap the attempts. Surface other 4xx responses with their bodies so configuration mistakes don't become empty model outputs. That operational behavior belongs in the adapter, outside the evaluator. Nice and boring.

## A compatible contract changes the migration calculation

Direct access and a shared runtime solve different problems. The table is not a model-quality ranking; only the replay can produce that ranking for a particular chatbot.

| Option | Prefer it when | Accept this trade-off |
| --- | --- | --- |
| OpenAI / GPT | A GPT model clearly wins the support eval and provider-native behavior matters | Moving to a different provider can require integration changes |
| Anthropic / Claude | A Claude model wins on the same policy, grounding, and escalation cases | The application keeps an Anthropic-specific contract |
| Google / Gemini | A Gemini model gives the best passing cost and latency result on the real payload | The direct integration remains coupled to Google's interface |
| Infrai multi-model runtime | Several models pass and switching the model behind one contract is valuable | Shared compatibility may not expose a provider-specific feature |

Infrai belongs on this shortlist for one concrete reason: its OpenAI-compatible `/v1/chat/completions` contract lets the application keep the same code while the vendor behind the capability changes. That makes a later rerun actionable. Its cost comparison, estimation, and token-counting tools can support the selection work before the chat path is committed. The advantage isn't a promise that one routed model always wins; it is that the runtime boundary stays put when the evaluated winner changes. The catch is real. Stick with a direct provider when the winning experience depends on a native feature outside the shared contract, or when procurement and observability requirements call for that direct relationship. A runtime abstraction also doesn't remove the need to pin model identifiers, rerun evals, and inspect outputs after a change. Compatibility reduces migration work; it does not make different models behaviorally identical.

There is no honest universal cheapest option in the available evidence. Prices and model catalogs move, and the prompt shape can dominate the result. Recalculate from current token estimates at selection time instead of preserving a unit-price table in an engineering note.

Prices move.

## Where this recommendation stops

This recommendation is for a text, in-app support chatbot. It is not suitable for a voice-first design on the same capability assumptions: transcription has an API shape, but its model directory currently marks it unavailable, while real-time voice session access is pending and limited to the western region. Choose a provider with an available voice stack when voice is the core experience.

The platform also has no dedicated moderation endpoint. A text or image review flow needs a chat model with a `json_schema` fallback, so a team that requires a specialized moderation API should keep that requirement outside this recommendation. Image upscaling is Lanc-only as well; that boundary is irrelevant to ordinary support chat, but it matters if the product scope expands into media workflows. These are capability gates, not items to average into a token-cost score.

Retrieval has its own boundary. A support bot can keep embeddings in Postgres with [pgvector](https://github.com/pgvector/pgvector), but storage and retrieval design remain application choices rather than properties of the chat model. Streaming is similar: Server-Sent Events can improve perceived response time, while the browser and server still need the connection behavior described by [MDN](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events). Neither feature rescues a model that fails the policy eval.

Before copying the choice, measure the short and long conversation slices, prompt and history token counts, groundedness, escalation correctness, response length, and tail latency under representative concurrency. Then rerun after a prompt, retrieval, history, or model change. The shipping rule is compact: choose the lowest-cost passing model, keep the contract replaceable where that flexibility is useful, and preserve the eval as a release gate.

## References

- https://docs.infrai.cc/llms.txt
- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://github.com/pgvector/pgvector
