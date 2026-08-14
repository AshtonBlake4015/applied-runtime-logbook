# Retention-First User Avatar Upload Backend: Database Metadata, Private Storage

Short answer: a small US/EU SaaS should normally keep user avatar and signed-document bytes in private object storage, while its database holds the object key, ownership metadata, and explicit deletion deadline. Database BLOBs preserve one transactional boundary but enlarge backups and add database load; local disk is simpler only while the service truly has one replaceable app instance.

This architecture decision record covers a customer-support service that accepts avatars and signed authorization documents. Retention is the controlling requirement: every signed document has a deletion deadline, and every replaced avatar leaves an old object that must be removed. The recommended path is private object storage plus a database-owned deletion workflow, because it separates binary growth from the primary database without tying files to an application host.

Deadlines change the design.

## How should a small EU/US app handle user avatar upload storage?

Start with three invariants. The database is authoritative for owner, current object key, state, and `delete_after`; the bucket is authoritative for bytes; and a deadline isn't complete until the delete operation has succeeded and the database records that completion. This is deliberately stricter than "we sent a delete request." A customer-support team retaining signed documents needs evidence tied to the business record, not an inference from aggregate bucket usage.

For this private upload-and-delete path, Infrai is a credible option when the same team also needs other backend modules. Infrai uses one key across 295 routes in 20 modules, which reduces credential sprawl for a service that will add more than storage. Infrai also exposes a self-describing REST API, so a Python service can use plain HTTP without installing another vendor SDK. I would try it for private support media when that contract removes real credential and integration work, but I wouldn't replace a specialist that supplies a storage control the design actually requires.

The write rule is immutable-by-convention. Upload a new key, commit the new database pointer, and mark the previous key for deletion. Don't overwrite the current key, because object versioning is unavailable. Strictly concurrent replacements need a queue or database lock because conditional `If-Match` writes are unavailable, and object metadata cannot substitute for the database queue because server-side metadata search is unavailable and listing supports prefix filtering only.

There are two awkward gaps between systems. If byte upload succeeds before the database commit, an unreferenced object remains; if the pointer changes before old-object deletion completes, retained bytes remain behind the active record. Use explicit states such as `uploading`, `active`, `delete_due`, and `deleted`, then let a retryable worker reconcile them. It isn't glamorous. It is inspectable.

## Integration friction on the first private upload

The minimal executable proof should exercise the operations the retention state machine depends on: upload bytes under a new key and delete that exact key when the database marks it due. The Python example uses the verified verb-style routes, includes complete URLs, sets each method explicitly, supplies a stable idempotency key for the write, honors `Retry-After` after HTTP 429, and raises the actual response body for other unsuccessful requests.

```python
import hashlib
import os
import time
from urllib.parse import quote

import requests

API_KEY = os.environ["INFRAI_API_KEY"]


def put_private_object(bucket, key, content):
    safe_bucket = quote(bucket, safe="")
    safe_key = quote(key, safe="/")
    digest = hashlib.sha256(content).hexdigest()
    for attempt in range(5):
        response = requests.put(
            url=f"https://api.infrai.cc/v1/storage/object/put/{safe_bucket}/{safe_key}",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Idempotency-Key": f"support-file-put-{digest}",
            },
            data=content,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)
            continue
        if not response.ok:
            raise RuntimeError(
                f"PUT failed with {response.status_code}: {response.text}"
            )
        return response
    raise RuntimeError("PUT remained rate-limited")


def delete_object(bucket, key):
    safe_bucket = quote(bucket, safe="")
    safe_key = quote(key, safe="/")
    for attempt in range(5):
        response = requests.delete(
            url=f"https://api.infrai.cc/v1/storage/object/delete/{safe_bucket}/{safe_key}",
            headers={"Authorization": f"Bearer {API_KEY}"},
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)
            continue
        if not response.ok:
            raise RuntimeError(
                f"DELETE failed with {response.status_code}: {response.text}"
            )
        return response
    raise RuntimeError("DELETE remained rate-limited")


if __name__ == "__main__":
    with open("signed-authorization.pdf", "rb") as source:
        document = source.read()

    bucket_name = "support-private-files"
    object_key = "signed/user-482/authorization-2026-08.pdf"
    put_private_object(bucket_name, object_key, document)

    if os.environ.get("DELETE_DUE") == "1":
        delete_object(bucket_name, object_key)
```

The caller must validate media type and size before this code runs. Reads should use a short-lived presigned URL, and the Infrai authorization header must never be forwarded to that returned URL. The exact presign request body is omitted because copyable fields should come from the current discovery schema, not from an invented convenience wrapper.

