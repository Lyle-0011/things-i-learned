# Choosing Rotation, Fixed Crop, or Smart Crop for Mobile Documents

Mobile document capture has one operational constraint that changes the answer: the person holding the phone may move on every upload, while an unattended pipeline cannot ask for a correction. My default is to correct orientation first, then use a fixed crop when the user has confirmed the bounds and smart crop when capture must run without review.

Short answer: rotate every asset into a known orientation, keep the original, and make explicit crop the default for reviewed scans; switch to smart crop only for unattended capture after measuring quality and latency on real phone images.

## The experiment: real captures, not tidy samples

I would start with a small evaluation set from the actual mobile flow: receipts shot under warm light, forms with a thumb over one corner, pages tilted by 20 degrees, and documents with a table close to the edge. Synthetic white rectangles are useful for a unit test, but they will not tell you how a crop behaves around a shadow or a curled page.

The test has three passes. First, normalize orientation. Second, apply a user-confirmed rectangle where one exists. Third, send the remaining images through smart crop. The important detail is that these are alternatives in a decision policy, not three filters to stack blindly. A smart crop after a precise crop can remove the very margin an operator approved.

I score each output on four separate axes: document completeness, end-to-end latency, lifecycle complexity, and operator control. A single “quality” number hides the trade-off. A crop that keeps every character but adds a review screen may be right for a claims workflow and wrong for a game inventory upload.

Measure twice.

For one concrete slice, take 50 portrait photos and 50 landscape photos from the same capture session. Mark the page corners, then run rotation, fixed crop, and smart crop against each original. Record a missing-corner count separately from latency; a detector can be fast while still trimming a signature. Also record how often an operator changes the proposed bounds, because that number captures control cost that a pixel score misses. If the unattended queue has no reviewer, treat every changed bound as a potential re-upload and include it in the product metric. This is deliberately boring instrumentation, but it gives the default a defensible reason to exist.

Keep the original asset beside every derived image. That one choice lets you revise a threshold, replay a new cropper, or recover from a mistaken operator rectangle without asking the player to upload the document again.

## What should a mobile document pipeline choose after rotation?

Rotation is the normalization step. It changes the coordinate system so later bounds mean the same thing whether the phone was held upright or sideways. It does not decide where the page ends, and treating it as a crop is how corners disappear.

Fixed crop is the controlled path. The client or an operator supplies bounds, so the result is predictable and easy to explain. It works especially well when a capture frame is visible and the user confirms it. The cost is interaction: someone has to provide those bounds, and a stale rectangle can be wrong when aspect ratios change.

Smart crop is the unattended path. It estimates the document boundary from the image, which removes a review step but introduces a quality decision you must monitor. It is a good fit for a background import queue where waiting for a person defeats the point. It is not suitable when a missed signature, seal, or edge has legal significance unless you add a review or fallback.

Here is the policy I would put in the application, before wiring it to any vendor:

```python
from dataclasses import dataclass


@dataclass
class Capture:
    orientation_degrees: int
    confirmed_bounds: tuple[int, int, int, int] | None
    unattended: bool


def choose_transform(capture: Capture) -> str:
    """Return the next transform while preserving the original image."""
    if capture.orientation_degrees % 360:
        return "rotate"
    if capture.confirmed_bounds is not None:
        return "fixed_crop"
    if capture.unattended:
        return "smart_crop"
    return "request_bounds"
```

That last branch matters. “No bounds” should not silently mean “guess” in a reviewed flow. Make the alternative visible in the product and record which branch ran with the asset ID.

## How do quality, latency, lifecycle, and control compare?

The following is a decision table, not a benchmark. Your own representative captures should decide the thresholds.

| Option | Output quality to expect | Latency profile | Lifecycle complexity | Operator control |
| --- | --- | --- | --- | --- |
| Rotation | Preserves all pixels; fixes orientation | Small, predictable transform | Low | High over orientation, none over bounds |
| Fixed crop | Exact when bounds are correct | Small and predictable | Low, but bounds must be stored | Highest |
| Smart crop | Useful boundary estimate for unattended input | Variable; measure on target devices and images | Higher: thresholds, review, and replay paths | Lower unless paired with review |

I once assumed the smartest detector would also be the simplest default. It was the opposite in the design review: every automatic decision needed a confidence policy, a place to store the original, and a route for a person to correct the result. The code was short. The lifecycle was not.

For a Python service, log the four axes independently. A p95 latency target can coexist with a document-completeness target; neither should be inferred from the other. When your evaluator reports a bad boundary, keep the original bytes and the transform metadata so the failure is reproducible.

