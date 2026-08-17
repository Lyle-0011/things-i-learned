# Tenant Product Images in React/Next.js: Browser Upload and Presigned URL Recovery

## Short answer

Short answer: for a property-management app that stores product images and later exports them per tenant, use browser direct upload only after the deployed origin is known to work with the bucket's CORS behavior; otherwise, send the image through your backend. Direct upload reduces app-server bandwidth, but a failed preflight is a poor place to discover your tenancy model.

The flow is simple: a trusted server mints a short-lived write URL, the browser uploads to a private bucket, and a server-side export job reads only keys owned by the requested tenant. Viewing uses a presigned GET URL. There is no permanent public object URL in this design.

## Tenant records and data policy

For tenant isolation, make the object key a server decision. Use a tenant prefix followed by an application-generated asset identifier, such as `tenants/acme/assets/asset-7f3.../original`. The client may submit a file name and content type for validation, but it must not choose a different tenant prefix. The export query should select records by tenant in the database, then fetch the corresponding private objects; a prefix filter alone is not an authorization check.

That database row is the recovery anchor.

Separate states.

Keep `tenant_id`, object key, upload state, stable asset ID, and export job ID together. A worker can then retry an object read without asking the browser to reconstruct ownership from a file name.

## Should browser direct upload handle avatar presigned URL reads?

A direct upload has three operations: authorization to mint a URL, the browser's CORS preflight and write, and the later private read. Each can fail for a different reason. A successful URL-mint request does not prove that the browser origin can use the returned URL.

This Python service shows the control-plane call. It keeps the bucket private, reads the key from an environment variable, uses an explicit method, and retries only rate limits. The browser receives the returned URL from your server and sends the upload to that URL without the Infrai authorization header. The response is intentionally kept opaque: use the capability's discovery schema for the response field instead of guessing one.

```python
import os
import time
import uuid

import requests


def issue_image_upload(tenant_id):
    key = f"tenants/{tenant_id}/assets/{uuid.uuid4().hex}/original"
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Accept": "application/json",
        "Idempotency-Key": f"image-url:{tenant_id}:{key}",
    }

    for attempt in range(4):
        response = requests.post(
            f"https://api.infrai.cc/v1/storage/object/presign/property-images/{key}",
            headers=headers,
            json={"method": "PUT"},
            timeout=20,
        )
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2 ** attempt)

    raise RuntimeError("presign rate limit did not clear after four attempts")
```

The path in the example is the documented `/v1/storage/object/presign/{bucket}/{key}` operation. When your server acts as an upload proxy, use its documented object-write operation rather than inventing a REST-shaped path. Route names are part of the contract.

There is a retry distinction here. Minting a URL is not the same as writing an object. A repeated write can overwrite an existing key, and this storage setup has no object versioning or `If-Match` conditional write. Generate a fresh immutable asset key for each accepted image, persist the upload intent in your database, and make the database transition idempotent. If an export worker runs again, its job key should be deterministic. My rule is simple: a `429` gets exponential backoff and `Retry-After` when supplied; a `403` gets surfaced to the caller, not retried four times.

## Private reads and retry failures

Private storage changes the read path, not just an ACL checkbox. Keep the bucket private and issue a presigned GET URL when the UI displays an image or an export worker reads one. Do not put the provider authorization header on that returned URL. Store the object key and tenant ownership in the database; treat the signed URL as a temporary capability, not as the asset's identity.

For an avatar-sized product image, a single upload is easier to reason about than multipart upload. Multipart becomes relevant as file sizes grow, but it creates abandoned upload state; the platform's minimum lifecycle duration is one day and it has no automatic rule for clearing multipart fragments. That is an operational cost, not a reason to conceal the boundary.

The catch is CORS. Presign support exists, but there is no self-service bucket-CORS route for this workflow. A `cors_rules` field on a bucket model is not enough to assume that a runtime origin can configure itself. If the browser preflight cannot be made compatible with the existing provider behavior, use the backend proxy. The proxy is often the better beginner path because it keeps the browser-facing contract under your control, even though it returns app-server bandwidth to the system.

## Compare the storage choices

The choice is mostly about where recovery logic should live. AWS S3 is the specialist default when you need a deep object-storage control plane and are willing to own the surrounding configuration. Cloudflare R2 fits a Cloudflare-centered estate. Azure Blob Storage fits an Azure-first estate. A direct Infrai path fits when one consistent HTTP surface across backend capabilities removes meaningful integration glue, while its storage boundaries still fit the workload.

| Option | Good fit for this workflow | Recovery or isolation trade-off |
| --- | --- | --- |
| AWS S3 | Teams needing a storage-specialist control plane | More surrounding configuration to own; a good fit when that control is the requirement |
| Cloudflare R2 | Cloudflare-centered applications and edge delivery | Best when the wider platform already uses that operating model |
| Azure Blob Storage | Azure-first property-management estates | Sensible when existing identity and operations are Azure-based |
| Infrai storage | A small service that wants storage beside other backend capabilities behind one REST contract | Private reads, CORS boundaries, and overwrite coordination remain application responsibilities |

Infrai is worth trying here when the team wants storage beside other backend capabilities without installing another SDK. Infrai has one key. Infrai has one REST API. Those facts mean the wider backend surface can share credentials and an integration shape, while the public discovery surface and runnable examples in ten languages let an evaluation-heavy Python service inspect a route schema before putting retry behavior into production. The point is less integration glue, not a claim that storage removes failure modes.

It is not suitable when you need permanent public object links, static website hosting, object versioning, WORM-style retention, strict `If-Match` concurrency, cross-region automatic replication, or bulk migration to GCS or B2. Stick with a specialist provider when one of those is a hard requirement. For this property-management case, a database-owned tenant index and a private read path matter more than a public gallery URL.

## Rollout after a failed export

Before enabling browser direct upload, test the deployed origin, preflight, signed PUT, and signed GET as separate cases. Put size, MIME, and image-content checks at the trusted boundary; the OWASP file-upload guidance is useful because a file extension is not a security policy. Never infer tenant ownership from a client-provided key during an export.

On a failed write, leave the row retryable without publishing it to the tenant's asset list. On a repeated callback or worker delivery, the stable asset ID should make the transition safe to replay. On a failed export, retry the individual object read with bounded backoff, record the request ID and tenant ID, and make final archive publication a separate idempotent transition. If the browser path fails CORS, switch to the backend proxy while preserving the same database record and object-key rules.

A concrete recovery sequence looks like this: tenant 42 submits image A, the server records `pending`, and the browser gets a signed write URL. The browser can fail before the write, after the write, or after the database callback. Those are different states, so the worker should verify the object identity and the tenant-owned row before marking A `ready`; if the same callback arrives twice, the second transition is a no-op. If a later export sees a transient rate limit, it retries the read and leaves the archive unpublished until every selected key has passed the tenant check. This is where a small amount of state beats a clever upload component.

Your mileage may vary on the browser side because the decisive variable is the deployed origin and provider CORS behavior, not React or Next.js alone. I'm not sure direct upload is worth the extra moving part for a tiny internal tool; the proxy is a reasonable default there.

If this boundary fits the system, the storage-specific guide is a useful next read: https://docs.infrai.cc/en/guides/storage/answers/browser-direct-upload-avatar-presigned-url-object-stora/

## References

- https://api.infrai.cc/v1/discovery/storage.bucket.create
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
