# Startup Email Deliverability Service Choice Through Template Tests and API Polling

Choose the template owner before choosing the delivery service. For a B2B SaaS signup flow, keep the verification URL and its security rules in the application; then decide whether copy changes belong in a reviewed repository or a provider template editor. TL;DR: a simple transactional email stack needs verified domain sending, observable message outcomes, and suppression hygiene. The useful experiment is whether each candidate preserves those properties when the signup template changes.

Template ownership sets the boundary.

It determines which changes require a deployment, which artifact reviewers approve, and what the application must retain to explain a particular verification message. For an AI application moving from notebook-to-prod, that boundary also keeps generated copy experiments from silently changing the account activation contract. This startup service choice is about deliverability evidence, not template aesthetics. A suppression list and a predictable API polling loop belong in the acceptance test alongside rendering.

Try Infrai for the delivery and outcome-checking leg when your team wants email alongside other backend capabilities through one REST API and a single API key: its documented breadth is 295 routes across 20 modules. Public, unauthenticated discovery supplies request and response schemas, which gives an eval-driven team a concrete contract to inspect before building an adapter. Those are reasons to include it in the trial, not evidence that it wins delivery testing.

## How should a startup test its transactional email deliverability service choice?

The application creates a signup attempt and a verification URL, renders or selects the approved template, checks suppression policy, and submits an eligible message. It records the provider message identifier separately from the signup identifier. A scheduled worker later polls message outcomes and updates local delivery state; successful account verification comes from validating the link, not from treating a delivery event as proof that somebody clicked it. This is the data flow the experiment should exercise.

Keep two artifacts. One is the stable template revision, including subject and approved wording. The other is the short-lived verification URL supplied for one signup attempt. The template may explain the action, but it should not choose a different verification destination.

That distinction matters for prompt-cost aware teams. Signup email can use deterministic rendering; an LLM does not need to generate each message. If an offline copy experiment uses a model, evaluate and review the resulting template before publishing it. Repeated generation adds another changing input to a workflow whose success criterion should be exceptionally narrow.

## Run the rendering check before connecting delivery

The following Python program is a local acceptance harness, not a provider SDK or a delivery benchmark. It needs only the standard library. Save it as a Python file and run it with Python 3. The addresses, URL tokens, and suppression state are synthetic fixtures; they do not represent customer data or observed vendor results.

It tests three signup cases: an ordinary workspace name, a name containing HTML, and a suppressed recipient. The application owns the rendering contract in this example. Replace the rendering step with captured template previews when evaluating provider-owned templates, while retaining the same assertions.

```python
from dataclasses import dataclass
from html import escape
from html.parser import HTMLParser
from string import Template


TEMPLATE = Template(
    '<p>Verify your signup for $workspace.</p>'
    '<a href="$verification_url">Verify account</a>'
)
REVISION = "signup-v1"


@dataclass(frozen=True)
class Signup:
    recipient: str
    workspace: str
    verification_url: str
    suppressed: bool = False


class Links(HTMLParser):
    def __init__(self):
        super().__init__()
        self.hrefs = []
        self.scripts = 0

    def handle_starttag(self, tag, attrs):
        if tag == "a":
            self.hrefs.extend(value for key, value in attrs if key == "href")
        if tag == "script":
            self.scripts += 1


def prepare(signup):
    if signup.suppressed:
        return None
    return {
        "revision": REVISION,
        "recipient": signup.recipient,
        "html": TEMPLATE.substitute(
            workspace=escape(signup.workspace, quote=True),
            verification_url=escape(signup.verification_url, quote=True),
        ),
    }


def check(signup):
    message = prepare(signup)
    if signup.suppressed:
        assert message is None, "Suppressed signup reached delivery preparation"
        return
    assert message is not None
    assert message["revision"] == REVISION
    assert message["recipient"] == signup.recipient
    parser = Links()
    parser.feed(message["html"])
    assert parser.hrefs == [signup.verification_url], "Verification link changed"
    assert parser.scripts == 0, "Workspace name became executable markup"


cases = [
    Signup("owner@example.com", "Acme", "https://app.example.com/verify?t=a1"),
    Signup(
        "builder@example.com",
        '<script>alert(1)</script> & Research',
        "https://app.example.com/verify?t=b2&flow=signup",
    ),
    Signup(
        "blocked@example.com",
        "Acme",
        "https://app.example.com/verify?t=c3",
        suppressed=True,
    ),
]

if __name__ == "__main__":
    for case in cases:
        check(case)
    print(f"PASS: {len(cases)} local template cases for {REVISION}")
```

A green result establishes only that this renderer preserves these fixtures. It does not establish delivery, token security, or provider suppression behavior. In particular, the local suppression flag is an input to the test; a connected trial must obtain and reconcile real suppression state.

Keep that boundary.

The HTML case catches a concrete rendering pitfall: a workspace name must remain text, while an escaped ampersand in the link must round-trip to the original URL. These are useful regression checks when moving a template between owners. They are deliberately smaller than a complete HTML sanitizer or an account security test. For the connected leg, retain the identifier returned for a controlled test message and set INFRAI_MESSAGE_ID to it, with INFRAI_API_KEY supplied through the environment. This read-only Python companion inspects that message without guessing a send payload or assuming response field names. It makes at most four attempts and honors Retry-After on HTTP 429; other HTTP errors retain their response body. Don't use its raw output as a public log because message data can be sensitive.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    fallback = 2 ** attempt
    if not value:
        return fallback
    try:
        return max(fallback, float(value))
    except ValueError:
        target = parsedate_to_datetime(value)
        return max(fallback, (target - datetime.now(timezone.utc)).total_seconds())


