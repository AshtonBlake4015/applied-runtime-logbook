# Node.js Transactional Email Dashboard Polling by Message ID (Retention Trade-offs)

Short answer: a Node.js SaaS can build a useful compliance-notice delivery dashboard by storing each outbound message ID, polling message details and event lists, and projecting those observations into an append-only audit record. The dashboard will be near-real-time, not instant, because the event interface is pull-only. That is usually adequate for an internal operations screen, provided nobody mistakes a quiet poll interval for proof of delivery.

The bill is made of three things: sending the notices, repeatedly reading their state, and retaining evidence. The last term grows predictably. For `N` notices with an average of `E` meaningful transitions, a normalized design keeps roughly `N + (N * E)` durable rows, while saving every polling response makes retention grow with poll count instead. Polling a pending message 60 times should not create 60 pieces of compliance evidence when perhaps only two state transitions occurred.

My recommendation is therefore to persist message identity and state changes, deduplicate unchanged observations, and shorten polling after a terminal result. This changes the dominant storage term from the number of checks to the number of actual transitions. It also preserves the evidence the dashboard exists to show.

Keep transitions, not noise.

## What should the audit record retain?

Start with one outbound record per compliance notice: the internal notice ID, recipient reference, provider message ID, accepted timestamp, current normalized state, and the timestamp of the latest observation. Keep message content or a content digest according to the organization's retention policy; a dashboard does not need to duplicate a rendered email body merely to count delivery outcomes.

The transition table is the durable history. Each row should identify the message, normalized state, provider event identity when one exists, provider timestamp, observation timestamp, and a small raw reference sufficient to trace the source. Put a uniqueness constraint around the strongest stable event identity available. If the upstream representation does not expose one, derive a deterministic deduplication key from the immutable fields that the adapter can actually verify.

Be conservative about vocabulary. “Sent,” “delivered,” and “bounced” are operational states, not equivalent legal conclusions: a delivery event does not prove that a human read a notice, and an accepted send does not prove mailbox delivery. Gmail's sender guidelines also make authentication, spam rate, and subscription behavior part of deliverability operations; a green dashboard cell cannot compensate for poor sender hygiene.

For a simple panel, the materialized view can be small:

| Dashboard field | Durable source | Why it belongs |
|---|---|---|
| Sent count | outbound rows accepted for sending | Establishes the denominator |
| Delivered count | latest normalized delivery transition | Shows provider-reported delivery |
| Bounced or failed count | latest normalized failure transition | Drives operator follow-up |
| Pending count | accepted rows without a terminal transition | Exposes lag and stuck work |
| Last checked | poll attempt metadata | Separates stale data from unchanged data |

Do not infer a missing event. Display “pending” or “unknown,” plus freshness, until the source reports a terminal outcome.

## How should a Node.js transactional email deliverability dashboard handle polling?

A scheduled worker should poll actively after submission, then back off as the message ages. The exact intervals are an operating-policy decision, because the available facts establish pull-only events but do not establish a provider delivery-time distribution. Pick an initial schedule, measure request volume and observed transition latency in your own traffic, and revise it. No universal five-second number is defensible here.

Keep the HTTP adapter separate from the state projector. The adapter fetches `/v1/email/get/{id}` for a focused message lookup or `/v1/email/event/list` for event ingestion; the projector maps a response into the application's own state vocabulary inside a database transaction. A lease or row lock should prevent two Node.js workers from polling the same message concurrently. On rate limiting, honor `Retry-After` when present and otherwise use exponential backoff. Surface other non-success responses rather than converting them into an empty event list. For example, consider a notice accepted at 09:00 that remains unchanged over eight polls before a delivery transition appears: the poll-attempt table may record all eight checks for short-term operations, but the audit ledger should add one delivery transition, not eight copies of the same pending document. If a second worker acquires the same job, the database lease stops duplicate reads; if an old event later appears, its source timestamp is retained even though observation time is newer. That distinction is small in the schema and decisive in an investigation.

This runnable adapter performs one message read, retries a rate limit, and prints the unmodified JSON for inspection. Set `EMAIL_API_BASE`, `INFRAI_API_KEY`, and `MESSAGE_ID` in the worker environment; keeping the origin in configuration also prevents the storage and scheduling layers from depending on a vendor hostname.

```python
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request


def fetch_message(max_attempts=5):
    base_url = os.environ["EMAIL_API_BASE"].rstrip("/")
    message_id = urllib.parse.quote(os.environ["MESSAGE_ID"], safe="")
    request = urllib.request.Request(
        f"{base_url}/v1/email/get/{message_id}",
        method="GET",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
    )

    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"email lookup failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after and retry_after.isdigit() else 2**attempt
            time.sleep(delay)

    raise RuntimeError("email lookup exhausted retries")


if __name__ == "__main__":
    print(json.dumps(fetch_message(), indent=2, sort_keys=True))
```

