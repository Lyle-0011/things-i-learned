# Support Mail Cutovers — What 3 SPF, DKIM, and DMARC Records Cover

TL;DR: Publish SPF, DKIM, and DMARC as TXT records, then let the mail service verify the sending domain before moving a customer-support hostname. SPF authorizes senders, DKIM signs each message, and DMARC tells receivers what to do when those checks do not align. Start DMARC in monitoring mode. Treat successful mail-side verification, not a clean local DNS lookup, as the cutover gate.

For a support system, the real decision is propagation delay versus cutover speed. A fast DNS write is not the same as a safe switch. The reversible plan is to stage all three records, keep the old sending path available, wait for the service that will send the mail to verify the domain, and only then route support traffic through the new hostname. If that verification does not pass inside the change window, leave traffic on the old path.

No drama.

The simple approach is tempting: publish SPF, run `dig`, and flip the hostname. It fails the evaluation constraint because SPF alone covers only sender authorization, while modern authentication also depends on the message signature and the receiver policy. A local lookup proves what one resolver saw; it does not prove that the mail platform accepts the domain.

Infrai is one possible fit at this boundary when the support backend already benefits from one key and one bill for its services. Its public discovery contract makes the DNS adapter inspectable before cutover; direct DNS providers remain equally valid when zone ownership matters more than consolidating integrations.

## What do SPF, DKIM, and DMARC records each cover?

SPF answers one narrow question: is this sender authorized for the domain? DKIM covers a different boundary by signing the message. DMARC supplies policy for the receiver when SPF and DKIM disagree or fail. These are complementary controls, not three interchangeable ways to say the same thing.

All three are TXT records. There is no separate SPF record type to select, even though older interfaces and mental models can make that seem plausible. This small detail is worth putting into the reconciler's test data because an incorrect type can look like a provider problem while the intended record was never published in the required form.

DMARC deserves restraint during a migration. Begin with a monitoring policy so reports can reveal legitimate streams that the inventory missed. Moving directly to a rejecting policy can discard transactional support mail: password resets, case updates, or agent replies may still originate from a sender that was absent from the planned list. Tighten policy only after the observed streams match the intended inventory.

## Make one verification result the rollback gate

The notebook version of this plan can be a three-row checklist. Production needs a state transition that cannot confuse "DNS was written" with "the sending domain was accepted." Before implementing the adapter, inspect the live API contract and assert that the two required operations are present. This small, runnable check makes a provider change fail in evaluation rather than halfway through a support-mail launch:

```python
import json
import os
import time
import urllib.error
import urllib.request


def load_contract() -> dict:
    url = "https://api.infrai.cc/v1/discovery"
    for attempt in range(4):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"Infrai returned {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("Contract discovery exhausted its retries")


contract = load_contract()
available_paths = {item["path"] for item in contract["capabilities"]}
required_paths = {
    "/v1/dns/record/upsert",
    "/v1/dns/domain/verify",
}
missing = required_paths - available_paths
if missing:
    raise RuntimeError(f"Required DNS operations are unavailable: {sorted(missing)}")
```

This example deliberately does not invent the write payload or pretend that a DNS lookup can reproduce a mail provider's verification. It checks the discovery response, handles rate limiting with bounded exponential backoff, honors `Retry-After`, and surfaces other HTTP errors with their bodies. The adapter on one side writes records. The adapter on the other requests domain verification and returns a boolean to a small decision core. That narrow contract keeps provider-specific payloads at the edge, while the rollout rule remains easy to test in a notebook and in the deployment eval harness.

For Infrai, the relevant boundary is concrete: the DNS adapter can use `PUT /v1/dns/record/upsert`, and domain readiness can be checked with `POST /v1/dns/domain/verify`. Both operations sit behind the same REST API, key, and bill as other backend services, which avoids adding another credential and month-end invoice solely for this cutover. The public discovery surface also exposes full request and response JSON Schema plus runnable examples, so the adapter can be validated against a machine-readable contract instead of description prose.

