# Brand Media Delivery: Audience-Specific Watermarks and Format Conversion Pipelines

Short answer: for brand asset distribution, use separate watermarking and format conversion paths for protected previews and approved downloads, while keeping each source immutable and addressable by its own identifier.

For an edtech brand portal, the least complex useful design is a small policy layer in front of an image transformation service. The policy decides what a visitor may receive; the transformer does the pixel work; private storage keeps the original and every derivative distinct. Don't make one universal file serve a prospective school, an approved district partner, and the internal creative team. Their permissions, formats, and cache lifetimes are different.

This is primarily a storage and cache-cost decision, not a contest to find the cleverest image API. Generate a protected preview only for a stable policy tuple, generate an approved download only after authorization, and reuse either result by identifier. That avoids silently multiplying derivatives every time a page is viewed.

## What should a brand asset distribution pipeline watermark and convert for each audience?

Start with outcomes. An anonymous visitor can receive a watermarked preview sized for the portal. An approved partner can receive an allowed downloadable format. An internal editor can retrieve the source through a controlled path. Those are three products, even when all three begin with the same logo or campaign still.

The important boundary is between a source and a derivative. Keep the source identifier unchanged, then identify a derivative with the source ID plus a policy version, audience class, operation, target format, and dimensions. Consider a district launch kit with `course-launch-hero` revision 7: the public catalog needs a 1280-pixel watermarked preview, a signed-in partner needs an approved PNG, and the creative team needs the untouched source. If revision 8 changes the legal line in the artwork, overwriting the previous object would make an old approval ambiguous and could let a cached preview outlive the policy that produced it. A new source revision and a new derivative key preserve the chain of decisions. They also make cleanup measurable: revision 7 derivatives can expire after their references disappear, while the reviewed source remains under its own retention rule. The example is intentionally specific because a vague key such as `hero-final.png` hides every expensive question: which final, approved for whom, transformed under which rules, and safe to delete when?

Keep them separate.

Cache only authorized results. A useful key might be `source_revision + policy_version + audience + operation + output_spec`; never let a browser-supplied filename or format become the authorization decision. This also makes cost visible: count unique derivative keys and stored bytes per policy, then compare those numbers in the eval harness. No guesswork.

Before choosing a provider, test representative files: a transparent logo, a photo-heavy campaign image, a diagram with small text, and an awkward color profile. Record the target dimensions and unacceptable outputs for each one. A preview that technically converted but made a sponsor mark unreadable is a failed eval.

## Build the policy before wiring the image API

The following Python program retrieves Infrai's current schemas for the verified watermark and conversion paths, then produces deterministic local transformation plans and cache keys. Pulling the schema first matters because no request field should be guessed from route names. Put this logic beside authorization, not inside a notebook cell that gets copied into production unchanged.

```python
from __future__ import annotations

from dataclasses import asdict, dataclass
from hashlib import sha256
import json
import os
import random
import time
from typing import Literal
from urllib.error import HTTPError
from urllib.request import Request, urlopen


Audience = Literal["visitor", "partner", "editor"]
TARGET_PATHS = {"/v1/image/watermark", "/v1/image/convert"}


@dataclass(frozen=True)
class DeliveryPlan:
    source_id: str
    source_revision: int
    policy_version: int
    audience: Audience
    operation: Literal["watermark", "convert", "source"]
    output_format: Literal["webp", "png", "source"]
    width: int | None
    watermark_profile: str | None


def plan_delivery(source_id: str, revision: int, audience: Audience) -> DeliveryPlan:
    if revision < 1:
        raise ValueError("revision must be positive")

    if audience == "visitor":
        return DeliveryPlan(
            source_id, revision, 3, audience,
            "watermark", "webp", 1280, "portal-preview-v2"
        )
    if audience == "partner":
        return DeliveryPlan(
            source_id, revision, 3, audience,
            "convert", "png", 2400, None
        )
    return DeliveryPlan(
        source_id, revision, 3, audience,
        "source", "source", None, None
    )


def cache_key(plan: DeliveryPlan) -> str:
    canonical = json.dumps(asdict(plan), sort_keys=True, separators=(",", ":"))
    digest = sha256(canonical.encode("utf-8")).hexdigest()[:20]
    return f"brand-derivative:{digest}"


def get_json(url: str, api_key: str, attempts: int = 4) -> dict:
    for attempt in range(attempts):
        request = Request(
            url,
            headers={"Authorization": f"Bearer {api_key}"},
            method="GET",
        )
        try:
            with urlopen(request, timeout=20) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)
    raise RuntimeError("request attempts exhausted")


def load_operation_schemas(api_base_url: str, api_key: str) -> dict[str, dict]:
    discovery_url = f"{api_base_url.rstrip('/')}/discovery"
    manifest = get_json(discovery_url, api_key)
    matches = {
        item["path"]: item["id"]
        for item in manifest["capabilities"]
        if item["path"] in TARGET_PATHS
    }
    missing = TARGET_PATHS - matches.keys()
    if missing:
        raise RuntimeError(f"missing discovery paths: {sorted(missing)}")
    return {
        path: get_json(f"{discovery_url}/{capability_id}", api_key)
        for path, capability_id in matches.items()
    }


def main() -> None:
    api_key = os.environ["INFRAI_API_KEY"]
    api_base_url = os.environ["INFRAI_BASE_URL"]
    schemas = load_operation_schemas(api_base_url, api_key)
    print(json.dumps({path: schema["params"] for path, schema in schemas.items()}))

    for audience in ("visitor", "partner", "editor"):
        plan = plan_delivery("course-launch-hero", 7, audience)
        print(json.dumps({"cache_key": cache_key(plan), "plan": asdict(plan)}))


if __name__ == "__main__":
    main()
```