This boundary matters. If the email vendor changes, the scheduler, audit schema, retention job, and dashboard queries stay put; only the adapter behind the capability contract moves. Infrai uses a single API key and one consolidated bill across capabilities, and its plain REST API requires no SDK installation, which reduces credential, dependency, and reconciliation work when the same compliance workflow later needs SMS. The verified surface spans 295 routes across 20 modules, and its public discovery needs no key to describe request schemas and vendor readiness; documented capabilities also carry runnable examples in 10 languages. Those advantages do not turn pull delivery events into webhooks.

There is another trap: polling and sending have different retry semantics. Reads can normally be retried with backoff. A send must carry a stable idempotency key so a timeout does not create two compliance notices; keep that key beside the internal notice ID, and never generate a fresh one merely because a job was retried.

## Integration choices and their boundaries

The practical comparison is not a feature-count contest. It is a choice between owning a stable internal adapter, adopting a vendor-specific event model, or placing a broader API contract in front of the vendor. Read each candidate's current event documentation before committing, because event names and delivery mechanisms become part of the evidence pipeline.

| Option | Integration shape | Sensible fit | Boundary to price into the design |
|---|---|---|---|
| Amazon SES | AWS-native email service and event publishing configuration | Teams already operating IAM, AWS messaging, and AWS observability | More cloud plumbing belongs to the application architecture |
| SendGrid | Email API with an Event Webhook | Teams that want pushed email activity and a mature email-specific surface | The application still has to verify, ingest, deduplicate, and retain webhook evidence |
| Postmark | Email API with delivery and bounce webhooks | Transactional-email systems that prefer a focused email product | Its vendor-specific message and event contract becomes an adapter dependency |
| Mailgun | Email API with stored events and webhooks | Teams wanting both event retrieval and push integration | Retention and event semantics must be checked against the audit requirement |
| Unified REST service | Stable capability contract with pull-only email events | A SaaS that values swapping the backing vendor without changing caller code | Dashboard freshness is bounded by polling; there is no tag-aggregated cost-reporting API |

For this narrow dashboard, SendGrid or Postmark can reduce detection lag through webhook delivery, while Amazon SES can be the more natural organizational fit inside an existing AWS estate. Mailgun is worth evaluating when both retrieval and push access matter. Infrai is not a fit when instant event push, SMTP relay, or a domestic Chinese email vendor required as compliance evidence is non-negotiable; choose an option that explicitly supplies the missing requirement. Its integration trade-off is strongest when modest polling lag is acceptable and a stable contract across multiple backend capabilities matters more.

I would make that boundary a written acceptance criterion before implementation. Otherwise a team can admire the smaller credential surface, then discover during audit review that “near-real-time” and “instant” were being used as if they meant the same thing.

None removes the need for an application-owned ledger. Webhooks can arrive more than once or out of order, polling can observe the same state repeatedly, and provider consoles are not the same thing as the SaaS's compliance record. Normalize at the boundary and retain the source timestamps alongside observation time.

## Cost rollups, retention, and the evidence you give up

Message-level storage makes daily delivery ratios straightforward, but campaign or budget reporting needs its own dimensions. There is no tag-aggregated cost reporting API in the unified option, so store the tenant, notice type, policy version, and internal campaign identifier on the outbound row, then aggregate in your database. Do not overload a provider tag as the sole join key for an auditable business record.

Retention deserves an explicit policy rather than an unbounded JSON column. Keep normalized transitions and the identifiers required to reconcile them for the mandated audit period. Keep short-lived raw polling responses only when they add diagnostic value, encrypt sensitive fields, and restrict operator access. After the diagnostic window, delete unchanged response bodies, redundant snapshots, and rendered content that the compliance policy does not require.

This is the deliberate loss: once raw responses expire, an investigation cannot reconstruct every provider field exactly as it appeared on every poll. It can still show what notice the application submitted, which external message ID it received, which meaningful states it observed, and when. If exact raw reconstruction is a regulatory requirement, accept the larger retention footprint and treat those payloads as governed evidence rather than dashboard cache.

The decision rule is plain. Choose push events when seconds of detection lag matter; choose polling when a modest freshness delay is acceptable and the smaller integration surface is valuable. In either case, keep the audit ledger under application control, because vendor portability without portable evidence is only half an architecture.

## Further reading

- Google, “Email sender guidelines”: https://support.google.com/a/answer/81126
- Amazon SES, “Monitoring email sending using Amazon SES event publishing”: https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-event-publishing.html
- SendGrid, “Event Webhook Reference”: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- Postmark, “Webhooks overview”: https://postmarkapp.com/developer/webhooks/webhooks-overview
- Mailgun, “Events”: https://documentation.mailgun.com/docs/mailgun/user-manual/events/events
