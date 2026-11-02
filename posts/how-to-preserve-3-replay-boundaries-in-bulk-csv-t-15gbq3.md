# How to Preserve 3 Replay Boundaries in Bulk CSV Text Classification Batch Jobs

For bulk CSV text classification, turn sales calls into reviewable CRM actions by retaining a compact decision ledger and a time-limited source archive. The bill to examine includes model input and output, repeated attempts, retained transcripts and responses, and the work of reconciling uncertain CRM writes. A batch discount cannot tell you which of those terms dominates.

Short answer: measure repeated input before choosing a cheap LLM text classification API for bulk CSV tagging. Keep three replay boundaries: source identity, classification decision, and CRM delivery. Use a provider-independent action schema so switching the classification backend doesn't require rebuilding the CRM ledger. The runnable example below uses Python and a local adapter fixture; its CSV contract can also be consumed by a Node.js worker.

Consider an explicitly hypothetical developer-tools workload: 1,000 sales calls, each with a 12,000-character transcript, classified separately for 4 action categories. Sending the full transcript for each category exposes 48,000,000 input characters before retries. Sending it once and requesting the categories in one response exposes 12,000,000. These are character counts, not token estimates, billing units, or measured production results. In this example, repeated transcript input is the largest counted payload term; actual financial dominance needs usage records and the applicable billing rules.

Count the repeats.

## What should a bulk CSV text classification batch job retain?

A useful sales-call result might identify a request for a security review, a pricing follow-up, or no supported action. The CRM needs a stable action and its delivery status; an engineer investigating a disputed label needs the source and the classification configuration. Those needs have different lifetimes. Keeping every transcript forever makes an incident easier to inspect, but it also commits the system to retaining the largest source material long after the action has been reviewed. Keeping only the final label removes the evidence needed to decide whether the classifier misunderstood the call.

The design here treats retention as a recovery decision.

A source digest establishes which exact input was used, an action record preserves the accepted interpretation, and a delivery receipt records what the CRM acknowledged. None substitutes for the others. A digest cannot reconstruct a deleted transcript, and a successful classification doesn't prove that an external write succeeded.

| Boundary | Retained record | Recovery it enables | Deliberate limit |
| --- | --- | --- | --- |
| Source | Tenant, call ID, digest, expiry, temporary transcript | Reclassify while source remains available | Digest alone cannot explain disputed wording |
| Decision | Input digest, adapter identity, configuration revision, action | Inspect and compare accepted decisions | Same revision doesn't guarantee identical future model output |
| Delivery | Stable action key, state, CRM receipt | Reconcile a write with its destination | Local state alone cannot settle a lost acknowledgment |

Don't infer durability from a table named ledger. The small example uses a local database to make state transitions inspectable; deployment still needs a documented storage failure domain, recovery procedure, and tested backup policy. Those choices determine whether the records survive a machine loss. Provider portability doesn't settle them.

## Step 1 Measure the repeated payload before scheduling work

Save the following blocks, in order, into one Python file and run it with Python 3. They use the standard library, create a local demonstration database, and make no network requests. The fixture is deliberately small: two calls from one developer-tools sales team and a repeated copy of the first row. The repetition tests ingestion identity, rather than pretending that a toy dataset measures classifier quality.

```python
import csv
import hashlib
import io
import json
import sqlite3
from datetime import datetime, timedelta, timezone

source = io.StringIO(
    'tenant,call_id,transcript\n'
    'demo,c1,Please arrange a security review for our SDK rollout.\n'
    'demo,c2,No follow-up requested during the developer tools demo.\n'
    'demo,c1,Please arrange a security review for our SDK rollout.\n'
)
rows = list(csv.DictReader(source))
required = {"tenant", "call_id", "transcript"}
assert required.issubset(rows[0])
assert all(row[name].strip() for row in rows for name in required)

calls = {}
for row in rows:
    key = (row["tenant"], row["call_id"])
    if key in calls and calls[key] != row["transcript"]:
        raise ValueError("Conflicting transcripts for one call identity")
    calls[key] = row["transcript"]

example_calls = 1000
example_characters = 12000
example_categories = 4
print({
    "illustrative_separate_input_characters": (
        example_calls * example_characters * example_categories
    ),
    "illustrative_combined_input_characters": (
        example_calls * example_characters
    ),
    "fixture_rows": len(rows),
    "fixture_unique_calls": len(calls),
})
assert len(calls) == 2
```

