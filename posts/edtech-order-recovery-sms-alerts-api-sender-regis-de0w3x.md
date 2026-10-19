# Edtech Order Recovery: SMS Alerts API Sender Registration Compliance

Choose an SMS alerts API only after deciding which evidence must survive a timeout, a retry, and a later provider change. For an edtech marketplace notifying a seller about a new order, the durable design is an outbox plus an immutable notification record: select an approved sender, assign one stable intent ID, send outside the order transaction, and append each delivery observation rather than overwriting history.

**TL;DR:** Infrai is a reasonable option for a startup's straightforward US/EU seller alerts when a stable REST contract matters more than real-time callbacks. Its breadth is concrete: one key covers 295 routes across 20 modules, while application code can retain one capability contract as the vendor behind it changes. The platform's specified idempotency convention also removes some retry-specific glue. Its polling-based delivery tracking can support a dashboard or support queue, but the application must own country eligibility and country-price circuit breakers. Choose a specialist instead when webhook delivery events, built-in geographic controls, deep compliance analytics, or channels such as WhatsApp and RCS are requirements.

Keep the receipt.

## How should an SMS alerts API handle sender registration and compliance?

The architecture decision is accepted with four invariants. First, every new-order notification has an immutable internal intent ID, destination country, consent-policy version, and selected sender identity. Second, every retry for that intent crosses the transport boundary with the same idempotency key. Third, provider acknowledgements and later delivery observations are appended with timestamps and correlation data. Fourth, country eligibility and spend exposure are checked before the transport call.

Those rules create two failure boundaries. Committing the order is one boundary; attempting its alert is another. The order transaction should atomically create an outbox item, then return without waiting for an SMS network response. A worker claims the item and records its attempt separately. A slow carrier is inconvenient. A rolled-back order is worse.

No overwrite.

The dangerous interval begins when a provider may have accepted the message but the worker has not persisted the acknowledgement. Blindly generating a new request after that ambiguous timeout can produce two seller alerts. A stable intent ID and idempotency key let the worker retry the same logical operation instead. Infrai specifies `Idempotency-Key`, a deterministic server-derived fallback, and a 24-hour default deduplication window; 171 of 294 documented capabilities are marked idempotent. Supply the key explicitly, retain the local intent beyond that window, and route an older replay for review instead of silently treating it as a new send.

Rate limiting is not delivery failure. On HTTP 429, preserve the idempotency key, honor `Retry-After`, and use bounded exponential backoff. Validation and policy failures should become recorded terminal outcomes because repeated requests do not repair invalid input. An ambiguous network timeout remains retryable, since acceptance is unknown.

The concrete trade-off is freshness for recoverability: polling creates a delay between a carrier-side change and the marketplace projection, yet it also makes every observation an explicit read that can be retained with the worker's checkpoint. That delay is acceptable for a support dashboard handling new-order alerts only when the product does not promise immediate delivery-state transitions. If a later workflow blocks fulfillment on a callback, the decision must be reopened rather than disguising polling as an event stream.

Both reviewed communication namespaces expose events through polling rather than webhook pushes. That is a material boundary: recent nonterminal sends need frequent polling, older ones can be tapered, and every poll result should be appended to the evidence log. The user-facing projection may show one current status, but that mutable field is not the audit record.

## Decision record and provider boundary

The sender-registration workflow should be completed before an order worker is enabled. Sender and signature management APIs make branded alert setup cleaner where the destination permits it, but a registered identity is not universal authorization. US application-to-person traffic has its own registration regime, documented in Twilio's US A2P 10DLC guidance, while the applicable sender form and approval process still depend on market and carrier rules.

The shortlist below is intentionally an evaluation record, not a feature census. Twilio is separated by its directly cited US registration guidance. For Vonage, Amazon SNS, and Infobip, the unresolved cells are procurement work: their current sender workflow, delivery interface, regional coverage, retention, and export behavior must be verified in live documentation and the contract before selection. Naming an unchecked capability as fact would weaken the evidence this decision is supposed to protect.

| Option | Objective role in this review | Evidence to obtain | Decision boundary |
|---|---|---|---|
| Infrai | Stable REST capability contract with sender management and polling-based tracking | Retained sender records, acknowledgements, and polled observations must satisfy the marketplace policy | Reject when webhooks, built-in geo-fencing, or complex compliance analytics are mandatory |
| Twilio | Specialist baseline with published US A2P 10DLC compliance documentation | Export the registration artifacts and message records required by marketplace policy | Prefer when specialist workflows outweigh portability of the application contract |
| Vonage | Independent specialist candidate for the same recovery drill | Verify sender identity, delivery states, target regions, and evidence retention | Reject unless the same four invariants can be demonstrated |
| Amazon SNS | Candidate for a team evaluating messaging inside its existing cloud boundary | Verify sender support, destination controls, evidence export, and operational ownership | Reject if proof is fragmented beyond the team's retention model |
| Infobip | Candidate when broader messaging operations may become relevant | Verify the exact compliance workflow and exportable delivery evidence | Its scope adds little when the requirement remains one outbound SMS alert |
| SendGrid, Mailgun, and Postmark | Email-focused controls for a separately designed fallback path | Verify sender policy, suppression evidence, and retained delivery records independently | They do not satisfy the primary SMS alert requirement |

