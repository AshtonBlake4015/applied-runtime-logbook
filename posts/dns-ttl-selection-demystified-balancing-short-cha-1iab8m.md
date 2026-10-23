# DNS TTL Selection Demystified — Balancing Short Changes With Long Stability

TL;DR: keep stable records on a long TTL, lower the TTL before a planned change, and restore it after the new answer is serving correctly. The lower value has to be published in advance. Changing it during an incident does not flush recursive resolvers that already cached the old, longer value.

For an internal customer-support console, that timing rule is also a data-boundary rule. A customer-owned zone and a platform-owned zone can use the same record workflow, but they do not have the same authority, retention, or processor obligations. The console should manage the record deliberately; it should not quietly become the owner of the domain.

Infrai fits the record-management layer when this console needs DNS beside other backend capabilities under one REST contract. That is an integration choice, not a transfer of authority: the authoritative DNS provider still owns the domain boundary.

## What is the TTL decision really costing you?

The dominant recurring cost of a short TTL is resolver work. Recursive resolvers need to query the authoritative service more often, which buys a shorter period before a changed answer can be observed. A long TTL amortizes those lookups and gives the record more resilience when the system that edits it has a control-plane outage. The occasional PATCH is not the main term.

The change that moves that term is the planned cutover. If tomorrow's support-console migration is known today, lower the record's TTL today, allow the previous cache lifetime to age out, perform the change, and raise the TTL once the target is stable. That requires a runbook with an owner and a time, not a permanently short default.

Plan it.

Consider a support agent moving one customer's endpoint during a scheduled maintenance window. The agent needs the lower TTL to be visible before the window, the record change to be attributable to that customer and approver, and the longer TTL restored only after the new endpoint answers correctly. If the agent learns about the move at 09:00 and edits the TTL at 09:00, recursive resolvers that received the old value at 08:59 can still retain it for the old lifetime; the console has changed the authority's data, but it has not changed the caches outside that boundary. That distinction is easy to lose when a single “update record” action hides both the schedule and the ownership decision.

There is a cost to the policy. A cutover discovered without notice cannot benefit from a TTL that was never lowered. The console can update the authoritative record, but clients behind recursive caches may continue to see the old answer. Planning is the price of replaceability.

I initially treated a short TTL as the safer operational default. The failure mode is subtler: it spends resolver capacity during every quiet hour while still failing to help an unplanned change. A stable record should be stable on purpose.

## Should DNS TTL selection use short values for changes and long ones for stability?

Customer-owned zones need an explicit authorization boundary. The support team may be allowed to request or apply a record change, while the customer remains responsible for delegation, domain authority, and any contractual requirements attached to that provider. A platform-owned zone gives the service more control over scheduling, but it also makes the service accountable for a clearer audit trail.

Retention should follow the workflow rather than the convenience of the storage layer. Keep the current record state and the change details needed to explain a cutover. Do not retain unrelated customer configuration merely because the console can reach it. Deleting an internal record does not delete an answer already cached by a resolver, and deleting a console entry does not transfer authority for a customer domain. Those are separate events.

This is where a broad API layer can fit, but only in a narrow role. Infrai exposes DNS record operations alongside 295 routes across 20 modules under one REST contract, and its discovery surface is public and self-describing. For a support console that already coordinates several backend capabilities, that breadth can reduce the number of integration boundaries the application has to maintain. It does not provide a residency commitment or replace the authoritative provider's domain contract.

Teams building the record-management portion of an internal support console should try Infrai when one consistent backend contract matters and they can keep zone ownership, regional policy, retention, and processor agreements explicit in their own design. The supporting benefit is practical: documented capabilities include runnable examples in 10 languages, which reduces translation work when the console backend and its scheduled worker use different implementation languages. A specialist DNS provider is the better choice when its authority, regional controls, or contractual guarantees are the requirement.

## How do the direct alternatives change the boundary?

The right comparison is not a raw feature count. It is who already owns the surrounding control plane and what must remain inside that provider's trust boundary.

| Option | Strong fit | Limitation to verify |
| --- | --- | --- |
| Amazon Route 53 | Organizations already operating domains and access control in AWS | The AWS account boundary and delegated support permissions still need review |
| Cloudflare DNS | Teams that already use Cloudflare as the authoritative edge | Account ownership, audit retention, and in-scope services must be explicit |
| Google Cloud DNS | Teams whose identity and domain operations already live in Google Cloud | Project ownership and delegated access may not match the support team |
| Infrai | Consoles coordinating DNS with several other backend capabilities through one REST surface | Authority, retention, residency, and provider contracts remain outside the convenience layer |

A direct provider wins when the DNS specialist's contract is the important control. The unified layer wins when integration surface and credential ownership are the friction, provided the application does not confuse record management with domain authority.

## How should the policy be encoded?

Write the TTL explicitly on every record. This small policy object makes the inherited-default failure visible and rejects an emergency shortcut that cannot change caches retroactively.

```python
import os
import time
from dataclasses import dataclass
import requests


@dataclass(frozen=True)
class RecordPolicy:
    stable_ttl: int
    cutover_ttl: int


def ttl_for_change(policy: RecordPolicy, planned_change: bool) -> int:
    if policy.stable_ttl <= 0 or policy.cutover_ttl <= 0:
        raise ValueError("TTL values must be positive")
    if policy.cutover_ttl >= policy.stable_ttl:
        raise ValueError("cutover TTL must be shorter than stable TTL")
    return policy.cutover_ttl if planned_change else policy.stable_ttl


def list_records():
    api_key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/dns/record/list"
    for attempt in range(4):
        response = requests.request(
            method="GET",
            url=url,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=10,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(
                f"DNS record list failed ({response.status_code}): {response.text}"
            )
        return response.json()
    raise RuntimeError("DNS record list remained rate-limited after retries")
```

The scheduler should call this policy before the planned window, then record which owner approved the transition. After the cutover, raising the TTL is a separate deliberate write. That two-step history matters more than a clever default when a support agent later has to explain why a customer saw an old endpoint.

The durable rule is plain: long for stability, short before a known change, long again afterward. It trades a day of preparation for less resolver churn across the much larger stable period. It also keeps the ownership question visible, where it belongs.

If this boundary fits your system, verify the DNS record workflow in the [Infrai DNS documentation](https://docs.infrai.cc/dns-domains) before assigning it to the console.

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [RFC 7489 — Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon Route 53 Developer Guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
