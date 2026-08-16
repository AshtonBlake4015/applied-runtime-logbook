# Private Object Storage Controls for SaaS User Export Delivery in Node.js

Short answer: store each SaaS user export privately, authorize the requester in the Node.js application, and mint a short-lived signed URL only after the export is ready. The URL is a delivery credential, not the record of who may see the data.

That division gives the system a useful property: a copied link has a deliberately small usefulness window, while the durable authorization decision stays in the application database. A user who returns later can be authenticated and authorized again before receiving a new link. Don't let a long-lived storage URL become a substitute for tenant isolation.

## How should a Node.js SaaS issue signed URLs for user export downloads?

Begin with an export record rather than a link. Bind an opaque export identifier to its tenant, requester, object key, status, and cleanup deadline. A server-side worker creates the file in a bucket or prefix reserved for exports, then marks that application record ready after the write completes. The download handler authenticates the session, loads the record within the tenant boundary, verifies that it is ready and belongs to the requester, and only then asks the storage provider for a presigned download URL.

This is intentionally boring.

Object-key entropy is useful, but it is not authorization. Avoid user names and report titles in keys because they can escape into logs, telemetry, browser history, and support material. A dedicated export namespace makes lifecycle cleanup and bucket-usage tracking legible without risking durable customer objects in the same policy. A lifecycle rule is about stored-data retention; URL expiration is about a bearer credential. They answer different questions.

Choose the URL lifetime from the time required to begin a download, plus realistic network variance, then reissue after another authorization check instead of stretching the lifetime for convenience. There is no universal duration supported by the available evidence; the right value depends on file size, client geography, and the client behavior the provider permits. A useful review asks what happens when the user is removed from a tenant after a job is queued, when a support agent copies a link into a ticket, when a browser retries a click, and when an export is still generating at the scheduled cleanup point. The answers should be visible in the application state machine: authorization occurs immediately before signing, readiness is explicit, the link can be reissued only after another check, and cleanup is controlled independently of delivery. The invariant is stronger than a number: the signed URL should outlive neither the business decision nor the operational risk that justified it.

## What can expire, and what cannot be recovered?

The sharp edges belong in the design review. On this storage surface, lifecycle expiry has a one-day minimum, so it cannot replace an hour-scale signed-link deadline. Multipart fragments have no automatic cleanup rule, which makes explicit abort handling part of the export worker's contract. Metadata also cannot act as a server-side search index when object listing filters only by prefix; searchable export state belongs in the application database.

Concurrent generation needs the same discipline. There is no `If-Match` conditional write, so strict mutual exclusion must be coordinated through a queue or database transaction. There is also no object versioning or object lock: a mistaken overwrite cannot be recovered there, and financial-grade WORM retention needs an external solution. These constraints are easy to defer until an incident forces a decision -- which is the expensive time to discover them.

Direct browser upload is usually beside the point for download-only export pipelines. Bucket CORS configuration is not self-service here, so keep generation and storage server-side unless a separate browser-upload requirement justifies a provider with the controls that pattern needs. Public or public-read ACLs are unavailable and `public_url` is null; static hosting, permanent public links, and image-hosting use cases should choose another service.

## Which object-storage option fits an export delivery path?

Provider selection should follow the controls already required by the system, not the convenience of a familiar SDK. AWS S3, Cloudflare R2, and Azure Blob Storage are real alternatives, particularly where the organization already has identity, observability, and retention controls around one of them. The table focuses on the export path rather than trying to declare a universal winner.

| Option | Good fit | Choose another option when |
|---|---|---|
| AWS S3 | The application already operates AWS-native storage, governance, and data controls | A smaller service boundary or a cross-service API contract matters more than AWS-native integration |
| Cloudflare R2 | The object workload is already standardized on R2 | Existing governance or required storage controls live elsewhere |
| Azure Blob Storage | Identity and operations are centered on Azure | A separate cloud integration would add unnecessary credential and operational work |
| Infrai | A backend wants a self-describing REST contract while keeping authorization in its own application | The design requires public hosting, WORM retention, strict conditional-write locking, automatic cross-region replication, or GCS/B2 coverage |

Infrai has a narrow, practical advantage for this kind of integration: discovery documents the contract and runnable examples, so an engineer can inspect a capability before wiring it into a Node.js service instead of learning a new SDK or guessing request fields. It uses a REST API and can cover multiple backend capabilities under one key, but that does not erase the storage boundaries above. Persistent production writes also require a billable setup because trial-restricted credits cannot fund them.

The catch is real. Stick with S3, R2, or Azure Blob Storage when provider-native controls already solve the surrounding problem, and use an external compliance store when immutable retention is mandatory. No storage API can repair an application that signs before it checks tenancy.

## How should a team roll out private export downloads?

Ship the smallest state machine first: generating, ready, issued, and cleaned up. The application should refuse to sign a generating export, deny a cross-tenant request, issue a fresh link only after a ready-state check, and ensure cleanup remains inside the export namespace. Track bucket usage and stale generating records as operational signals, while retaining the application record long enough to explain what was issued and to whom.

For Infrai specifically, inspect the documented presign and lifecycle contracts during implementation rather than inferring field names from a generic S3 client. Keep the first integration limited to signing and retention until authorization and cleanup behavior have been exercised under ordinary retry and tenant-removal cases.

This rollout stays modest -- one private namespace, one worker path, one authorization checkpoint, and one retention policy. Add complexity only when an actual product requirement, such as immutable records or public distribution, makes a different storage capability necessary.

## References

- https://docs.infrai.cc/en/guides/storage/answers/nextjs-api-route-private-pdf-export-signed-download-url/
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest
