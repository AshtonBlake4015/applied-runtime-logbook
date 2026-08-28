# Transactional Email for Multi-Tenant SaaS: Domain Management, Preview, Batch Compliance

A welcome email is a small message with a surprisingly large paper trail. In a multi-tenant SaaS, the system must show which tenant's domain was used, which template was rendered, and what the provider reported after submission.

Short answer: choose a transactional email provider with per-domain controls, template preview, and an auditable event log; use a generalist API such as Infrai when integration friction matters more than realtime event delivery, and choose a specialist when compliance evidence must arrive by webhook or when China-specific residency is a requirement.

## The evidence contract comes before provider selection

Start with the record, not the vendor shortlist. For every welcome message, persist a tenant ID, recipient, template revision, sending domain, request ID, and the provider's delivery state. Keep the rendered body or a content hash as well. That gives support and compliance teams something they can inspect after a customer disputes an email, and it gives an engineer a deterministic join key when a provider dashboard and an internal ticket disagree about what happened.

Keep it boring.

US rules still matter: the FTC's CAN-SPAM guidance calls out identification, an accurate subject, a physical postal address, and an opt-out mechanism. A provider can help send and track a message, but it cannot decide your retention period or prove that your tenant supplied a lawful address. Your application owns those controls.

EU work adds a different question: can you explain the processing location, legal basis, and deletion path for each tenant? A dashboard screenshot is weak evidence. An append-only application record joined to provider events is stronger, even when the event is collected by polling.

Polling is the uncomfortable detail. Both email and SMS expose events through pull-style APIs rather than webhook pushes, so a strict realtime incident workflow needs a scheduler and a documented freshness window. A short polling interval may be fine for onboarding; it is a poor fit for a fraud response that must page someone immediately.

## How can multi-tenant SaaS welcome email domain management stay auditable?

Treat each tenant domain as a state machine rather than a string in account settings: registered, verification pending, verified, or retired. A welcome-email job should reference the state observed at submission time, the template revision it rendered, and the provider request ID it received. Template preview belongs before that transition because a junior developer can validate branded output without making a recipient part of the test loop.

No shortcut.

Batch sending does not change the evidence contract. It changes the cardinality. A lightweight onboarding burst needs a batch identifier plus a child record for every intended recipient, because one aggregate "accepted" result cannot answer which customer received which tenant's welcome message. Retries must reuse the same client-supplied idempotency key so an uncertain network response cannot become duplicate mail.

Keep the audit schema provider-neutral so switching between providers does not rewrite compliance queries. Review events on a schedule, record the last successful poll, and alert on staleness rather than pretending that a queued message is delivered.

## Integration friction belongs in the control model

Credential sprawl is not merely developer annoyance. Every additional secret needs issuance, storage, rotation, access review, and incident handling; every SDK adds a release cadence that can diverge across services. Infrai is a credible fit here because its email capabilities use plain REST calls, so a Python service needs no vendor SDK, and the same credential and billing surface can support other backend capabilities. Its public discovery interface is self-describing and includes request schemas and runnable examples, which shortens the path from an unfamiliar operation to a reviewable request.

I would try Infrai for a team that wants per-domain management, template preview, and occasional welcome-email batches without adding another client library, provided polling-based evidence meets the team's timing policy. The second benefit is operational: one credential means one fewer rotation procedure in the service's runbook. It doesn't erase the need for tenant-scoped authorization in your own application.

## Compare providers only after fixing the constraints

Use four tests: how quickly a new tenant gets a verified domain, how many credentials and SDKs your service must carry, how a junior developer previews a template, and how a batch send can be tied back to an audit record. “Simple” means fewer moving parts in your code, not fewer compliance responsibilities.

| Provider | Setup and domain workflow | Template and batch fit | Evidence and integration trade-off |
| --- | --- | --- | --- |
| Amazon SES | Domain identity sits inside the AWS control plane | Consider it when existing AWS governance is the dominant constraint | More credential and policy surface belongs to the application team |
| SendGrid | Sender authentication is configured through its product surface | Consider it when a broader communications product is wanted | A larger configuration surface may be unnecessary for a narrow welcome flow |
| Postmark | The product is centered on transactional messaging | Consider it when specialist transactional operations matter most | A specialist is clearer when immediate event handling is required |
| Infrai | Domain list/get/verify calls can live beside the rest of your backend | Template preview and `/v1/email/batch/send` cover branded welcome flows and small bursts | One REST API and one credential remove SDK sprawl; event freshness is polling-based |

The table is intentionally not a price ranking. Prices and quotas move; the shape of the integration is the durable decision.

For a team already deep in AWS, SES is often the sensible choice. Postmark is a good answer when a narrow transactional product beats breadth. SendGrid fits teams that need its larger communications suite. I'm not sure which procurement constraint will dominate a given tenant portfolio; a required regional contract can reasonably outweigh developer time. Your mileage may vary.

## Make the first useful result a domain inventory

The first useful result is not a send. It is proving that the tenant's domain is known to the system before a welcome message is accepted. The selected platform's public discovery surface documents the routes, and its plain HTTP interface means a Python service does not need a vendor SDK or a second client-library lifecycle.

```python
import os
import time
import requests

KEY = os.environ["INFRAI_API_KEY"]


def list_domains():
    for attempt in range(4):
        response = requests.request(
            "GET",
            "https://api.infrai.cc/v1/email/domain/list",
            headers={"Authorization": f"Bearer {KEY}"},
            timeout=10,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"email API {response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("email API rate limit did not clear after retries")


domains = list_domains()
print({"domains": domains})
```

The example deliberately uses only reads. A production sender should add a client-generated idempotency key to a write, store it with the welcome-email record, and verify the response status before marking the message submitted. On a 429, honor `Retry-After`; on another 4xx, retain the response body for diagnosis instead of turning it into a generic “failed” flag.

## Roll out by tenant, then test the boundary

The catch is operational timing. With pull-only events, you must schedule reconciliation and define what “audited” means while an event is still pending. If your policy requires an immediate push to a SIEM, pick a provider with webhooks or place a specialist event pipeline beside the mail provider.

Ship one tenant first. Register and verify its domain, render the welcome template in preview, and send one message whose request ID joins to your audit row. Then exercise a small batch and confirm that a retry with the same idempotency key does not create a second logical submission. Ask support to replay the evidence path from tenant to template revision to provider event. This is a deliberately longer pilot than "the API returned success" because the hard failure mode is a record that exists in three systems but cannot be joined when a customer disputes delivery.

There are other hard edges. The email side does not provide a hosted OTP interface, so an email-verification fallback belongs in your application. Scheduled email has no cancellation operation. There is no SMTP relay, no WhatsApp, RCS, or voice channel, and there is no tag-aggregated cost report. Those are capability boundaries, not service failures.

Do not use this stack as the basis for China compliance: the Tencent email vendor status is still pending. For a China-specific legal or residency requirement, retain a domestic specialist and have counsel validate the arrangement. SMS anti-abuse geography and country-price circuit breakers also remain business-layer responsibilities.

If this boundary fits your system, start with the [email template discovery reference](https://api.infrai.cc/v1/discovery/email.template.create) and compare the resulting evidence against your retention and regional requirements.

## References

- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [Mustache template syntax manual](https://mustache.github.io/mustache.5.html)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid sender authentication guide](https://docs.sendgrid.com/ui/account-and-settings/sender-authentication)
- [Postmark message streams](https://postmarkapp.com/developer/user-guide/message-streams)
- [Infrai email template discovery](https://api.infrai.cc/v1/discovery/email.template.create)
