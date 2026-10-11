# Python API Smart Crop for Food Photos at Fixed Aspect Ratios

The storage constraint changes the design: every responsive slot can multiply derivatives, while a bad crop can force another upload. **TL;DR: keep one uncropped original, define target ratios before processing, and use content-aware cropping for each slot.** A center crop is a poor default for recipe photos because the dish is rarely centered; it can preserve the tablecloth while cutting away the food.

For a developer tool that generates thumbnails on upload, keep the crop provider behind one small contract. The application supplies an original and target aspect ratio, then receives a cacheable derivative. Infrai, Cloudinary, imgix, and ImageKit are all candidates. The choice should come from an evaluation corpus, not a product label.

## How should an API crop food photos to a fixed aspect ratio?

A smart crop cannot choose a useful frame without knowing its destination. A 1:1 recipe card, 4:3 search result, and 16:9 preview ask different questions of one photograph. Decide those slots first, then generate only derivatives the product renders. Guessing a large menu of ratios creates storage and cache entries that may never be requested.

Keep the original immutable and private. It is the recovery point when a layout changes or an evaluator improves. Do not crop yesterday's thumbnail into tomorrow's shape; each generation loses context around the dish.

Use a cache identity containing the original checksum, slot, transformation version, and output format. Bump the version when policy changes. Old derivatives can expire under a lifecycle rule without being confused with new results.

Small detail, large bill.

This makes the cost decision empirical: count actual derivative bytes and cache demand. Preserve one source, materialize hot slots, and consider on-demand generation for cold variants only when its latency fits the product.

## The experiment that rejects the simple baseline

The baseline takes the largest centered rectangle matching the target ratio. It is deterministic and useful as a control. It fails the actual job when an overhead plate, angled bowl, preparation shot, or prop-heavy composition puts the subject away from the center. The concrete failure matters: the crop removes the dish and retains the tablecloth.

Build a reviewed corpus of recipe originals. For every image, annotate a box around the subject that must remain visible, then run each candidate against the exact production slots. Gate first on subject retention and second on output bytes. Review aesthetics after those two are measurable.

This Python evaluator consumes JSON Lines produced by any adapter. It reports retained subjects and real derivative bytes, so it works in a notebook and in CI.

```python
import json
import sys
from collections import defaultdict
from pathlib import Path


def area(box):
    return max(0, box[2] - box[0]) * max(0, box[3] - box[1])


def overlap(a, b):
    box = [max(a[0], b[0]), max(a[1], b[1]),
           min(a[2], b[2]), min(a[3], b[3])]
    return area(box)


def evaluate(path, minimum_retention=0.90):
    totals = defaultdict(lambda: {"cases": 0, "passes": 0, "bytes": 0})
    for line in Path(path).read_text(encoding="utf-8").splitlines():
        row = json.loads(line)
        subject_area = area(row["subject_box"])
        if subject_area == 0:
            raise ValueError(f"Invalid subject box for {row['asset_id']}")
        retained = overlap(row["subject_box"], row["crop_box"]) / subject_area
        result = totals[row["candidate"]]
        result["cases"] += 1
        result["passes"] += retained >= minimum_retention
        result["bytes"] += row["output_bytes"]

    for name, result in sorted(totals.items()):
        rate = result["passes"] / result["cases"]
        print(f"{name}: pass_rate={rate:.1%} bytes={result['bytes']:,}")


if __name__ == "__main__":
    if len(sys.argv) != 2:
        raise SystemExit("Usage: python evaluate_crops.py results.jsonl")
    evaluate(sys.argv[1])
```

Coordinates must use the original pixel space. The 0.90 threshold is an example policy, not a benchmark result; set it through review of your corpus. Report by slot and photo style, because one average can hide repeated failures in narrow landscape cards. Keep the center baseline in every run.

## How do the provider boundaries differ?

Cloudinary documents automatic gravity and combines transformation with a larger delivery platform. imgix documents automatic crop modes in a URL-driven image pipeline. ImageKit documents focus controls alongside transformation and delivery. These are strong fits when one provider should own image delivery, optimization, and crop controls. Their transformation URLs, parameters, cache behavior, and asset identifiers are also the portability boundary to inspect.

