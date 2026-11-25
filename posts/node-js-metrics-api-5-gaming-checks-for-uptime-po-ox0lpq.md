# Node.js Metrics API: 5 Gaming Checks for Uptime Poll Failures

Keep a gaming experiment reversible with a small worker that polls aggregated health metrics every 60 seconds, compares treatment and control within each tenant, and passes a decision to a separate notification sender. TL;DR: use deterministic thresholds for the first implementation, attach log evidence when needed, and require human approval before a cohort rollback. A Python reference implementation can define the decision contract for a Node.js service without adding an LLM to the alert path.

The useful question is which cohort needs attention, with enough evidence to act safely. A fleet-wide average can hide the tenant whose experimental game assistant is failing. Conversely, rolling back every tenant after a shared dependency failure changes far more than the evidence justifies.

## 1. How should uptime alerts poll metrics every minute?

The data flow is short: a scheduler starts the worker, a metrics adapter supplies completed windows, a pure decision function compares cohorts, and a sender delivers an event to Slack, email, or a webhook. Logs supply incident details and timestamps after the metric check identifies a problem. Keep the adapter and sender outside the decision function so the same inputs can run in a notebook and in production.

Treat 60 seconds as the polling cadence, not an uptime guarantee. Collection delay, window completion, and delivery time all sit between a failure and a notification. A successful metrics query also says nothing about whether a player can complete the game action being measured. Define that action first: for this example, it is a request to the experimental in-game assistant.

Missing is different.

A missing control window, zero traffic, or an incomplete interval should produce a monitor-data warning rather than a rollout recommendation. The example below uses invented fixture counts to demonstrate that distinction; they are test inputs, not measurements of a product or game.

## 2. Make the decision executable before wiring notifications

Start with the read boundary. This standard-library Python example queries aggregated metrics with an explicit GET and no filter parameters. Configure INFRAI_BASE_URL with the service's documented versioned HTTPS API base and INFRAI_API_KEY with a credential in the process environment. The program prints the returned JSON unchanged; it does not guess a metrics response schema. It runs one query per scheduled invocation, with a bounded retry budget for rate limits.

```python
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            retry_at = parsedate_to_datetime(value)
            return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())
    return float(2 ** attempt)


def query_metrics():
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    if not base_url.startswith("https://") or not base_url.endswith("/v1"):
        raise ValueError("Use the documented HTTPS API base ending in /v1")
    request = Request(
        base_url + "/metrics/query",
        headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
        method="GET",
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=10) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected HTTP {response.status}")
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            delay = retry_delay(error.headers.get("Retry-After"), attempt)
            if delay > 20:
                raise RuntimeError("Retry exceeds this tick's wait budget") from error
            time.sleep(delay)
    raise RuntimeError("Retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(query_metrics(), indent=2))
```

Have the scheduler prevent overlapping runs and start a query every 60 seconds. A delay beyond the wait budget surfaces as a failed tick rather than retrying earlier than the server requested. Do not treat that failure as player traffic. Map successful responses into normalized observations only after checking the service's response contract; do not assume server-side cohort filters. HTTP timeouts and connection failures should likewise reach the worker supervisor as failed ticks.

The next standard-library program runs as a single poll evaluation with local fixtures. Every observation must describe the same completed window for its tenant and cohort. Keep this part independent of HTTP so it remains easy to test.

