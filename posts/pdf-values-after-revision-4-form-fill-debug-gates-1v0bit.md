# PDF Values After Revision: 4 Form Fill Debug Gates for Invoices

A form-filling job that finishes without an error but leaves an invoice value blank should be treated as a template-contract failure first. **The fastest diagnosis is to inventory the fields in the exact input PDF, compare their full identities with the mapping used by the job, and record the template digest with every batch.** Do that before changing rendering engines, retrying, or adding more workers. Retries reproduce a deterministic mismatch and consume capacity that should be generating valid invoices.

TL;DR: put four gates between an invoice template revision and a production batch: extract field metadata, reject missing or ambiguous mappings, fill and read back a probe document, then visually inspect a small rendered sample. This protects throughput because malformed templates fail once during promotion rather than once per order.

## Why does a PDF form fill silently ignore values?

A PDF form is a structured document, not a picture with obvious input boxes. ISO 32000-2 defines the Portable Document Format, including the interactive-form structures on which field discovery depends. The label a person sees near a box is therefore not sufficient evidence of the field identity a program must target. A revision can preserve the visible layout while changing the underlying form structure.

That distinction explains the unnerving symptom: the pipeline opens a file, applies a mapping, saves output, and reports success, yet `purchase_order` is empty on the page. The write call completing proves only that the library accepted the operation. It does not prove that the intended widget existed, that the selected field had the expected type, or that a viewer will render its current value as expected.

Start with the artifact. Hash the precise template bytes received by the worker and enumerate its fields before looking at order data. If yesterday's template digest and today's digest differ, a revision entered the system even if both files are named `invoice.pdf`. A mutable filename is a poor release identifier.

Stop the batch.

Consider one concrete revision path. The accounts team moves the visible “PO number” box two centimeters to make room for a longer address, then exports the template under the same filename. The new file looks right in review. Its field inventory, however, contains `po_number_v2` where the released mapping still targets `purchase_order`; a worker that treats an absent target as a no-op can save an apparently healthy PDF. Replaying that order won't reconcile the two names. Comparing the new digest and inventory with the released contract will, and it does so before a queue of 20,000 orders amplifies one template defect into 20,000 bad outputs. Those quantities describe the scenario being tested, not measured performance.

## Make field identity an explicit contract

The useful contract is small: a versioned template digest, a logical business key, the exact PDF field identifier discovered from that template, the expected field type, and whether the value is required. Keep business keys such as `invoice_number` stable even when a template editor changes the underlying identifier. The adapter between those two namespaces is reviewable data, not an assumption buried in filling code.

Do not normalize discovered names by trimming, lowercasing, or taking only the final segment. Normalization can turn two distinct identities into one apparent match, making a revision harder to diagnose. Compare exact strings first. If aliases are needed during a planned migration, list them explicitly and give the compatibility window an owner.

A compact promotion record might capture these gates:

| Gate | Evidence | Reject when |
|---|---|---|
| Inventory | Exact discovered identifiers and field types | A required logical key has no unique compatible target |
| Mapping | Reviewed logical-key-to-field table | An identifier is stale, duplicated, or implicit |
| Probe | Written values read back from a saved test PDF | A required sentinel differs or cannot be read |
| Render | Sample pages inspected as images | A required value is absent, clipped, or visually stale |

The read-back gate and render gate test different things. Reading the saved form checks the document structure. Rendering checks what the recipient is likely to see. Neither should be quietly substituted for the other.

They catch different failures.

## A focused preflight for every revision

The following Python sketch keeps the library-specific work behind an interface. The important behavior is the order of operations and the evidence emitted. The adapter's `inspect`, `fill`, and `read_values` methods must operate on the saved artifact, while the renderer used by the evaluation harness should be pinned and recorded separately.

