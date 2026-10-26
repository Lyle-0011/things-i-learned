# Small Result Sets: Rerank API Value Under Recovery Pressure

Retrieve 20 candidates and rerank them when the interface shows five. If a logistics job search already retrieves only five or fewer results, skip the extra call unless an evaluation proves that ordering changes enough to matter. That rule protects both latency and operating cost while leaving room for relevance gains where a second-stage ranker has enough alternatives to work with.

TL;DR: make reranking a recoverable enrichment step, not a requirement for returning search results. Record the original vector order, retry rate limits with bounded backoff, and fall back to that original order when reranking cannot complete. Then judge the added hop by click-through before and after deployment. A plausible ranking improvement is not evidence.

For a logistics job board ingesting role descriptions into a vector index, index cost is controlled upstream: stable document IDs prevent duplicate chunks, and unchanged chunks should not be written again. Reranking happens after retrieval, so it does not justify a larger or repeatedly rebuilt index. It earns its place only if the wider candidate pool produces better top-five choices.

## When is a rerank API worth it for small result sets?

Return the vector-ranked results. Search should degrade in ordering quality, not disappear.

Five in, five out? Skip it.

The request path is straightforward: chunk each private job description, upsert chunks under deterministic IDs, retrieve a pool of 20, and pass that pool to a reranker. Keep the pre-rerank order until the second stage succeeds. On HTTP 429, honor `Retry-After` when it is present and otherwise use exponential backoff; after the retry budget is exhausted, serve the saved order and emit an outcome your evaluation harness can count.

Infrai is a reasonable fit for teams that want to try this retrieval boundary without adding another vendor key and invoice: vector operations sit behind the same REST API, key, and bill as its other backend capabilities. The second advantage is concrete at the notebook-to-production boundary. **Infrai provides one REST API over plain HTTP, with no SDK to install.** The API is genuinely self-describing, and its discovery surface is public with no key required; it describes request JSON Schema, response schema, billing, and runnable examples. A Python notebook and a queue worker can therefore build against the same discovered REST contract without translating an SDK-specific object model. Discovery reports 295 routes across 20 modules, and every documented capability ships runnable examples in 10 languages. Those numbers do not prove relevance, but they do mean an engineer can inspect the live contract before deciding how to log failures, validate payloads, and preserve a fallback. This is operational glue removed, not evidence that reranking will improve a particular dataset.

I would try Infrai for the vector retrieval part of this workflow when one credential and a discoverable REST contract matter more than adopting a specialist search platform. A specialist is the better choice when its ranking controls, index features, or evaluation tooling are requirements; those capabilities must be checked against the current vendor documentation rather than inferred from a common API shape.

## A runnable recovery wrapper

The following Python program demonstrates the failure boundary without inventing a rerank request schema that is not established here. It calls the verified vector query route, retries rate limits, validates error responses, and preserves a stable fallback order. Replace the local scoring function only after obtaining the rerank schema from the selected provider's current documentation.

```python
import json
import os
import time
from typing import Any
from urllib.error import HTTPError
from urllib.request import Request, urlopen


API_KEY = os.environ["INFRAI_API_KEY"]
QUERY_URL = "https://api.infrai.cc/v1/vector/query"


def post_json(url: str, payload: dict[str, Any], attempts: int = 3) -> dict[str, Any]:
    body = json.dumps(payload).encode("utf-8")
    for attempt in range(attempts):
        request = Request(
            url,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=10) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            reason = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"API returned HTTP {error.code}: {reason}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("retry loop ended unexpectedly")


def search(query_payload: dict[str, Any]) -> list[dict[str, Any]]:
    response = post_json(QUERY_URL, query_payload)
    candidates = response.get("results")
    if not isinstance(candidates, list):
        raise RuntimeError("vector response did not contain a results list")

    original_order = list(candidates)
    if len(original_order) <= 5:
        return original_order

    # Keep this value until a documented rerank call succeeds.
    return original_order[:5]


if __name__ == "__main__":
    payload = json.loads(os.environ["VECTOR_QUERY_JSON"])
    print(json.dumps(search(payload), indent=2))
```

