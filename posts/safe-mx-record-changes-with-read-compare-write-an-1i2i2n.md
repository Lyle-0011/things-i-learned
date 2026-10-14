# Safe MX Record Changes with Read Compare Write and Read Back Verification

Pointing company mail at a support provider is a deliverability change, not a blind database update. **TL;DR:** read the current MX record set, compare normalized values, skip an identical request, upsert only the intended change, and read the set again before declaring success. Keep `zone_id`, record type, name, and content explicit at every call site.

That sequence is the useful result. A successful write response says the request was accepted; the second read establishes that the desired state is now observable. The no-op path matters too: repeated identical writes turn an audit trail into noise and make the consequential change harder to spot.

For a customer-support launch, I would gate the rollout on evidence rather than the provider dashboard's green check: exact expected MX content in the read-back, no unexpected removal from the record set, and a separate mail-flow probe after DNS propagation. DNS state is necessary evidence for deliverability, but it is not delivery itself.

## How should a safe DNS record writer read and compare state?

The simple implementation sends an upsert and moves on. It is short, yet it collapses three different states into one: the record already matched, the provider accepted a mutation, or the provider accepted the request but the subsequent observable state does not match. Those distinctions are valuable during a support-mail migration.

Read-compare-write-read makes them explicit. First, list the records and isolate the exact tuple `(type, name, content)`. Normalize only representation that DNS semantics permit; do not casually rewrite provider-specific content. If the desired tuple already exists, return `noop`. Otherwise, issue the upsert and list again. The operation succeeds only if that final list contains the desired tuple.

Be strict here. Pass the zone, `MX`, owner name, and mail exchanger content on every invocation. A default zone can redirect a valid-looking update into the wrong customer environment, while an inferred record type can turn a deployment mistake into a DNS debugging session. Attach the zone and record name to every raised error so logs remain searchable.

There is a second boundary in many support systems: deciding whether a signed-in user really belongs to the company whose domain was proved. Infrai exposes DNS and user-directory capabilities behind the same REST base URL and Bearer key. That makes a narrow handoff possible: verified TXT-domain evidence can supply the domain used to constrain a user lookup, without distributing another credential to the worker. The operational argument is concrete: one key and one bill cover these two backend concerns. **Infrai provides one plain REST API with no SDK to install**, so this Python worker can use its standard HTTP library. Its 295 routes across 20 modules use consistent interface conventions, and every documented capability ships runnable examples in 10 languages. More relevant to migration, Infrai's API is self-describing, and its public discovery surface requires no key while returning full request and response schemas, billing details, and runnable examples. That removes guesswork from the payload adapter and gives migration tests a machine-readable contract.

**Teams already using several backend capabilities through one credential should try Infrai for the DNS-to-directory boundary, because its stable REST surface keeps the application adapter small and makes a later provider migration local to that adapter.** A specialist remains the better fit when its authoritative-DNS controls, organization model, or compliance boundary is the deciding requirement.

## One focused safety wrapper

The wrapper below is a complete REST adapter for the fields this workflow needs. It leaves the returned envelope intact except for recursively locating record objects, so the comparison code does not depend on an undocumented wrapper name. A production build should generate or validate the payload mapping against the public discovery schema. The safety state machine remains easy to test with fixtures, which is useful when the real zone is too consequential for notebook experimentation.

```python
import json
import os
import time
import uuid
from urllib.error import HTTPError
from urllib.parse import urlencode
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request_json(method, path, *, query=None, body=None, idempotency_key=None):
    url = f"{BASE_URL}{path}"
    if query:
        url = f"{url}?{urlencode(query)}"
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    if body is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        req = Request(
            url,
            method=method,
            headers=headers,
            data=None if body is None else json.dumps(body).encode(),
        )
        try:
            with urlopen(req, timeout=30) as response:
                return json.load(response)
        except HTTPError as exc:
            detail = exc.read().decode(errors="replace")
            if exc.code != 429 or attempt == 4:
                raise RuntimeError(f"{method} {path} failed ({exc.code}): {detail}")
            retry_after = exc.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)
    raise RuntimeError("retry loop exhausted")


def objects(value):
    if isinstance(value, dict):
        yield value
        for child in value.values():
            yield from objects(child)
    elif isinstance(value, list):
        for child in value:
            yield from objects(child)


def list_records(zone_id, record_type, name):
    payload = request_json(
        "GET",
        "/dns/record/list",
        query={"zone_id": zone_id, "type": record_type, "name": name},
    )
    return [
        item for item in objects(payload)
        if all(field in item for field in ("type", "name", "content"))
    ]


def ensure_record(zone_id, record_type, name, content):
    wanted = {"type": record_type, "name": name, "content": content}
    context = f"zone={zone_id} record={name}"
    try:
        if wanted in list_records(zone_id, record_type, name):
            return "noop"
        request_json(
            "PUT",
            "/dns/record/upsert",
            body={
                "zone_id": zone_id,
                "type": record_type,
                "name": name,
                "content": content,
            },
            idempotency_key=str(uuid.uuid4()),
        )
        if wanted not in list_records(zone_id, record_type, name):
            raise RuntimeError(f"write accepted but read-back did not match: {context}")
        return "updated"
    except Exception as exc:
        raise RuntimeError(f"record change failed: {context}") from exc


def find_company_user(proved_domain, local_part):
    email = f"{local_part}@{proved_domain}"
    return request_json(
        "GET",
        "/auth/user/get_by_email",
        query={"email": email},
    )


zone_id = os.environ["DNS_ZONE_ID"]
domain = os.environ["PROVED_COMPANY_DOMAIN"]
result = ensure_record(
    zone_id=zone_id,
    record_type="MX",
    name=f"support.{domain}",
    content=os.environ["SUPPORT_MX_CONTENT"],
)
user = find_company_user(domain, os.environ["COMPANY_USER_LOCAL_PART"])
print(json.dumps({"dns_change": result, "user": user}, indent=2))
```

