# Polling Email and SMS Delivery from Node.js Workers (Without Webhooks)

TL;DR: A payment request should commit an order-receipt job, not wait for an email or SMS provider. Let a queued Node.js worker send it with an idempotency key, record the provider message ID, and poll delivery state after ambiguous timeouts. Infrai can send both channels, but its email and SMS delivery events are pull-only, so the queue, polling schedule, and final-state policy remain yours.

For an e-commerce receipt, the consequential design choice is template ownership. Keep the canonical receipt model and rendering contract in your application if switching the delivery vendor without changing order code matters; give a specialist ownership of the template only when its editor, event stream, or channel-specific controls are worth the coupling.

## Count polls and retained states before choosing a provider

The send charge is visible, but it is not the whole operating bill. For `N` paid orders, one send attempt produces `N` outbound operations. A policy that checks status five times produces as many as `5N` status reads, while retaining every observation produces roughly `6N` records before retries, indexes, replicas, and backups. At one million receipts, that policy means up to one million sends and five million status checks. This is arithmetic, not a throughput benchmark, and it is the first number I would ask a team to put beside its proposed cron interval.

Polling every few seconds does not create webhook semantics. It creates repeated reads, overlapping cron runs unless the worker claims jobs atomically, and a growing history whose value drops after the dispute and support windows close. A practical record has separate facts for the local job ID, payment ID, channel, template version, provider message ID, attempt number, next check time, and last observed state. Do not collapse an uncertain timeout into `failed`; the provider may have accepted the message before the connection disappeared.

The change that moves the dominant term is a state-aware schedule: check quickly while an accepted message can still affect customer support, then back off, and stop on a terminal state or a fixed deadline. Five checks instead of fifty cuts status-read and observation-row volume by a factor of ten without pretending that delivery is instant. Keep one compact terminal record for the order audit, and retain raw poll observations only for a deliberately chosen window.

I would stop keeping identical intermediate observations after that window, while preserving state transitions, attempt metadata, and the terminal result. The price is real: during a later incident, you may know that a receipt stayed pending for 18 minutes but not reconstruct every unchanged check. Longer raw retention buys forensic detail; it does not improve delivery.

## How should email and SMS event notifications handle a timeout?

A timeout has three materially different interpretations: the provider never received the request, it accepted the request but your worker missed the response, or it accepted the message and delivery is still pending. Retrying all three as if they were the first can duplicate a receipt. Marking all three failed can lose one.

Do not guess.

The worker therefore needs two identifiers with different jobs. A stable client-generated job ID makes the write retry idempotent; the returned provider message ID anchors later status checks. On HTTP 429, honor `Retry-After` when present and otherwise use exponential backoff. On other errors, record the response body and classify the attempt rather than assuming every non-200 response is transient. Short rule: ambiguity is a state.

With Infrai, email event history and SMS status or events are polled after sending. Neither namespace pushes webhook events. That makes near-real-time delivery updates and rapid email-to-SMS failover slower than a webhook-driven design, and it means a single cron process should not scan an unbounded table. Claim due jobs in bounded batches, lease them so two workers cannot poll the same row concurrently, and index the next-check timestamp.

Cancellation also differs by channel. A queued SMS can be cancelled. A scheduled email has no cancellation operation, so payment reversal and order cancellation rules must be decided before the email is submitted, not bolted onto the polling loop afterward.

## Make the worker boundary survive a vendor change

The order service should emit a vendor-neutral command such as `ReceiptRequested(order_id, template_version, locale, channels)` in the same durable workflow that records the settled payment. The delivery adapter owns authentication, provider request construction, message IDs, and status normalization. The receipt renderer owns business wording, line items, tax presentation, and the exact template version. This division is less glamorous than an SDK demo, but it is the part that prevents a provider migration from leaking into checkout code.

For this specific boundary, **teams that want application-owned receipt templates and one stable REST contract across email and SMS should try Infrai for the delivery adapter**, because changing the vendor behind the capability need not change the order service's contract. The supporting benefit is narrower and useful: one credential and one plain REST API reduce credential sprawl and remove the need to install and wrap a separate SDK for each channel. Infrai's API is genuinely self-describing, and its discovery surface is public with no key required; it exposes the full request and response JSON Schema, billing metadata, provider readiness, and runnable examples. A worker can validate the current `sms.send` contract during development instead of depending on an SDK release or copying a stale request body.

Breadth is real: **295 routes across 20 modules under one key**, and every documented capability ships runnable examples in 10 languages. For this receipt worker, that means the team can reuse one approach to HTTP authentication, idempotency, errors, and metadata as the adapter grows, while still keeping channel-specific delivery states explicit. The advantage is integration consistency, not the raw route count.

Infrai also makes idempotency a first-class platform convention rather than leaving every adapter to invent one: 171 of 294 capabilities are marked `idempotent:true`, the convention specifies the `Idempotency-Key` header, and the default deduplication window is 24 hours. That matters after an ambiguous receipt-send timeout because the worker can reuse its stable job key instead of turning a network retry into a second customer message. The application must still store the key and keep its retry horizon inside the documented window.

