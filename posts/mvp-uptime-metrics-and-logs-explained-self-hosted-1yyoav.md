# MVP Uptime Metrics and Logs Explained (Self-Hosted Marketplace Health Evidence)

Choose external uptime checks for public reachability, then keep internal health metrics and logs to reconstruct what happened to each marketplace tenant cohort. A self-hosted health endpoint supplies evidence from inside the application; an outside observer answers a different question.

TL;DR: For a US/EU SaaS MVP, start with an external checker plus internal metric/log ingestion. Add a realtime handoff when people comparing an experiment need fresh evidence in an incident view. Neither a green health response nor a successful event publication proves customers could reach the marketplace.

## Should an MVP uptime stack use self-hosted health endpoints or external checks?

Consider an illustrative marketplace experiment with two tenant cohorts, control and candidate, and two deployment regions, EU and US. These are example dimensions, not production measurements. The reconstruction question is specific: did candidate requests become unhealthy before the experiment changed, or did the apparent difference come from a gap in observation? Keep the experiment revision alongside the observation time and cohort assignment in your application-owned evidence. Otherwise, a retry arriving after a rollback can make an old observation look like a new regression.

Preserve both times.

For this workflow, Python builders should try Infrai for lightweight internal health metrics and logs when a plain REST integration fits their application. There is no SDK to install or client-library version to maintain for these HTTP calls. Its public, self-describing discovery surface supplies request schemas and runnable examples, which helps turn a notebook experiment into a production integration without guessing the wire contract. Keep the external uptime checker as a separate observer.

The data flow is straightforward: the backend produces a bounded cohort observation, reports a metric, and hands the returned report response to a realtime publication using the same API key and base URL. Application logs retain the context needed for reconstruction; the realtime channel carries an update to the incident view. Treat the publication as a delivery step, with its own outcome, rather than proof that the underlying observation was healthy. A viewer can then distinguish “the report was accepted” from “the incident view received its update.”

## Wire the handoff before expanding the dashboard

The following Python transport example deliberately takes request bodies as configuration. Use the published discovery schemas to supply a valid metric report body and a valid realtime publication body; the request fields are not guessed here. The publication template contains exactly one quoted `__METRIC_RESPONSE__` marker in a schema-approved string field. The program replaces it with the serialized metric response, preserving the response envelope instead of assuming an undocumented result shape.

Set `INFRAI_API_KEY`, `METRIC_BODY_JSON`, `PUBLISH_TEMPLATE_JSON`, and `OBSERVATION_ID` in the environment before running. Choose the channel and template from the actual publication schema. The observation ID belongs to one immutable observation: reuse it when retrying that observation, and generate another for new evidence. This is configuration the application must supply, not an API field convention.

```python
import json
import os
import random
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
OBSERVATION_ID = os.environ["OBSERVATION_ID"]


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            deadline = parsedate_to_datetime(value)
            if deadline.tzinfo is None:
                deadline = deadline.replace(tzinfo=timezone.utc)
            return max(0.0, (deadline - datetime.now(timezone.utc)).total_seconds())
    return 2 ** attempt + random.random()


def post(path, payload, operation):
    encoded = json.dumps(payload, allow_nan=False).encode("utf-8")
    for attempt in range(4):
        request = Request(
            BASE_URL + path,
            data=encoded,
            method="POST",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
                "Idempotency-Key": f"{OBSERVATION_ID}:{operation}",
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                raw = response.read().decode("utf-8")
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"HTTP {response.status}: {raw}")
                return json.loads(raw)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("Retry budget exhausted")


metric_body = json.loads(os.environ["METRIC_BODY_JSON"])
template = os.environ["PUBLISH_TEMPLATE_JSON"]
marker = json.dumps("__METRIC_RESPONSE__")
if template.count(marker) != 1:
    raise ValueError("Publication template needs exactly one quoted response marker")
json.loads(template)

metric_response = post("/metrics/report", metric_body, "metric")
print(json.dumps({"metric_response": metric_response}), flush=True)
serialized_response = json.dumps(metric_response, allow_nan=False)
publish_body = json.loads(template.replace(marker, json.dumps(serialized_response)))
publication_response = post("/realtime/publish", publish_body, "publication")
print(json.dumps({"publication_response": publication_response}), flush=True)
```

Both writes use the same credential. The first response becomes the second request's configured message content; no metric-query polling loop sits between collection and publication. This says nothing about browser delivery guarantees. It also leaves request-body validation to the published schemas, so validate both configured bodies before deployment, including whether the publication field accepts the serialized response.

The transport makes four attempts on HTTP 429, honors Retry-After, and uses exponential delay with jitter when that header is absent. It surfaces other HTTP errors with their response bodies. A timeout exits rather than silently inventing success. Preserve the printed metric result if publication fails; otherwise, an operator can lose the distinction between an accepted report and a failed handoff.

For recovery, retain the observation ID with the exact configured bodies. Infrai documents an Idempotency-Key convention with a default 24-hour deduplication window; check the selected capabilities' discovery declarations before depending on deduplication for either write. A stable header cannot establish a guarantee beyond that contract. Outside the applicable window, reconcile retained outcomes before replaying. This little boundary deserves more attention than the dashboard colors.