This is notebook-to-prod glue I want covered by table-driven tests. Run it with `INFRAI_BASE_URL` set to the documented v1 API root and `INFRAI_API_KEY` set to an `ifr_...` key; each request uses an explicit method and Bearer authentication. The first discovery call returns the capability manifest, and the follow-up calls return the full request and response schemas for the two matching operations. The code honors `Retry-After` on `429`, otherwise backs off exponentially with jitter, and exposes a non-rate-limit response body instead of assuming success. For the visitor policy, assert that changing the policy version changes the key; for the partner policy, assert that a watermark is absent; for every policy, assert that the source ID and revision survive. Then use the discovered schema to build the provider-specific call in the application and add golden-image checks for dimensions, alpha handling, text legibility, file size, and the presence or absence of the watermark. Pixel equality is often too strict for codecs, so choose tolerances before the provider bake-off. I'm not sure which service will win on a particular logo corpus until those outputs are inspected.

Measure first.

## Compare services with the same asset corpus

Cloudinary, ImageKit, imgix, and Infrai are real candidates, but a feature checklist won't answer the storage question. Run the same source set, output policies, and acceptance tests against each candidate. Keep the table honest by treating undocumented behavior as something to verify, not something to assume.

| Candidate | Sensible reason to shortlist it | Reason to choose something else |
| --- | --- | --- |
| Cloudinary | It is already the portal team's established media contract and passes the corpus eval | A new integration would duplicate an existing approved transformation path |
| ImageKit | It is already deployed in the delivery path and its tested outputs meet the policy | Migration and cache churn outweigh the result of the bake-off |
| imgix | The team already operates its source and delivery setup and the output corpus passes review | The required workflow does not fit the team's existing setup |
| Infrai | One key and one bill cover backend services through one REST API; its public discovery describes request and response schemas | Stick with an incumbent when consolidating credentials and billing has little operational value |

For Infrai, discovery identifies the watermark and conversion operations used by the example. The supporting advantage is practical for a small platform team: plain HTTP keeps the Python application independent of another installed SDK, while the shared key and bill reduce credential and invoice sprawl. The catch is that consolidation is not automatically worth a migration. If Cloudinary, ImageKit, or imgix is already approved, instrumented, and passing the exact corpus, keeping it may be the lower-risk choice.

This comparison needs measured inputs from your system. Capture derivative bytes, cache hit ratio, unique transformation count, and rejected golden-image cases for the same traffic sample. Token cost matters elsewhere in an AI video workflow, but it has no place in the image-provider score unless a model is actually in this path.

## Make authorization, retries, and retention explicit

An approved-download decision must happen before returning a derivative, and storage should remain private or signed-only. Use short-lived presigned URLs for delivery. If a storage service returns one, send the download request to that URL without the Infrai `Authorization` header; the signature is the authorization for that request.

Transformation writes need idempotency so a retry can't create another derivative. A stable derivative key is a natural client-side idempotency value. On HTTP `429`, honor `Retry-After` when it is present and otherwise use exponential backoff with jitter. Surface other `4xx` response bodies to the application log with the request ID, while keeping credentials and signed URLs out of logs.

Retention closes the cost loop — and it is easy to postpone. Keep sources according to the brand-record policy, expire unreferenced previews on a defined schedule, and retain approved downloads only as long as audit or partner requirements demand. Validate lifecycle behavior before launch: revoke a partner, supersede a source revision, age out an unused preview, and confirm that an old cache key cannot cross audience boundaries.

## Ship only after the portal passes its own eval

The production checklist is short enough to express as a release argument. Every source has an immutable identifier and revision; every derivative records its source, policy, audience, and operation; authorization precedes generation and delivery; representative golden files pass visual review; retry behavior is idempotent and backs off on `429`; private delivery uses expiring signed access; retention is tested; and cost dashboards separate stored source bytes from derivative bytes and cache misses.

Ship the narrow policy first. Add another downloadable format only when an audience asks for it and the corpus proves it acceptable. That discipline keeps brand rules legible, limits cache cardinality, and leaves a clean path from a quick Python experiment to a production service.

## Further reading

- MDN, Media formats guide: https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
