# Tenant Isolation Testing for Private SaaS User Documents and Signed Download Links

Short answer: for private SaaS documents, select object storage only after the application can prove tenant isolation, issue narrow signed download links, restore verified bytes, and model the actual read path; “cheapest” is a result of that test, not a service property.

The difficult constraint is that a document has two lives. Its bytes live in object storage, while its permission, legal retention, owner, and visibility live in the application. A design that lets a client select an arbitrary key, or treats an expiring URL as proof of identity, can be inexpensive and still be wrong. Start with the access contract, derive the storage behavior it demands, then compare current offerings against the same trace. The price page comes last.

This is a security boundary, not a naming convention.

## What should private SaaS document storage and signed download links guarantee?

Keep objects private by default. The database should map an opaque, immutable object key to a tenant, document version, checksum, state, and retention rule; it should not put email addresses, original filenames, or customer names in keys that may later surface in logs, support screenshots, inventories, or metrics. An authenticated request first resolves that metadata and verifies tenant ownership. Only then may a signing adapter receive the object key.

A signed download link is a bearer credential. The recipient need not present the application's normal credentials while the grant remains valid, so authorization must happen before issuing it, its lifetime should match the realistic transfer, and full query strings must stay out of telemetry. AWS describes presigned URLs as time-limited access that can be used without providing cloud credentials. That property is useful; it does not turn the URL into an identity system.

The catch is revocation. A direct link can remain valid until expiry after a user's role changes, so this pattern is not suitable when access must disappear immediately. Use short grants and an application-controlled delivery path, or another revocable mechanism, for that document class. Don't hide this requirement behind a long default expiration.

Browser access is a separate boundary. CORS controls which cross-origin responses browser code may read; it is not authorization. Test the exact production origin, method, response headers, redirect behavior, and `Range` requests used by a viewer. Some cross-origin requests require a preflight, as MDN explains. A permissive development rule is a poor substitute for an access policy.

## Derive the write and read state machine before comparing providers

The application needs explicit states because a database transaction and an object upload do not commit together. A useful minimum is `reserved`, `uploading`, `available`, `deleting`, and `deleted`. The metadata row is reserved first, an upload is restricted to its generated key, bytes are checked against the expected digest, and only then does the row become available to the download endpoint. A retry must be safe at every transition.

Those details sound procedural until a user retries an upload just as a worker times out, or a delete races an active download. Then the missing state machine becomes the incident. The failure modes worth naming in the design record are an object row whose upload did not complete, completed bytes whose metadata was never published, duplicate version creation after a retry, a grant that outlives an entitlement, stale cached content after replacement, and a delete that conflicts with retention. Object storage cannot decide these application states for you.

Use immutable object keys for versions. Replacement means publishing a new version and retiring the old one under a defined policy, rather than overwriting a key while clients, caches, and replication paths may still observe it. Conditional writes, checksums, pagination, multipart cleanup, and deletion semantics all belong in acceptance tests. Compatibility language alone is not evidence.

Here is the boundary I would keep stable while the storage implementation changes:

```python
from dataclasses import dataclass
from datetime import timedelta
from typing import Protocol


@dataclass(frozen=True)
class Document:
    tenant_id: str
    object_key: str
    download_name: str
    state: str


class DocumentRepository(Protocol):
    def get(self, document_id: str) -> Document | None: ...


class ObjectSigner(Protocol):
    def sign_download(
        self,
        *,
        object_key: str,
        expires_in: timedelta,
        download_name: str,
    ) -> str: ...


def issue_download_url(
    *,
    tenant_id: str,
    document_id: str,
    repository: DocumentRepository,
    signer: ObjectSigner,
) -> str:
    document = repository.get(document_id)
    if document is None or document.tenant_id != tenant_id:
        raise LookupError("document not found")
    if document.state != "available":
        raise LookupError("document not found")

    return signer.sign_download(
        object_key=document.object_key,
        expires_in=timedelta(minutes=5),
        download_name=document.download_name,
    )
```

The identical application error for a missing document and a cross-tenant lookup avoids disclosing ownership. The concrete signer must safely encode the filename rather than concatenate untrusted text into a response header, and it should bind the intended operation and explicit expiry. Bucket details remain outside the domain model. Small boundary, large consequence.

## How can a team compare object storage cost, private files, and download behavior?

Use one sanitized workload trace for every candidate and collect the same evidence. Run it from the regions where the application and users operate, with fresh connections as well as warm ones, realistic object-size buckets, bursty concurrency, and a mix of direct downloads, failed uploads, replacements, and deletes. An average transfer time is a weak signal. Tail setup latency and time to first byte are usually closer to what the user notices.

| Decision axis | Evidence to collect | Reject when |
|---|---|---|
| Tenant isolation | Private defaults, generated keys, authorization tests | A caller-selected key crosses a tenant boundary |
| Delivery | Expiry, altered-path tests, `Range`, headers, CORS | A grant or browser path differs from the contract |
| Integrity | Digest checks, conditional writes, retry behavior | A retry can publish uncertain document state |
| Recovery | Inventory, sampled restore, byte and metadata comparison | The team cannot independently verify a restore |
| Operations | Request correlation, access review, lifecycle visibility | A document operation cannot be traced safely |
| Cost | Byte-months, operations, transfer paths, retention, retries | The model excludes a material traffic path |

For cost, calculate stored byte-months, write and read operation counts, transfer direction, retrieval behavior where applicable, cleanup of incomplete multipart uploads, and retention. Run a base case plus a download spike and a growth case. The result should be a versioned model beside the test trace, not a remembered headline. I'm not sure any public pricing comparison stays useful for long, because rates, free allowances, and network paths change; the way to resolve that uncertainty is a dated model built from the contract terms and the product's own measured request mix.

There is no universal winner. Keep an existing storage deployment when its controls, recovery practice, and team knowledge outweigh the migration exposure. Change the design when measured latency, data-location requirements, or transfer patterns conflict with the document contract. For strict revocation or audit needs, a proxy can be the correct architecture even when direct signed links look simpler.

## How should a team migrate document storage without breaking downloads?

Move a new, low-risk document class first. Store one authoritative placement per version so reads do not silently drift between locations, and keep the application authorization path unchanged while the signer is selected from placement metadata. Do not combine a storage migration with key rotation and an authorization redesign in one release; each changes a different failure surface.

Before widening the cohort, gate expansion on checksum matches, grant-denial behavior, tail latency, incomplete-upload age, deletion backlog, and sampled restores. Put the gates in the migration control plane rather than a spreadsheet: each copied version needs a source digest, a destination digest, the placement selected for reads, the time it entered the cohort, and a state that cannot advance on a partial comparison. A reader that asks for a document while that comparison is incomplete should still use its known authoritative placement. This matters because an apparently harmless retry can otherwise switch one reader to a destination before its metadata has been checked, while another reader continues using the source; the resulting mismatch is hard to diagnose after the fact because both locations contain plausible bytes. Correlation IDs should connect the application decision to the object operation without recording the signed URL. The rollback decision needs an owner and a written threshold before the first migration job starts.

For existing data, copy immutable versions, compare bytes and required metadata, switch a cohort's reads, observe through its retention window, and schedule source deletion only under the retention policy. A final decision record can stay short: required semantics, workload trace, test results, cost assumptions, rejected risks, and the exit mechanism. That record will outlast any storage comparison.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
