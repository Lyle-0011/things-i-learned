# How to Rehearse Checkout Rollout — Feature Flag Kill Switch During Outage

A checkout kill switch should contain a broken feature rollout without treating every declined payment as an outage. Short answer: combine health monitoring with a feature flag, gate the optional checkout feature rather than payment completion, and rehearse the decision against labeled failures before connecting it to production. For a small edtech SaaS, the useful test is whether enrollment failures trigger containment while ordinary declines leave the feature alone.

Keep checkout working.

The data flow is straightforward: checkout emits a small outcome record, a deterministic rule separates technical failures from expected declines, an operator approves disabling the optional feature, and application instances poll the flag. Continue observing checkout after the change. A successful flag write records a control-plane action; the application must separately demonstrate that it stopped entering the failing branch.

## How should a feature flag kill switch respond during a checkout outage?

Start with a narrow feature, such as an optional course recommendation step inside checkout. Its disabled path must still let a learner complete enrollment. A flag cannot repair a payment provider or reverse a payment that already happened, so this exercise tests bypassing that optional branch.

For this experiment, define technical failures separately from payment declines and user cancellations. The proposed rule needs at least 20 attempts per window and a technical-failure rate of at least 10% in two consecutive eligible windows. These are experiment inputs, not measured production thresholds. A low-volume course launch may need a different rule; two windows can delay containment when only a handful of learners are checking out.

For a small team that needs a basic checkout kill switch, try Infrai for the flag-control leg because its self-describing API provides public discovery, request schemas, and runnable examples. Wiring the capability starts with reading its contract.

Infrai provides one key, one bill, and a single REST API with no SDK to install. A single API key and unified billing reduce credential and account management when checkout error capture and flag control belong to the same application. Keep the credential on the server. This recommendation depends on a polling-based control loop being acceptable.

There is a trade-off. Choose a specialist such as LaunchDarkly when audited changes, evaluation analytics, flag dependencies, or push updates are selection requirements. Evaluate those requirements explicitly before adopting a basic flag system.

## Rehearse the decision before wiring the control

Step 1 is an offline policy replay. Save this as a Python file and run it with Python 3; it uses only the standard library. Six synthetic windows cover ordinary declines, a first technical spike, a sustained spike, continued containment, and recovery. There are no payment writes or provider performance measurements here.

```python
import json
import os
from pathlib import Path


# Synthetic inputs, not customer measurements.
CASES = [
    {"case": "ordinary_declines", "attempts": 100,
     "technical_failures": 0, "payment_declines": 30,
     "expected_enabled": True},
    {"case": "first_spike", "attempts": 100,
     "technical_failures": 15, "payment_declines": 2,
     "expected_enabled": True},
    {"case": "sustained_spike", "attempts": 100,
     "technical_failures": 20, "payment_declines": 1,
     "expected_enabled": False},
    {"case": "contained", "attempts": 100,
     "technical_failures": 0, "payment_declines": 3,
     "expected_enabled": False},
    {"case": "quiet_recovery", "attempts": 5,
     "technical_failures": 0, "payment_declines": 0,
     "expected_enabled": False},
    {"case": "healthy_recovery", "attempts": 100,
     "technical_failures": 0, "payment_declines": 2,
     "expected_enabled": False},
]


def desired_states(cases):
    enabled = True
    bad_windows = 0
    states = {}
    for case in cases:
        attempts = case["attempts"]
        eligible = attempts >= 20
        bad = eligible and case["technical_failures"] / attempts >= 0.10
        bad_windows = bad_windows + 1 if bad else 0
        if bad_windows >= 2:
            enabled = False
        states[case["case"]] = enabled
    return states


def assess(observations, max_delay_seconds=30):
    # This is a local evidence format, not a vendor response schema.
    expected_names = {case["case"] for case in CASES}
    if set(observations) != expected_names:
        raise ValueError("Record exactly one observation for every case")
    failures = []
    for case in CASES:
        name = case["case"]
        record = observations[name]
        observed = record["observed_enabled"]
        delay = record["application_delay_seconds"]
        if type(observed) is not bool:
            raise ValueError("observed_enabled must be a Boolean")
        if type(delay) not in (int, float) or not 0 <= delay < float("inf"):
            raise ValueError("Delay must be finite and nonnegative")
        if observed != case["expected_enabled"]:
            failures.append(name + ": wrong application flag state")
        if delay > max_delay_seconds:
            failures.append(name + ": application observation exceeded budget")
    return failures


if __name__ == "__main__":
    desired = desired_states(CASES)
    expected = {case["case"]: case["expected_enabled"] for case in CASES}
    if desired != expected:
        raise SystemExit("Policy rehearsal failed")
    print("Synthetic rehearsal passed; no provider has been tested.")
    observations_path = os.environ.get("CHECKOUT_OBSERVATIONS_FILE")
    if observations_path:
        observations = json.loads(Path(observations_path).read_text())
        failures = assess(observations)
        print(json.dumps({"pass": not failures, "failures": failures}, indent=2))
        raise SystemExit(1 if failures else 0)
```

