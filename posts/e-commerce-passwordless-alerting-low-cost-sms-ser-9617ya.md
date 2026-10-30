# E-commerce Passwordless Alerting: Low-Cost SMS Service Retention Rules

**Short answer:** For a reliable low-cost passwordless SMS alert service, choose the route that exposes regional delivery evidence and supports deduplication, then retain only a compact event ledger rather than every message body.

E-commerce support queues need SMS for the few events that cannot wait for an inbox: a passwordless backup alert, an account-change notification, or a contact form that needs human ownership. The low-cost design is not the one with the lowest message rate. It is the design that sends only an authorized alert, records enough evidence to explain delivery, and deletes message content on a schedule.

The bill is made of delivery attempts, carrier or route surcharges, and the storage and operational work created by retries and audit records. The first term normally dominates. The useful change is to suppress duplicate sends before they reach a provider, while retaining a compact delivery event after the content expires. That improves reliability and retention discipline together.

A support queue has two destinations: the customer who submitted the form and the internal queue that must take ownership. Those are different notification policies. A customer-facing passwordless backup alert should contain no reusable secret. An internal queue alert can contain a case identifier, but not a copied contact-form body.

## What makes a low-cost passwordless SMS alert service reliable?

Start with a message ledger, not a provider dashboard. For each notification, retain an opaque event ID, a hashed recipient identifier, the purpose, creation time, current state, provider-neutral attempt IDs, and the last error class. Retain the rendered body only for the shortest period needed to investigate an active support case; that period belongs in policy, not in an accidental database default.

This matters when a contact form contains an address, order reference, or free-form complaint. A delivery record can prove that an alert was attempted without becoming a second customer database. Consent evidence also belongs in the ledger when optional notifications are involved. GDPR Article 7 describes conditions for consent, including the ability to demonstrate consent, so the record should identify the notification purpose and consent version rather than storing a vague `true`.

The retention trade-off is concrete:

| Data | Operational value | Retention consequence |
| --- | --- | --- |
| Event ID and timestamps | Reconcile attempts and queue ownership | Keep for the audit period |
| Recipient hash and route region | Detect duplicate sends and investigate routing | Keep with access controls |
| Message body | Reproduce an exact rendering | Expire quickly, then delete or isolate |
| Provider response metadata | Classify retryable failures | Keep only fields needed for diagnosis |
| Raw provider payload | Fast debugging during an incident | Avoid long-term retention |

Deleting the body has a cost. A support engineer may need to ask the customer to restate the issue, and a forensic review may have less context. I accept that cost because an indefinitely retained copy of every account notification creates a larger exposure than a missing convenience field. A compact event trail is the compromise.

## How do you prevent a contact form from becoming a message storm?

Reliability begins before the SMS API. The form service should create one notification intent with a stable idempotency key, such as a server-generated event ID, and place it on a durable queue. A worker claims the intent, applies recipient and purpose policy, then sends through an adapter. If the worker crashes after acceptance but before the local state changes, the adapter should reuse the same idempotency key where the selected route supports it; otherwise the system should mark the outcome as uncertain rather than pretending a second send is safe.

`unknown` is not a failure class that should be retried forever. It needs a bounded reconciliation path, a visible queue-age metric, and an escalation rule for an operator.

The route adapter is deliberately boring. It accepts a destination, a message, and an idempotency key, then returns normalized metadata.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Protocol


@dataclass
class DeliveryResult:
    state: str
    attempt_id: str | None
    error_class: str | None


class SmsAdapter(Protocol):
    def send(self, destination: str, body: str, key: str) -> DeliveryResult:
        ...


def dispatch_alert(ledger, adapter: SmsAdapter, event: dict) -> DeliveryResult:
    existing = ledger.find_by_key(event["idempotency_key"])
    if existing and existing["state"] in {"accepted", "delivered", "unknown"}:
        return DeliveryResult(existing["state"], existing["attempt_id"], None)

    ledger.record_attempt(event["event_id"], datetime.now(timezone.utc))
    result = adapter.send(
        destination=event["destination"],
        body=event["body"],
        key=event["idempotency_key"],
    )
    ledger.record_result(
        event["event_id"], result.state, result.attempt_id, result.error_class
    )
    return result
```

Retry transient route failures with a deadline, not a fixed count detached from the business event. A passwordless backup alert that arrives thirty minutes late may be harmful even if transport eventually reports success. Permanent recipient failures should stop quickly. Rate limits should apply per account, recipient, IP, and event purpose; OWASP's forgot-password guidance also calls for uniform responses, expiring tokens, and abuse defenses. SMS is an alert channel, not proof of identity.

## Which route differences change the decision?

A comparison is useful only after the message contract is stable. Twilio, Vonage, and Telnyx all expose SMS APIs, but their sender registration, regional routing, status callbacks, retry behavior, and account controls are not interchangeable. Those differences belong behind the adapter and in route tests, not in business logic. Their documentation describes each service's API and delivery-status model; none of those models should be treated as a universal guarantee of handset delivery.

The useful comparison is operational:

| Question | Why it affects support alerts | Evidence to collect |
| --- | --- | --- |
| Can the route return a stable attempt identifier? | Links a queue event to an external result | Response and callback documentation |
| Are US and EU sender rules different? | Registration and sender choice can change deliverability | Country-specific requirements |
| Can callbacks be authenticated and replay-protected? | Prevents a forged status from closing a case | Signature and replay tests |
| What are the documented filtering and throughput limits? | An accepted message can still be delayed or filtered | Route limits and status definitions |
| Can data residency and deletion be controlled? | Processing may cross the application boundary | Data-processing documentation |

Do not compare advertised unit prices before answering those questions. A route that needs repeated manual intervention has a cost absent from a rate card, while precise status events may reduce duplicate sends and investigation time. The decision axis is delivery reliability against a notification deadline and a known retention policy.

## What does a reliable rollout measure?

Test the complete path with synthetic recipients in each target region. Record enqueue latency, time to accepted, callback latency, terminal status, duplicate rate, and the age of unresolved `unknown` events. Test deliberate failures: a malformed destination, a throttled route, a callback replay, a worker restart after acceptance, and an expired passwordless token. A green API response is not enough.

Keep dashboards split by purpose. A support-queue alert can tolerate a different delay than an account-security alert, and aggregation hides the failure that matters. Alert on rising duplicate attempts, increasing queue age, missing callbacks, and regional divergence. Store hashes and classifications in metrics labels, never raw phone numbers or message text.

A deployment should also prove that the retention job works. Create a disposable event containing a marker, verify the ledger has only permitted fields after expiry, and verify that the body cannot be retrieved through the support console. This catches a common failure mode: database cleanup succeeds while a log, dead-letter queue, or analytics export keeps the original payload.

The practical rule is narrow: send a minimum-content alert through a replaceable adapter, make duplicate prevention a property of the event ledger, and preserve delivery evidence after the message body is gone. That gives an e-commerce team a defensible way to compare routes across the US and EU without turning a provider choice into an architectural dependency.

## Further reading

- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- GDPR Article 7, Conditions for consent: https://gdpr-info.eu/art-7-gdpr/
- Twilio Messaging API documentation: https://www.twilio.com/docs/messaging/api
- Vonage SMS API documentation: https://developer.vonage.com/en/messaging/sms/overview
- Telnyx Messaging API documentation: https://developers.telnyx.com/docs/messaging