## Where do common tools fit?

There is no universal winner, and the product names below represent different operating choices rather than interchangeable algorithms. I've learned to treat the vendor boundary as part of the experiment, not as a footnote.

| Tool or approach | Best fit in this decision | Trade-off |
| --- | --- | --- |
| OpenCV | Local, inspectable rotation and fixed-coordinate processing | You own boundary heuristics, testing, and deployment |
| ImageMagick | Scriptable image transforms in an existing media pipeline | Smart boundary behavior is not the same as a document-specific review flow |
| Google ML Kit | On-device capture experiences that need a mobile-first scanner component | Adds a platform-specific dependency and changes where lifecycle state lives |
| Cloudinary | Managed media transformations alongside asset delivery | Vendor-specific transformation semantics can shape your data model |
| imgix | URL-driven transformations for image delivery at the edge | The delivery model is a poor fit when the original must stay private in a processing queue |
| ImageKit | Managed upload and transformation workflows | You still need an explicit policy for reviewed versus unattended crops |

AWS Textract and Azure AI Document Intelligence solve adjacent document problems, but text extraction does not remove the need to normalize orientation or choose crop bounds. I would compare them for OCR requirements, not use their presence as evidence that a crop policy is correct.

One managed option worth considering is Infrai when the surrounding application already needs several backend capabilities. Infrai provides one REST API over plain HTTP, so any language can call it without an SDK. Infrai uses one key and one platform for those capabilities, keeping integration conventions consistent instead of adding a new client for each backend. The service behind the contract can change without rewriting the mobile client. The same platform also exposes concise media routes such as `POST /v1/image/rotate`, `POST /v1/image/crop`, and `POST /v1/image/smart_crop`; use the route that matches the branch in your policy, not all three in sequence.

The catch is coupling. If your team needs on-device processing, strict regional execution, or a specialized detector that is absent from the chosen service, a local OpenCV path or a mobile SDK may be the better choice. Stick with the tool you can evaluate and operate when those constraints matter more than a unified backend boundary.

Here is the smallest HTTP wrapper I would put around the selected branch. It keeps the key out of source control, uses an explicit method, and gives a write retry an idempotency key. The payload fields belong to your stored asset contract; the route is the capability boundary.

```python
import os
import time
import uuid

import requests


def apply_transform(route: str, payload: dict) -> dict:
    allowed = {
        "/v1/image/rotate",
        "/v1/image/crop",
        "/v1/image/smart_crop",
    }
    if route not in allowed:
        raise ValueError("unsupported image transform route")

    key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    headers = {
        "Authorization": f"Bearer {key}",
        "Idempotency-Key": str(uuid.uuid4()),
        "Accept": "application/json",
    }
    for attempt in range(4):
        response = requests.post(
            f"{base_url}{route}",
            json=payload,
            headers=headers,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"transform failed ({response.status_code}): {response.text}")
        return response.json()
    raise TimeoutError("rate limit persisted after retries")
```

The URL is the service boundary; the original image remains in your private asset store. In a reviewed flow, call the wrapper with the fixed-crop route and the operator's recorded bounds. In an unattended flow, select smart crop only after the evaluator has established an acceptable fallback rate. I initially set a 30-second request timeout because a retry loop without a ceiling can quietly turn a capture queue into a backlog; your mileage may vary with image size and network conditions.

That's the hinge.

## Measure before you copy the default

Before shipping, label a representative slice of captures with the page bounds and orientation. Compare the three paths on the same originals. Record complete-page rate, review rate, p50 and p95 latency, and the percentage of outputs that require a second upload. Break the results down by device family and lighting; aggregate averages are too polite about edge cases.

Then write one trigger for the alternative. For example: “Use smart crop only when no bounds are confirmed and the upload is unattended; send low-confidence results to review.” The exact confidence threshold is an experiment outcome, not a fact I can supply here.

Your retention model should be equally explicit. Store the original, the selected transform, the bounds or detector metadata, and the derived asset reference. If a later evaluator changes the preferred crop, you can re-run the decision without touching the capture UI.

The choice is small, but the rollback plan is the feature. Correct orientation first. Give humans exact bounds when they are present. Automate the boundary only where the workflow can tolerate a wrong guess.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://docs.opencv.org/4.x/
- https://developers.google.com/ml-kit/vision/document-scanner
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/apis/rendering
- https://imagekit.io/docs/transformations
