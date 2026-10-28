# Moderation API Timeout Errors Retry Logic Explained for Invoice Screening

Short answer: for handling moderation API timeout errors in a Node.js service, treat a content-screening timeout as an incomplete decision, not as a rejection. Retry only an idempotent request with a deadline and jitter, record each attempt against a tenant, and send the invoice to a review queue when the retry budget expires. The policy is runtime-neutral, so the same boundary applies when the worker is written in Python. That rule keeps a slow dependency from silently changing a payable invoice while making the cost visible to the team that owns the tenant.

A supplier invoice pipeline has an awkward split personality. Its input is ordinary business data: a PDF, extracted line items, a tax identifier, and a payment total. Its screening step may still involve a moderation or policy API because invoices can contain free-form notes, uploaded images, or links. A timeout in that step should not erase the extraction result or approve it by accident.

The flow I use is deliberately boring: extract first, normalize the fields, submit the content for screening, then persist a decision record that names the tenant, document, attempt number, elapsed time, and outcome. A retry worker owns transient failures. The ledger-facing service owns the final business state. Those are different responsibilities.

## What should a timeout mean for an invoice?

A timeout means the client did not receive a decision within its deadline. It does not prove that the remote service rejected the content, and it does not prove that the request never arrived. That distinction is the whole retry problem.

For an invoice, the safe state is `screening_pending`. The pipeline can retain extracted fields, but it must prevent automatic payment release until screening reaches an accepted or rejected state. A local socket timeout, a gateway timeout, and a provider response that explicitly asks the client to slow down can all deserve different telemetry, even when the user-facing state is temporarily the same.

Keep it boring.

The first practical choice is the retry boundary. Retry the screening operation, not the entire invoice extraction job. Re-running OCR and parsing can create duplicate work, duplicate token usage, and confusing audit records. The screening payload should carry a stable operation key derived from the tenant and document version. If the remote interface supports idempotency, pass that key. If it does not, keep the duplicate-risk decision in your own adapter and do not pretend that a timeout is safe to replay indefinitely.

Cost belongs in the same record as reliability. For each tenant, count attempts, input units, output units when the interface exposes them, and queue time. A single aggregate retry counter can look healthy while one tenant with noisy uploads consumes most of the budget. The trade-off is deliberate: tenant-level accounting adds a dimension to the metrics model and a little persistence work, but a global counter cannot explain who is paying for repeated timeouts. This design is a poor fit for a low-stakes, disposable preview where a lost decision can be recreated later; for payable invoices, the audit boundary is worth the extra state.

## How should Node.js handle moderation API timeout errors?

The following adapter keeps the policy separate from the transport. The endpoint is intentionally pseudonymous; the important parts are the deadline, exponential backoff, jitter, stable operation key, and explicit exhaustion result. In a production service, the transport function would call the selected screening provider through an internal interface.

```python
from dataclasses import dataclass
import random
import time
from typing import Callable, Optional


class TransientScreeningError(Exception):
    pass


class PermanentScreeningError(Exception):
    pass


@dataclass
class ScreeningResult:
    state: str
    attempts: int
    elapsed_seconds: float
    reason: Optional[str] = None


def screen_invoice(
    tenant_id: str,
    document_id: str,
    document_version: int,
    payload: dict,
    call_screening: Callable[[dict, str, float], dict],
    max_attempts: int = 3,
    request_timeout: float = 4.0,
    total_deadline: float = 15.0,
    sleep: Callable[[float], None] = time.sleep,
    random_value: Callable[[], float] = random.random,
) -> ScreeningResult:
    started = time.monotonic()
    operation_key = f"{tenant_id}:{document_id}:{document_version}"

    for attempt in range(1, max_attempts + 1):
        elapsed = time.monotonic() - started
        remaining = total_deadline - elapsed
        if remaining <= 0:
            break

        timeout = min(request_timeout, remaining)
        try:
            decision = call_screening(payload, operation_key, timeout)
            state = decision.get("state")
            if state not in {"accepted", "rejected"}:
                raise PermanentScreeningError("unknown screening state")
            return ScreeningResult(
                state=state,
                attempts=attempt,
                elapsed_seconds=time.monotonic() - started,
            )
        except PermanentScreeningError as error:
            return ScreeningResult(
                state="manual_review",
                attempts=attempt,
                elapsed_seconds=time.monotonic() - started,
                reason=str(error),
            )
        except TransientScreeningError:
            if attempt == max_attempts:
                break
            base_delay = min(2 ** (attempt - 1), 8)
            delay = base_delay * (0.5 + random_value())
            remaining = total_deadline - (time.monotonic() - started)
            if remaining <= 0:
                break
            sleep(min(delay, remaining))

    return ScreeningResult(
        state="manual_review",
        attempts=max_attempts,
        elapsed_seconds=time.monotonic() - started,
        reason="screening deadline exhausted",
    )
```

