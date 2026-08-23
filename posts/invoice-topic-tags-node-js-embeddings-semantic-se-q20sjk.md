# Invoice Topic Tags: Node.js Embeddings, Semantic Search, and Reranking

Short answer: use embeddings to retrieve a small candidate set, rerank that set only when the distinction matters, and let the classifier assign a topic from a closed schema. For a fintech invoice pipeline, keep retrieval, classification, and per-tenant usage accounting as separate steps. That is the least complex design that stays testable from notebook to production.

Keep it boring.

The example below uses Python because the boundary matters more than the SDK: an embedding service, a vector store, and a language model can sit behind ordinary functions. The same contract can be implemented in a Node.js worker. The important part is that every document carries a tenant ID, a stable document ID, a content hash, and a versioned classification schema.

## Invoice topic classification starts with a contract

Start with a small, explicit flow. Normalize the invoice text, embed it, retrieve topic examples or policy passages, optionally rerank those candidates, and then ask the classifier for structured output. Store the raw decision inputs alongside the result. Without that record, a changed prompt or model turns a one-line label into an unrepeatable guess.

For supplier invoices, semantic search is useful for finding nearby examples such as `software`, `logistics`, `office supplies`, or `professional services`. It should not be the final judge. Similarity finds language that looks alike; it does not understand the accounting policy that makes one topic valid for a tenant and invalid for another.

Here is a compact implementation sketch. The interfaces are intentionally generic so the evaluation harness can replace each dependency with a fake.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass
class Candidate:
    text: str
    topic: str
    score: float


class Embedder(Protocol):
    def embed(self, text: str) -> list[float]: ...


class VectorIndex(Protocol):
    def search(self, vector: list[float], limit: int) -> list[Candidate]: ...


class Reranker(Protocol):
    def rank(self, query: str, candidates: list[Candidate]) -> list[Candidate]: ...


class Classifier(Protocol):
    def classify(self, text: str, evidence: list[Candidate], topics: list[str]) -> dict: ...


TOPICS = ["software", "logistics", "office_supplies", "professional_services", "unknown"]


def classify_invoice(tenant_id: str, invoice_text: str, embedder: Embedder,
                     index: VectorIndex, reranker: Reranker,
                     classifier: Classifier) -> dict:
    vector = embedder.embed(invoice_text)
    candidates = index.search(vector, limit=12)

    # Rerank only the retrieved evidence, where its extra latency is bounded.
    evidence = reranker.rank(invoice_text, candidates)[:4]
    result = classifier.classify(invoice_text, evidence, TOPICS)

    return {
        "tenant_id": tenant_id,
        "topic": result["topic"],
        "confidence": result.get("confidence"),
        "evidence": [candidate.text for candidate in evidence],
    }
```

The classifier should return a topic from the schema, not invent a new one because an invoice contains an unfamiliar phrase. `unknown` is a useful output: it gives the review queue a defined path and prevents a plausible-looking label from silently entering a ledger. The application should validate the response before persisting it, and it should reject evidence from another tenant. If a supplier changes its invoice template, the stored content hash and evidence still show exactly which input produced the label, which is the difference between an explainable correction and a mysterious reclassification.

## What should semantic search and embeddings prove before an LLM classifier sees reranked evidence?

Chunking is the first quiet source of bad labels. A whole invoice may contain a vendor name, payment terms, tax lines, and several line-item categories. One vector for all of that can blur the signal. Chunk line items or sections when the classification decision is local, but retain the invoice ID and section coordinates so a reviewer can see where the evidence came from.

Consider an invoice with a hosting subscription, a one-time migration fee, and a line for expedited freight. A nearest-neighbor search may return a software example because the vendor description dominates the text. A reranker may move the freight example upward, but the classifier still needs a rule for whether the document receives one topic or several line-item topics. That choice belongs in the schema and the evaluation set, not in an improvised prompt. The same record should preserve the selected span, the candidate examples, the schema version, and the tenant policy used at decision time. When a reviewer changes the label, the team can tell whether the problem was chunking, retrieval, ranking, or policy. Without those distinctions, every error looks like “the model was wrong,” and the next prompt change has no reliable target.

The index needs a metadata filter for `tenant_id`, document type, and schema version. A nearest neighbor from the wrong tenant is a data isolation failure even if the final label happens to look right. A Postgres deployment can use pgvector for vector similarity and ordinary relational columns for these filters; the choice of storage is less important than making the filter part of the query contract rather than an afterthought.

Reranking has a narrower job. It can improve the order of a dozen retrieved candidates, but it cannot repair an incomplete index, a bad tenant filter, or a topic schema that does not reflect the business. Set a maximum candidate count and record the pre-rerank and post-rerank scores. Those fields make it possible to see whether reranking changes decisions or merely adds latency and token usage.

There is a practical limit here. If the labels are already separated by strong lexical rules, a reranker is not suitable for every invoice. A deterministic rule or a human review queue may be the better choice when the evidence is sparse, the invoice is malformed, or the tenant has too few labeled examples.

## Per-tenant usage is an audit record

A classifier that looks good in a notebook can still be expensive and hard to audit. Build an evaluation set with representative invoices per tenant, including near-boundary pairs: freight versus professional services, recurring software versus one-time implementation, and invoices with multiple tax or currency formats. Measure exact topic accuracy, unknown rate, and review rate. Keep the test set fixed while prompts and models change.

Per-tenant cost visibility belongs in the same event record as the decision. Record the tenant ID, operation name, model identifier, input and output token counts when available, embedding dimensions, candidate count, reranker invocation, latency, retry count, and schema version. Do not estimate a tenant's bill from aggregate dashboard totals after the fact; that loses the relationship between one invoice and one operation.

Prompt cost is an architectural input. Sending twelve full candidate passages to the classifier every time can dominate the spend and make context noisy. Trim evidence to the fields that support the label, cap the number of passages, and measure the accuracy change. A cheap first pass can send only the invoice header and line-item names; escalate to richer evidence when confidence is low or the topic is `unknown`.

A useful decision record looks like this:

```python
from time import time


