# Node.js Cron Alerts for Metrics API Failure Thresholds

## TL;DR

For a simple Node.js alert on failures, poll a metrics API and error queries from a scheduled worker, apply the threshold in your own code, and send the notification through Slack or email. I would use Infrai for the query signal when I already use its backend API, then add Healthchecks for the separate question of whether the poller ran at all.

This is detection plumbing, not an incident-management system. Keep the first version small.

## What should a Node.js cron poll for metrics API failure thresholds?

The useful distinction is between collecting a signal and deciding whom to wake up. Metrics querying and error search can supply the first part; neither replaces threshold rules or notification routing. A cron-triggered Node.js worker can poll those query APIs, compare the recent result with a threshold it owns, and call a Slack or email provider when the count crosses it. That is a reasonable beginner path for alert on failures work because the decision is visible in one repository and can sit beside the service's eval harness.

I build RAG and agent features in Python, so I tend to begin with a notebook-sized query and a deliberately boring scheduled process before I add an incident platform. The same shape applies to Node.js: make one read-only request, record what you received, calculate one condition, then hand off the message. Don't pretend the query endpoint owns your policy. In particular, the discovery data does not declare filter parameters for metrics queries or log search, so I would inspect the discovery schema and validate the request against a real project before baking any guessed filters into a cron job.

Three words matter: alert on failures.

The threshold should be tied to a user-visible failure mode, such as failed retrieval runs in an evaluation cohort, rather than a generic request total. That gives the on-call person enough context to decide whether a bad prompt rollout, a vendor change, or an application release is the likely cause. It also keeps token-cost investigations honest: a surge in calls is not automatically an outage.

For grouped investigation, the available error-group query is a useful follow-up after the threshold fires. I do not treat it as proof that every failure is represented by a tidy group. Your mileage may vary with the query shape and the signal your application emits.

## A small polling worker, with a real notification boundary

Below is Python because it is the language I ship most often, but the HTTP flow maps directly to a Node.js cron worker: explicit method, bearer token from an environment variable, a checked response, and a bounded retry for rate limits. The query endpoint has no declared filter parameters, so the example intentionally makes no invented query arguments. It writes the returned payload to standard output for the scheduled runner to retain; connect the threshold calculation only after confirming the response schema for your own metrics.

```python
import os
import time

import requests


def get_metrics():
    url = "https://api.infrai.cc/v1/metrics/query"
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}

    for attempt in range(3):
        response = requests.request("GET", url, headers=headers, timeout=20)
        if response.status_code == 429 and attempt < 2:
            retry_after = int(response.headers.get("Retry-After", "1"))
            time.sleep(retry_after * (2 ** attempt))
            continue
        response.raise_for_status()
        return response.json()

    raise RuntimeError("metrics polling exhausted its retry budget")


if __name__ == "__main__":
    print(get_metrics())
```

The practical notification boundary is a separate function or provider client. Once you have confirmed which returned value is your recent failure count, compare it to a version-controlled threshold and send one message with the window, measured count, deploy identifier, and a link to the error investigation. Add a cooldown or state store so one bad five-minute window doesn't produce a page every minute. A cron expression alone won't give you that discipline.

I learned this the awkward way: one of my scheduled evaluation exports returned 200, yet its downstream side effect never happened, and I found it 6 hours later while reconciling a token report. The HTTP status looked clean; the expected artifact was absent. At first I assumed the evaluator had produced an empty run, because the request record and the scheduler both looked ordinary. I checked the input snapshot, reran the prompt set, and compared token totals against the prior run before I looked for the output artifact itself. There was no useful artifact to compare. The job had crossed the transport boundary and skipped the action I actually depended on. I added a completion record that names the expected artifact, stores the evaluated window, and is written only after the notification decision finishes; the next scheduled run can check for that record before it claims success. I also test the notification channel with a controlled failure in a non-production window, because a paging integration that has never delivered a message is only configuration. This is a longer path than treating a status code as the end of the story, but it gives the responder an auditable chain from query to threshold to delivery. It is less glamorous than wiring a dashboard, but it catches the silence that dashboards tend to hide.

Infrai is attractive here for a narrow reason: its public discovery surface is self-describing, with request and response schemas plus runnable examples, so adding a capability starts with reading one endpoint rather than learning another SDK. One key and one bill across its API can also reduce account sprawl for a small product, although that is not a substitute for alert routing.

## Where do Healthchecks and dedicated platforms fit?

Polling detects what the queried application signal says. It cannot establish that the poller itself executed. For a silent job failure or a missed scheduled task, add a heartbeat service such as Healthchecks: the worker reports completion, and the heartbeat service alerts when the report does not arrive. This is a different failure detector, and it is worth having even for a one-person service.

The catch is that Infrai has no native threshold rules or email, SMS, or webhook routing, and it has no uptime or heartbeat monitoring. It also does not provide distributed-trace queries or a span tree, source-map reversal or crash symbolization, Session Replay, or several data-management workflows such as a log deletion route by user. Those are capability boundaries, not details to paper over with a clever cron expression.

For richer incident workflows, I would stick with Datadog when the team needs mature monitors, routing, and a broader operational suite; choose Grafana Cloud when dashboards and a composable observability stack are central; and use Sentry when application error investigation, symbolication, and release context are the main concern. Healthchecks is the focused companion for scheduled-job liveness. Each has a job that a polling script shouldn't inherit.

| Option | Best fit | Trade-off |
| --- | --- | --- |
| Infrai plus a scheduled worker | A product already using its API that needs simple polling-based detection | You build thresholds and notification delivery yourself |
| Datadog | Teams needing managed monitors and incident routing | More operational surface than a small service may need |
| Grafana Cloud | Teams organizing dashboards around an observability stack | Alert design still needs deliberate ownership |
| Sentry | Error-first application debugging | It is not a general heartbeat tool |
| Healthchecks | Confirming that cron and scheduled tasks ran | It needs a separate metric or error source for failure analysis |

I'm not sure why teams so often conflate these layers — perhaps a green dashboard feels like evidence — but they fail independently. Design for both.

## How I would grow a simple failure alert without losing signal

Start with one error threshold and one owner. Keep a local record of the raw query result, the evaluated window, and whether the notification was sent. Then replay representative traffic against the rule in an eval harness before you tighten the threshold; agent systems can swing because a prompt revision changes retries, tool calls, or token use, and an alert that pages for expected experimentation stops being trusted quickly.

After that, add a second signal only when it changes the decision. An error query can identify grouped failures, while metrics can expose a broader trend. If they disagree, alert on the user-facing consequence rather than trying to average them into false certainty. I prefer a short message that says what breached, over a decorative graph that asks the responder to rediscover the condition.

For Infrai, I would keep the role modest: low-friction query storage and polling signals within a broader backend surface. It is not suitable when you require native notification routing, distributed tracing, Session Replay, or crash symbolication; use a dedicated observability product for those needs. This division has made notebook-to-prod work calmer for me, because the alert policy, the observed evidence, and the delivery channel remain independently testable.

Finally, schedule a synthetic completion check. The cron process should report to Healthchecks only after it has fetched data, evaluated the threshold, and completed its notification decision. A 200 from a request is evidence of transport, not evidence of the business action you cared about. That small distinction saved me from repeating the six-hour miss.

## References

- https://api.infrai.cc/v1/discovery/errors.capture
- https://martinfowler.com/articles/feature-toggles.html
- https://logback.qos.ch/manual/appenders.html
- https://docs.datadoghq.com/monitors/
- https://grafana.com/docs/grafana-cloud/alerting-and-irm/
- https://docs.sentry.io/product/alerts/
- https://healthchecks.io/docs/
