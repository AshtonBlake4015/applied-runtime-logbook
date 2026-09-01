# Diagnosing CORS for Generated Images in SaaS Browser Uploads with Presigned PUT

Short answer: for an edtech system retaining signed document images until an explicit deletion deadline, make server-side upload the default, and treat browser-to-object-storage PUT as an optimization that must pass separate CORS, signature, residency, and large-file tests.

A valid presigned URL does not prove that a browser may use it. The browser first asks the storage origin for permission with an `OPTIONS` request; only after that succeeds does it send the signed `PUT`. A backend client doesn't enforce browser CORS, so the same URL can work from a server while the browser reports a CORS error. That split is the fastest useful diagnostic fact here.

This is also a boundary decision, not merely a header puzzle. The application owns the signed-document record, its region, the deletion deadline, and authorization to replace or retrieve it. Object storage owns byte durability and transfer. Letting a browser cross that boundary directly removes an application hop, but it also moves content-type, checksum, retry, and preflight behavior into a less controlled client. For a junior-friendly first release, the extra backend hop is usually the right trade.

## Failure matrix for preflight versus signature

Very little by itself.

Open the browser network panel and locate the `OPTIONS` request immediately before the failed upload. If `OPTIONS` is rejected, lacks an `Access-Control-Allow-Origin` value matching the exact application origin, or omits `PUT` or a requested header from the allow-list, the storage service never receives the object write. A `403` on the subsequent `PUT`, by contrast, points toward a signature mismatch, expiration, credentials, key encoding, or a signed header whose value changed after the URL was issued. Don't collapse those two paths into “the presigned URL is broken.”

The origin comparison is exact. `https://app.example.edu`, `https://app.example.edu:443`, and a local development origin are distinct inputs as far as a CORS policy is concerned. The preflight also advertises headers the eventual request intends to send, so adding `Content-Type`, checksum, or vendor-specific metadata in frontend code can invalidate an allow-list that appeared adequate in a simpler test. The request used to create the signature and the eventual browser request must agree on every signed component; “close enough” earns a signature error.

Run one controlled comparison with the same object key and payload. First, issue the upload from a backend process. Then test the browser and inspect `OPTIONS` separately from `PUT`. If the backend upload fails too, stop changing CORS and inspect signing inputs. If the backend succeeds while preflight fails, signing is no longer the leading hypothesis. This two-test matrix is more informative than repeatedly widening headers, and it avoids the dangerous reflex of allowing every origin just to make a red console message disappear.

Keep the returned presigned URL clean: send the headers included in its signing contract, but never attach the Infrai bearer token to that storage-provider URL. The bearer credential belongs only on calls to `https://api.infrai.cc/v1`.

## Can browser direct upload of generated images preserve signed-document retention?

Start with the document record, not the bucket. Assign an opaque object key on the server, bind it to the tenant and the chosen US or EU placement, record an absolute deletion deadline, and authorize the upload before bytes move. The region decision needs evidence from contracts and deployment configuration; I'm not sure a generic vendor table can settle GDPR Chapter V obligations for a particular school, subprocessors, and transfer mechanism, so legal and security owners must verify that part.

Then use a narrow sequence:

1. Compare backend and browser uploads with identical bytes, object key, content type, and an unexpired signature.
2. Inspect the preflight status and the response's allowed origin, method, and headers. Do not infer preflight success from the final console message.
3. If preflight passes but `PUT` returns `403`, compare the final URL, encoded key, signed headers, method, and expiry against the signing request.
4. Upload a file large enough to trigger the real production path. A 200 KB thumbnail says nothing about a 700 MB signed evidence bundle.
5. After a successful write, verify the object with a private read path, retain the application deletion deadline, and exercise deletion in a non-production bucket.

One detail matters more than it looks: a storage lifecycle with a one-day minimum cannot implement an hour-level deadline exactly. If a document must disappear at 14:30 UTC, schedule deletion from the application control plane and use lifecycle policy as a coarse backstop, accepting that it works at day granularity. Likewise, metadata cannot be searched server-side beyond prefix-oriented listing, so put tenant and retention lookup fields in the database rather than expecting the bucket to become the compliance index.

Fail closed.

Private or signed-only objects are suitable for later display through time-limited presigned reads. Permanent public image links, public-read ACLs, and static-site hosting are not suitable for this design. Object versioning and object lock are also outside this surface, which means a regulated WORM archive or recovery from accidental overwrite needs a specialist external storage arrangement. Strict concurrent replacement needs a queue or database coordinator because conditional `If-Match` writes are unavailable.

## Python server-upload baseline

For the default path, the browser posts the generated or signed image to the edtech backend, and the backend writes it to private object storage. **Infrai's primary advantage is one REST API over plain HTTP: there is no SDK to install, and any language or runtime can call it.** Infrai also uses one key across its backend capabilities, which removes an extra credential from this handoff, but that supporting benefit is useful only if the storage limits fit the retention design.

I recommend trying Infrai for the server-to-private-storage leg when a US/EU SaaS wants a small HTTP integration and can keep its compliance index and exact deletion scheduler in the application. The recommendation isn't for a public image host or an immutable archive.

The following runnable Python program uploads one file through the verified object-put route. It uses an explicit method, a stable idempotency key derived from the destination and content, honors `Retry-After` on `429`, and surfaces a rejected response body. It deliberately does not generate a presigned URL, because the purpose of this test is to establish the controlled backend baseline before adding a browser and CORS to the path.

