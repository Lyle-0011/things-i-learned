# Cheapest One-API-Key Router for Startup Apps: Comparing Token Cost

Short answer: the cheapest one-API-key setup is usually the one that meets a measured quality target with the fewest input and output tokens, not the router with the lowest advertised token price. For a customer-support app that extracts fields from supplier invoices, start with a strict schema, a small evaluation set, and a latency budget; route only after those constraints are visible.

That answer sounds less tidy than comparing the per-token prices of OpenAI, Claude, and Gemini. It is more useful in production. A router can simplify credentials and provider switching, but it cannot make an oversized prompt, a vague extraction contract, or a second retry disappear. Those are the costs that tend to sneak into a startup app.

This is an experiment note, not a vendor ranking. The simple approach is one model, one prompt, and a fallback on every timeout. The chosen approach is a provider-neutral interface with structured output, explicit budgets, and an eval-driven route policy. Before copying it, measure field accuracy, invalid responses, p95 latency, retries, and total tokens on your own invoice set.

## What should one API key, token cost, and latency mean for invoice extraction?

Treat the request as a constrained measurement problem. “Cheapest” needs a denominator. For each invoice, record the number of input tokens, output tokens, attempts, and the elapsed time from request start to validated result. Then calculate cost from the provider's current price sheet or your routing service's current terms. Do not compare a successful first attempt from one model with a three-attempt result from another.

The quality target also needs a definition. An extraction that gets the invoice number right but turns a tax amount into a string with the wrong decimal separator is not fully correct. I would score each field separately, then add a document-level pass condition: required fields are present, numeric values parse, dates use the agreed format, and the result preserves an explicit unknown when the source is unreadable. A confidence score is useful for triage, but it is not proof of correctness.

A minimal record might look like this:

```python
from dataclasses import dataclass


@dataclass
class ExtractionRun:
    provider: str
    model: str
    input_tokens: int
    output_tokens: int
    attempts: int
    latency_ms: int
    valid_schema: bool
    correct_fields: int
    total_fields: int

    @property
    def field_accuracy(self) -> float:
        return self.correct_fields / self.total_fields
```

The exact price is external data and changes over time. Your harness should load it as configuration, timestamp the snapshot, and report cost per successful document. A route that is cheaper per token can lose once it produces longer answers, rejects more schemas, or needs another request. Your mileage may vary, and the evaluation set is what resolves the uncertainty.

Measure twice.

## How can routing compare models without hiding invoice extraction failure modes?

Use a narrow adapter boundary. The application should ask for an invoice extraction, pass the schema and context, and receive either a validated object or a typed failure. It should not know which provider-specific request shape was used underneath. The adapter owns authentication, request serialization, response parsing, and provider errors; the application owns business validation and the retry policy.

The one-key promise is mostly an operations benefit: one secret to rotate, one place to audit, and one usage record to inspect. It doesn't remove the need to understand each model's context limits, structured-output behavior, safety filters, or latency distribution. A single credential also becomes a concentrated blast radius, so scope it to the runtime that needs it and keep it out of notebooks and client code.

Here is the shape I want the rest of a Python service to see. The endpoint is intentionally abstract because the decision is about the contract, not a commercial URL.

```python
from typing import Any, Protocol


class InvoiceModel(Protocol):
    def extract(self, *, text: str, schema: dict[str, Any]) -> dict[str, Any]:
        ...


def extract_invoice(model: InvoiceModel, text: str) -> dict[str, Any]:
    schema = {
        "type": "object",
        "properties": {
            "invoice_number": {"type": ["string", "null"]},
            "supplier_name": {"type": ["string", "null"]},
            "total": {"type": ["number", "null"]},
            "currency": {"type": ["string", "null"]},
        },
        "required": [
            "invoice_number",
            "supplier_name",
            "total",
            "currency",
        ],
        "additionalProperties": False,
    }
    return model.extract(text=text, schema=schema)
```

Structured output is valuable here because downstream code can reject a malformed object before it reaches a ticket, a finance queue, or a database. It still does not guarantee that the extracted value is true. The schema constrains shape; the evaluation harness checks meaning.

## Where does the simple cheapest setup break?

The first failure mode is prompt inflation. Teams append OCR text, a long instruction, examples for every supplier, and the previous failed response. Input tokens rise while field accuracy barely moves. Keep the prompt stable, normalize OCR upstream, and include only the evidence needed for the current document.

The second is an unbounded retry. A parser failure triggers a retry, then a timeout triggers another, and a queue worker eventually sends four billable requests for one invoice. Retry only transient transport failures, cap attempts, and log the reason for each attempt. A schema rejection may need a different model or a human review, not the same request again.

The third is a latency blind spot. Average latency can look fine while p95 makes an agent wait during a busy support shift. Record time to first byte when streaming is available, and time to validated JSON separately. Server-sent events can deliver incremental events over an HTTP connection, but invoice extraction still needs a complete, validated object before it is committed.

The fourth is silent semantic drift. A provider changes a model alias, OCR quality changes, or a new supplier layout arrives. The response remains valid JSON and the dashboard stays green, while a currency field is now wrong. Keep a fixed regression set, add newly reviewed invoices, and alert on per-field accuracy rather than only request success. This is the ugly case: the system can be healthy at the transport layer, return valid structured output, and still send a wrong total into a support workflow. That is why the evaluation set belongs beside deployment checks, not in a one-off notebook.

I like a small decision table because it forces the trade-off into the open:

| Situation | Route decision | Why |
| --- | --- | --- |
| Clean text, common layout, strict latency budget | Use the fastest model that clears the field-accuracy gate | Extra reasoning is not free |
| Messy OCR or unfamiliar supplier | Use the higher-quality route or human review | A second attempt may cost less than a wrong accounting value |
| Valid schema but low field accuracy | Reject and escalate | Shape validation cannot detect a plausible wrong value |
| Transport timeout | Retry within a small capped budget | Recovery is useful; unbounded retries are not |

## What should you measure before switching models or routing providers?

Start with a fixed corpus of representative invoices. Split it by difficulty: clean digital PDFs, noisy scans, tables with multiple totals, and documents with missing fields. Keep the labels outside the prompt so the system cannot accidentally learn the answer. Run the same extraction contract across candidate routes, then inspect the errors rather than sorting by a single score.

My notebook-to-prod path is deliberately boring. The notebook explores error categories and prompt changes. A checked-in test harness freezes the corpus and schema. A staging job records token counts and latency. Production sends a small sample through a shadow route, with no side effects, before changing the default. That sequence makes prompt-cost awareness practical: a shorter prompt is only a win if it preserves the fields that matter.

A useful acceptance rule is: choose the lowest-cost route whose required-field accuracy clears the business threshold, whose invalid-output rate stays below the operational threshold, and whose p95 latency fits the support workflow. If no route clears all three, change the workflow or add review; do not hide the miss in a fallback chain.

The catch is that a one-key router is not suitable when you need provider-specific features, contractual data residency, a precise vendor SLA, or direct control of every request. In those cases, keep a direct integration or split the workload by policy. Stick with a single-provider path when its audit, compliance, and observability requirements matter more than credential consolidation.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://platform.openai.com/docs/guides/structured-outputs