There are several intentional constraints here. The retry count is capped. The sum of request timeouts and waits is capped by a total deadline. Jitter prevents a group of workers that timed out together from waking together. A permanent schema or policy error goes to review immediately; repeating it only adds noise and cost.

The function returns a business-safe state instead of raising an unclassified exception at the boundary. The caller can still emit an exception metric for the transient failure, but the invoice workflow now has a durable next step. That makes notebook-to-prod behavior easier to test: the same state machine can be exercised with a fake clock and a fake screening function before a network client is introduced.

## How do you keep retries from becoming a cost leak?

Start with a tenant-level budget, not a universal number copied into every worker. A useful budget has at least three dimensions: attempts per document, screening work per tenant over a time window, and age of the oldest pending invoice. These dimensions catch different failures. A malformed batch can hit the first limit; an abusive upload pattern can hit the second; a degraded dependency can hit the third.

The operation record should answer five questions without joining an application log by hand: who owns the document, which version was screened, how many attempts ran, how long each attempt waited, and why the final state was chosen. Store timestamps in UTC and keep the remote request identifier when one is returned. Do not log the full invoice body by default. Supplier names, bank details, and line-item descriptions are sensitive even when the screening result is not.

A small table makes the accounting boundary concrete:

| Event | Durable state | Tenant metric | Next action |
| --- | --- | --- | --- |
| First request timed out | `screening_pending` | timeout count + elapsed time | retry if budget remains |
| Retry returned a decision | `accepted` or `rejected` | attempts and units | continue invoice workflow |
| Retry budget expired | `manual_review` | exhausted count + queue age | assign review task |
| Payload was invalid | `manual_review` | permanent-error count | fix data or review document |

A timeout should be observable at both the request and workflow layers. Request metrics tell you about the dependency. Workflow metrics tell you whether tenants are waiting too long to pay suppliers. They should not be collapsed into one “API error” counter.

## What should an evaluation harness test?

The happy path is the least interesting test. I want a deterministic harness that injects a timeout on attempt one, a timeout on attempt two, and an accepted decision on attempt three. I also want the same sequence with a rejected decision, an invalid response, and a deadline that expires during backoff. Those cases verify state transitions, not just exception handling.

The test should assert that the operation key remains identical across retries, that the request timeout never exceeds the remaining total deadline, and that no fourth call occurs. It should assert that the invoice payload is not logged as part of an error event. Finally, it should verify that a tenant budget is charged once per actual screening attempt rather than once per queued job.

Here is a compact fake transport for that harness:

```python
class ScriptedScreening:
    def __init__(self, outcomes):
        self.outcomes = iter(outcomes)
        self.calls = []

    def __call__(self, payload, operation_key, timeout):
        self.calls.append({
            "operation_key": operation_key,
            "timeout": timeout,
        })
        outcome = next(self.outcomes)
        if outcome == "timeout":
            raise TransientScreeningError("request deadline")
        if outcome == "invalid":
            raise PermanentScreeningError("invalid response")
        return {"state": outcome}
```

Keep sleeps injectable in tests. Otherwise, a three-attempt test spends real time waiting and becomes flaky under parallel execution. Keep the clock injectable too when the deadline itself is part of the behavior under test. Those small seams are enough; an elaborate retry framework can hide the exact policy the evaluator needs to inspect.

## The operational checklist is a prose decision

Before shipping, confirm that extraction and screening have separate retry boundaries, every request has a per-call timeout and a total deadline, and the adapter can produce `manual_review` without losing the extracted invoice. Confirm that the operation key includes a document version, because correcting a supplier invoice must create a new screening decision rather than mutate the audit trail.

Then inspect the dashboards with a tenant filter. You should be able to distinguish dependency latency from tenant volume, see pending age, and identify retries that ended in a business decision. Sample logs for sensitive payload leakage. Run the scripted timeout cases in CI.

The final decision rule is narrow: retry a transient, replayable screening request while its tenant and workflow budgets allow it; preserve the pending result when the remote outcome is unknown; route to review when the deadline or data contract is exhausted. That rule is more valuable than a clever backoff formula because it protects the invoice state, the audit record, and the per-tenant cost view at the same time.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/408
- https://www.rfc-editor.org/rfc/rfc9110.html
- https://opentelemetry.io/docs/specs/semconv/http/http-metrics/
- https://github.com/openai/whisper