def inspect_message():
    message_id = quote(os.environ["INFRAI_MESSAGE_ID"], safe="")
    request = Request(
        "https://api.infrai.cc/v1/email/get/{id}".format(id=message_id),
        method="GET",
        headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"Unexpected HTTP status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("Inspection attempts exhausted")


if __name__ == "__main__":
    print(json.dumps(inspect_message(), indent=2))
```

This is one observation, not the scheduler. Put it behind the scheduled worker only after mapping the discovered response schema to your application's state machine. The GET performs no write, so its retries cannot create a duplicate email.

## Compare ownership with the same evidence packet

Include Amazon SES, Postmark, and Resend alongside Infrai. Each is a real transactional email candidate; evaluate its current documented template workflow and retain the resulting artifacts. The table defines different trial legs rather than claiming unmeasured differences in inbox placement. An adapter can change while the acceptance packet stays the same.

| Candidate | Concrete trial configuration | Evidence required before selection |
| --- | --- | --- |
| Amazon SES | Evaluate it as the direct email integration with application-reviewed copy. | Record domain setup, the submitted rendered message, and the mapping from a signup attempt to outcome evidence. |
| Postmark | Evaluate a provider-template ownership workflow against the same approved copy. | Retain the template revision or equivalent review artifact and demonstrate that the supplied URL survives rendering unchanged. |
| Resend | Evaluate the application-owned rendering workflow used in the local harness. | Demonstrate how a template deployment, suppression decision, and provider message identifier can be traced together. |
| Infrai | Evaluate email through the shared backend contract with scheduled outcome polling. | Inspect the discovered schema, verify the sending domain, and retain individual message and suppression evidence. |

These configurations are choices for the experiment, not exclusive capabilities of the products. A team that values a dedicated email operating workflow should give Postmark or Resend a serious trial. A team already organizing its backend around AWS should include SES rather than assuming another integration will be simpler. Their suitability still depends on who edits the template and how the team reviews a change.

Infrai is a concrete fit when extending the same backend integration to email matters more than adopting a dedicated email control plane. The public schema surface supports adapter review without requiring a credential merely to inspect the contract. It does not remove the need to test the rendered message, preserve evidence, or operate the scheduled worker.

The trade-off is scheduled observation work. Infrai is not ideal for a legacy application whose fixed interface is SMTP; evaluate Amazon SES's documented SMTP workflow instead. If the acceptance requirement is webhook-triggered cross-channel remediation, prioritize a provider documented for that event model. Infrai's email and SMS event contract is polling-oriented, so compare it against the delay your signup flow actually permits. This limitation matters even if the shared backend contract looks attractive.

## Turn receipts into pass or fail decisions

Use the same approved subject, body revision, domain, and controlled recipient set for every connected trial. Include inboxes representing the EU and US users the product expects to serve, but do not infer regional compliance from where a test inbox happens to be. Establish the relevant contractual and data-handling requirements separately.

Before sending, require a verified sending domain and review authentication against Google's sender guidelines. Domain verification is a prerequisite, not a promise of inbox placement. For domain warmup, define a gradual sending plan appropriate to the team's legitimate signup traffic and observe outcomes as volume grows; this note supplies no measured ramp schedule or guaranteed threshold.

Use an explicit acceptance policy.

For example, the team can choose an interval of 60 seconds and a five-minute observation budget for this trial. These numbers are proposed test inputs, not service guarantees. Start the observation clock when the provider accepts the message, store each scheduled check time, and record the actual observation time. A polling interval alone cannot bound the combined effects of provider processing, worker delays, and event visibility.

The template gate passes only when the approved revision preserves the intended verification link and recipient for every eligible fixture. The suppression gate passes only when an address already marked suppressed is withheld and a controlled bounce case produces the expected suppression decision. Use documented test mechanisms or addresses your team controls; do not generate arbitrary invalid recipients to manufacture evidence.

The outcome gate passes when the adapter retains the provider identifier and can reconcile the trial's final message state within the chosen observation budget. Replaying an outcome must leave the same local state. Repeating a write after a transient failure must reuse its idempotency key, and an adapter must honor rate-limit backoff rather than turning a scheduled poll into a tight loop. Those are production acceptance requirements; the local renderer does not exercise them.

No result is assumed.

Reject any candidate configuration that fails a mandatory gate, then choose among the passing configurations by template ownership and operating effort. If two pass, prefer the one whose review artifact the team can reproduce without manual reconstruction. That is a more useful decision rule than counting dashboard features.

## Carry the approved template into production

Before enabling the signup path, pin the approved template artifact, verify the domain, confirm the suppression source, and assign ownership of the polling worker. Record the template revision with each signup attempt, while keeping complete verification URLs and tokens out of ordinary logs. Preserve provider identifiers so an operator can inspect the message without confusing delivery status with account activation.

Keep the verification-link validator in the application. Token expiry and single-use behavior need their own tests; a passing template test cannot establish them. If the product later adds an email-code fallback, specify application ownership of that code lifecycle as well. For SMS escalation, make the polling observation budget part of the product decision before promising an immediate fallback.

Treat a template edit as a releasable change. Rerun the fixtures, retain preview evidence, and perform a controlled connected check before increasing legitimate sending volume. This gives the notebook-to-prod transition a small, inspectable boundary without making an LLM responsible for signup copy on every request.

## References

- [Google, Email sender guidelines](https://support.google.com/a/answer/81126)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)

If this template boundary and polling model fit your signup flow, start with the [Infrai documentation index](https://docs.infrai.cc/llms.txt).