```python
from dataclasses import dataclass
import json


@dataclass(frozen=True)
class Window:
    tenant: str
    cohort: str
    window_id: str
    attempts: int
    failures: int
    complete: bool = True


def evaluate(rows, experiment, min_attempts=100, gap=0.05):
    groups = {}
    for row in rows:
        if row.cohort not in {"control", "treatment"}:
            raise ValueError("Unknown cohort")
        if row.attempts < 0 or not 0 <= row.failures <= row.attempts:
            raise ValueError("Invalid request counts")
        pair = groups.setdefault((row.tenant, row.window_id), {})
        if row.cohort in pair:
            raise ValueError("Duplicate cohort window")
        pair[row.cohort] = row

    events = []
    for (tenant, window_id), pair in sorted(groups.items()):
        event = {
            "event_id": f"{experiment}:{tenant}:{window_id}",
            "experiment": experiment,
            "tenant": tenant,
            "window_id": window_id,
            "automatic_rollback": False,
        }
        if set(pair) != {"control", "treatment"}:
            event.update(action="monitor_data", reason="missing_cohort")
        elif any(not row.complete for row in pair.values()):
            event.update(action="monitor_data", reason="incomplete_window")
        elif any(row.attempts < min_attempts for row in pair.values()):
            event.update(action="monitor_data", reason="insufficient_traffic")
        else:
            control = pair["control"]
            treatment = pair["treatment"]
            delta = (treatment.failures / treatment.attempts
                     - control.failures / control.attempts)
            event.update(
                action="review_rollback" if delta >= gap else "observe",
                reason="failure_rate_gap",
                gap=round(delta, 4),
            )
        events.append(event)
    return events


if __name__ == "__main__":
    fixture = [
        Window("arena-east", "control", "window-001", 200, 2),
        Window("arena-east", "treatment", "window-001", 200, 24),
        Window("arena-west", "control", "window-001", 200, 20),
        Window("arena-west", "treatment", "window-001", 200, 21),
        Window("arena-new", "treatment", "window-001", 8, 2),
    ]
    events = evaluate(fixture, experiment="assistant-prompt-b")
    expected = ["review_rollback", "monitor_data", "observe"]
    assert [event["action"] for event in events] == expected
    print(json.dumps(events, indent=2))
```

The east tenant has a treatment failure rate of 12% against 1% for control. That exceeds the illustrative 5 percentage point gap. The west tenant has failures in both cohorts but only a 0.5 percentage point difference; the comparison does not identify the experiment as the cause. Its absolute failure rate can still warrant a separate service-health alert. The new tenant lacks a control window and cannot support a comparison.

Those distinctions matter more than the delivery channel.

The threshold and 100-request minimum are policy choices for this fixture, not statistical confidence claims. Before production, evaluate them against representative traffic, sparse tenants, and known acceptable behavior. This is a useful notebook-to-prod boundary: keep the fixture and assertions, then expand the eval harness before allowing any action beyond a review request. A new prompt version should pass that harness too.

## 3. Choose the monitoring boundary deliberately

A backend API, a metrics stack, and a missed-job monitor solve different parts of this flow. Compare them by who owns the decision and who owns delivery, because rollback safety depends on that division more than on how few lines the first poll takes.

| Option | Useful role in this design | Boundary to evaluate |
| --- | --- | --- |
| Prometheus with Alertmanager | Metrics evaluation and notification routing in a dedicated monitoring stack | Requires operating or adopting that stack and defining useful metrics and rules. |
| Grafana Alerting | An alert-rule and notification workflow for teams using Grafana | Check data-source support and rule behavior against the actual cohort windows. |
| Datadog | A managed monitoring option to evaluate when the team prefers vendor-operated monitoring | Validate cohort grouping and notification behavior in a trial; adopting it adds another operational dependency. |
| Healthchecks | Detecting a scheduled poller that fails to check in | A check-in confirms worker activity; cohort comparison still needs its own decision logic. |

Infrai combines one REST API, one key, and one bill for metrics and logs. That single API key and unified billing reduce credential handoffs and invoice reconciliation between metric collection and log-evidence retrieval. The capability contract stays fixed when the vendor behind it changes, so swapping that vendor doesn't require changing application code. Its public, keyless discovery surface supplies request and response schemas plus runnable examples, which lets the team check the metrics adapter's contract before connecting it to the rollback eval harness. The platform covers 295 routes across 20 modules. Per-call cost, vendor, and latency metadata also give a prompt-cost-aware team concrete fields to retain alongside experiment results; those fields are context for investigation, not proof that a prompt caused a failure. Keep threshold evaluation and Slack, email, or webhook delivery explicit in the worker design.

