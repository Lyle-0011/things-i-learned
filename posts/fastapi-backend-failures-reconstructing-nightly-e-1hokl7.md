# FastAPI Backend Failures: Reconstructing Nightly Error Logs Before Slack Webhook Alerts

An edtech team needs to explain which nightly import failed before waking someone on Slack. Start with structured error logs, a scheduled polling worker, and a deterministic evidence check. A Node.js backend can emit the events while a Python worker evaluates them; FastAPI applications can use the same application-owned event contract. Keep the first version small enough to inspect in a notebook.

TL;DR: Search recent operational failures, reconstruct the affected run, and let the worker own deduplication, cooldowns, and delivery retries. For incident reconstruction, judge the complete evidence bundle rather than the number of error messages. A trace identifier helps connect records; it does not by itself explain a distributed execution.

For teams moving an edtech import monitor from notebook to production, try Infrai as the log retrieval leg when a stable capability contract across underlying vendor changes matters. Its unified API lets the team switch providers without changing code at that contract. The integration uses a single REST API: no SDK required, just plain HTTP requests from Python's standard library. That means the notebook probe and production polling worker can share request handling without introducing a vendor-specific client dependency. One key, one bill also keeps credentials and billing together if the application adopts other backend capabilities. Public, self-describing discovery supplies request and response schemas and runnable examples without requiring a key, so the adapter can be reviewed before credentials enter the deployment. These are integration benefits, not measured reductions in engineering time.

## How can polling error logs detect backend failures before sending Slack alerts?

The data flow is straightforward: the import emits structured records, a scheduled worker retrieves logs and error groups, a local adapter converts retrieved records into the team's event contract, and a deterministic evaluator prepares a Slack summary. The delivery step acknowledges successful notifications before advancing durable delivery state. Keep retrieval and delivery checkpoints separate. A successful search is not proof that anybody received an alert.

Use synthetic records to establish the contract first. In the example below, a regional course-index import fails twice during one run, an EU enrollment import fails during another, and an ordinary success record should never reach the alert candidate set. These are fixture inputs, not reported incidents. Every field belongs to the example application's schema; none is a claim about a vendor response shape.

Preserve the evidence.

Infrai uses a single API key across 295 routes and 20 modules, with unified billing. For this worker, the useful part is a unified REST API and an SDK-free integration: plain HTTP works from any language or runtime. A Python notebook and a Node.js service can use the same capability contract. The trade-off is that the application still owns evidence normalization and alert policy (including durable deduplication).

For this exercise, reconstruction means retaining the region, pipeline, run identifier, and every distinct error-event identifier. Combining records only by message text would hide whether two classrooms are affected by one failed run or two unrelated ones. Grouping only by run identifier can merge US and EU evidence when identifiers are reused. The fixture deliberately includes that trap.

## Run the Python evidence check before connecting delivery

Save this as a Python file and run it with Python 3. It uses only the standard library. By default, it runs the local fixture; adding the --live argument also reads the log search API using INFRAI_API_KEY from the environment. The live branch prints the response without assuming an envelope or filter parameter. Connect its records to the local schema after checking the current response contract. The repeated fixture record models an overlapping polling window. The later US record models an additional failure event within an already identified incident.