The judgment here is narrow: duplicate rows must not become additional calls, and conflicting source text under the same identity must stop ingestion.

Don't silently choose the last row.

A real source revision needs its own explicit version policy, because otherwise a rerun can overwrite the evidence behind an already delivered action. Tenant identity belongs in the key even when today's export contains only one tenant.

Combining categories changes the dominant counted term in the hypothetical workload, but it introduces a quality question: can one response still identify every supported action? Evaluate that question against reviewed sales-call examples before deployment. Track missed actions and unsupported actions separately; accepting an invented security-review request is a different operational mistake from overlooking a follow-up. A batch API is a scheduling choice, not evidence that either error is controlled.

## Step 2 Persist a portable decision with its source identity

Keep the adapter boundary small.

The rest of the job should receive an action list in the application's vocabulary, plus the identity of the adapter and configuration that produced it. Provider-specific responses belong inside the adapter or in a temporary diagnostic archive. They shouldn't become the CRM's schema.

```python
db = sqlite3.connect("sales_call_retention_demo.sqlite3")
db.execute("""
CREATE TABLE IF NOT EXISTS decisions (
    action_key TEXT PRIMARY KEY,
    tenant TEXT NOT NULL,
    call_id TEXT NOT NULL,
    source_digest TEXT NOT NULL,
    adapter TEXT NOT NULL,
    revision TEXT NOT NULL,
    transcript TEXT,
    expires_at TEXT NOT NULL,
    actions TEXT NOT NULL,
    delivery_state TEXT NOT NULL DEFAULT 'pending',
    receipt TEXT
)
""")

allowed = {"security_review", "pricing_follow_up"}
revision = "actions-v1"
adapter = "local-fixture-v1"

def classify_fixture(transcript):
    # This fixture checks plumbing, not language-model quality.
    if "security review" in transcript.lower():
        return {"actions": ["security_review"]}
    return {"actions": []}

def stable_digest(value):
    encoded = json.dumps(
        value, ensure_ascii=False, separators=(",", ":")
    ).encode("utf-8")
    return hashlib.sha256(encoded).hexdigest()

now = datetime.now(timezone.utc)
expires_at = (now + timedelta(days=7)).isoformat()

for (tenant, call_id), transcript in calls.items():
    source_digest = stable_digest(transcript)
    action_key = stable_digest(
        [tenant, call_id, source_digest, revision, adapter]
    )
    existing = db.execute(
        "SELECT 1 FROM decisions WHERE action_key = ?",
        (action_key,),
    ).fetchone()
    if existing:
        continue
    result = classify_fixture(transcript)
    actions = result.get("actions")
    if (
        not isinstance(actions, list)
        or any(not isinstance(item, str) for item in actions)
        or any(item not in allowed for item in actions)
        or len(actions) != len(set(actions))
    ):
        raise ValueError("Invalid classification result")
    with db:
        db.execute(
            """INSERT INTO decisions
            (action_key, tenant, call_id, source_digest, adapter,
             revision, transcript, expires_at, actions)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)""",
            (action_key, tenant, call_id, source_digest, adapter,
             revision, transcript, expires_at, json.dumps(actions)),
        )

assert db.execute("SELECT COUNT(*) FROM decisions").fetchone()[0] == 2
```

Seven days is a demonstration policy, not a recommended retention period. Choose the real window from review latency, contractual obligations, and recovery needs. The repeat-run check above keeps an existing decision without extending its expiry, so a routine rerun cannot silently reset the retention clock. Run this demonstration serially against its own database; concurrent ingestion needs a claim protocol before the remote classification call, because a unique database key prevents duplicate records but doesn't prevent two workers from paying for the same inference.

The fixture intentionally cannot summarize a conversation.

Replace it with a classifier adapter that extracts evidence-supported actions, validates the response, and leaves uncertain results for review. For a full pipeline, transcription and action classification are separate stages; reranking candidate text is another distinct task. The sources in References describe adjacent capabilities, not a validated end-to-end sales-call workflow. Neither supplies evidence for the quality of this fixture.