```python
from dataclasses import dataclass
from hashlib import sha256
from pathlib import Path
from typing import Mapping, Protocol


@dataclass(frozen=True)
class FieldInfo:
    name: str
    kind: str


class PdfFormAdapter(Protocol):
    def inspect(self, source: bytes) -> list[FieldInfo]: ...

    def fill(self, source: bytes, values: Mapping[str, str]) -> bytes: ...

    def read_values(self, document: bytes) -> Mapping[str, str]: ...


def preflight_revision(
    template_path: Path,
    mapping: Mapping[str, str],
    required_keys: set[str],
    adapter: PdfFormAdapter,
) -> dict[str, str]:
    source = template_path.read_bytes()
    fields = adapter.inspect(source)
    discovered = {field.name: field for field in fields}

    missing_keys = required_keys - mapping.keys()
    missing_fields = {
        pdf_name for pdf_name in mapping.values() if pdf_name not in discovered
    }
    if missing_keys or missing_fields:
        raise ValueError(
            f"contract mismatch: missing_keys={sorted(missing_keys)}, "
            f"missing_fields={sorted(missing_fields)}"
        )

    sentinels = {key: f"PROBE-{index:04d}" for index, key in enumerate(
        sorted(required_keys), start=1
    )}
    requested = {mapping[key]: value for key, value in sentinels.items()}
    output = adapter.fill(source, requested)
    observed = adapter.read_values(output)

    mismatches = {
        name: value for name, value in requested.items()
        if observed.get(name) != value
    }
    if mismatches:
        raise ValueError(f"saved-document mismatch: {sorted(mismatches)}")

    return {
        "template_sha256": sha256(source).hexdigest(),
        "field_count": str(len(fields)),
        "required_field_count": str(len(required_keys)),
    }
```

This probe uses deterministic sentinels rather than real order data. That makes failures reproducible and keeps customer values out of test artifacts. It also catches a subtle mapping error that a single repeated test string can hide: two logical keys accidentally targeting the same PDF field. Add an explicit uniqueness check when the invoice schema requires one-to-one mappings.

The example deliberately does not flatten the form or prescribe an appearance-repair call. Those operations are implementation choices, and applying them before proving field identity destroys useful diagnostic evidence. First establish that the expected target exists and retains the requested value in the saved file. Then investigate visual presentation.

The trade-off is promotion latency and extra artifacts. This method is a poor fit for an ad hoc, one-document workflow where a person already inspects every output and templates never enter a shared batch queue. In that case, direct field inspection plus a manual render check may be enough. For recurring invoice runs, I would accept the additional preflight because its cost is paid once per revision; full visual rendering of every invoice is the more expensive control, so sampling depth should follow the document risk and the observed defect rate.

## Promote templates, not filenames

A revised invoice form should move through the same sort of controlled promotion as code. Store an immutable template artifact, its digest, the extracted field inventory, the reviewed mapping, and the probe result together. A production job should select that released identity, not whichever bytes happen to sit behind a familiar path.

This changes the unit of failure. Without promotion, 20,000 queued orders can each discover the same incompatible revision after consuming parse, fill, storage, and retry capacity. With promotion, the incompatible artifact is rejected before it reaches the queue. The number is illustrative, not a throughput claim; teams should evaluate at their own expected batch shape.

Rollouts also need separation between template defects and data defects. Missing `invoice_number` in an order is an input-schema failure. A mapping that points to a nonexistent target is a template-contract failure. A stored value that reads back correctly but is absent from a rendered page is a presentation failure. Give each class a distinct error code and counter so an alert leads to the right owner.

Avoid logging invoice contents. Operational records usually need the order identifier, template digest, mapping revision, stage, duration, and failure class. If debugging requires a document, generate a synthetic probe and retain it under the same access and deletion rules applied to other build artifacts.

No payload dumps.

## What should you measure before adopting this pattern?

**Measure batch throughput as valid invoices per unit time, not merely completed jobs.** A fast worker that silently emits blanks inflates the wrong counter. Track promotion rejection rate, fill latency by template digest, valid-output rate, retry rate by failure class, queue age, and the fraction of sampled renders that pass visual checks.

Use an eval set that resembles the awkward edges of real order data: empty optional fields, long purchase-order references, non-ASCII customer names, multiline addresses, and values near layout limits. The set should contain expected structural values and expected visual outcomes. Pin the template digest and tool versions for each evaluation run so a regression can be reproduced.

Prompt and model calls do not belong in the deterministic filling path. If a model extracts order data upstream, evaluate that extraction separately and pass validated structured data across the boundary. This keeps token cost visible and prevents a form-identity defect from being mislabeled as an extraction failure. Notebook experiments are useful for building the first inventory and probe; production promotion needs a repeatable command, stored evidence, and a failing exit status.

Run a load test only after correctness gates pass. Record parse time separately from fill, save, render, and storage time, then test with the concurrency and document mix expected in the batch window. The decision to cache a parsed template, reuse workers, or pre-render samples should follow those measurements because each optimization changes memory use or isolation.

The practical rule is direct: release an invoice template only when its exact field contract, saved-value probe, and rendered sample agree. That turns a silent blank into a promotion error with a specific owner, while keeping production workers focused on valid orders.

## Further reading

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
