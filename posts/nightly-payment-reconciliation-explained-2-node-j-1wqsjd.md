# Nightly Payment Reconciliation Explained: 2 Node.js Paths for Cron, Queue, and Email

Short answer: choose cron plus a database-backed run ledger for one nightly payment reconciliation, then add a queue only when measured execution time, retry isolation, or fan-out requires it. The deciding constraint is not which scheduler looks more sophisticated; it is whether the report can finish inside its delivery window without paying the operational cost of another durable system.

**Decision:** let cron create a uniquely identified reconciliation run, let workers claim bounded pages from that run, and send the daily report email only after the ledger says every page is complete. For a small Node.js SaaS workload, those workers can begin in the cron-started process. The database remains the authority, so moving execution to a queue later does not change correctness.

This is an architecture decision record for a developer-tools company reconciling its internal payment records against a provider every night. Latency versus cost is the primary axis. A report that arrives five minutes sooner has little value if the delivery window is two hours, while an extra broker, worker pool, and on-call surface have costs even when message volume is low.

## Evaluation criteria for a 2-hour delivery budget

Start with a delivery budget, not a technology preference. Suppose the trigger becomes eligible at 02:00 and the report is due at 04:00. The useful inequality is `tail fetch time + comparison time + retry allowance + email time < 120 minutes`. A broker can reduce waiting between units of work, but it cannot improve the provider's response time or remove the need to compare complete data. If one checkpointed runner stays comfortably inside the budget, another durable service buys little user-visible latency.

Component count is only a proxy for cost. The actual bill includes deployment, upgrades, dashboards, retention rules, access control, backup policy, and the engineer who must distinguish a late message from a lost one at 03:30. A queue can be the cheaper choice when parallelism protects a contractual deadline; cron can be the expensive choice when a serial retry forces someone to reconstruct a partial run. Put both labor and delay in the record, even if neither can be reduced to a neat monthly figure.

No magic threshold exists.

Record page-fetch duration, page count, throttled attempts, claim age, and end-to-end completion across representative nights. I'm not sure what latency distribution your payment provider will produce, and no generic architecture diagram resolves that uncertainty. The observed tail, plus an explicit safety margin, does.

## Reliability failures across calendar, settlement, and acknowledgement state

A scheduler proves that a time condition was observed. It does not prove that reconciliation completed, that an email was sent once, or that two application instances did not observe the same minute. The durable object should therefore be a run record keyed by business date and account scope, such as `reconcile:2026-08-19:merchant-42`; a unique constraint turns concurrent triggers into one run, while stable page keys make provider pagination replayable and a separate send key guards final publication.

Walk one 02:00 run through the ledger. Instance A inserts the run key and stores the first provider page together with its next cursor. Instance B observes the same minute, attempts the same insert, sees the existing run, and exits without creating parallel reconciliation. A restarter claims page two, saves its settlements and cursor in one transaction, then loses its lease before page three; a later worker can resume from the last committed cursor without guessing which rows survived. Once every expected page is present, comparison moves the run from imported to reconciled. A mismatch stops publication because it is a business exception, not a transient delivery attempt. A balanced result permits one email reservation. If the process loses contact after asking the mail service to submit but before recording the response, the ledger marks that boundary as ambiguous for review unless the mail interface accepts an idempotency key. This one run names six different outcomes, and none of them is accurately described by a single `job_failed` flag.

Keep scheduled, imported, reconciled, and reported distinct.

Calendar semantics need equal care. The Linux crontab format matches minute, hour, day-of-month, month, and day-of-week fields, and its manual warns that civil times affected by daylight-saving changes may occur twice or not at all. Use a named business timezone only if the report follows that civil calendar; otherwise schedule in UTC and store the intended business date separately. The catch is that queue delivery does not repair a poorly defined date: a perfectly acknowledged message can still reconcile the wrong accounting day.

Acknowledgement is narrower than correctness. RabbitMQ consumer acknowledgements, for example, let a consumer report that a delivery was processed, and unacknowledged deliveries can be requeued after a channel or connection closes. A worker should acknowledge only after the page transaction commits. Even then, the broker knows nothing about balanced totals or ambiguous email submission; those facts belong in the run ledger.

## Integration boundary: persist page progress before parallel pickup

The following Python sketch is intentionally about the storage contract, even though Node.js is the named application stack. The same transaction boundaries apply in either runtime, and Python makes the pseudonymous interfaces compact. The scheduler calls `start_run`; a local loop or queue consumer calls `reconcile_page`; a finalizer evaluates the ledger before requesting email delivery. No vendor endpoint is assumed.