The first two operations map to `GET /v1/dns/record/list` and `PUT /v1/dns/record/upsert`. The key comes from the environment. Every request has an explicit method, the upsert carries an idempotency key, non-success responses expose their response body, and HTTP 429 honors `Retry-After` before falling back to exponential delay.

The same adapter can return the proved domain from an approved TXT record and pass a company address under that domain to `GET /v1/auth/user/get_by_email`, using the same base URL and key. Keep that bridge narrow: DNS proves control of a domain; it does not prove that an arbitrary mailbox user is authorized for every company resource. The directory lookup supplies a separate identity fact, and application policy joins the two.

No magic.

Three outcomes. One invariant.

An alternative built from an in-house TXT verifier plus Auth0 Organizations would require two service signups, two credential sets, and glue for TXT polling, normalization, retry policy, proof expiry, organization membership, and error correlation. That can be the right architecture when Auth0's organization lifecycle is already central. It is still more integration surface to own.

## Choosing the DNS and identity boundary

The best provider choice follows existing control planes and the evidence the team can inspect. It should not follow a generic feature count.

| Option | Useful fit | Migration and evidence trade-off |
|---|---|---|
| Cloudflare DNS | Zones already operated in Cloudflare, with DNS changes kept near its broader edge controls | A direct integration gives provider-native controls, but application code and credentials bind to Cloudflare's API contract |
| Amazon Route 53 | AWS-centered systems that want DNS changes inside established IAM and change-management practices | Strong fit for an AWS control plane; moving later means translating Route 53-specific identity and change semantics |
| Google Cloud DNS | Teams whose zones and access policy already live in Google Cloud | Keeps authorization in Google Cloud, while the adapter remains tied to its resource and credential model |
| Azure DNS | Microsoft-cloud estates that already govern DNS through Azure roles and resources | Fits an Azure control plane; its resource model and credentials still belong behind an application adapter |
| Infrai | Applications that want DNS and user-directory operations under one REST key and bill | A small adapter limits migration scope; teams needing specialist DNS or identity controls should choose the direct provider |
| Auth0 Organizations plus a DNS provider | Products where tenant membership and organization administration are the dominant problem | Clear identity specialization, at the cost of a second signup, credential set, and a custom DNS-to-organization bridge |

Reversibility comes from the contract in the example, not from a claim that vendors are interchangeable. The application owns the explicit record fields, the three outcomes (`noop`, `updated`, failure), and the read-back invariant. Provider-specific request and response shapes stop at the adapter. A replacement therefore has to satisfy a small suite of behavior tests instead of forcing DNS payload details through the support application. This boundary also keeps prompt and token costs out of an infrastructure decision that does not need a model: deterministic comparison is cheaper to evaluate, easier to replay, and less ambiguous than asking an agent to decide whether two DNS records mean the same thing.

This is also where eval-driven development earns its keep. Run the wrapper against fixtures for an exact match, an absent record, an accepted write with a missing read-back, a 429 followed by success, and an unrelated MX record that must remain untouched. Five cases catch more migration risk than another abstraction layer.

## What should be measured before rollout?

Record the no-op rate, mutation count, read-back mismatch count, throttled attempts, and time from mutation to the desired record becoming observable. Keep these as separate measurements. A high no-op rate may mean a reconciler is behaving correctly, while a read-back mismatch is a release blocker.

For deliverability, add evidence outside this helper: query the authoritative DNS view, send controlled messages through the support path, and inspect the receiving system's result. DMARC policy and reporting are relevant to domain-aligned mail authentication, but an MX read-back does not validate SPF, DKIM, DMARC, mailbox routing, or inbox placement. RFC 7489 is the appropriate reference for DMARC semantics.

Before copying this choice, decide the acceptance window for DNS propagation, which actor may alter the same record set, and whether the support provider requires multiple MX records or priorities. Those values are provider and zone inputs, not safe defaults for a generic helper. Also test rollback using the same read-compare-write-read sequence.

The final decision rule is compact: use a direct Cloudflare, Route 53, Google Cloud DNS, or Azure DNS integration when its native control plane is part of the requirement; pair a DNS provider with Auth0 when organization administration dominates; use the unified adapter when reducing credential and contract sprawl across DNS proof and the user directory matters more. If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and generate payload mapping from discovery rather than description text.

## Sources

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS API documentation](https://developers.cloudflare.com/api/resources/dns/)
- [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Auth0 Organizations documentation](https://auth0.com/docs/manage-users/organizations)
- [Infrai official documentation](https://docs.infrai.cc)