def usage_event(tenant_id: str, operation: str, result: dict,
                input_tokens: int, output_tokens: int,
                retry_count: int, started_at: float | None = None) -> dict:
    return {
        "tenant_id": tenant_id,
        "operation": operation,
        "topic": result["topic"],
        "input_tokens": input_tokens,
        "output_tokens": output_tokens,
        "candidate_count": len(result["evidence"]),
        "retry_count": retry_count,
        "schema_version": "invoice-topics-v1",
        "started_at": started_at or time(),
    }
```

This is not a pricing claim. It is an accounting shape. A tenant can then be compared by invoices processed, classifier calls, reranker calls, and tokens, which are different levers and should not be collapsed into one opaque number.

## Retry semantics protect the review queue

Network retries deserve the same design attention as prompts. RFC 9110 distinguishes methods and their retry semantics; a worker should also impose an application-level idempotency key, such as `tenant_id + invoice_id + content_hash + schema_version`. If a request is repeated after an ambiguous timeout, the consumer can recognize the prior decision or safely replace an unfinished attempt.

Don't retry every response. A transient transport failure may be retryable, while malformed structured output needs validation and a bounded repair or review path. The retry count must be visible in cost records, because a technically successful classification can still have consumed several attempts.

Low confidence is a workflow state, not a hidden failure. Put the invoice and its evidence into a review queue, preserve the model output, and let the corrected label become an evaluation example only after a person confirms it. Automatic feedback from an unverified label will contaminate the very data used to measure progress.

The catch is operational ownership. This pattern is not suitable when a team cannot maintain a topic schema, review ambiguous invoices, or inspect tenant-level usage. In that case, start with deterministic extraction and manual labeling, then add semantic retrieval after the review process can produce reliable examples. Your mileage may vary with invoice language and tenant policy; a small, tenant-specific evaluation set should settle that uncertainty before a wider rollout.

## The notebook-to-production handoff

Use semantic search when the evidence is distributed across examples or policy text. Use embeddings to narrow the search space. Add reranking when candidate order materially affects the classifier and the measured accuracy gain justifies its latency. Use a closed-schema LLM classifier for the final topic decision, with `unknown` and human review available from the start.

Before deployment, check that the tenant filter is enforced in the index query, the output schema rejects invented topics, and every attempt emits a usage event. Run the same corpus through the notebook and the worker. Compare labels, token counts, retries, and review outcomes—not just a single accuracy number. Then sample the boundary cases again after every schema or prompt change.

| Stage | Question to answer | Evidence to retain |
| --- | --- | --- |
| Retrieve | Did the right tenant and section enter the candidate set? | Filter, IDs, similarity scores |
| Rerank | Did ordering improve the relevant evidence? | Before and after ranks |
| Classify | Did the output follow the topic schema? | Label, confidence, schema version |
| Account | Can usage be attributed to one tenant? | Tokens, retries, operation, tenant ID |

That is the durable implementation: retrieval supplies bounded evidence, classification applies a declared policy, and the usage ledger keeps the cost question answerable per tenant. The exact model or vector backend can change without changing those contracts.

## Further reading

- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- pgvector, Postgres vector similarity extension: https://github.com/pgvector/pgvector