Changing adapters produces a new decision identity here. That makes comparisons inspectable, but it must not automatically authorize a second CRM write. Select an approved decision for each call before delivery, and treat revised actions as amendments requiring reconciliation with the existing CRM record. Otherwise a provider migration can turn a harmless replay into duplicate follow-up tasks. For example, both adapters might classify the same call as requiring a security review, yet produce different decision keys because the adapter identity changed; delivery must recognize the existing approved action before treating the new classification as new work. The trade-off is extra reconciliation state in exchange for preserving a readable history of how decisions changed.

## Step 3 Reconcile delivery and expire source material deliberately

A timeout after sending an action is an ambiguous outcome: the destination may have accepted it before the acknowledgment was lost. Keep that uncertainty visible. Retry with a stable key only when the destination actually supports the corresponding deduplication contract; otherwise reconcile against the destination's records or hold the action for review. A locally generated key doesn't establish exactly-once delivery across an external boundary.

```python
def record_receipt(connection, action_key, receipt):
    if not isinstance(receipt, str) or not receipt.strip():
        raise ValueError("A destination receipt is required")
    with connection:
        cursor = connection.execute(
            """UPDATE decisions
            SET delivery_state = 'acknowledged', receipt = ?
            WHERE action_key = ? AND delivery_state = 'pending'""",
            (receipt, action_key),
        )
        if cursor.rowcount != 1:
            raise ValueError("Decision is missing or already acknowledged")

# A simulated acknowledgment exercises the ledger without a CRM write.
key = db.execute(
    "SELECT action_key FROM decisions WHERE call_id = ?",
    ("c1",),
).fetchone()[0]
state = db.execute(
    "SELECT delivery_state FROM decisions WHERE action_key = ?",
    (key,),
).fetchone()[0]
if state == "pending":
    record_receipt(db, key, "fixture-receipt-c1")

def expire_sources(connection, cutoff):
    with connection:
        cursor = connection.execute(
            """UPDATE decisions SET transcript = NULL
            WHERE transcript IS NOT NULL AND expires_at <= ?""",
            (cutoff.isoformat(),),
        )
    return cursor.rowcount

# Advance the fixture clock to exercise logical expiry.
removed = expire_sources(db, now + timedelta(days=8))
assert db.execute(
    "SELECT COUNT(*) FROM decisions WHERE transcript IS NOT NULL"
).fetchone()[0] == 0
assert db.execute(
    "SELECT COUNT(*) FROM decisions WHERE receipt IS NOT NULL"
).fetchone()[0] == 1
print({"expired_source_records_this_run": removed})
db.close()
```

This demonstrates logical expiry from the active table. It doesn't demonstrate physical erasure from database pages, snapshots, backups, or upstream transcript exports. A deployed retention job needs an inventory of those copies and a deletion policy for each; leaving a full response in a diagnostic log defeats the boundary even if the primary table looks clean. Action records can also contain sensitive information, so their retention needs an explicit policy rather than an assumption that compact means harmless.

Before deploying a new adapter, replay reviewed calls into a separate decision set and compare action validity, missed actions, and schema rejection counts. During operation, record input volume, retry volume, queue age, review backlog, pending deliveries, and expired-source counts without copying transcripts into telemetry. These measures distinguish a payload problem from a reconciliation backlog. They also let the team check whether the proposed input consolidation actually changes the bill's dominant term.

The trade-off is precise: this design deliberately stops keeping transcripts after the chosen source window and avoids retaining provider response bodies indefinitely. It preserves decision identities and receipts for a separately defined audit window. If a disputed action arrives after source expiry, the ledger can identify the input digest and accepted result, but it cannot show the missing words or recreate the model's reasoning. Recovery then requires an authorized upstream source that still exists, or human review with incomplete evidence. Accept that loss explicitly before scheduling deletion.

## References

The linked material covers adjacent processing stages. It isn't a benchmark, a pricing comparison, or a recommendation of a classification provider. The workload arithmetic and retention windows above are labeled design examples.

- Speech-recognition project reference: https://github.com/openai/whisper
- Reranking documentation reference: https://docs.cohere.com/docs/rerank-overview

## Further reading

- https://github.com/openai/whisper
- https://docs.cohere.com/docs/rerank-overview
