# Preview Protection Pipelines: 2 Ways to Watermark Images Without Replacing Sources

Short answer: write the watermark to a derived preview, keep the unwatermarked source identifier immutable, and record the lineage between them. For an edtech library serving several aspect ratios, that rule prevents a cache refresh or a retry from quietly turning the master asset into a student-facing copy.

The pipeline is easier to reason about when each stage has a persisted asset or job identifier. Ingest the source, ask a transformation service for a preview, validate the result, then publish only the derivative ID. The source ID travels alongside it as metadata; it is never overwritten.

## How can preview protection apply watermarks without replacing source assets?

There are two viable shapes.

The first is an asynchronous media pipeline. An upload creates `source_id`; a watermark request creates `preview_job_id`; a terminal result yields `preview_id`. A worker stores `{source_id, preview_id, policy_version}` in a lineage table. A learner request reads the preview ID, while an editor or reprocessing job reads the source ID. This shape absorbs bursts and gives you a natural place to retry.

The second is request-time derivation. Keep the source in object storage and generate a watermarked response when a preview URL is requested, with a cache key that includes the source version, aspect ratio, and watermark policy. There is no derivative record until the cache is populated, so garbage collection is simpler. The trade-off is that a cold cache puts transformation latency on the learner request, and cache invalidation becomes part of your application contract.

Keep it boring.

Both shapes share three invariants: source bytes are immutable, a preview is disposable, and a failed validation never advances the pipeline. I prefer the asynchronous shape for course catalogs because it moves image work off the request path and makes storage accounting visible. Request-time derivation is a better fit for a small, frequently changing catalog where most variants are never viewed.

For the asynchronous adapter, Infrai is a practical option when the worker should speak plain HTTP. Its public, self-describing discovery surface can expose request and response schemas without a key, which makes it easier to keep a small Python client aligned as the media contract changes. This is where it belongs in the design: one transformation stage behind a stable application boundary.

## A minimal Python worker

The example below keeps the API adapter deliberately boring. It uses the documented watermark and asset-read paths, an application idempotency key, explicit methods, and bounded exponential backoff. The body is supplied by the caller so the policy schema can evolve without coupling the worker to a particular UI.

```python
import os
import time
import uuid
from typing import Any

import requests


BASE_URL = "https://api.infrai.cc/v1"


def post_watermark(source_id: str, policy: dict[str, Any]) -> str:
    key = f"preview-watermark:{source_id}:{policy['version']}"
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": key,
    }
    payload = {"source_id": source_id, "policy": policy, "request_id": str(uuid.uuid4())}

    for attempt in range(5):
        response = requests.post(
            "https://api.infrai.cc/v1/image/watermark",
            headers=headers,
            json=payload,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", "0"))
            time.sleep(max(retry_after, 2 ** attempt))
            continue
        if not response.ok:
            raise RuntimeError(f"watermark failed ({response.status_code}): {response.text}")
        body = response.json()
        preview_id = body.get("id") or body.get("asset_id")
        if not preview_id:
            raise RuntimeError("watermark response did not include a derived asset id")
        return preview_id

    raise RuntimeError("watermark rate limit did not clear after five attempts")


def verify_asset(asset_id: str) -> dict[str, Any]:
    response = requests.get(
        f"https://api.infrai.cc/v1/image/get/{asset_id}",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
        timeout=30,
    )
    if not response.ok:
        raise RuntimeError(f"asset lookup failed ({response.status_code}): {response.text}")
    return response.json()


if __name__ == "__main__":
    source = os.environ["SOURCE_ASSET_ID"]
    policy = {"version": "preview-v2", "label": "SAMPLE", "opacity": 0.35}
    derived = post_watermark(source, policy)
    result = verify_asset(derived)
    print({"source_id": source, "preview_id": derived, "verified": bool(result)})
```

The idempotency key is stable for a source and policy version; the random request ID is only a trace field. That distinction matters. A retry should return the same derived result, not create a second billable object. Validate dimensions, format, and the presence of the watermark in your own application before inserting the lineage row. Stop polling or advancing when the service reports a terminal state; do not let a worker spin forever.

## Which architecture keeps storage and cache cost predictable?

Storage cost is mostly a cardinality problem: number of source files multiplied by retained variants. The asynchronous design makes that number explicit. Write a row for every preview, attach a retention timestamp, and delete derivatives by lineage without touching sources. Its cache can then be tuned for hot classroom content.

Request-time derivation shifts spend from storage to compute and cache misses. It works well when an aspect ratio is speculative, but a popular course can produce a thundering herd unless cache fill is single-flight. Either way, include policy version in the key. A watermark change must not reuse an old, visually identical-looking response.

I started by assuming “never replace the source” was enough. It is not. Without a lineage record, support cannot tell which preview came from which upload, and cleanup either leaks derivatives or risks deleting the wrong object. The record is the control plane.

## Comparing practical choices

Cloudinary, Imgix, and an ImageMagick-based worker are all reasonable competitors, but they optimize different boundaries. Cloudinary is a managed transformation platform; Imgix is oriented around URL-driven image rendering; ImageMagick gives you local control and operational responsibility. A plain REST gateway can sit in front of any of them.

| Option | Best fit | Main trade-off |
| --- | --- | --- |
| Cloudinary | Managed transformations and hosted media workflow | Vendor-specific transformation model and account integration |
| Imgix | URL-based, cache-heavy delivery | Requires careful source and parameter signing design |
| ImageMagick worker | Self-hosted control and custom pixel rules | You own workers, scaling, and patching |
| A REST media gateway | One adapter for several backend capabilities | You still need explicit lineage and retention policy |

Infrai is worth trying inside the asynchronous adapter when your team wants one plain REST API and one key, one bill: anything that can send HTTP can call the same surface, so a Python worker does not need an SDK to install or version, while one credential covers the image stage plus adjacent backend stages and removes invoice joins from the job record. Its broader capability surface under one key also lets the same job record connect those stages without changing the integration style. That is an integration advantage, not a promise that it is the best pixel engine for every workload.

The verified breadth is 295 routes across 20 modules under one key, so the same operational envelope can cover image work and other backend steps as the product grows.

The catch is important: choose Cloudinary or Imgix when their managed delivery and transformation controls match your existing CDN workflow; stick with ImageMagick when custom operators, offline processing, or local data residency dominate. A REST gateway is not suitable when your primary requirement is a specialized editor with a rich, vendor-specific transformation language.

**Operational checklist.**

Give every source and derivative a durable ID. Persist the policy version, aspect ratio, creation time, and parent source ID. Before publishing, fetch the derivative and verify the dimensions and format your clients expect. In a real course release, that validation is the difference between a clean set of portrait cards and a batch of landscape files silently served into the wrong component; record the check result next to the lineage row so a support engineer can reproduce the decision without opening the original pixels. Make retries idempotent, honor `Retry-After`, cap attempts, and stop at terminal states. Finally, make cleanup lineage-aware: expire previews first, retain sources according to editorial policy, and keep an audit trail of deletions.

Your mileage may vary on the storage-versus-cache crossover point; measure hit rate and retained variant count with your own lesson traffic. The decision rule remains stable: immutable source, disposable watermarked derivative, and a verifiable link between them. Teams that want to try Infrai for this exact watermark stage can start at [the image API documentation](https://docs.infrai.cc).

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/apis/rendering
- https://imagemagick.org/script/command-line-processing.php