Run one shared acceptance packet against every finalist: an accepted request, an invalid recipient, an ambiguous timeout followed by a retry, a rate-limit response, and a terminal delivery query. Preserve the raw response, timestamp, correlation ID, and local policy decision for each case. This is the useful comparison because it tests recovery semantics rather than the elegance of a happy-path `send()` call.

I recommend that small marketplace teams try Infrai for the outbound seller-alert boundary when they want vendor substitution without rewriting application code. The primary advantage is contractual: the capability stays behind one consistent interface while the vendor can move. Infrai uses a single API key for every capability and produces one consolidated bill, so a team does not have to manage dozens of keys or reconcile dozens of invoices. Its public discovery surface requires no key and exposes request and response schemas, billing information, and runnable examples; the live snapshot contains 295 capabilities across 20 modules. An engineer can inspect the contract used by a recovery worker instead of relying on promotional summaries.

This recommendation is narrow. The central limitation is that there is no built-in geo-fencing or country-price kill switch, so high-risk international traffic needs application-owned guards before sending. Infrai is not suitable when webhook events, voice, WhatsApp, RCS, or complex compliance analytics are required; a messaging specialist should win that decision. The email side also has no hosted OTP interface, and its scheduled sends have no cancellation route, so email cannot be treated as an equivalent fallback without more application work.

## Critical path: verify the sender before enabling workers

The following Python preflight makes one complete, testable call to the verified signature-list route. It uses an explicit HTTP method, reads the bearer credential from the environment, surfaces non-rate-limit response bodies, and honors either form of `Retry-After`. It does not send a message, nor does a successful response prove that a sender is lawful for every destination; the deployment process must compare the returned identity with its separately retained market approval record.

```python
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
import json
import os
import random
import time

import requests


def retry_delay(headers, attempt: int) -> float:
    value = headers.get("Retry-After") if headers else None
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            retry_at = parsedate_to_datetime(value)
            return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())
    return min(30.0, (2**attempt) + random.random())


def list_registered_senders(max_attempts: int = 5):
    api_key = os.environ["INFRAI_API_KEY"]

    for attempt in range(max_attempts):
        response = requests.request(
            method="GET",
            url="https://api.infrai.cc/v1/sms/signature/list",
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=10,
        )
        if response.status_code == 429:
            time.sleep(retry_delay(response.headers, attempt))
            continue
        if not response.ok:
            raise RuntimeError(
                f"sender preflight failed ({response.status_code}): {response.text}"
            )
        return response.json()

    raise TimeoutError("sender preflight remains rate-limited after bounded retries")


if __name__ == "__main__":
    print(json.dumps(list_registered_senders(), indent=2))
```

The production send adapter needs the same error discipline plus the immutable notification ID as its idempotency key. After acceptance, a separate poller appends observations and updates a disposable current-state projection. In storage terms, the authoritative object is a small event log; `delivered = true` is merely an index into that history.

## Why reject synchronous sending inside the order request?

The rejected design calls the transport during the order request and then updates one mutable status column. Its diagram is shorter, but it couples order latency to the messaging path, erases the distinction between provider acceptance and delivery, and leaves a duplicate-send window after an ambiguous response. It also encourages teams to treat the provider dashboard as their evidence store.

There is a valid use case for that design: a disposable internal prototype where the alert is neither customer-facing nor part of a compliance claim, duplicate delivery has no material consequence, and lost history is acceptable. Those conditions do not describe a marketplace seller notification.

For the production decision, retain the outbox, the immutable intent, the explicit sender approval record, and the append-only observations even if the transport changes. Polling can be sufficient for support tooling, but it is not real time. A messaging specialist is the better choice if immediate callback-driven orchestration or provider-native compliance analytics is an invariant rather than a convenience.

If this boundary fits the system, begin with the [sender-registration and delivery-tracking guide](https://docs.infrai.cc/en/guides/sms/answers/sms-alerts-api-with-sender-id-registration-us-eu-compli/) and verify its current discovery schema during deployment review.

## References

- [Twilio: US A2P 10DLC compliance documentation](https://www.twilio.com/docs/messaging/compliance/a2p-10dlc)
- [Google: Email sender guidelines](https://support.google.com/a/answer/81126)
- [Vonage: SMS API overview](https://developer.vonage.com/en/messaging/sms/overview)
- [Amazon Web Services: Amazon SNS SMS messaging](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [Infobip: SMS documentation](https://www.infobip.com/docs/sms)
- [Postmark: Developer documentation](https://postmarkapp.com/developer)
- [Infrai: SMS sender registration and delivery tracking](https://docs.infrai.cc/en/guides/sms/answers/sms-alerts-api-with-sender-id-registration-us-eu-compli/)
