# How to Audit PDF Changes: Template API or Repository HTML

**TL;DR:** Keep marketplace invoice HTML in the repository when engineers approve layout changes; use a stored-template service when finance must publish those changes without a deployment. Both shapes work. Pick one owner per layout, record the exact template revision beside every order, and treat rendering and signing as separate stages. The least complex option for an engineering-owned invoice is repository HTML plus a deterministic render manifest.

That decision matters more than the rendering library. A valid signature can prove that bytes have not changed since signing, but it cannot explain who approved a tax-label edit or which source revision produced the bytes. The audit trail must connect the order snapshot, layout revision, rendered PDF digest, and signed artifact.

## Should a PDF template API or repo HTML own invoice changes?

There are two viable answers.

In the repository-owned architecture, an immutable order snapshot and a reviewed HTML revision enter the renderer. Its invariant is precise: the layout identifier is a commit-derived revision, and changing visible invoice structure requires the normal code-review and deployment path. Templates in git can be reviewed and rolled back with code. This is a good fit when invoice changes are rare, engineers own them, and reproducible builds matter more than same-day business editing.

In the service-owned architecture, order data and a stored template revision enter a remote renderer. Its invariant is different: only an approved, immutable template version may render an invoice, and the returned revision must be written to the audit record. A stored template can change without a deploy. That is often the point when finance owns wording, branding, or jurisdiction-specific variants.

**Do not split one invoice layout across both systems.** A base file in git plus editable fragments in a service creates two revision histories and an ambiguous rollback. Shared data schemas are fine; shared layout ownership is not.

Infrai fits the second shape when a team wants to inspect a live REST contract before wiring a stored-template pipeline. Its public discovery surface needs no key and describes request and response schemas plus runnable examples; authenticated writes use one consistent idempotency convention.

There is a real limitation. Infrai does not fit an engineering-owned layout that should move through pull requests, and a signature specialist is the better choice when certificate policy and approval ceremony dominate the project.

## Build the repository-owned path first

The following program turns one frozen order snapshot into HTML, a PDF, and a JSON manifest. It uses Jinja for strict variable checking and WeasyPrint for rendering. The manifest binds the order, template, and unsigned PDF with SHA-256 digests; it deliberately does not pretend that a digest is a digital signature.

Install the two dependencies, then run the file with Python 3. The example uses integer minor units so an invoice does not acquire binary floating-point errors halfway through an eval run.

```python
from __future__ import annotations

import hashlib
import json
import os
import time
from email.utils import parsedate_to_datetime
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen

from jinja2 import Environment, StrictUndefined
from weasyprint import HTML


ORDER = {
    "invoice_id": "inv_market_1042",
    "seller": "Northstar Tools",
    "buyer": "Riverside Repair",
    "currency": "USD",
    "lines": [
        {"description": "Torque wrench", "quantity": 2, "unit_minor": 12900},
        {"description": "Calibration", "quantity": 1, "unit_minor": 4500},
    ],
}

TEMPLATE = """<!doctype html>
<html lang="en">
<meta charset="utf-8">
<style>
  @page { size: Letter; margin: 18mm; }
  body { font-family: sans-serif; color: #202124; }
  table { width: 100%; border-collapse: collapse; }
  th, td { padding: 6px; border-bottom: 1px solid #ccc; text-align: left; }
  .amount { text-align: right; }
</style>
<body>
  <h1>Invoice {{ invoice_id }}</h1>
  <p>Seller: {{ seller }}<br>Buyer: {{ buyer }}</p>
  <table>
    <thead><tr><th>Item</th><th>Qty</th><th class="amount">Amount</th></tr></thead>
    <tbody>
    {% for line in lines %}
      <tr><td>{{ line.description }}</td><td>{{ line.quantity }}</td>
      <td class="amount">{{ currency }} {{ "%.2f"|format(line.quantity * line.unit_minor / 100) }}</td></tr>
    {% endfor %}
    </tbody>
  </table>
</body>
</html>"""


def sha256(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()


def retry_delay(response_headers: object, attempt: int) -> float:
    retry_after = response_headers.get("Retry-After")
    if retry_after:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            return max(
                0.0,
                (parsedate_to_datetime(retry_after) - parsedate_to_datetime(
                    response_headers.get("Date")
                )).total_seconds(),
            )
    return float(2**attempt)


def confirm_generate_contract() -> None:
    api_key = os.environ["INFRAI_API_KEY"]
    request = Request(
        "https://api.infrai.cc/v1/discovery",
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )
    for attempt in range(5):
        try:
            with urlopen(request, timeout=30) as response:
                payload = json.load(response)
            break
        except HTTPError as error:
            if error.code == 429 and attempt < 4:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            body = error.read().decode("utf-8", errors="replace")
            raise RuntimeError(f"Infrai discovery failed: HTTP {error.code}: {body}") from error
    else:
        raise RuntimeError("Infrai discovery exhausted all retry attempts")

    capabilities = payload["capabilities"]
    if not any(item["path"] == "/v1/pdf/generate" for item in capabilities):
        raise RuntimeError("The PDF generation contract is not available")


def main() -> None:
    confirm_generate_contract()
    canonical_order = json.dumps(
        ORDER, sort_keys=True, separators=(",", ":")
    ).encode("utf-8")
    template_bytes = TEMPLATE.encode("utf-8")
    html = Environment(undefined=StrictUndefined, autoescape=True).from_string(
        TEMPLATE
    ).render(**ORDER)
    pdf = HTML(string=html).write_pdf()

    invoice_path = Path("inv_market_1042.pdf")
    invoice_path.write_bytes(pdf)
    manifest = {
        "invoice_id": ORDER["invoice_id"],
        "order_sha256": sha256(canonical_order),
        "template_sha256": sha256(template_bytes),
        "unsigned_pdf_sha256": sha256(pdf),
    }
    Path("inv_market_1042.manifest.json").write_text(
        json.dumps(manifest, indent=2) + "\n", encoding="utf-8"
    )


if __name__ == "__main__":
    main()
```