For a small experiment, a scheduled worker is an understandable starting point. The trade-off is maintaining the decision state, scheduler, and sender yourself. This approach is not a good fit for a team that needs a managed alerting workflow without owning those components; evaluate Grafana Alerting or Datadog instead. A team already operating Prometheus may get a clearer ownership model by keeping alert evaluation there. Prefer Healthchecks-style monitoring for the separate question of whether the worker ran at all. No amount of request-failure arithmetic can detect a scheduled job that never supplies a window.

## 4. Make notification delivery safe to repeat

A poll should create an event, not directly mutate a rollout. Persist the event identity and delivery state so retries can reuse the same identity. The example's identifier combines experiment, tenant, and window; a production incident record should also group continuing failures across windows, or a one-minute cadence can become one message per minute for the same unresolved incident.

The printed JSON is an event contract, not a Slack, email, or webhook payload. Write a sender for the chosen destination and verify its documented authentication and payload shape. Set explicit HTTP methods, check response status, honor Retry-After when present, and use bounded exponential backoff on rate limits. Persist pending notifications so a failed send does not erase the decision. A local delivery ledger alone cannot guarantee exactly-once delivery if the process stops after a remote send succeeds but before recording success; use destination deduplication where supported and design messages to tolerate duplicates.

Keep secrets out of events.

An alert should carry the experiment, tenant, window, comparison, and a reference to supporting evidence. Raw prompts and player messages rarely belong in notification text. For a RAG or agent feature, retrieve the smallest relevant log slice for diagnosis instead of sending every log through a summarization model on every tick. The alert decision stays deterministic; an optional summary can explain evidence after the decision. This preserves a focused eval surface and keeps prompt consumption tied to investigation.

## 5. Rehearse the rollback boundary before enabling the worker

Start the operational review with the east fixture: confirm that the event identifies only that tenant and asks for review without changing its rollout. Then check the west fixture, where both cohorts have failures, and verify that a separate absolute-health rule can still notify the service owner. Neither comparison establishes causality. Before an operator approves rollback, inspect matching log timestamps and confirm the experiment assignment and current rollout version. An approval for an older version must not change a newer deployment.

Test absence too.

The new tenant should produce a monitor-data warning. A wholly absent tenant produces no event in this example, so production needs an expected-tenant inventory and a freshness check outside the comparison function. Rehearse a missed scheduler invocation with a heartbeat monitor, a repeated window with the delivery ledger, and an interrupted send with the actual destination. These checks expose different failure modes; passing the arithmetic assertions alone does not establish operational readiness.

Finally, review telemetry scope before moving from notebook to production. Keep cohort labels bounded, avoid player identifiers in metric labels, and use logs for detailed incident evidence. Correlated log identifiers help investigation, but a full span tree, browser source-map decoding, and session replay require dedicated diagnostic tooling. Name the person who receives monitor-data warnings and the person authorized to approve a tenant rollback. Enable the minute schedule only after both can recognize the fixture outcomes and find the evidence behind them.

## References

These primary sources cover the monitoring boundaries and implementation practices used in this note.

- Prometheus instrumentation and cardinality guidance: https://prometheus.io/docs/practices/instrumentation/
- Alertmanager notification grouping and routing: https://prometheus.io/docs/alerting/latest/alertmanager/
- Grafana Alerting documentation: https://grafana.com/docs/grafana/latest/alerting/
- Datadog monitor documentation: https://docs.datadoghq.com/monitors/
- Healthchecks check-in and missed-job monitoring: https://healthchecks.io/docs/

## Further reading

- Python HTTP request methods and error handling: https://docs.python.org/3/library/urllib.request.html
- Slack incoming webhook payloads: https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks/
- CLI telemetry opt-out convention: https://consoledonottrack.com/
