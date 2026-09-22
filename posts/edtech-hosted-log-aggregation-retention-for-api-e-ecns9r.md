# Edtech Hosted Log Aggregation Retention for API Errors and Background Jobs

TL;DR: Use one hosted log destination, emit the same structured event shape from web requests and workers, and make `deployment_id`, `notification_id`, `attempt`, `region`, and a trace identifier searchable. Keep verbose success events briefly, retain delivery failures and rollout evidence longer, and test export before committing to a backend. That is the least complex design that can answer the rollback question: did this release increase failed student notifications, and can the team prove which attempts were affected?

The bill is usually a multiplication problem: events emitted x average encoded bytes x retained days, plus indexing, query, archive, and egress terms defined by the service. Before comparing interfaces, measure those terms from representative traffic. Suppose a planning sample, not a benchmark, contains 10 million request and job events per day at 1.2 KB each. That is about 12 GB/day before transport compression or provider-specific accounting. If routine success traffic contributes 92% of those bytes, trimming or sampling that class changes the dominant term; shaving fields from the 8% failure slice does not.

For an edtech notification service, the useful unit is a delivery attempt, not a line of application text. An API request may schedule a job, the job may retry after a provider timeout, and a later deployment may change template selection. A store that makes the first line easy to find but cannot connect those stages leaves rollback decisions to guesswork.

Connections matter.

## What must survive long enough to make a rollback safe?

A rollback needs comparable evidence from before and after a release boundary. Record an immutable deployment identifier on every request and worker event; do not infer it from a mutable environment label such as `production`. Preserve the outcome vocabulary across releases, because renaming `provider_timeout` to `upstream_error` halfway through a comparison creates a false improvement unless queries normalize both values.

The minimum event should answer who performed the work, what logical operation it represented, when it happened, where data was processed, which release produced it, and how the attempt ended. “Who” should be an internal opaque identifier rather than an email address or message body. Keep secrets and notification content out at emission time. Redaction after ingestion is too late for copies already present in queues, indexes, or archives.

Use severity as a routing signal, not as the data model. RFC 5424 defines severity numerically from Emergency (0) through Debug (7), but an `error` level alone cannot distinguish a permanent invalid destination from a retryable timeout. Put machine-stable failure classes in fields and reserve the human message for diagnosis.

Levels are not outcomes.

## A small event contract for requests and jobs

The following Python example keeps the contract generic. It produces JSON that can go to standard output or a transport selected by the deployment environment; the transport is deliberately outside the application event schema.

```python
import json
from datetime import datetime, timezone
from typing import Any

ALLOWED_OUTCOMES = {
    "accepted", "delivered", "retryable_failure", "permanent_failure"
}

def emit_delivery_event(
    *,
    notification_id: str,
    deployment_id: str,
    trace_id: str,
    region: str,
    attempt: int,
    outcome: str,
    failure_class: str | None = None,
    duration_ms: int | None = None,
) -> None:
    if outcome not in ALLOWED_OUTCOMES:
        raise ValueError(f"unsupported outcome: {outcome}")
    event: dict[str, Any] = {
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "event_name": "notification.delivery_attempt",
        "notification_id": notification_id,
        "deployment_id": deployment_id,
        "trace_id": trace_id,
        "region": region,
        "attempt": attempt,
        "outcome": outcome,
        "failure_class": failure_class,
        "duration_ms": duration_ms,
        "schema_version": 1,
    }
    print(json.dumps(event, separators=(",", ":"), sort_keys=True))
```

Keep `notification_id` stable across retries and increment `attempt`. Carry a W3C Trace Context identifier across the HTTP-to-queue boundary when the queue and worker model allow it, while remembering that a trace identifier is correlation data, not proof of delivery. The terminal delivery event remains authoritative for this workflow.

This contract has a sharp limit: a process can terminate after an external provider accepts a message but before the worker emits the corresponding event. Logs alone cannot close that ambiguity. Reconciliation needs a provider receipt, callback, or idempotent status check, joined by a non-secret delivery identifier. Name that state `unknown`; do not silently count it as success.

## Retention tiers and their failure costs

Treat retention as an evidence policy rather than one global number. The values below are an illustrative starting hypothesis to validate against incident frequency, organizational obligations, and the actual billing definition of a candidate service. They are not universal compliance periods.

| Event class | Illustrative retention | Why keep it | What is lost when it expires |
|---|---:|---|---|
| Debug and successful request detail | 3 days | Immediate rollout diagnosis | Slow regressions and old per-request reconstruction |
| Aggregated success counts by release and region | 90 days | Baselines without raw-event volume | Individual attempt detail |
| Retryable and permanent delivery failures | 30 days | Failure clustering and support investigation | Older recipient-level timelines |
| Deployment and schema-change markers | 180 days | Rollback comparison boundaries | Long-range release attribution |

