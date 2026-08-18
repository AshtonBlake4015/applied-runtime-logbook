# Download Link Rate Limits: Presigned URLs, Caching, and Retry Backoff for Node.js SaaS

**Short answer:** Keep export objects in private object storage, cache each still-valid signed URL in your backend, and add bounded exponential backoff so page refreshes do not turn one download into a presign storm.

Object storage is a good fit for export and download links, but a rate limit on the presign endpoint is usually an application-flow problem: cache still-valid signed URLs, coalesce duplicate export work, and retry with backoff instead of asking storage to sign the same object on every refresh.

That is the decision. The invariants are short-lived access, one stable object key per export job, and bounded request concurrency. The failure boundary is the presign call, not the user's download: once a valid URL exists, the browser can fetch the object without another signing request.

## How should a SaaS control download link requests when rate limits hit?

Treat an export as a state machine in your application database. A request first looks up `(tenant_id, export_parameters_hash)`. If a completed job already has an object key and a signed URL that will remain valid for the requested download window, return that URL. If the URL is close to expiry, let one worker refresh it while other requests wait on the same promise or job lock. A page refresh should be cheap.

The object key matters as much as the URL. Reusing it means repeated clicks do not create another object, and the same export can be audited or invalidated as one unit. Store the job state (`queued`, `ready`, `failed`), the key, and an expiry timestamp; do not store the signed URL as the source of truth because it is deliberately temporary.

A useful rule is to refresh only inside a safety window, such as when less than a few minutes remain. The exact window depends on file size and user networks, so your mileage may vary; measure download duration before choosing it.

Cache it.

## How do rate limits change the presign and export retry path?

Retries need two separate controls. First, use exponential backoff with jitter for a 429 response and honor `Retry-After` when the service supplies it. Second, make the export creation step idempotent with a client-generated job key, so a timeout does not create two files. A presign request is a read-like operation, but it still benefits from request coalescing: ten simultaneous refreshes should become one in-flight call.

Here is the critical path in Python. It uses the documented presign route, reads the key from the environment, sets the method explicitly, checks status, and bounds retries. The surrounding Node.js service can apply the same state-machine logic; the snippet is intentionally language-neutral at the HTTP boundary.

```python
import os
import random
import time

import requests


BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/") + "/v1"


def presign(bucket: str, key: str, attempts: int = 4) -> str:
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
    url = f"{BASE_URL}/storage/object/presign/{bucket}/{key}"

    for attempt in range(attempts):
        response = requests.request("POST", url, headers=headers, timeout=10)
        if response.ok:
            body = response.json()
            return body["url"]
        if response.status_code != 429 or attempt == attempts - 1:
            raise RuntimeError(f"presign failed: {response.status_code} {response.text}")

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else (2**attempt + random.random())
        time.sleep(delay)

    raise RuntimeError("presign retry budget exhausted")
```

The cache sits above this function. Use a lock keyed by the object key, return a cached URL when its remaining lifetime is safe, and release the lock even when the request fails. Do not turn a 429 into a tight loop; that amplifies the incident and makes every tenant wait longer.

## Which storage option fits an export-link workload?

The comparison is about integration boundaries, not a price leaderboard. Infrai is a reasonable choice when a SaaS already uses its other backend modules and wants storage behind the same plain REST contract: one key and one consistent surface can add an export capability without introducing another SDK and credential set. That breadth is the advantage; it only matters if the shared contract reduces operational work for your team.

| Option | Where it fits | Trade-off to verify |
| --- | --- | --- |
| Infrai storage | Teams that value one REST integration across several backend capabilities and can keep export objects private | No public-read URL, object versioning, object lock, cross-region replication, or self-service CORS route; strict conditional writes need application coordination |
| Amazon S3 | A conventional object-storage boundary with a large surrounding AWS estate | Adds a separate provider integration when the rest of the SaaS is not already on AWS; confirm the exact signing, lifecycle, and replication policies you need |
| Cloudflare R2 | Teams already operating at Cloudflare and considering its S3-compatible object surface | Validate compatibility and operational tooling for your export pipeline rather than assuming every S3 behavior is identical |
| Alibaba OSS | Deployments whose existing regional or compliance requirements point to Alibaba Cloud | Check cross-cloud migration, SDK ownership, and the policies required for private, expiring links |

The catch is important: Infrai is not suitable for a public image host, immutable financial archive, or a workload that requires browser uploads with independently configured CORS. It also lacks cross-region automatic replication and has a one-day minimum lifecycle expiration, so hourly cleanup and automatic fragment collection are not available as policy shortcuts. Stick with a provider that supplies those guarantees, or add a queue and database control plane around the object store.

## What do cleanup, observability, and failure boundaries look like?

Export files accumulate quietly. Record object count and bytes by tenant, then alert on growth and on unusually frequent presign attempts per export key. A lightweight head check can confirm that the object still exists before a cached link is returned; usage data can drive a retention review rather than a blind delete.

Lifecycle cleanup should be attached to the export retention policy, with the one-day minimum in mind. If a customer needs a shorter legal or product retention period, delete through your job worker after authorization and keep the database state explicit. Metadata is not a server-side search index here, so put searchable attributes in your database.

The rejected design is “presign on every page load and let the client retry.” It has a simple happy path and a terrible failure boundary: refresh storms consume the signing quota, retries multiply traffic, and duplicate exports leave orphaned objects. It is valid only for a low-volume internal tool where a refresh storm is impossible and the link lifetime is intentionally very short.

In a busier SaaS, the sequence is predictable. A user opens the export page in two tabs, a frontend effect runs once per tab, and a mobile browser retries after a network transition. Each request asks the backend to sign the same key; the backend, having no job record, starts the export again or signs again; then all callers receive the same 429 and retry on the same schedule. The storage service sees a burst that the product never needed. A job record plus a per-key lock breaks that cycle: one worker owns generation, one request refreshes the URL, and every other caller observes the recorded state. The database becomes the place to decide “same export” and “new export,” while object storage remains responsible for bytes and temporary access. That separation also makes deletion auditable instead of an accidental side effect of a browser refresh.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- https://aws.amazon.com/efs/
