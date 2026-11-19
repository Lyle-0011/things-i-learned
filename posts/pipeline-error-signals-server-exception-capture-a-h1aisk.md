# Pipeline Error Signals: Server Exception Capture and Repeated Alert Polling Explained

Capture server exceptions centrally, then poll error groups and alert when the same group crosses a threshold in a recent window. For an e-commerce nightly data pipeline, the deciding constraint is signal quality: repeated failures should produce an actionable notification, while another read of the same failures should stay quiet.

Short answer: connect Express or background-worker exceptions to error capture first. Use a small polling worker to evaluate recent grouped failures, keep notification state between polls, and search structured logs for the affected pipeline run. Evaluate this against labeled examples before taking the notebook-to-prod step.

This engineering note uses a synthetic catalog-enrichment failure to explain the decision. It reports no production benchmark. The experiment asks a narrower question: does a repeated-error rule identify a failing stage without turning each exception, retry, or polling cycle into another interruption?

## How should server exceptions trigger an alert on repeated errors?

A simple approach sends a notification for every exception. That preserves individual events, but a pipeline retry can produce another event for the same underlying problem. Polling adds a second trap: a worker can encounter the same unresolved group repeatedly and notify every time. Neither repetition establishes that a new incident has started. The useful unit for notification is a group crossing a recent-window threshold, with enough context to find the underlying log records.

Count recent events.

An unresolved group's accumulated count answers a different question. It can describe a long history of failures even after the current run has recovered. For a trial rule, choose a 10-minute window, a threshold of 3 matching events, and a 60-second polling interval. These are illustrative settings, not measured recommendations. The trade-off: polling every 60 seconds limits read frequency but can delay detection until the next successful poll. The threshold trades sensitivity against noise. A quiet import and a high-volume catalog rebuild need different calibration.

For the local experiment below, the grouping key combines a normalized exception signature with the pipeline stage and run identifier. A request identifier would fragment the group because each retry could receive a different identifier. A broad label such as error would merge unrelated failures. Including the run identifier makes each nightly run independently actionable in this experiment; teams that want one incident to span multiple runs should choose a different notification key. Preserve the underlying events so that changing this policy doesn't erase evidence.

Keep the raw evidence.

The capture service's grouping behavior and this worker's notification key are separate decisions. Inspect the documented grouping contract before adapting the rule. Don't assume that a field in a local fixture is an accepted API parameter.

## Choose the collection and notification boundary

The application boundary comes first. Express server exceptions and background-worker exceptions need a central capture path before a polling process can evaluate grouped failures. Structured logs then supply the run and stage context for investigation. Searching logs helps explain the alert; exception capture supplies the repeated-failure signal. Mixing those responsibilities too early makes the rule hard to test.

There are several reasonable starting points. Sentry is a comparison candidate for a workflow centered on application exceptions; evaluate its grouping and alert behavior against the same labeled failures. Prometheus is the metric-oriented alternative in this comparison: decide what aggregate signal represents the failing stage, and follow its metric naming guidance. Healthchecks addresses the complementary question of whether a scheduled job ran at all. An exception threshold cannot identify a missing run if that run emits no exceptions. These are different decision boundaries, so testing all three against one generic feature checklist would obscure the actual choice.

Infrai pairs a self-describing REST API and public discovery, including request and response schemas and runnable examples, with unified billing and a single key: one key and one bill cover both exception capture and log investigation. That combination fits a small capture-and-poll workflow because developers can inspect the integration before wiring it, while keeping credentials and billing together for those two steps.

Keep scope explicit. Notification thresholds and Slack or email delivery belong to the polling worker in this design. Log trace and span identifiers provide correlation context; a distributed trace investigation requires a tracing tool. Browser or mobile diagnosis that depends on source-map unminifying, crash symbolication, Electron minidump parsing, or session replay belongs in a tool selected for that work. None of those concerns is resolved by changing an error-count threshold.

For this pipeline, first decide whether the failure is an exception burst, a metric anomaly, or a silent missed run. Then evaluate the corresponding tool. The capture-and-poll approach is attractive when a small, understandable rule is enough and the team can own its notification state. A workflow needing richer incident handling should include that operational work in its comparison.

## A focused experiment with structured log records

The following Python program is a complete local experiment using synthetic records. It makes no network calls and uses only the standard library. Its field names are local fixture fields, not a vendor request or response schema. The fixture represents three catalog lookup failures from one nightly e-commerce pipeline run. Searching the structured records by run and stage is what gives the alert an investigation boundary.

The first poll should emit one notification. The next poll sees the same records and should emit none. Once those events leave the recent window, the key becomes eligible for a future threshold crossing. Run the program as a Python file or in a notebook cell.