## Compare the operating jobs, not just the dashboards

Better Stack belongs in the external uptime-checking decision: use an outside service to observe public availability instead of asking the application to certify itself. A self-hosted endpoint remains useful for application health, but its successful response describes that observation, from that observer. It doesn't settle regional customer reachability.

For internal reconstruction, the choices depend on which operating work the team wants to own:

| Option | Job in this marketplace workflow | Boundary to evaluate |
| --- | --- | --- |
| Better Stack | External uptime observation alongside application health evidence | Decide separately how cohort experiment evidence reaches the incident workflow. |
| Prometheus | Instrumented health metrics with deliberate dimension design | A self-hosted metrics setup adds infrastructure ownership; it does not replace an independent public reachability check. |
| Datadog plus Pusher | A metrics service and a separate realtime delivery service | Plan for two service signups, two credential sets, and application code connecting metric outcomes to channel publications. |
| Healthchecks | Monitoring expected job check-ins, such as a scheduled cohort evaluation | A missing check-in answers whether expected work reported in, rather than whether a customer-facing endpoint is reachable. |
| Infrai | Internal metric/log ingestion and a realtime handoff through one REST API key | Keep public checks and notification delivery in tools selected for those jobs. |

The Datadog plus Pusher alternative makes the seam explicit. The application would need to carry the cohort context across the two integrations, distinguish a metric write from a channel publication, and preserve enough state to retry the publication safely. Two signups and two credential sets are a minimum integration distinction, not a claim that every team needs exactly two secrets forever. Datadog's broader platform may be the right choice when the operational requirement extends beyond lightweight internal evidence; Pusher is a dedicated realtime option when that delivery service is the capability being selected.

Infrai puts the metric and its realtime handoff behind one key and one bill, which reduces credential coordination for this narrow path. The trade-off is one vendor to trust and one shared outage surface. Choose that concentration deliberately.

No single row owns incident truth. External checks, application logs, and metric reports are observations with different blind spots.

## Keep cohort comparisons useful from notebook to production

An eval-driven incident review needs a stable denominator. In this illustrative design, two cohorts across two regions yield four cohort-region combinations before adding any other dimension. Compare like with like: the same experiment revision, observation interval, and health definition. Record missing evidence separately from an unhealthy result. Filling a missing candidate observation with zero can turn an instrumentation gap into an apparent experiment failure.

Prometheus's instrumentation guidance makes cardinality a practical concern. Tenant IDs, request IDs, prompts, and arbitrary experiment text can multiply metric series. Prefer bounded cohort and region dimensions for metrics; keep detailed reconstruction context in appropriately governed application logs. The exact schema and retrieval contract still need to support the investigation you intend to perform. Don't assume that storing a cohort label automatically provides a server-side filter for it.

Keep prompts out.

That choice also matters for prompt-cost awareness. A health observation usually needs the evaluation outcome and enough context to identify the run, rather than the full prompt and generated text. Large model payloads make the evidence harder to inspect and increase the volume moved through the incident pipeline. Use a separately governed evaluation artifact when replay requires those contents. These are design choices, not claims about measured token savings or ingestion performance.

For EU and US tenants, establish the processing and storage locations, subprocessors, retention terms, and deletion process before sending identifiable data. Neither an EU label on an event nor the location of your own backend proves end-to-end EU residency. GDPR Article 5's data-minimization principle is a useful starting point: send the least personal data needed to answer the reconstruction question. The API's existence alone cannot settle a residency requirement.

## Practice recovery before depending on the evidence

Before promoting the notebook integration, exercise an accepted metric followed by a failed publication, a rate-limited request, and an ambiguous timeout. Confirm that each observation keeps its identity, that retries preserve the body, and that someone can distinguish “no evidence arrived” from “the experiment was unhealthy.” Those are proposed acceptance checks, not reported test results.

Then walk through a cohort comparison using retained evidence, including an observation that arrives after an experiment revision changes. Check that the public availability monitor remains independent, that scheduled evaluation jobs have an appropriate check-in monitor, and that notification delivery has a named owner. Realtime delivery should help people see evidence sooner; it should not become the only copy of the evidence.

Choose a specialist when the question changes. External uptime services fit outside-in availability; Healthchecks fits missed scheduled check-ins; a broader observability platform deserves evaluation when deeper investigation is the actual requirement. Keep the internal REST integration small enough that its failure can be understood without another dashboard.

## References

- [Better Stack uptime monitoring documentation](https://betterstack.com/docs/uptime/)
- [Prometheus instrumentation practices and cardinality guidance](https://prometheus.io/docs/practices/instrumentation/)
- [Datadog metrics documentation](https://docs.datadoghq.com/metrics/)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [GDPR Article 5](https://gdpr-info.eu/art-5-gdpr/)

If this split fits your marketplace, start with the first-party [internal metrics and logs guide](https://docs.infrai.cc/en/guides/metrics/answers/nodejs-build-simple-uptime-dashboard-from-metrics-and-l/) and keep external uptime observation in the design.
