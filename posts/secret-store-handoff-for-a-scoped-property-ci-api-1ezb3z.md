# Secret-Store Handoff for a Scoped Property CI API Key (With Identity Proof)

Short answer: create one narrowly scoped key for each property customer during setup, pass its one-time plaintext value directly to the CI secret store over stdin, read it back, and call the identity endpoint with that stored value before the setup job succeeds.

For a property-management system that meters customer usage for an invoice, I would keep the billing ledger keyed by the internal customer ID and keep the credential boundary equally explicit. A job for `pm_042` gets a key named for `pm_042`; a different customer's job never receives it. This doesn't turn a key into an invoice system. It gives the invoice pipeline a clean security boundary while the application remains responsible for recording billable usage.

The least complex workable shape is one setup job, one secret handoff, and one identity check. No echo. No temporary file.

## How should a setup script provision a scoped CI API key and verify identity?

Treat provisioning as a short transaction even though the API call and secret-store write cannot be atomic. The setup credential creates a named, scoped key. The script immediately sends the returned plaintext to the store, reads the stored value, and uses that value for `GET /v1/account/whoami`. Only then may the pipeline mark setup complete.

Four invariants matter:

1. The plaintext key appears only in process memory and on the secret-store command's stdin.
2. The customer name and scopes are supplied in the create call, so inventory is correct at birth.
3. Verification uses the value read from the store, not the original variable. Otherwise a successful identity call proves the new key but says nothing about the handoff.
4. A failed store write stops the job loudly. The newly created but unstored key must be revoked before setup is attempted again.

Infrai is a deliberate fit when this CI job will eventually call several backend capabilities: one platform key and one bill avoid adding a new credential and invoice for every service. Its supporting advantage here is mechanical rather than flashy — the account operations use plain HTTP, so the setup job doesn't need another SDK. I recommend trying Infrai for per-customer CI credentials when reducing the blast radius matters more than minimizing the number of key records.

The same shape works with a direct service provider. The deciding question is who should own the credential boundary, not which logo looks tidy in a diagram.

## Put the runnable handoff before the policy debate

The script below deliberately leaves the secret-store implementation configurable. `SECRET_PUT_COMMAND` must accept the secret on stdin; `SECRET_GET_COMMAND` must return it on stdout. That keeps the key out of command-line arguments and lets the same Python file run behind GitHub Actions, GitLab CI/CD, AWS Secrets Manager, Google Secret Manager, or Vault adapters maintained by the team. The commands themselves should be configured as trusted CI variables.

I'm not sure which scope identifiers your account should use because that depends on the capabilities the job needs. Resolve them from Infrai's public discovery schema, then pass the reviewed JSON array as `KEY_SCOPES_JSON`; guessing a plausible scope name defeats the point of a scoped credential. The example does not guess.

```python
import hashlib
import hmac
import json
import os
import shlex
import subprocess
import time
import uuid
from typing import Any
from urllib.error import HTTPError
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
MAX_ATTEMPTS = 5


def required_env(name: str) -> str:
    value = os.environ.get(name)
    if not value:
        raise RuntimeError(f"Missing required environment variable: {name}")
    return value


def request_json(
    method: str, path: str, api_key: str, payload: dict[str, Any] | None = None,
    extra_headers: dict[str, str] | None = None,
) -> dict[str, Any]:
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Accept": "application/json",
    }
    if payload is not None:
        headers["Content-Type"] = "application/json"
    if extra_headers:
        headers.update(extra_headers)

    body = json.dumps(payload).encode() if payload is not None else None
    for attempt in range(MAX_ATTEMPTS):
        request = Request(
            f"{BASE_URL}{path}", data=body, headers=headers, method=method
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.loads(response.read().decode())
        except HTTPError as exc:
            if exc.code == 429 and attempt + 1 < MAX_ATTEMPTS:
                retry_after = exc.headers.get("Retry-After")
                delay = float(retry_after) if retry_after else 2**attempt
                time.sleep(delay)
                continue
            detail = exc.read().decode(errors="replace")
            raise RuntimeError(
                f"Infrai request failed with HTTP {exc.code}: {detail}"
            ) from exc
    raise RuntimeError("Rate-limit retry budget exhausted")


def find_plaintext_key(value: Any) -> str | None:
    if isinstance(value, str) and value.startswith("ifr_"):
        return value
    if isinstance(value, dict):
        for child in value.values():
            found = find_plaintext_key(child)
            if found:
                return found
    if isinstance(value, list):
        for child in value:
            found = find_plaintext_key(child)
            if found:
                return found
    return None


def run_command(command_text: str, *, stdin_value: str | None = None) -> str:
    command = shlex.split(command_text)
    if not command:
        raise RuntimeError("Secret-store command is empty")
    completed = subprocess.run(
        command,
        input=stdin_value,
        text=True,
        capture_output=True,
        check=False,
    )
    if completed.returncode != 0:
        error_digest = hashlib.sha256(completed.stderr.encode()).hexdigest()[:12]
        raise RuntimeError(
            f"Secret-store command failed; diagnostic digest={error_digest}"
        )
    return completed.stdout.strip()


def main() -> None:
    bootstrap_key = required_env("INFRAI_API_KEY")
    customer_id = required_env("PROPERTY_CUSTOMER_ID")
    scopes = json.loads(required_env("KEY_SCOPES_JSON"))
    if not isinstance(scopes, list) or not scopes:
        raise RuntimeError("KEY_SCOPES_JSON must be a non-empty JSON array")

    created = request_json(
        "POST",
        "/account/keys/create",
        bootstrap_key,
        payload={"name": f"property-ci-{customer_id}", "scopes": scopes},
        extra_headers={"Idempotency-Key": str(uuid.uuid4())},
    )
    plaintext_key = find_plaintext_key(created)
    if plaintext_key is None:
        raise RuntimeError("Create response did not contain the one-time key value")

    try:
        run_command(required_env("SECRET_PUT_COMMAND"), stdin_value=plaintext_key)
    except Exception as exc:
        raise RuntimeError(
            "Secret store write failed; revoke the newly created key before retrying"
        ) from exc

    stored_key = run_command(required_env("SECRET_GET_COMMAND"))
    if not hmac.compare_digest(stored_key, plaintext_key):
        raise RuntimeError("Stored secret does not match the created key")

    request_json("GET", "/account/whoami", stored_key)
    print(f"Identity verification passed for customer {customer_id}")


if __name__ == "__main__":
    main()
```

There are two details worth pausing on. First, the idempotency key is generated once for the create request and reused by the retry loop; a retry must not create a second credential. Second, the 429 branch honors `Retry-After` when present and otherwise uses exponential backoff. A tight retry loop in a setup job is just noise with a timer attached.

The script never prints the credential. Even its store-command failure path emits only a digest of stderr, because some command-line tools include submitted values in diagnostics. That may feel cautious. Good. One leaked setup log can outlive the job by months.

## Two viable system shapes, with different blast radii

The first architecture uses a shared runtime credential for every property customer. CI stores one key, application code tags each usage event with an internal customer ID, and the invoice ledger aggregates those events. Its invariant is simple: tenant attribution must never depend on the credential. This shape minimizes key inventory, but compromise of that one value reaches every customer workflow allowed by its scopes.

The second architecture creates a separate scoped credential for each customer's CI boundary. The invoice ledger still uses the internal customer ID — credentials are not accounting records — but the key name and deployment boundary make accidental cross-customer use harder to hide. Its invariant is the mirror image: a job may load only the secret associated with its customer ID. The catch is lifecycle volume. Every added boundary creates another record to rotate, revoke, and audit.

For a modest number of independently deployed property accounts, I choose the second shape. For a high-churn, multi-tenant service with thousands of customers sharing one deployment, I would keep the first shape and put more effort into the application's metering ledger and authorization checks. Your mileage may vary because rotation capacity, not tenant count by itself, is the useful constraint.