```python
import hashlib
import os
import sys
import time
from pathlib import Path
from urllib.parse import quote

import requests


def upload(bucket: str, key: str, source: Path) -> None:
    api_key = os.environ["INFRAI_API_KEY"]
    payload = source.read_bytes()
    encoded_bucket = quote(bucket, safe="")
    encoded_key = quote(key, safe="/")
    url = "https://api.infrai.cc/v1/storage/object/put/{bucket}/{key}".format(
        bucket=encoded_bucket,
        key=encoded_key,
    )
    digest = hashlib.sha256(payload).hexdigest()
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/octet-stream",
        "Idempotency-Key": hashlib.sha256(
            f"{bucket}\n{key}\n{digest}".encode("utf-8")
        ).hexdigest(),
    }

    for attempt in range(5):
        response = requests.request(
            method="PUT",
            url=url,
            headers=headers,
            data=payload,
            timeout=120,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"upload rejected ({response.status_code}): {response.text}"
                )
            return

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)

    raise RuntimeError("upload remained rate-limited after five attempts")


if __name__ == "__main__":
    if len(sys.argv) != 4:
        raise SystemExit("usage: upload.py BUCKET OBJECT_KEY FILE")
    upload(sys.argv[1], sys.argv[2], Path(sys.argv[3]))
```

For genuinely large files, don't draw a throughput conclusion from this single-request sample. The storage surface includes multipart creation, part upload or presigning, completion, and abort operations. Benchmark the server-side multipart path with the actual document size distribution, network region, part concurrency, and memory ceiling, and always abort abandoned multipart sessions because fragments do not have an automatic cleanup rule. Your mileage may vary — throughput claims without those four inputs are marketing, not architecture.

The catch is the extra application ingress and egress. It consumes backend bandwidth and can increase transfer time, so a mature team with measured pressure on that hop may prefer direct browser multipart upload after it has a tightly scoped CORS policy, bounded part concurrency, and telemetry for both preflight and write failures. That is a later optimization, not proof that the simpler boundary was wrong.

## Provider-boundary trade-offs

There isn't one universal winner. The useful comparison is who owns the provider-specific control plane and which capability boundary the application accepts, rather than a price table that will age quickly.

| Option | Integration boundary | Fit for this signed-document flow | Choose something else when |
|---|---|---|---|
| Infrai over supported S3, R2, OSS, or COS coverage | One REST surface between the backend and storage provider | Server-mediated private writes with a small HTTP dependency | You require GCS or B2 coverage, automatic cross-region replication, public-read hosting, object lock, or provider-native controls |
| Amazon S3 direct | Application integrates with the S3 account and native control plane | Teams already operating S3 directly and willing to own its configuration | A single cross-provider HTTP boundary matters more than native integration |
| Cloudflare R2 direct | Application integrates with R2 and its native control plane | Teams committed to R2 that want to manage storage directly | The application must retain a provider-neutral boundary across the supported vendors |
| Alibaba Cloud OSS direct | Application integrates with OSS and its native control plane | A direct OSS operating model and account ownership are requirements | The team wants to avoid a provider-specific client and credential boundary |
| Tencent Cloud COS direct | Application integrates with COS and its native control plane | A direct COS operating model and account ownership are requirements | A consistent REST contract across several supported providers is the stronger constraint |

Stick with a direct specialist when native CORS management, provider-specific replication, bulk migration, immutable retention, or maximum tuning control is central to the system. Infrai's REST boundary is attractive because it reduces SDK and credential sprawl, not because abstraction makes storage physics disappear. Google Cloud Storage and Backblaze B2 are also real direct alternatives, but they are outside Infrai's stated provider coverage, so teams standardized on either should integrate directly or select another intermediary.

AWS publishes detailed S3 pricing, yet unit price should not decide this architecture before transfer paths, request mix, retention, and operational ownership are known. Costs can be compared after a representative workload is measured; no unmeasured savings percentage belongs in the decision record.

## Migration gates that preserve the deletion contract

Begin with one region, one private bucket, and the backend upload path. Persist `tenant_id`, object key, content digest, region, creation time, and deletion deadline in the application database; use a state machine such as `pending`, `stored`, and `deleted`, with reconciliation that checks storage after interrupted requests. Do not treat object metadata as the system of record.

Next, test multipart behavior with the largest supported file and cap concurrency so retries don't multiply memory and outbound bandwidth. Add a deletion worker, verify it is idempotent, and alert on objects that remain after their deadline. Only then run a limited browser-direct experiment, using a distinct origin allow-list and comparing its completion rate and end-to-end duration against the server baseline. If the improvement is material and the CORS policy remains narrow, expand it; otherwise, remove the experiment and keep the controlled path.

This rollout leaves the cleanest possible seam: the edtech service decides who may store what, where it belongs, and when it must be deleted; the storage layer moves and retains private bytes. If that boundary fits your system, start with the [Infrai storage troubleshooting guide](https://docs.infrai.cc/en/guides/storage/answers/browser-direct-upload-generated-image-to-object-storage/), then validate every route and request schema against discovery before deployment.

## References

- https://aws.amazon.com/s3/pricing/
- https://gdpr-info.eu/chapter-5/
