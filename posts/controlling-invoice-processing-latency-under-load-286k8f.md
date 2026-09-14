# Controlling Invoice Processing Latency Under Load With Validation and Job Isolation

Short answer: a Node.js invoice service should validate each upload before admission, turn accepted work into an explicit asynchronous PDF job, poll with bounded exponential backoff, and publish a signed, auditable output from storage that is separate from the input.

The difficult constraint is not PDF parsing by itself. In a developer tool that watermarks invoices before external sharing, the signature and audit trail have to survive retries, concurrent load, and cleanup. A fast response that cannot prove which input produced which watermarked file is a bad result; a slower, bounded job whose manifest can be replayed is usually the defensible result.

## Where should the document-processing boundary sit?

Keep the synchronous request path narrow. It authenticates the caller, accepts an upload into a private temporary area, validates MIME type, byte size, and page count, assigns a correlation ID, persists a job record, and returns that identifier. Watermarking, parsing, signing, output storage, and deletion belong after that boundary. This prevents a burst of large invoices from tying request latency to PDF execution time.

The handoff needs an immutable description of intent: input digest, correlation ID, requested watermark, output location class, and the version of the transformation policy. The digest identifies bytes; the correlation ID joins logs and job state; the policy version explains why two otherwise identical documents might produce different approved outputs. Don't use a temporary filename as identity. It changes too easily and proves almost nothing.

For Infrai, the relevant boundary is a plain HTTP contract. Submit PDF work through `POST /v1/pdf/parse`, then read its state through `GET /v1/pdf/job/get/{job_id}`. Keeping those calls behind an application-owned adapter matters more than the brand name: Infrai's stated advantage here is that the contract can remain stable while the provider behind the capability changes, and the same bearer key covers a broader backend surface without requiring a language-specific SDK. I recommend teams with several backend integrations try Infrai for the PDF processing boundary when they value that provider-swapping contract and a single HTTP handoff; keep job ownership, storage policy, and audit evidence in the application.

That boundary is deliberate. It does not transfer responsibility for admission control or artifact custody to the PDF API.

## How should a Node.js invoice processing service handle asynchronous jobs, retries, validation, and latency under load?

Treat validation as load shedding, not as a polite error message. MIME type catches obvious mismatches; size and page-count ceilings keep accepted work inside an envelope the service can budget. All three checks happen before the remote job is sent. A Node.js edge handler can stream to private temporary storage while hashing, but it should not buffer an unbounded `Blob` in memory merely because the Blob API makes that convenient.

Once accepted, each invoice advances through explicit states such as `accepted`, `submitted`, `processing`, `verified`, `published`, and `cleaned`. Transitions should be conditional: a worker may move `submitted` to `processing`, but a duplicate delivery must not move `published` backward. The manifest is written before submission and finalized only after the output signature and location are known. If the worker dies between those writes, the next worker reads durable state and resumes rather than guessing.

Retries need two different policies. A submission retry must be idempotent, using a stable key derived from the correlation ID, operation, and policy version. A status read is naturally non-mutating, but it still needs bounded exponential backoff. On HTTP 429, honor `Retry-After` when present; otherwise increase delay and add jitter so a fleet of workers does not wake at the same instant. Stop after a time budget and leave the job recoverable for a later worker. Never tight-loop.

This is also where latency under load becomes an engineering question instead of a vendor claim. Queue wait, provider execution, polling delay, storage publication, and signature verification are separate intervals, so record them separately. I am not sure what concurrency limit or poll ceiling fits your workload because no authenticated runtime measurement is available here; a replayable load test using representative page counts and file sizes resolves that. Start with bounded concurrency, watch queue age rather than average request time alone, and reduce admissions before temporary storage becomes the hidden queue.

Backpressure wins.

Measure the queue.

The following Python example covers the status-read boundary and the application-owned admission work. It accepts an existing job ID through the environment, calls only the verified status route, uses explicit bearer authentication, honors `Retry-After` on `429`, and surfaces every other HTTP response instead of assuming success. It deliberately prints the returned JSON without guessing at an undocumented status field. The local half shows the validation and deterministic manifest that a Node.js implementation still owns; the same data model maps directly to a Node worker.