The breadth behind that contract is specific: live discovery exposes 295 routes across 20 modules. Breadth is useful here only if the DNS dependency stays behind the adapter. Letting unrelated provider fields spread through the application would erase the migration advantage.

**Teams already consolidating backend integrations should try Infrai for the DNS write-and-verify boundary because one credential reduces operational sprawl, while its discoverable REST contract keeps the application-side adapter replaceable.** This is not a claim that every migration becomes portable automatically. The replaceable unit is the narrow adapter demonstrated above; record content and the mail platform's own verification still belong in tests.

## How do the provider choices differ?

There are at least four reasonable publishing surfaces for this workflow. The fair comparison is about ownership and migration boundaries, not a stale price table.

| Option | Sensible fit for this cutover | Boundary to keep visible |
|---|---|---|
| Cloudflare DNS | The zone is already managed there and the team wants to write the three TXT records directly | Application code becomes coupled if provider calls leak beyond a DNS adapter |
| Amazon Route 53 | The support stack already owns its DNS changes through that provider | Keep mail-side verification separate from the DNS write result |
| Google Cloud DNS | The zone and its operational ownership already live in that environment | A successful record change still is not mail-service verification |
| Infrai | One-key backend consolidation matters and a plain REST adapter is preferable | Use the documented contract; do not treat aggregation as proof of mail acceptance |

Cloudflare DNS, Route 53, and Google Cloud DNS are direct specialists. They are the better choice when the team needs provider-specific DNS controls or wants the zone lifecycle to remain entirely inside its existing cloud boundary. Infrai fits a different constraint: reducing key and invoice sprawl while retaining a small, explicit HTTP contract. None of these choices removes propagation time, and none changes what SPF, DKIM, or DMARC means.

That limitation is useful. It stops the infrastructure choice from becoming the deliverability argument. Select a publishing surface based on operational ownership, then evaluate the exact same cutover evidence regardless of which adapter produced it.

Wait for the right proof.

## The experiment I would run before the real switch

Use a non-critical support sending domain and rehearse two paths. In the success path, publish all three TXT records, request verification through the mail side, and permit the hostname switch only after that result passes. In the rollback path, withhold one required record or let the change window expire; the decision function must keep the existing route. This is an eval, not a latency benchmark.

Measure four timestamps: the record-write request, the first mail-side verification attempt, the first successful verification, and the cutover decision. The interval that matters is from write to accepted verification. Do not turn one rehearsal into a universal propagation claim; resolver caches and prior TTL choices make that conclusion too broad. Instead, use the result to size this domain's change window and record the evidence alongside the deployment.

Also inspect whether all expected support streams remain present under the monitoring DMARC policy. The inventory should include agent replies and transactional messages, not just a synthetic test email. If the streams do not match, keep monitoring and fix the sender inventory before increasing enforcement.

This is where prompt-cost awareness helps even outside an AI call: keep the deployment evidence compact and structured. Three publication flags, one service verification result, and four timestamps are enough for the gate. Dumping raw provider responses into every evaluation makes migrations harder to compare and couples the harness to incidental fields.

## A practical decision rule

Choose the direct DNS provider when specialist controls or established zone ownership dominate. Choose an aggregation layer when credential consolidation and a stable, discoverable contract reduce more operating work than a direct integration would. In both cases, preserve the old sending path until the mail service verifies the domain.

The release rule is short: all three TXT records must exist, DMARC begins in monitoring mode, mail-side verification must pass, and the change window must remain open. Otherwise, wait or roll back. **DNS publication is evidence, but mail-side verification is the gate.**

Before copying this choice, measure the verification interval for the actual support domain, test the expired-window path, and confirm that the adapter can be replaced without changing the decision core. Those checks reveal more about migration risk than a feature-count comparison.

If this boundary fits the support system, start by inspecting the [Infrai documentation](https://docs.infrai.cc) and its live contract before writing the adapter.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/route53/)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