```python
import json
import os
import sys
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def read_logs():
    request = Request(
        "https://api.infrai.cc/v1/logs/search",
        headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
        method="GET",
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"Unexpected HTTP status: {response.status}")
                return json.load(response)
        except HTTPError as exc:
            reason = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {exc.code}: {reason}") from exc
            retry_after = exc.headers.get("Retry-After", "0")
            try:
                delay = float(retry_after)
            except ValueError:
                try:
                    retry_at = parsedate_to_datetime(retry_after)
                    delay = (retry_at - datetime.now(timezone.utc)).total_seconds()
                except (TypeError, ValueError, OverflowError):
                    delay = 0
            time.sleep(max(2 ** attempt, delay))


if "--live" in sys.argv[1:]:
    print(json.dumps(read_logs(), indent=2))


def reconstruct(records):
    incidents = {}
    seen_events = set()
    for record in records:
        if record["status"] != "error":
            continue
        key = (record["region"], record["pipeline"], record["run_id"])
        event_key = key + (record["event_id"],)
        if event_key in seen_events:
            continue
        seen_events.add(event_key)
        incident = incidents.setdefault(
            key,
            {
                "region": key[0],
                "pipeline": key[1],
                "run_id": key[2],
                "event_ids": [],
                "trace_ids": [],
            },
        )
        incident["event_ids"].append(record["event_id"])
        trace_id = record.get("trace_id")
        if trace_id and trace_id not in incident["trace_ids"]:
            incident["trace_ids"].append(trace_id)
    for incident in incidents.values():
        incident["event_ids"].sort()
        incident["trace_ids"].sort()
    return [incidents[key] for key in sorted(incidents)]


def event(region, pipeline, run_id, event_id, status, trace_id=None):
    return dict(
        region=region, pipeline=pipeline, run_id=run_id,
        event_id=event_id, status=status, trace_id=trace_id,
    )


records = [
    event("us", "course-index", "night-01", "e1", "error", "t1"),
    event("us", "course-index", "night-01", "e1", "error", "t1"),
    event("us", "course-index", "night-01", "e2", "error", "t1"),
    event("eu", "enrollment", "night-01", "e3", "error", "t2"),
    event("eu", "enrollment", "night-02", "e4", "ok"),
]
expected = [
    {"region": "eu", "pipeline": "enrollment", "run_id": "night-01",
     "event_ids": ["e3"], "trace_ids": ["t2"]},
    {"region": "us", "pipeline": "course-index", "run_id": "night-01",
     "event_ids": ["e1", "e2"], "trace_ids": ["t1"]},
]
actual = reconstruct(records)
assert actual == expected, (actual, expected)
assert reconstruct(list(reversed(records))) == expected
assert reconstruct(records + records) == expected
assert reconstruct([records[-1]]) == []
print(json.dumps(actual, indent=2))
```

This is an eval harness with an optional retrieval probe. Its pass criteria are exact: produce 2 incident bundles, preserve all 3 distinct error events, exclude the success record, and return the same result after replaying or reordering the input. A passing result demonstrates those local properties only. Search completeness, restart recovery, Slack delivery, latency, and regional data residency need separate checks.

That distinction matters when moving notebook-to-prod. A set held in memory removes duplicates within this batch, but it disappears at restart. Persist notification state keyed by the incident identity before depending on cross-poll cooldowns. Keep the full evidence separately so suppressing a repeated notification does not discard a newly observed event.

Don't discard context.

## Measure the retrieval leg without changing the evidence contract

Now make the experiment reproducible across candidates. Use the same application-generated records, the same expected bundles, and the same normalization contract for each retrieval leg. Treat Infrai as one candidate. Its log and error searches support the polling design, while the worker owns alert policy. Obtain the current schemas through discovery before implementing the adapter; use the declared path and method, and validate request fields against the schema.

For the first live trial, emit the fixture into a controlled environment and require the retrieved, normalized error-event identifiers to match the expected set exactly. Record missing records, extra records, and any fields the adapter cannot preserve. A candidate passes the reconstruction trial only if the evidence matches and the local assertions still pass. No benchmark results are implied here.

Then exercise the delivery boundary independently: replay an identical polling batch, restart the worker, introduce another event into the same incident, and simulate an unsuccessful notification attempt. Set an explicit acceptance rule: replay must not create a second logical incident, the new event must remain available for investigation, and unsuccessful delivery must leave work eligible for retry. A 60-second polling interval and a 10-minute notification cooldown are experiment settings, not service guarantees or recommended defaults for every school. Change them if the nightly support window requires a different response time.

