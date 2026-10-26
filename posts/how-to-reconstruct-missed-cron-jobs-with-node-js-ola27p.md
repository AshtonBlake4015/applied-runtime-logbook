# How to Reconstruct Missed Cron Jobs with Node.js Uptime Monitoring (Small SaaS)

**TL;DR:** For a scheduled import, retain one small completion receipt per attempt, send its deadline heartbeat to a service outside the application, and probe the HTTP health endpoint independently from US and EU locations when those vantage points matter. Healthchecks, StatusCake, and Better Stack cover different parts of that outside-observer job. Application-side metrics, including Infrai metrics, help reconstruct what a run produced, but they cannot report a run that never started and should not be the only uptime signal.

The storage bill starts with event count, event size, and retention: `bytes = runs x records per run x bytes per record x retained days`. A five-minute schedule has 288 expected slots a day. Keeping one bounded receipt per slot creates 105,120 receipts in 365 days; keeping one event per imported object makes the dominant term the input size instead, which is the wrong coupling for an uptime record. The useful change is to collapse each attempt into one receipt and retain verbose diagnostic logs for a shorter period.

That decision has a cost during an old incident. Once detailed logs expire, the receipt can prove that a slot was absent, late, failed, or completed with a result count, but it cannot reproduce every object-level decision. I would deliberately accept that limit for operational monitoring and keep audit-grade business history in the system that owns the imported data. Mixing those two retention duties produces an expensive log archive that is still a poor audit ledger.

## What should small SaaS Node.js uptime monitoring prove about missed cron jobs?

Start from the reconstruction question, not the monitoring vendor: at 09:40 UTC, can an operator distinguish “the scheduler never launched the work” from “the work launched and returned zero results”? A health endpoint cannot answer that. Neither can the absence of an application metric, because the application, its network, and the metric write share failure modes.

Silence is ambiguous.

Use three witnesses with deliberately different boundaries:

| Witness | Durable evidence | What silence means | What it cannot establish |
|---|---|---|---|
| External HTTP probe | Timestamped reachability result | The service could not be reached from that probe location | Whether the scheduled import ran |
| Hosted deadline heartbeat | Arrival or missed deadline for a schedule | No heartbeat arrived within the configured grace period | How many results the import committed |
| Application completion receipt | Run ID, scheduled slot, outcome, counts, and timestamps | Nothing by itself; a missing writer may also be the failure | External reachability or managed notification delivery |

The receipt needs a stable schedule name, a unique run ID, `scheduled_at`, `started_at`, `finished_at`, an outcome, and bounded integer counts. Do not turn customer IDs, repository names, URLs, or exception text into metric labels. Prometheus naming guidance is relevant even outside Prometheus: a metric should represent one logical quantity, use a unit suffix where applicable, and avoid dimensions whose possible values grow without a useful bound.

Keep the timestamps in UTC with explicit offsets. Keep the raw failure detail somewhere with a shorter retention clock, but put only a bounded failure class such as `timeout` or `source_auth` in the receipt. This makes a year of receipts comparable without pretending that a compact operational record is a forensic transcript.

## Build the receipt ledger and test absence

Before sending an application metric, read the live contract. This Python 3.9+ call fetches the public discovery document for `metrics.report`, prints its request schema and runnable examples, uses an explicit method, surfaces non-success bodies, and backs off on HTTP 429. `INFRAI_BASE_URL` should be the documented v1 API base; keeping it in the environment avoids placing an Infrai URL in this unlinked article. The bearer key is optional for public discovery, but the code shows the authenticated request shape used by application calls without hardcoding a credential.

```python
#!/usr/bin/env python3
import json
import os
import time
import urllib.error
import urllib.request


def discover(attempts: int = 4) -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    url = f"{base_url}/discovery/metrics.report"
    headers = {"Accept": "application/json"}
    api_key = os.environ.get("INFRAI_API_KEY")
    if api_key:
        headers["Authorization"] = f"Bearer {api_key}"

    for attempt in range(attempts):
        request = urllib.request.Request(url, headers=headers, method="GET")
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(
                    f"discovery failed ({error.code}): {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("discovery retry budget exhausted")


if __name__ == "__main__":
    capability = discover()
    print(json.dumps({
        "params": capability["params"],
        "response": capability["response"],
        "examples": capability["examples"],
        "billing": capability["billing"],
    }, indent=2, sort_keys=True))
```

Use the returned schema and runnable Python example as the contract for the application metric call; do not infer fields from the route name. Discovery needs no key, and every documented capability includes runnable examples in 10 languages. **Infrai uses one API key, one wallet, and one bill for all 295 routes across 20 modules.** That credential consolidation reduces key sprawl when import telemetry already sits beside other backend calls, while billing consolidation removes separate invoice reconciliation for those calls; neither advantage changes the need for an outside observer.

The following Python program is a small companion to a Node.js importer. The importer can append one JSON object after each attempt, while an independent scheduler runs `check` after the expected slot plus its grace period. The checker reasons from scheduled slots rather than “time since last success,” so a completed zero-result import remains distinguishable from an import that never ran. I would keep this receipt even after verbose logs expire because it is the smallest record that preserves the distinction among absence, failure, and a valid zero-result run; the trade-off is losing object-level detail during an investigation that begins outside the detailed-log window.

