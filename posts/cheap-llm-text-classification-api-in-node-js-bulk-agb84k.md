# Cheap LLM Text Classification API in Node.js: Bulk CSV Tagging Batch Job Failure Boundaries

Customer-support moderation changes the answer: a label is a data record, not a chat reply. Short answer: use an asynchronous batch boundary for large CSV tagging, give every row a durable identity, constrain the model to a versioned label set, and publish results only after reconciliation. A cheap request that cannot explain which input produced a decision is expensive to operate.

This is an architecture decision before it is a model decision. The system has to classify moderation reports before human review, preserve the original evidence, and make uncertainty visible to the reviewer. Three words matter here: identity, state, and proof.

## Decision record: protect the classification contract

The input row needs an application key that survives sorting and re-exporting. A line number is a weak identity; it changes when an analyst filters the file. Store a stable report ID, the source-file checksum, the prompt version, the label-set version, and the batch submission ID in a manifest. That manifest lets a replacement worker determine what was accepted, what is still pending, and what must be reviewed.

The output contract should be closed. For example, the allowed moderation labels might be `self_harm`, `harassment`, `spam`, and `needs_human_review`. The importer rejects unknown labels, missing report IDs, duplicate IDs, and malformed JSON into a review queue. Explanations can be retained as evidence, but an explanation must not silently become a new category.

No guesswork.

The final write should be idempotent. Stage results by batch ID, verify that each source key occurs once, compare the source checksum, and promote only the reconciled set. Running promotion twice should produce the same canonical rows. This is where object-storage habits help: immutable inputs and append-only manifests are easier to audit than a mutable “latest.csv” that has lost its lineage.

## What should a Node.js CSV tagging batch job verify before human review?

The production worker may be Node.js, but the important interface is language-neutral. It should separate submission latency from completion latency, because a web request should not remain open while a queue processes a large export. Persist the accepted job ID, poll deliberately, and treat a terminal result as data that still needs validation.

Before a full run, sample the file and measure the shape of the input: empty reports, unusually long reports, missing locale, repeated report IDs, and labels that are rare enough to need a reviewer. I’m not sure any fixed sample size is honest across datasets; the label risk and available review capacity should determine the threshold. The sample is a gate, not a promise of accuracy.

Record it.

Retry policy needs a boundary. A `429` is flow control: honor `Retry-After` when it exists and use bounded exponential backoff. A timeout after submission is not proof that the job was rejected. Look up the manifest and the idempotency key before creating another job. A malformed payload or an authentication failure should be recorded for intervention, not retried until the queue fills with copies.

Consider a nightly export with 80,000 moderation reports. The importer has already written the checksum and label vocabulary when the submission client times out after sending its request. If the on-call engineer treats that timeout as rejection, the next run creates a second job; both jobs can later produce plausible labels, and a downstream “last write wins” rule hides the duplication. A manifest-first worker takes a slower but defensible path: it checks whether the immutable idempotency key has an accepted job, records the last observed state, and resumes polling that job. If no accepted job exists, it submits once and records the returned ID before doing anything else. After completion, it compares the returned report IDs with the manifest, sends missing or duplicate IDs to review, and promotes only the reconciled rows. The extra records are not ceremony. They are the evidence needed to distinguish a transport uncertainty from a classification decision.

The operational record should answer four questions: which source was classified, with which prompt and label vocabulary, by which job, and why a row was sent to a person. Those fields matter more than a dashboard showing only average latency.

## A small adapter for the critical path

The adapter below illustrates the durable boundary without binding the CSV pipeline to a commercial SDK. The endpoint is intentionally supplied by configuration; the application owns schema validation and reconciliation rather than guessing that a successful HTTP response means every row is safe to publish.

```python
import hashlib
import json
import os
import time
import urllib.error
import urllib.request


API_URL = os.environ["CLASSIFICATION_BATCH_URL"]
API_TOKEN = os.environ["CLASSIFICATION_API_TOKEN"]


def source_fingerprint(path):
    digest = hashlib.sha256()
    with open(path, "rb") as source:
        for chunk in iter(lambda: source.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def submit_once(payload, idempotency_key):
    request = urllib.request.Request(
        API_URL,
        data=json.dumps(payload).encode("utf-8"),
        headers={
            "Accept": "application/json",
            "Authorization": f"Bearer {API_TOKEN}",
            "Content-Type": "application/json",
            "Idempotency-Key": idempotency_key,
        },
        method="POST",
    )
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            return json.load(response)
    except urllib.error.HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        if error.code == 429:
            delay = min(60, 2 ** 3)
            time.sleep(delay)
        raise RuntimeError(f"batch submission failed: HTTP {error.code}: {detail}") from error


def build_manifest(source_path, prompt_version, label_version, payload):
    return {
        "source": source_path,
        "source_sha256": source_fingerprint(source_path),
        "prompt_version": prompt_version,
        "label_set_version": label_version,
        "payload": payload,
        "submitted": False,
    }


def main():
    manifest = build_manifest(
        os.environ["SOURCE_CSV"],
        os.environ["PROMPT_VERSION"],
        os.environ["LABEL_SET_VERSION"],
        {"records": []},
    )
    result = submit_once(manifest["payload"], os.environ["IDEMPOTENCY_KEY"])
    manifest["job_id"] = result["job_id"]
    manifest["submitted"] = True
    print(json.dumps(manifest, indent=2))


if __name__ == "__main__":
    main()
```

There is a deliberate omission: this sample does not pretend to know a provider’s response schema. In a real worker, persist the manifest before submission, persist the accepted job ID immediately afterward, and make a restart load that record first. Validate every returned report ID and label against the manifest before promotion. A code sample that hides those steps is shorter, but it teaches the wrong failure model.

## Comparing designs by failure boundary

The useful comparison is who owns the contract and the operational burden, not which service has the longest feature list.

| Design | Application owns | Suitable when | The catch |
| --- | --- | --- | --- |
| Direct hosted model API | Provider adapter, retries, normalization, and migration | Native controls are essential and the provider choice is stable | Provider-specific behavior becomes part of the application contract |
| Shared model gateway | A normalized request, label contract, and reconciliation layer | The team needs one integration boundary across changing backends | A provider-only control may be unavailable through the shared contract |
| Self-hosted model | Serving, capacity, upgrades, and incident response | Governance or specialization justifies infrastructure ownership | Operations can outweigh the inference call |

The catch is important. A normalized gateway is not suitable when moderation depends on a control it does not expose; use the direct interface in that case. Self-hosting is not automatically safer either: it moves custody and capacity decisions into your team. The choice should follow the failure boundary you are prepared to own.

## Why per-row calls are still useful sometimes

One request per row is a poor default for a large backfill because a worker crash leaves many ambiguous restart points. It is a reasonable fit for a support agent waiting on one report, a small administrative correction, or a workflow where the human response is more valuable than queue throughput.

Batching also has limits. A delayed result is unacceptable when a reviewer needs an immediate answer. Large payloads can make individual bad records harder to isolate. And if the label taxonomy is changing during the run, a faster batch only produces a larger reconciliation problem. Freeze the vocabulary for the job, or split the input by taxonomy version.

The acceptance test is intentionally boring: every input ID appears exactly once in accepted output or the review queue; every label is in the versioned vocabulary; the source checksum matches; evidence is retained; and rerunning promotion is a no-op. That is the standard I would use to judge a classification API, regardless of its price or model brand.

## References

- https://docs.cohere.com/docs/rerank-overview
- https://github.com/openai/whisper
- https://www.rfc-editor.org/rfc/rfc7231
- https://www.rfc-editor.org/rfc/rfc9110