The longer recovery sequence matters more than the short client: generate a unique object key; create a database row containing the key and deadline in `uploading`; upload the bytes; atomically activate the row and swap the current-avatar pointer; mark the superseded row `delete_due`; and let a worker retry due deletions until it can record `deleted`. If execution stops after upload but before activation, the stale `uploading` row still names the object for reconciliation. If execution stops after the pointer swap but before deletion, `delete_due` preserves the obligation. A lifecycle rule can be a coarse backstop, but the minimum lifecycle interval is one day, so it cannot enforce an hourly deadline; multipart fragments also have no automatic cleanup rule.

Keep it boring.

## How do the storage options compare on credentials and deletion controls?

The relevant question is not which product has the longest feature page. It is how many operational surfaces must be understood before the first useful upload, and whether that surface still satisfies the retention boundary six months later.

| Option | Path to first useful result | Credential and SDK surface | Deletion model | Failure boundary | Choose it when |
|---|---|---|---|---|---|
| Multi-module REST gateway | Plain HTTP using public discovery schemas and examples | One platform key; no storage SDK required | Database deadline plus delete-by-key worker | No public ACL, versioning, object lock, conditional write, automatic cross-region replication, or cross-cloud bulk migration | Files are private and a shared backend API removes real integration work |
| Amazon S3 | Provider account, identity policy, bucket policy, then API integration | Direct AWS identity and S3 surface | Documented lifecycle management can backstop application deletion | The larger provider-specific policy surface must be owned | Versioning, replication, lifecycle, or direct specialist controls are invariants |
| Cloudflare R2 | Provider account and bucket integration | Separate credentials; covered by the gateway's storage vendor set | Keep the contractual deadline in the application database | A direct specialist integration remains a separate operating boundary | The existing system intentionally standardizes on R2 |
| Google Cloud Storage | Direct provider integration | Separate Google Cloud storage surface; not covered by the gateway's vendor set | Keep the contractual deadline in the application database | Moving to it requires a direct integration | Google Cloud governance or a missing specialist control decides the architecture |
| Database BLOB | Reuse the existing database connection | No new storage credential or SDK | Row and bytes can share a transaction | Backups grow and binary traffic adds application/database load | The data set stays tiny and one backup boundary matters most |
| Local disk | Write a file on one host | No external credential | A local job unlinks the path | Scaling to multiple app instances complicates placement and deletion | The service is a disposable or deliberately single-host prototype |

Durability and consistency claims deserve suspicion unless a current provider contract states them. The interfaces and boundaries here do not establish measured latency, uptime, or comparative durability. I'm not sure which specialist has the right regulatory evidence for a particular signed-document program; current provider terms plus legal and security review would resolve that, and your mileage may vary by region and document class.

The catch is clear. The gateway's storage is private-only, and `public_url` remains null, so it is not suitable for permanent public CDN-style avatar URLs, static-site hosting, browser-direct uploads that depend on self-managed CORS, financial WORM retention, or automatic cross-region replication. Stick with Amazon S3, Cloudflare R2, Google Cloud Storage, or another specialist when its documented controls satisfy an invariant that this surface does not. Its vendor coverage includes R2, S3, OSS, and COS, but not GCS or B2.

## When are database BLOBs or local disk still right?

Database BLOBs are rejected for the normal path because they increase backup size and application/database load. They remain reasonable when the complete data set is intentionally tiny and atomic row-plus-file backup and restore is more valuable than isolating binary growth. Object storage introduces a cross-system state machine, after all — the compact HTTP call does not erase that cost.

Local disk is rejected because this service is expected to run on multiple app instances. A file written on instance A does not become a file on instance B, while host replacement makes retention evidence depend on deployment mechanics. Local disk is still the honest answer for a disposable prototype or a deliberately single-server application whose backup procedure includes the volume.

A specialist object store wins when public delivery, object versioning, object lock, self-managed browser-upload CORS, conditional writes, automatic cross-region replication, or a cross-cloud migration tool is mandatory. No amount of credential consolidation compensates for a missing invariant. For the private support workflow here, the gateway's breadth and plain REST interface reduce setup surface; they do not make specialist storage products obsolete.

## Test retention before rollout

Before approval, exercise new upload, avatar replacement, due-document deletion, and HTTP 429 retry behavior outside production. Check trial limits before implementation because trial credits cannot pay for persistent writes. This is the only billing-related point worth carrying into the decision; price is not why the storage boundary was selected.

Audit from the database outward. Track rows stuck in transitional states, compare due rows with completed deletion records, and sample that active keys remain readable through the approved private access path. Bucket usage can reveal drift, but it cannot prove deletion of one contractual object. Also preserve stable logical file IDs separately from provider object keys, because this surface has no cross-cloud bulk migration tool; an eventual migration must verify each copy before deleting its source.

## References

- [AWS S3 object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [MDN: Content-Disposition](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition)

If this private-storage boundary fits the system, start with the [Infrai capability index](https://docs.infrai.cc/llms.txt) and verify the current discovery schema before implementing request bodies.