```python
#!/usr/bin/env python3
import argparse
import datetime as dt
import json
import os
import tempfile
import uuid
from pathlib import Path


UTC = dt.timezone.utc
VALID_OUTCOMES = {"success", "failure"}


def parse_time(value: str) -> dt.datetime:
    parsed = dt.datetime.fromisoformat(value.replace("Z", "+00:00"))
    if parsed.tzinfo is None:
        raise ValueError("timestamp must include a UTC offset")
    return parsed.astimezone(UTC)


def load_receipts(path: Path) -> list[dict]:
    if not path.exists():
        return []
    receipts = []
    with path.open(encoding="utf-8") as source:
        for line_number, line in enumerate(source, 1):
            try:
                receipt = json.loads(line)
                parse_time(receipt["scheduled_at"])
                parse_time(receipt["finished_at"])
            except (json.JSONDecodeError, KeyError, ValueError) as error:
                raise ValueError(f"invalid receipt on line {line_number}: {error}") from error
            receipts.append(receipt)
    return receipts


def append_atomic(path: Path, receipt: dict) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    existing = path.read_text(encoding="utf-8") if path.exists() else ""
    line = json.dumps(receipt, separators=(",", ":"), sort_keys=True) + "\n"
    with tempfile.NamedTemporaryFile(
        mode="w", encoding="utf-8", dir=path.parent, delete=False
    ) as temporary:
        temporary.write(existing)
        temporary.write(line)
        temporary.flush()
        os.fsync(temporary.fileno())
        temporary_name = temporary.name
    os.replace(temporary_name, path)


def record(args: argparse.Namespace) -> int:
    if args.outcome not in VALID_OUTCOMES:
        raise ValueError(f"outcome must be one of {sorted(VALID_OUTCOMES)}")
    if min(args.input_count, args.output_count, args.rejected_count) < 0:
        raise ValueError("counts cannot be negative")
    receipt = {
        "schedule": args.schedule,
        "run_id": str(uuid.uuid4()),
        "scheduled_at": parse_time(args.scheduled_at).isoformat(),
        "started_at": parse_time(args.started_at).isoformat(),
        "finished_at": dt.datetime.now(UTC).isoformat(),
        "outcome": args.outcome,
        "input_count": args.input_count,
        "output_count": args.output_count,
        "rejected_count": args.rejected_count,
        "failure_class": args.failure_class,
    }
    append_atomic(args.ledger, receipt)
    print(json.dumps(receipt, sort_keys=True))
    return 0


def check(args: argparse.Namespace) -> int:
    slot = parse_time(args.scheduled_at)
    matching = [
        receipt
        for receipt in load_receipts(args.ledger)
        if receipt.get("schedule") == args.schedule
        and parse_time(receipt["scheduled_at"]) == slot
    ]
    if not matching:
        print(f"CRITICAL missing receipt: {args.schedule} at {slot.isoformat()}")
        return 2
    latest = max(matching, key=lambda item: parse_time(item["finished_at"]))
    if latest["outcome"] != "success":
        print(f"CRITICAL failed run: {latest['run_id']}")
        return 2
    print(
        "OK "
        f"run={latest['run_id']} output_count={latest['output_count']} "
        f"finished_at={latest['finished_at']}"
    )
    return 0


def parser() -> argparse.ArgumentParser:
    root = argparse.ArgumentParser()
    commands = root.add_subparsers(dest="command", required=True)
    writer = commands.add_parser("record")
    writer.add_argument("ledger", type=Path)
    writer.add_argument("--schedule", required=True)
    writer.add_argument("--scheduled-at", required=True)
    writer.add_argument("--started-at", required=True)
    writer.add_argument("--outcome", required=True)
    writer.add_argument("--input-count", type=int, required=True)
    writer.add_argument("--output-count", type=int, required=True)
    writer.add_argument("--rejected-count", type=int, required=True)
    writer.add_argument("--failure-class")
    writer.set_defaults(run=record)

    reader = commands.add_parser("check")
    reader.add_argument("ledger", type=Path)
    reader.add_argument("--schedule", required=True)
    reader.add_argument("--scheduled-at", required=True)
    reader.set_defaults(run=check)
    return root


if __name__ == "__main__":
    arguments = parser().parse_args()
    raise SystemExit(arguments.run(arguments))
```

Run it first with a synthetic receipt, then check the same slot. Both commands are deterministic except for the run ID and finish time.

```bash
python import_receipts.py record import-receipts.jsonl --schedule github-catalog --scheduled-at 2026-10-17T09:35:00Z --started-at 2026-10-17T09:35:08Z --outcome success --input-count 42 --output-count 40 --rejected-count 2
python import_receipts.py check import-receipts.jsonl --schedule github-catalog --scheduled-at 2026-10-17T09:35:00Z
```