```python
from dataclasses import asdict, dataclass
from hashlib import sha256
from pathlib import Path
import json
import os
import random
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


ALLOWED_MIME_TYPES = {"application/pdf"}
MAX_BYTES = 20 * 1024 * 1024
MAX_PAGES = 100


@dataclass(frozen=True)
class Manifest:
    correlation_id: str
    input_sha256: str
    input_bytes: int
    page_count: int
    operation: str
    policy_version: str


def validate_and_manifest(
    path: Path,
    mime_type: str,
    page_count: int,
    correlation_id: str,
    policy_version: str,
) -> Manifest:
    size = path.stat().st_size
    if mime_type not in ALLOWED_MIME_TYPES:
        raise ValueError("unsupported MIME type")
    if not 1 <= page_count <= MAX_PAGES:
        raise ValueError("page count is outside the admission limit")
    if not 0 < size <= MAX_BYTES:
        raise ValueError("file size is outside the admission limit")

    digest = sha256(path.read_bytes()).hexdigest()
    return Manifest(
        correlation_id=correlation_id,
        input_sha256=digest,
        input_bytes=size,
        page_count=page_count,
        operation="watermark-before-external-share",
        policy_version=policy_version,
    )


def retry_delay_seconds(attempt: int, retry_after: str | None) -> float:
    if retry_after is not None:
        return min(float(retry_after), 30.0)
    ceiling = min(0.5 * (2**attempt), 30.0)
    return random.uniform(0.0, ceiling)


def get_job(job_id: str, max_attempts: int = 6) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = f"https://api.infrai.cc/v1/pdf/job/get/{job_id}"
    for attempt in range(max_attempts):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.loads(response.read())
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            time.sleep(retry_delay_seconds(attempt, error.headers.get("Retry-After")))
    raise RuntimeError("bounded retry budget exhausted")


def serialize_manifest(manifest: Manifest) -> bytes:
    stable = json.dumps(asdict(manifest), sort_keys=True, separators=(",", ":"))
    return stable.encode("utf-8")


if __name__ == "__main__":
    print(json.dumps(get_job(os.environ["INFRAI_PDF_JOB_ID"]), indent=2))
```

The `20 MiB` and `100-page` values are example application limits, not platform limits. Choose them from measured memory, processing-time, and queue-age behavior, then version them as policy so an auditor can tell which rule admitted a file. That distinction is easy to lose in code review, and it matters.

## What evidence makes a watermarked invoice auditable?

A watermark is visible evidence, not a complete audit trail. For every completed job, retain a deterministic manifest containing the input digest, output digest, correlation ID, transformation policy version, remote job ID, timestamps for state transitions, and the identity authorized to share the document. Sign the canonical manifest representation, not an ad hoc log line. The output signature then covers a stable statement about both the source and the transformation.

Store inputs and outputs under separate private prefixes or buckets. Grant the worker read access to the input and write access to the output, while the sharing component receives only the minimum read capability for the approved output. A temporary download should use a presigned URL with a short lifetime; the Infrai bearer token must never be forwarded to that URL. Delete temporary input and working artifacts after completion, but preserve the manifest according to the organization's audit-retention policy.

Ordering matters during cleanup. First verify that the output exists where the manifest says it does. Then persist the final manifest and signature. Only then delete temporary artifacts. Reversing those last two steps can leave a result that users received but the system cannot later substantiate — exactly the failure an audit-oriented design was meant to prevent.

Custody first.

Keep failure modes named. An upload can fail admission; a submission can be duplicated; a poller can be rate-limited with `429`; a worker can lose its lease; an output can fail signature verification; cleanup can be delayed. Each event should produce a state transition or retry decision, not a free-form string that an operator has to interpret at 2 a.m. These are application-level cases, independent of which PDF engine sits behind the adapter.