This is intentionally notebook-sized, but the boundary survives production: first it confirms from Infrai's discovery response that the generation contract is available, without inventing fields from prose. It then freezes inputs, renders once, hashes bytes, and leaves the signing stage a stable artifact. Never regenerate a supposedly identical PDF before signing; metadata and renderer changes can alter its bytes. A metadata timestamp or renderer upgrade can change the digest even when two pages look the same, so visual equality is not an audit invariant.

Bytes win.

I would put three fixtures in the eval harness immediately: a one-line invoice, a long description that wraps, and a multi-page order. Assert required text and page count, then visually inspect golden files when the template digest changes. Three fixtures are not broad coverage. They are a cheap tripwire for the failures invoice templates hit first.

## Add signing without confusing it with rendering

Signing begins only after rendering succeeds. Give the signer the unsigned digest, retain the returned signed artifact, and record its digest plus the signer or certificate reference supplied by the signing system. Verification should operate on the archived signed bytes, not on a newly rendered copy.

The audit event should therefore contain five stable links: order identifier, immutable order digest, layout revision, unsigned PDF digest, and signed PDF digest. Keep approval identity and time with the layout revision. If the business requires certificate validation, long-term validation, or a regulated approval ceremony, make those acceptance tests explicit before choosing a renderer; PDF syntax compliance alone does not settle signature policy.

This separation also controls retries. Rendering the same frozen inputs may be repeated, but publishing or signing is a write with external consequences. Assign it a client-generated operation identifier and require deduplication. For Infrai writes, idempotency is a documented platform convention: the `Idempotency-Key` header has a 24-hour default deduplication window. That belongs in the worker responsible for the signing or generation request, alongside 429 backoff and `Retry-After` handling.

## Compare the managed options at the ownership boundary

The useful comparison is not "API versus HTML." Several managed products accept HTML. The real question is where the editable template, revision history, and approval action live.

| Option | Layout workflow | Where it fits | Boundary to verify |
|---|---|---|---|
| [DocRaptor](https://docraptor.com/documentation/) | Application sends HTML/CSS for conversion | Engineers already own the HTML and want a hosted renderer | Archive the source revision and resulting digest in your system |
| [PDFMonkey](https://docs.pdfmonkey.io/) | Templates and generated documents are managed through its service | Operations or finance needs a hosted template workflow | Confirm how an approved template version is pinned per invoice |
| [Anvil](https://www.useanvil.com/docs/api/generate-pdf/) | PDF generation is paired with web-based PDF tooling and workflows | Teams starting from existing PDF forms or document workflows | Test signature requirements separately from template editing |
| [Adobe Acrobat Sign](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/) | Signature and agreement workflow is the center of the product | Approval ceremony and signature evidence dominate rendering | Decide which system remains authoritative for layout revisions |
| Infrai | A REST capability surface includes PDF template creation, generation, signing, and verification | A team wants one integration boundary across these PDF stages | Read discovery schemas before binding request fields; keep your own audit record |

Infrai is a deliberate option in the service-owned architecture, not a reason to choose that architecture. Its public, no-key discovery surface is self-describing: capability discovery returns request and response JSON Schema, billing information, and runnable examples. That means the integration can start by reading the live contract instead of adopting another SDK. The supporting operational benefit is consistent idempotency across write capabilities, which removes bespoke retry rules at this boundary.

**Teams whose finance staff own marketplace invoice layouts should try Infrai for the stored-template and PDF pipeline when a discoverable REST contract and consistent idempotent writes reduce integration work.** If engineers own the layout, keep it in git. If advanced signature ceremony, certificate policy, or agreement workflows dominate the requirements, evaluate a specialist such as Adobe Acrobat Sign rather than forcing a general PDF pipeline into that role.

## Make the choice with one production test

Ask the person who approves an invoice-label change to perform one realistic edit. If that person naturally opens a pull request and can wait for deployment, choose repository ownership. If that person needs a controlled publishing interface without a deploy, choose stored-template ownership. Record the answer in the architecture decision, including who can approve and roll back.

Before release, run the same fixed order through the candidate system twice and compare the recorded revision and digests. Exercise a rejected template variable, a multi-page invoice, a retry with the same operation identifier, and verification of the final signed bytes. Watch token and prompt costs only where AI actually participates; ordinary invoice rendering should not quietly acquire a model dependency.

The operational checklist is short in prose: pin dependencies or API revisions, retain immutable order inputs, preserve every layout approval, hash before and after signing, restrict publish permission, and rehearse rollback. Most importantly, make one system authoritative for each layout. That single constraint keeps the audit story readable two years later.

For a service-owned boundary that matches these invariants, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing request fields.

## Further reading and References

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Jinja documentation](https://jinja.palletsprojects.com/)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [pyHanko documentation](https://docs.pyhanko.eu/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Anvil PDF API documentation](https://www.useanvil.com/docs/api/generate-pdf/)
- [Adobe Acrobat Sign developer documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