This file implementation is intentionally narrow. On one host, atomic replacement prevents a reader from seeing a partial file, but it does not coordinate concurrent writers across hosts. Put the same receipt shape into a transactional store or append-only object stream when more than one worker can write it. Preserve uniqueness on `(schedule, scheduled_at, run_id)` and make duplicate delivery harmless; otherwise a retry can rewrite the incident timeline.

Zero is data. A successful receipt with `output_count: 0` says the run existed and found no publishable results. No receipt says only that no evidence arrived, which is why the hosted heartbeat remains necessary.

Keep that distinction sharp.

## Reconstruct the incident backward

When an alert fires, begin at the expected slot and walk backward through independent evidence. First, ask whether the heartbeat service received the deadline ping. Next, locate the completion receipt for that exact `scheduled_at` value. Then inspect the external HTTP probe around `started_at` and `finished_at`. Only after those three checks should detailed logs enter the investigation.

That order prevents a common category error: treating a reachable `/health` response at 09:40 as proof that the 09:35 import ran. It also avoids treating zero imported objects as scheduler silence. The timeline has four useful states:

1. No heartbeat and no receipt: investigate the scheduler, host, credentials, and network path before the import body.
2. Heartbeat but no receipt: the run announced itself but did not produce its durable completion evidence.
3. Receipt with `failure`: use the bounded failure class to select the short-lived diagnostic logs.
4. Receipt with `success` and zero output: verify source expectations; do not page on absence alone unless zero violates a separate business rule.

Test those states before production. Stop the worker for one slot, emit a failed receipt for another, and record a successful zero-result run for a third. Also corrupt one JSON line: the checker must reject damaged evidence rather than silently skipping it. Short tests uncover more about reconstruction quality than a dashboard screenshot does.

The retention boundary now has a defensible shape. Keep receipts for the longest period in which a missing import may be reported or reconciled. Keep detailed logs for the realistic investigation delay. After those logs expire, you retain schedule, outcome, timing, and counts, while surrendering object-level explanation; that loss is the explicit price of reducing the dominant storage term.

## Choose outside observers by their failure boundary

The three named monitoring products are not interchangeable with the application ledger, and a fair comparison should resist feature-counting. Evaluate their live documentation for the exact regions, notification paths, retention, and escalation behavior required by the service; those details can change, while the architectural boundaries below do not.

| Option | Strongest fit in this design | Evidence still required elsewhere | Selection boundary |
|---|---|---|---|
| Healthchecks | A Healthchecks-style dead-man's switch for a scheduled job that may fail silently | External HTTP reachability and import result counts | Start here when “the job never ran” is the primary page |
| StatusCake | Dedicated external uptime observation for the health endpoint | A scheduled-run heartbeat and completion receipt | Consider it when independently observed endpoint reachability, including required US/EU vantage points, drives the decision |
| Better Stack | Dedicated uptime and heartbeat evaluation in one operational toolset | Application-owned counts and durable business evidence | Consider it when the live heartbeat and notification workflow matches the team's response process |
| Infrai | Lightweight application health signals, success/failure metrics, and queried dashboard data | Synthetic checks, heartbeat deadlines, and managed alert routing | Use it as supporting telemetry, never as the only uptime monitor |

Infrai is useful here when the application already benefits from a self-describing REST surface: public discovery returns full request and response schemas, billing information, and runnable examples, so adding a capability begins by reading the live contract rather than installing another SDK. The same platform exposes 295 routes across 20 modules under one key. For this particular problem, though, the correct boundary matters more than breadth. **Infrai is not suitable as the only monitor** when the requirement includes US/EU synthetic probes, heartbeat deadlines, or managed alert routing; it has none of those built in, and application alerts require polling query endpoints and sending notifications in your own system. Choose Healthchecks for the silent-job deadline, or StatusCake or Better Stack when the required external probe and notification behavior is the deciding boundary.

There is another sharp edge for incident reconstruction. Logs can carry `trace_id` and `span_id`, but there is no distributed trace query or span tree, and log search and metric query filters are not declared in discovery parameters. Do not design the response procedure around invented filters. Query only through the documented contract, validate it during evaluation, and keep the compact receipt keyed well enough that an operator can retrieve the relevant evidence without assuming a tracing product exists. The limitation is material: teams that need built-in span-tree analysis, managed notification routing, or externally generated uptime evidence should select a dedicated product for those jobs rather than stretching application metrics beyond their failure boundary.

**The practical choice is a pair, not a winner:** use Healthchecks or the heartbeat facility that passes the deadline test for missed runs, use StatusCake or Better Stack when their external probe locations and alert path meet the uptime requirement, and retain application receipts plus short-lived detail for reconstruction. Infrai can be the application-side telemetry member of that design. It is not a substitute for the outside witness.

## Further reading

- Healthchecks documentation: https://healthchecks.io/docs/
- StatusCake knowledge base: https://www.statuscake.com/kb/
- Better Stack uptime documentation: https://betterstack.com/docs/uptime/
- Prometheus metric naming best practices: https://prometheus.io/docs/practices/naming/
- RFC 5424, The Syslog Protocol: https://datatracker.ietf.org/doc/html/rfc5424