The aggregate must be computed before raw successes expire, and its dimensions must remain bounded. Aggregate by release, region, outcome, and a controlled failure class; do not use notification identifiers as metric labels. Keep enough overlap to compare a new release with a representative prior window, but avoid claiming that a fixed number of days guarantees representativeness. School calendars and scheduled campaigns can make adjacent days incomparable.

There is a storage trap here. Dual-writing every event to two hosted backends during evaluation appears rollback-friendly, yet it doubles delivery paths and can produce two incomplete histories with different timestamps and retry behavior. A bounded parallel test can reveal discrepancies, but the durable escape hatch is a documented, regularly tested export of raw structured events into an independently readable format. An export that has never been restored is an aspiration.

Test the restore.

## How should hosted log aggregation connect API requests and background jobs?

Start with the data-flow diagram, not a region selector in a dashboard. For each tenant or deployment, identify where the application executes, where buffers spool, where logs are ingested, where indexed data and archives reside, where support personnel can access them, and where exports land. “EU storage” does not by itself describe every transfer or subprocess; the service contract and architecture must resolve those points.

A useful trial replays synthetic events, including retries and deliberately malformed payloads, through the same collector and queue boundaries used in production. Do not send real student data. Verify overload buffering, duplicate handling, clock-skew visibility, field-level redaction, and export fidelity. Then interrupt the network, terminate a worker after its external call, and roll from schema version 1 to version 2. Search by deployment and compare terminal outcomes across the boundary.

Run that trial separately through the US and EU paths under consideration. A nominally identical event can take a different route because an application region, collector endpoint, buffer, archive destination, or support-access boundary differs; the test record should therefore include arrival time, observed region, schema version, deployment identifier, and a checksum of the exported representation. Compare the records field by field. If one path drops an unknown field, rewrites timestamps, truncates a failure class, or delays the candidate deployment beyond the rollback window, document the behavior as a limit rather than averaging it into a score.

The selection table should record limits rather than scores. Scores conceal vetoes.

| Decision boundary | Evidence to request or test | Rollback risk if unclear |
|---|---|---|
| Ingestion backpressure | Documented queue limits plus an overload test | Failure evidence disappears during the event that triggers rollback |
| Regional data path | Contractual terms and a component-level flow diagram | Logs cross an unacceptable boundary or become inaccessible to responders |
| Search freshness | Measured delay during normal and burst traffic | A harmful rollout remains live while evidence is pending |
| Export and deletion | Restore test, format inspection, deletion verification | Migration loses history or retains data beyond policy |
| Access control and audit | Role test using least privilege | Notification metadata is exposed too broadly |
| Schema handling | Mixed-version replay | A field change breaks the pre/post comparison |

No single “simple” service wins every row. The shortlist should exclude any backend that cannot satisfy a hard regional, access, or recovery boundary, then compare the survivors using the same synthetic corpus. Interface polish comes later.

## The rollback runbook is part of the logging design

Define the decision before deployment: compare `permanent_failure` and exhausted `retryable_failure` rates for the candidate deployment against the chosen baseline, segmented by region and notification class. Set thresholds from historical distributions and service objectives, not invented industry constants. A minimum event count prevents one failure in a tiny sample from presenting as a crisis, while an absolute failure ceiling still catches low-volume but severe breakage.

During a rollback, stop or drain producers according to the queue's delivery semantics, record the rollback deployment marker, and preserve idempotency keys so replay does not send duplicate student notifications. Afterward, reconcile `unknown` attempts rather than immediately replaying all non-successes. This is where request logs, job logs, provider receipts, and deployment events become one operational record.

**The deliberate trade-off is that raw successful requests expire first.** When an incident is discovered after that window, the team retains aggregate baselines, failures, deployment markers, and exported evidence, but it may no longer reconstruct every successful request or inspect its transient fields. That loss is acceptable only if the retention decision is written down, tested against realistic investigation delays, and approved by the people responsible for privacy, operations, and support.

## Further reading

- RFC 5424, The Syslog Protocol: https://datatracker.ietf.org/doc/html/rfc5424
- W3C Trace Context Recommendation: https://www.w3.org/TR/trace-context/
- OpenTelemetry logs data model: https://opentelemetry.io/docs/specs/otel/logs/data-model/
- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- NIST SP 800-92, Guide to Computer Security Log Management: https://csrc.nist.gov/pubs/sp/800/92/final