```python
import json
from collections import Counter

WINDOW_SECONDS = 600
THRESHOLD = 3

records = [
    {
        "timestamp": timestamp,
        "run": "nightly-catalog-demo",
        "stage": "catalog-enrichment",
        "signature": "CatalogLookupError",
        "level": "error",
    }
    for timestamp in (600, 700, 800)
]


def evaluate(rows, now, active_keys):
    counts = Counter(
        (row["run"], row["stage"], row["signature"])
        for row in rows
        if row["level"] == "error"
        and now - WINDOW_SECONDS < row["timestamp"] <= now
    )
    above_threshold = {
        key for key, count in counts.items() if count >= THRESHOLD
    }
    notifications = [
        {
            "run": key[0],
            "stage": key[1],
            "signature": key[2],
            "recent_count": counts[key],
        }
        for key in sorted(above_threshold - active_keys)
    ]
    return notifications, above_threshold


active = set()
for poll_time in (1000, 1060, 1600):
    notifications, active = evaluate(records, poll_time, active)
    for notification in notifications:
        print(json.dumps(notification, sort_keys=True))
```

This emits one JSON notification record. Printing represents the notification decision, not delivery to Slack or email. No external service receives a message. The function keeps evaluation independent of collection and delivery, which makes it useful for an eval harness: pass in labeled records and compare decisions without sending alerts during a test.

State determines correctness.

The deliberate limitation is state. Here, active keys live in memory for one process. A deployed worker needs durable state and coordination if several workers can evaluate the same group. Otherwise a restart can notify again, or two workers can send the same message. Treat evaluation, state changes, and delivery acknowledgments as a workflow with explicit retry behavior. A failed send should remain eligible for retry; a completed send should have a stable deduplication identity.

## Move the rule from a notebook to production

Keep the experiment's input contract small, then implement a collector against the selected service's documented schemas. Verify whether a returned count covers the chosen time window or the group's full lifetime. Where event timestamps are needed, derive the recent count from those events rather than relabeling a cumulative count as recent. Fetching the same event twice must not add two occurrences to the evaluated window.

Start with a read.

This collector performs one grouped-error query and prints the response without assuming undocumented fields. Set OBSERVABILITY_API_BASE to the service's documented HTTPS API base, including its v1 prefix, and INFRAI_API_KEY to your key in the environment. The request uses the verified group-list path. No filter parameters are assumed. Run this separately from the synthetic experiment; inspecting the returned schema is the next step before mapping real records into the evaluator.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen

base = os.environ["OBSERVABILITY_API_BASE"].rstrip("/")
if not base.startswith("https://") or not base.endswith("/v1"):
    raise ValueError("Use the documented HTTPS API base ending in /v1")

request = Request(
    base + "/errors/groups",
    headers={
        "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
        "Accept": "application/json",
    },
    method="GET",
)

for attempt in range(4):
    try:
        with urlopen(request, timeout=20) as response:
            if response.status != 200:
                raise RuntimeError(f"Unexpected status: {response.status}")
            print(json.dumps(json.load(response), indent=2))
        break
    except HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"HTTP {error.code}: {detail}") from error
        delay = float(2 ** attempt)
        retry_after = error.headers.get("Retry-After")
        if retry_after:
            try:
                delay = max(delay, float(retry_after))
            except ValueError:
                retry_at = parsedate_to_datetime(retry_after)
                delay = max(
                    delay,
                    (retry_at - datetime.now(timezone.utc)).total_seconds(),
                )
        time.sleep(delay)
```

This is a read-only query. It does not capture an exception or deliver a notification. Wire application capture first, then adapt the documented group and event responses to the local evaluation contract; the synthetic field names above must stay out of API requests.

An empty result and a failed read need different handling. Only a successful evaluation should update the worker's view of active groups; a collection failure must not silently mark everything recovered. The production loop also needs an explicit HTTP method, status checks, and bounded retries that back off on rate limits and honor Retry-After. These are adapter responsibilities, so keep them outside the pure evaluation function.

There is a timing consequence, too. A group can cross the threshold between polls. Decide whether the next successful poll can still observe that crossing, especially after a delayed worker run. Use event time consistently, define the window boundary, and test delayed observations before trusting the rule overnight.

Keep prompts bounded.

For a RAG or agent feature that summarizes incidents, start with the group identity, run, stage, and a small relevant selection of log records. Sending every repeated stack trace to a model can repeat the same evidence in the prompt. Measure input size and summary usefulness together. Deterministic threshold evaluation should remain inspectable even if a generated explanation is added to the notification.

## Measure usefulness before copying the settings

Build a labeled fixture set containing isolated exceptions, a repeated catalog failure, unrelated failures in another stage, duplicate observations, delayed events, and a missed nightly run. The last case should exercise a separate heartbeat decision. Record which conditions should notify, then evaluate false notifications, missed expected notifications, detection delay, and duplicate notifications across repeated polls. These measurements reveal whether the chosen grouping boundary preserves the signal that matters.

Also test recovery followed by recurrence. The local example rearms as soon as a group falls below threshold; a noisy boundary can therefore cause repeated crossings. If the eval shows that behavior, consider a recovery interval or a different threshold policy and compare the results. Don't add complexity without an observed reason.

The production choice should follow those results: centralized exception capture plus a polling rule for repeated failures, structured logs for the affected run's evidence, and a heartbeat check for silence. Carry the same fixtures from notebook to deployment. The useful outcome is an alert that points to a failing catalog stage and stays quiet when another poll adds no new reason to act.

## References

- [Sentry documentation](https://docs.sentry.io/)
- [Prometheus metric naming best practices](https://prometheus.io/docs/practices/naming/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [OpenTelemetry sampling concepts](https://opentelemetry.io/docs/concepts/sampling/)