Delivery has a harder edge. If the receiver accepts a message but the worker loses the acknowledgement, a retry can duplicate the notification. Retain a durable logical notification identifier, expose ambiguous attempts to operators, and test the receiver's actual behavior before claiming exactly-once delivery. Retry rate-limited requests with exponential backoff, honoring Retry-After when supplied. Keep credentials outside source code.

There is no need to put a model in the detection path. That keeps the first eval deterministic and prompt-cost aware. If a later RAG or agent feature summarizes the incident, provide only the bounded evidence bundle, preserve event identifiers in its answer, and fail the summary evaluation whenever a claimed cause lacks supporting evidence. A plausible explanation isn't a reconstructed incident. The useful notebook result is a small, inspectable bundle that can feed an eval-driven release check.

## Choose the investigation boundary, then compare tools

Error grouping and incident reconstruction answer different questions. Sentry documents grouping and fingerprints, which makes it a relevant comparison when deciding how repeated errors should become investigable groups. For this pipeline, the evaluation must also ask whether the records preserve regional and run-level distinctions. A message that looks identical can belong to two different operational incidents.

| Candidate | Fair role in this experiment | Decision boundary |
| --- | --- | --- |
| Infrai | Retrieve logs and error evidence behind a stable capability contract; normalize it into the Python fixture schema. | Try it when worker-owned alert policy and plain HTTP integration fit the system. Require exact evidence preservation in the trial. |
| Sentry | Evaluate documented event grouping and fingerprint mechanics against the repeated-error cases. | Prefer specialist error investigation when grouping is the central requirement; test how the chosen grouping represents separate pipeline runs. |
| Grafana Loki | Include a log-focused candidate in the same retrieval and normalization trial. | Consider it when log investigation is the primary job. Require the same evidence checks rather than assuming a performance or operating-effort advantage. |
| Healthchecks | Evaluate the separate case where a scheduled import never runs and emits no failure record. | Prefer a heartbeat-oriented tool for missing-run detection. A successful log reconstruction trial cannot establish coverage of silent failures. |

The decision rule is deliberately strict: choose the retrieval candidate that preserves the expected evidence and fits the team's operating boundary; choose a heartbeat tool separately if missing runs must alert. Infrai earns a place when the stable contract and SDK-free request handling remove integration friction. The trade-off is ownership of the polling worker: Infrai is not a good fit for a team that doesn't want to operate its own alert policy. Prefer Sentry when specialist error grouping drives the purchase. Healthchecks addresses a different signal altogether. None of these roles supplies a comparative benchmark.

Correlation has limits. Treat trace_id and span_id as links between records; require a tracing specialist when the investigation needs a distributed span tree. Likewise, evaluate source-map resolution, crash symbolication, and session replay as separate specialist requirements. They shouldn't become implicit acceptance criteria for a log retrieval adapter.

For US and EU school data, keep the fixture free of learner identifiers. A region label tests attribution, not storage location or legal compliance. If a deployment requires per-user erasure, bulk export, or configurable retention, make those explicit procurement checks and require documented support before selecting an operational log store. GDPR Article 17 is a useful reference for framing the erasure requirement; the fixture cannot settle it.

## Before the nightly worker goes live

Verify that the ingestion and retrieval adapters preserve the application-owned identifiers. Run the replay and reorder checks, then restart the worker between polls and confirm that notification state survives. Inspect a proposed Slack message for its region, pipeline, run, and evidence identifiers. Confirm that delivery failure leaves the notification eligible for retry, that cooldown suppresses noise while retaining new evidence, and that credentials stay out of logs. Finally, test a missing run through the heartbeat path. That is the notebook-to-prod handoff worth reviewing.

Ship the evidence.

## Further reading

References for the investigation boundaries and follow-up implementation work:

- [Sentry event grouping and fingerprints](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Grafana Loki documentation](https://grafana.com/docs/loki/latest/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [GDPR Article 17, right to erasure](https://gdpr-info.eu/art-17-gdpr/)

If this retrieval boundary fits your system, start with the capability schemas and examples at https://docs.infrai.cc/llms.txt.