```python
from dataclasses import dataclass
from datetime import date
from decimal import Decimal
from typing import Protocol, Sequence


@dataclass(frozen=True)
class Settlement:
    external_id: str
    amount: Decimal


class PaymentProvider(Protocol):
    def settlements(self, business_date: date, cursor: str | None) -> tuple[Sequence[Settlement], str | None]: ...


class RunStore(Protocol):
    def create_once(self, run_key: str, business_date: date) -> bool: ...
    def save_page_once(self, run_key: str, cursor: str, rows: Sequence[Settlement], next_cursor: str | None) -> None: ...
    def mark_compared(self, run_key: str) -> None: ...
    def comparison_is_balanced(self, run_key: str) -> bool: ...
    def reserve_email_once(self, run_key: str) -> bool: ...
    def mark_email_submitted(self, run_key: str) -> None: ...


class Mailer(Protocol):
    def submit_report(self, run_key: str) -> None: ...


def start_run(store: RunStore, business_date: date) -> str:
    run_key = f"reconcile:{business_date.isoformat()}"
    store.create_once(run_key, business_date)
    return run_key


def reconcile_page(
    store: RunStore, provider: PaymentProvider, run_key: str,
    business_date: date, cursor: str | None,
) -> str | None:
    rows, next_cursor = provider.settlements(business_date, cursor)
    page_key = cursor or "first"
    store.save_page_once(run_key, page_key, rows, next_cursor)
    return next_cursor


def finalize(store: RunStore, mailer: Mailer, run_key: str) -> None:
    store.mark_compared(run_key)
    if not store.comparison_is_balanced(run_key):
        return
    if store.reserve_email_once(run_key):
        mailer.submit_report(run_key)
        store.mark_email_submitted(run_key)
```

There is a deliberate limit in this example: `reserve_email_once` prevents ordinary duplicate callers, but it cannot prove what happened if the process loses contact with the mail service after submission and before `mark_email_submitted`. Resolve that ambiguity with a mail API idempotency key when available, or record the state as unknown and reconcile it before retrying. Don't hide the uncertainty behind a boolean named `sent`.

The page transaction must persist imported rows and the next cursor together. Otherwise, a process exit can advance the cursor without storing its settlements, creating a quiet gap, or store a page without advancing and duplicate its rows on retry. `save_page_once` therefore implies a uniqueness constraint on `(run_key, page_key)` and one atomic commit.

Commit first. Acknowledge second.

## Should a Node.js SaaS schedule a daily report email with cron or a queue?

Use cron for the clock in either design. Keep execution in a checkpointed runner while its measured tail plus retry budget remains well inside the delivery window; move page or account work onto a durable queue when independent retries, backpressure, or parallel pickup are necessary to protect that window. The comparison is about the execution path, not cron versus queue as interchangeable schedulers.

| Execution path | Latency behavior | Ongoing cost | Failure isolation | Choose it when |
|---|---:|---:|---|---|
| Cron starts one checkpointed runner | Serial completion | Fewest moving components | A slow account extends the run | Volume is small and overnight slack is generous |
| Cron creates database work claims | Polling adds pickup delay | Uses primary database capacity | Leases isolate account attempts | Moderate fan-out fits existing database headroom |
| Cron enqueues durable work units | Workers can pick up promptly | Broker and worker operations | Message-level retries and backpressure | The deadline is tight or account latency is uneven |

Database claims deserve scrutiny rather than automatic acceptance as a compromise. They can provide controlled parallelism with transactions and expiring leases, but aggressive polling consumes primary database capacity, priority is awkward, and old work needs retention rules. A queue has its own operational surface. A single runner has head-of-line blocking. None is free; the table identifies where each cost lands.

The rejected option for this reconciliation is the bare in-memory cron callback that fetches every settlement, compares totals, and immediately sends mail. It has low pickup latency and almost no infrastructure overhead, yet it loses progress on restart, has no durable answer to duplicate triggers, and couples publication to a complete refetch. Stick with that shape when output is disposable, recomputation is cheap, recipients tolerate duplicates, and a missed day has no financial or audit consequence. A cached personal activity digest may fit. A payment exception report usually doesn't.

## Rollout signals and the exit rule

Begin with the ledger schema, fixed provider fixtures, and one runner. Test a duplicate trigger, repeated page, expired claim, mismatch, and interruption before finalization. During deployment, old and new workers must agree on run states and uniqueness keys; otherwise a zero-downtime release can create two interpretations of the same durable row.

Watch the deadline, not mere process liveness.

Emit run age, oldest unprocessed page age, attempt count, imported and internal totals, discrepancy amount, and email state. A restarted worker may be harmless, while a live worker holding a stale lease can still make the report late. If the delivery budget is repeatedly tight, first determine whether concurrency is allowed by the provider and whether the primary database can absorb claims. Queue adoption is justified when isolated pickup and retry reduce the measured tail enough to cover their operating cost. It is not suitable when a few serial accounts already finish with wide slack and the team would be adding its first broker solely for this batch.

The exit rule is explicit: observed tail duration plus retry allowance must remain below the delivery window. If that inequality fails, partition using the existing run and page keys; if it continues to hold, keep the smaller execution surface. This decision can change without changing the reconciliation model, which is the point of making progress durable before making it parallel.

## References

- Linux `crontab(5)` manual: https://man7.org/linux/man-pages/man5/crontab.5.html
- RabbitMQ consumer acknowledgements: https://www.rabbitmq.com/docs/confirms