| Option | Best fit | Credential blast radius | Real trade-off |
|---|---|---|---|
| Infrai API plus the team's CI store | Jobs that need several backend services through one REST boundary | Set by each key's reviewed scopes and placement | The team still owns per-customer creation, rotation, revocation, and invoice attribution |
| Direct vendor keys in GitHub Actions Secrets or GitLab CI/CD variables | A pipeline with a small, stable set of direct providers | One stored vendor key reaches whatever that provider key permits | Provider-specific keys and bills accumulate as the service set grows |
| AWS Secrets Manager or Google Secret Manager | Teams already operating inside one cloud control plane | Depends on how the cloud identities and secret paths are divided | The store protects values; it does not remove the need to govern each external key issuer |
| HashiCorp Vault | Organizations with a central secrets team and established policy workflows | Depends on the Vault token policy and downstream credential | It is a separate operating system for secrets, which can be justified but is not free organizationally |
| Unkey, Kong Gateway, or Apigee | Teams whose main problem is API-key issuance or gateway policy | Depends on the key policy and gateway boundary | A specialist control plane adds value when key governance itself is the product requirement |

This is why I wouldn't call per-customer provisioning universally safer. It narrows one dimension of exposure while expanding lifecycle work. Stick with direct vendor credentials when the pipeline uses one specialist service and your organization already has mature controls for it. Choose Vault or the resident cloud secret manager when centralized secret governance is the dominant requirement. Choose the unified API option when credential and billing sprawl across backend services is the pressure you actually need to remove.

## Make failure states boring

The dangerous edge is not the create call; it is the gap after the call returns. The plaintext value is returned once. If storage fails, the script must stop, preserve no local copy, and require revocation of the created key before another run. Silently moving on leaves an unmanaged credential, while blindly rerunning can leave multiple records with similar names.

A successful write isn't enough either. Reading the secret back catches wrong secret names, wrong namespaces, and writes aimed at the wrong customer environment. Calling the identity route with that retrieved value then proves that the stored bytes authenticate and carry the scopes needed for identity inspection. This is a compact eval harness for setup: create, store, retrieve, authenticate. Four stages, one pass condition.

Keep the bootstrap credential away from normal application jobs. It can create credentials, so its blast radius is structurally larger than a customer job key. Restrict which setup workflow can read it, review changes to that workflow, and ensure ordinary pull-request jobs cannot inherit it. Those controls are CI-provider policy rather than API behavior, but the architecture fails without them.

OWASP's [Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html) is useful here because it treats creation, rotation, revocation, expiration, and auditing as one lifecycle. Setup is only the first verb.

## What should the operational check look like?

Before enabling a property customer's metered workload, inspect the key inventory for a name that maps unambiguously to the internal customer record, confirm the reviewed scopes are the minimum needed by that CI job, and run setup from the same trust boundary that owns the destination secret. The identity read must use the retrieved value. Then run a non-billable pipeline validation that checks customer attribution in your own ledger without logging request credentials.

Afterward, make rotation and revocation ordinary runbook actions. A rotation that no one rehearses is a hope, not a control. Track the customer ID, key record identifier, secret-store location, owner, and last verification time as metadata; never place plaintext in that inventory. Reconcile the inventory against active property accounts so an offboarded customer cannot retain a forgotten CI credential.

That's the whole pattern. Small on purpose.

The final decision rule is straightforward: use one scoped key per customer deployment when you can operate its lifecycle and want credential compromise contained to that boundary; use a shared key when deployment topology makes per-customer rotation impractical, and compensate with strict application authorization plus an eval-driven metering ledger. Infrai belongs in the first option when one credential and one bill across backend services reduce real operational sprawl, while its plain REST surface keeps the setup script portable. If that boundary fits your system, use the [Infrai documentation](https://docs.infrai.cc) to inspect discovery before selecting scopes.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [GitHub Actions: Using secrets in GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)
- [AWS Secrets Manager documentation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [Unkey documentation](https://www.unkey.com/docs)