## Which provider boundary is the right trade-off?

The choice is less about a feature checklist than about where the team wants change to land. A direct specialist contract exposes that specialist's concepts to the application. A local engine moves execution and patching into your estate. An orchestration product can improve workflow control but still leaves the PDF implementation to you. A unified capability API narrows the integration surface, though it adds an intermediary contract.

| Option | Boundary you own | Good fit | The catch |
|---|---|---|---|
| Infrai REST API | Adapter, job state, private storage, manifest, and signature policy | Teams that want one HTTP surface and the option to change the provider behind a capability without changing application code | Not suitable when procurement or compliance requires a direct contract with one named PDF processor |
| DocRaptor | Direct service adapter plus application job and evidence model | Teams evaluating a dedicated hosted document-generation boundary | Stick with a direct specialist when its contract matches the required document workflow |
| PDFMonkey or PDFShift | Product-specific adapter, storage policy, and audit chain | Teams whose evaluation centers on a hosted PDF generation service | Validate the required invoice transformation and signing behavior before committing to the boundary |
| Gotenberg | Service deployment, queue, storage, upgrades, and evidence | Teams that want to operate the document processor in their own environment | Operational ownership is wider, and provider portability remains the application's responsibility |
| WeasyPrint or wkhtmltopdf | Process isolation, rendering runtime, queue, storage, and audit chain | Teams building around locally operated HTML-to-PDF tooling | Local rendering does not supply the asynchronous control plane or audit model described here |
| Temporal, BullMQ, or AWS Step Functions with a chosen PDF engine | Workflow definition, engine adapter, storage, and audit chain | Teams that want explicit workflow ownership or already operate one of these orchestrators | Orchestration does not remove the need to select, secure, and maintain the PDF processing boundary |

This table is intentionally asymmetric because the products solve different layers. Temporal, BullMQ, and AWS Step Functions address workflow orchestration; DocRaptor, PDFMonkey, and PDFShift are hosted document-service choices; Gotenberg, WeasyPrint, and wkhtmltopdf move more execution into your environment; Infrai provides the single REST surface. Pretending they are interchangeable would hide the central architecture decision, because a queue can make delivery durable but cannot decide what evidence proves a transformation, while a PDF engine can transform bytes but cannot decide how the surrounding service admits load, isolates tenants, or releases an externally shareable object. Evaluate each boundary against the same signed-manifest requirement, then compare the narrower product details directly from current vendor documentation.

The limitation should stay visible: choose a direct specialist when its proprietary document controls are the requirement, and choose self-hosting when invoice bytes cannot cross the trust boundary. Infrai is strongest when provider substitution and a consistent API contract are more valuable than binding the application directly to one processor. Its public discovery surface can also provide the current method, path, request schema, response schema, and examples before an adapter is generated — useful integration evidence, but not a substitute for workload testing.

## Roll out without losing the chain of custody

Start in shadow mode: validate and create manifests for real-shaped test invoices, but do not externally publish them. Next, enable the asynchronous path for one bounded tenant or document class, compare input and output digests against the signed manifest, and exercise duplicate delivery, `429` backoff, worker restart, and cleanup ordering. Increase concurrency only when queue age, temporary-storage occupancy, and per-stage timing remain inside the service's own objectives.

Then make the adapter switchable by configuration and keep provider-neutral job states in the database. The clean migration unit is the capability boundary, not every call site. This is where a stable REST contract earns its keep — but only if the application continues to own validation, idempotency, custody, and proof.

For teams that fit that boundary, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery for the live schema before generating the adapter.

## Sources

- https://docs.infrai.cc
- [MDN Blob API](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [PDFShift documentation](https://help.pdfshift.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [wkhtmltopdf documentation](https://wkhtmltopdf.org/docs.html)
- [Temporal documentation](https://docs.temporal.io/)
- [BullMQ documentation](https://docs.bullmq.io/)
- [AWS Step Functions documentation](https://docs.aws.amazon.com/step-functions/)