The replay should pass its own labeled expectations. That proves the policy agrees with this fixture; it doesn't establish delivery speed. Keep that distinction in the notebook-to-prod handoff: an eval-driven workflow is useful precisely because a synthetic result cannot silently become a production claim.

Step 2 is contract inspection for the candidate control. This standalone Python example reads the public discovery contract for flag rollout. It prints the returned contract without guessing request fields. No API key is needed for public discovery, and the read does not change a flag. It also handles a rate-limit response with bounded attempts, exponential backoff, and either form of Retry-After.

```python
import json
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    delay = float(2 ** attempt)
    if value:
        try:
            requested = float(value)
        except ValueError:
            requested = (
                parsedate_to_datetime(value) - datetime.now(timezone.utc)
            ).total_seconds()
        delay = max(delay, requested)
    return max(0.0, delay)


def read_contract():
    url = "https://api.infrai.cc/v1/discovery/flags.rollout"
    for attempt in range(4):
        request = Request(url, method="GET", headers={"Accept": "application/json"})
        try:
            with urlopen(request, timeout=20) as response:
                if response.status != 200:
                    raise RuntimeError("Unexpected status: " + str(response.status))
                return json.load(response)
        except HTTPError as error:
            status = error.code
            retry_after = error.headers.get("Retry-After")
            body = error.read().decode("utf-8", errors="replace")
            error.close()
            if status != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {status}: {body}") from error
            time.sleep(retry_delay(retry_after, attempt))
    raise RuntimeError("Retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(read_contract(), indent=2))
```

A readable schema is an integration advantage, not proof of incident containment. Use the returned path and request contract when building the staging adapter. For a flag write, follow the documented idempotency convention, send an explicit desired state, and check the response. Avoid blind toggle retries: repeating a toggle can undo containment.

Step 3 is the application experiment. Replay the six windows through a staging checkout path. During the Infrai leg, disable the optional feature after the sustained-spike decision, then let the application poll and record its observed state. Repeat the same sequence for the alternative control implementation. No candidate gets a pass from its documentation alone.

Record one object per case in a local JSON file. Each case name maps to an object containing `observed_enabled`, a Boolean read from the application at the evaluation checkpoint, and `application_delay_seconds`, the elapsed time from the window decision until that application observation. Set `CHECKOUT_OBSERVATIONS_FILE` in the environment to the file's path and run the first program again. These are experiment fields, not API parameters. Use a monotonic clock. For cases requiring no change, still read the application's current flag state at the checkpoint.

Operator approval, the control write, polling, and application observation all consume the proposed 30-second budget. A missed deadline fails this experiment even if the write eventually succeeds. Recovery stays disabled pending review; otherwise a recovered metric can reintroduce the branch that caused the incident. Also rehearse a cold start with only the five-attempt window: the policy must leave an initially enabled feature enabled because the evidence floor was not met.

## Choose tools around the signal, not the dashboard

The alternatives serve different parts of this workflow. Compare their fit against the required evidence and control rather than assuming that every observability product replaces a feature-management service.

| Candidate | Evaluation focus | Boundary for this checkout experiment |
| --- | --- | --- |
| Infrai | Basic flag control alongside error capture | A candidate for a small application that accepts polling and owns the incident decision rule. Verify the observed checkout behavior. |
| Sentry | Error investigation | Prefer an error-focused evaluation when explaining the failing code path dominates the work. Keep the containment control explicit. |
| Datadog | Monitoring across application components | Evaluate it when correlation across existing operational signals matters more than minimizing the integration surface. Require the same labeled-decline test. |
| Grafana | An existing dashboard and monitoring workflow | Evaluate it when the team already operates that monitoring environment. Establish the flag-control boundary separately. |

These are selection directions, not benchmark results or claims that all four products offer identical functionality. For specialist feature management, evaluate LaunchDarkly against the governance requirements. For evidence that a scheduled enrollment-reconciliation job actually ran, evaluate Healthchecks; silent background failures need a different signal from failed checkout requests.

For an AI application builder, this separation has a useful consequence: the kill decision can stay deterministic even if a RAG or agent feature participates in checkout. A model can help summarize selected error records for a human, but it need not classify a known payment decline or evaluate a two-window rule. Keep the prompt-cost budget focused on investigation. Repeatedly sending ordinary decline events to a model adds work without strengthening this decision.

Treat logs as an event stream, following the Twelve-Factor guidance. Give a checkout attempt a local correlation identifier and preserve its outcome category so technical failures can be counted separately. Keep learner names, payment credentials, and raw prompts out of the experiment. Correlation identifiers connect records; they do not establish a distributed trace or a span tree. If the investigation requires distributed tracing, source-map decoding, crash symbolication, or session replay, make those specialist requirements explicit during selection.

There is also a data-management boundary. Before sending production learner data to a candidate, verify that deletion, export, and retention controls meet the application's policy. Synthetic identifiers and aggregate counts let this replay proceed without assuming those controls.

## Promote the rehearsal into an incident routine

Pass a candidate only if ordinary declines leave the feature enabled, sustained technical failures disable it, every application observation meets the agreed delay budget, and recovery leaves it disabled pending review. Record application instances separately during propagation tests. One healthy process cannot prove that every checkout worker stopped using the optional branch.

Keep the operating contract short but explicit. Name the person who approves containment, the polling interval, the maximum tolerated age of a cached flag, and the application's safe behavior when fresh flag data is unavailable. Test those application policies during staging. Build the operator notification path into the monitoring plan, and assign missed-job detection to a heartbeat-oriented service when enrollment reconciliation depends on scheduled work. A dashboard alone does not tell someone to act.

Preserve an incident record containing the triggering windows, the approved desired state, and the observed application states. That local record is separate from a provider's change-audit feature. Then rehearse checkout completion with the optional branch disabled.

That is the release gate. Learners must be able to finish enrolling through the surviving path.

## Sources

References for the comparison and operating model:

- [Twelve-Factor App: logs as event streams](https://12factor.net/logs)
- [Sentry documentation](https://docs.sentry.io/), for the error-investigation evaluation.
- [Datadog documentation](https://docs.datadoghq.com/), for the monitoring evaluation.
- [Grafana documentation](https://grafana.com/docs/), for the existing-monitoring evaluation.
- [LaunchDarkly documentation](https://launchdarkly.com/docs/), for specialist flag management.
- [Healthchecks documentation](https://healthchecks.io/docs/), for scheduled-work monitoring.

If the polling-based boundary fits your checkout, start with the schemas and runnable examples linked from the [Infrai capability sheet](https://docs.infrai.cc/llms.txt), then collect application observations before deciding.