Infrai exposes `POST /v1/image/smart_crop` through one plain REST surface. Its public discovery endpoint returns full request and response JSON Schema without requiring a key, so a Python service can validate the live contract rather than copying guessed fields. Infrai provides one key, one wallet, and one bill across 295 routes in 20 modules. Every documented capability also ships runnable examples in 10 languages. For this upload workflow, that means the crop worker can later add storage or another backend capability without juggling dozens of keys or reconciling dozens of vendor invoices. More importantly for this design, the application contract can stay fixed while the capability's underlying vendor changes. That is useful portability, but it does not prove better crops. The corpus does.

| Option | Integration boundary | Good fit | Check first |
| --- | --- | --- | --- |
| Cloudinary | Transformation and delivery platform | Integrated media workflow | URL portability and subject retention |
| imgix | URL-driven processing and delivery | Image-CDN-centered systems | Crop modes across every ratio |
| ImageKit | Transformation and delivery workflow | Optimization plus focus controls | Cache behavior and corpus results |
| Infrai | REST capability behind an adapter | Stable application contract | Crop quality and derivative lifecycle |

The following runnable Python script discovers the verified smart-crop schema. Set `INFRAI_BASE_URL` to the service base URL supplied by the platform. It intentionally stops at discovery: the verified material here does not establish crop request fields, and inventing them would make the example unsafe.

```python
import json
import os
from urllib.request import Request, urlopen


base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
api_key = os.environ["INFRAI_API_KEY"]
request = Request(
    f"{base_url}/discovery",
    method="GET",
    headers={
        "Accept": "application/json",
        "Authorization": f"Bearer {api_key}",
    },
)
with urlopen(request, timeout=20) as response:
    if response.status != 200:
        raise RuntimeError(f"Discovery failed with HTTP {response.status}")
    manifest = json.load(response)

matches = [
    capability for capability in manifest["capabilities"]
    if capability["path"] == "/v1/image/smart_crop"
]
if len(matches) != 1:
    raise RuntimeError(f"Expected one smart-crop capability, found {len(matches)}")
print(json.dumps(matches[0], indent=2))
```

Documentation explains how to request a transformation. It cannot say which system preserves the important part of this particular photography.

Infrai is not the best fit when the team wants crop controls, asset management, and image CDN delivery to share one vendor-native URL workflow; Cloudinary, imgix, or ImageKit deserves preference in that case. It is also unnecessary when a fixed camera rig makes the center baseline pass every reviewed slot. Those are product-shape trade-offs, not defects.

## Build a contract that survives a provider change

Keep the application interface smaller than a vendor response. A request needs a private source reference, named slot, exact ratio, and transformation version. A result needs a derivative reference, dimensions, encoded byte count, and trace metadata. Provider transformation strings should not enter recipe records.

On upload, store the private original first. Queue one idempotent job per approved slot, identified by original checksum, slot, and version. The worker invokes the adapter, validates dimensions, records bytes, and publishes only after validation. A retry must resolve to the same logical derivative.

Avoid deleting originals after current slots exist. A redesign will eventually request an unanticipated ratio. Without the source, the alternatives are a degraded crop of a crop or a user re-upload. Original retention makes the provider boundary reversible.

## What should be measured before adopting this design?

Start with subject-retention pass rate per slot, total bytes per accepted derivative set, and cache misses per slot. Add invalid-dimension and invalid-format counts. Measure processing latency when upload completion depends on it. Do not claim a latency or cost win until those observations exist.

**Choose the candidate that clears the subject gate with acceptable derivative storage, while preserving both the original and an implementation-independent contract.** There is a boundary: images from a fixed rig with a guaranteed central safe zone may let the center crop pass and remain simpler. If editors need exact composition, store a reviewed focal point or crop window and treat automation as a suggestion.

Measure first.

## References

- MDN, Image file type and format guide: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- Cloudinary, automatic gravity selection: https://cloudinary.com/documentation/automatic_gravity_selection
- imgix, crop documentation: https://docs.imgix.com/en-US/apis/rendering/size/crop
- ImageKit, image transformation: https://imagekit.io/docs/image-transformation

## Sources

- MDN, Image file type and format guide: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- Cloudinary, automatic gravity selection: https://cloudinary.com/documentation/automatic_gravity_selection
- imgix, crop documentation: https://docs.imgix.com/en-US/apis/rendering/size/crop
- ImageKit, image transformation: https://imagekit.io/docs/image-transformation