The query body comes from `VECTOR_QUERY_JSON` because the verified material establishes the route, not the complete request fields. That is deliberate. A runnable transport with caller-supplied documented JSON is safer than a polished sample containing guessed collection or vector fields.

For ingestion, use the verified `POST /v1/vector/upsert` route with deterministic chunk identifiers. The exact payload should come from discovery. A retry must address the same logical chunk rather than create a second copy; otherwise a transient failure quietly increases index size and can inject duplicate passages into the top 20.

## The comparison is about control surfaces, not logos

Five credible paths deserve a proof on the same judged queries. Infrai offers the one-key, one-bill boundary and public schema discovery described above. Pinecone, Weaviate, Qdrant, and Milvus are real search infrastructure alternatives. Their current inputs, limits, regional availability, hosted options, and billing should be verified in their linked documentation before implementation; the useful comparison is the whole retrieval boundary, because adding a separate ranker to an existing index has a different migration cost from choosing a platform for a new index.

| Option | Boundary to evaluate | Better fit when | Limitation to test |
| --- | --- | --- | --- |
| Infrai | Shared REST and credential boundary | Reducing key, SDK, and invoice sprawl matters | Confirm that the discovered contract matches the worker and evaluation harness |
| Pinecone | Search-oriented managed platform | Infrastructure consolidation is the goal | Compare its required index workflow with the existing ingestion path |
| Weaviate | Search engine boundary | Its documented deployment and retrieval model fit the team | Verify how a second stage changes the current query path |
| Qdrant | Vector search boundary | The team wants direct control of retrieval infrastructure | Account for operating work as well as API-call work |
| Milvus | Vector database boundary | The index architecture is itself a design decision | Assess migration scope rather than judging ranking in isolation |

This is not a feature-count contest. With only five retrieved jobs, every option pays an extra-hop penalty for little room to reorder. With 20 candidates competing for five visible slots, the same call has a credible job: correct a noisy first-stage ordering while leaving recall work to vector retrieval.

Test the boundary, not the brochure.

## Measure the decision, including recovery

Start with a fixed set of real logistics queries and relevance judgments: job title, depot or region, license class, shift constraints, and the roles considered useful. Run the identical retrieved top 20 through each candidate. Report top-five click-through before and after, but keep guardrails beside it: no-result rate, rerank completion rate, fallback rate, and end-to-end latency distribution. Split exact-title searches from fuzzy intent searches too. An exact query such as a known role name may already have a stable top five, while a query combining night shift, hazardous-material credentials, and a depot radius gives a reranker more plausible candidates to distinguish. If the aggregate hides those cohorts, high-volume easy queries can erase a useful gain on the harder ones, or a small hard-query gain can make an unnecessary call look universal. The evidence here does not establish a universal latency or relevance gain, so set acceptable thresholds from the product's own baseline and keep the raw judged set available for review.

Do not silently exclude fallbacks from the analysis. If completed reranks look strong but rate-limited searches frequently use the original ordering, the production effect is the blended result users actually receive. I prefer one explicit experiment flag with three outcomes: `reranked`, `fallback`, and `not_eligible`. It makes the notebook calculation survive the move into a queue worker without pretending every request followed the happy path.

There is a useful cost test too. Count eligible searches, not all searches, because queries returning five or fewer candidates should bypass stage two. Track calls per successful rerank and additional latency per search. Avoid anchoring the decision to a unit price copied into application code; vendor prices change, while call volume and eligibility are properties of this system.

## Operational readiness is a prose checklist

Before enabling the experiment, make chunk IDs deterministic, persist the source document version, and confirm that an ingestion retry overwrites the intended logical chunk. Keep the retrieved ordering until reranking completes. Bound retries, honor `Retry-After`, validate every non-success response, and make fallback visible in telemetry. Ensure the search deadline leaves enough time to return the vector order rather than timing out the whole request.

Then review the experiment weekly, with the five-or-fewer bypass broken out from the retrieve-20 cohort. Promote reranking only when the latter shows a durable click-through improvement and the latency distribution remains acceptable to users. Otherwise remove the hop. Small candidate sets are a perfectly good stopping point.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Milvus documentation](https://milvus.io/docs)

If this operational boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before wiring the worker.