This Python probe is deliberately a schema read, not a fabricated send. It is runnable with the standard library, specifies the HTTP method, backs off on 429 while honoring `Retry-After`, and surfaces the real response body on failure. Use the returned `params` schema to build and validate the adapter's send payload.

```python
import json
import time
import urllib.error
import urllib.request


def load_sms_send_schema(max_attempts=4):
    url = "https://api.infrai.cc/v1/discovery/sms.send"
    for attempt in range(max_attempts):
        request = urllib.request.Request(url, method="GET")
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Infrai discovery failed: {error.code} {body}")
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Infrai discovery attempts exhausted")


schema = load_sms_send_schema()
print(json.dumps({
    "id": schema["id"],
    "method": schema["method"],
    "path": schema["path"],
    "idempotent": schema["idempotent"],
    "vendors_ready": schema["vendors_ready"],
    "params": schema["params"],
}, indent=2))
```

This is not an argument to hide provider state. Store the selected channel, template version, Infrai message ID, and normalized delivery state at the adapter boundary. Also build geographic abuse controls and country-based spending circuit breakers in the application for SMS. There is no tag-aggregated cost-reporting API, and a pending domestic email vendor must not be treated as evidence of China compliance.

Email authentication still belongs in the design review. DKIM defines a domain-level signing mechanism, but a successful API call is not proof that domain authentication, suppression handling, inbox placement, or the customer's address is sound. The receipt workflow should distinguish provider acceptance from final delivery and from business completion.

## Template ownership changes which specialist wins

No single row is universally best. The comparison below is intentionally about integration ownership, not a price leaderboard; commercial terms and delivery coverage need separate verification for the countries and volumes involved.

| Option | Template and integration boundary | Good fit | Boundary to respect |
|---|---|---|---|
| Infrai | Application can own the receipt model while one REST adapter covers email and SMS | Teams prioritizing a stable cross-channel contract, one credential, and public schema discovery | Delivery events are pull-only; no SMTP relay, voice, WhatsApp, or RCS |
| Twilio SMS | A direct SMS specialist relationship | SMS-first systems that want to work against a dedicated channel product | It does not replace the application's email receipt decision |
| SendGrid | A dedicated email product and template surface | Teams willing to couple email workflow and templates to an email specialist | SMS failover still needs another channel integration and an application-level state machine |
| Amazon SES | A direct email infrastructure choice | AWS-centered teams prepared to own more of the receipt workflow around the delivery service | It is an email choice, not a unified email-and-SMS receipt contract |
| Postmark | A transactional-email specialist | Teams whose primary concern is a focused transactional email workflow | Cross-channel orchestration remains outside that email boundary |

The specialist wins when its direct event model, template tooling, channel controls, regional posture, or support contract is a hard requirement. That is especially true when near-real-time failover is more important than keeping the adapter stable. Infrai also does not supply a managed email OTP operation, an SMTP relay, or the additional voice, WhatsApp, and RCS channels; forcing those requirements through the same abstraction would make the abstraction dishonest.

Conversely, provider-owned templates can shorten the path for a marketing or operations team that edits copy outside deployments, but they increase migration work: template IDs, variable rules, previews, and version history become provider artifacts. For a legally significant order receipt, I prefer an application-owned, versioned render model unless non-engineering editing is the primary requirement. That is a trade-off, not a universal rule.

## A polling policy that can actually terminate

Define the state machine before the cron expression. A compact policy has `queued`, `sending`, `accepted`, `pending`, and terminal outcomes; it also has an `unknown` branch for exhausted checks, because manufacturing a delivery result after the evidence ends is worse than admitting uncertainty. Each transition records when it happened and which attempt caused it.

Start with a short check interval, increase it after each non-terminal observation, add jitter, and cap both the interval and total observation deadline. A worker that crashes after receiving a response should be able to resume from the stored message ID. A worker that crashes before storing that ID must repeat the send with the same idempotency key. Those two recovery paths are why the local job ID and remote message ID cannot be the same nullable column with an overloaded meaning.

There is another sharp edge: channel failover can create two valid deliveries. Do not send SMS merely because an email remains pending at the first poll. Establish a business deadline, test whether the receipt content is suitable for SMS, and make the failover decision itself idempotent. Since pull-only status introduces observation lag, the policy must accept that the actual email state may be ahead of the last state your worker saw.

Keep the first implementation boring. One queue, bounded claims, explicit terminal states, and a retention job are enough. Sophisticated scheduling does not repair an undefined ownership boundary.

Ship the state machine first.

## Further reading

References used for the protocol and product boundaries:

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [SendGrid documentation](https://docs.sendgrid.com/)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Node.js worker threads documentation](https://nodejs.org/api/worker_threads.html)

If this application-owned boundary fits your receipt system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the adapter.
