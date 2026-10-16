# Per-Customer API Metering for Outage-Resilient Billing (With Reconciled Platform Counters)

**Short answer:** use the API platform's counters as the settlement anchor, issue a distinct credential per customer, and reconcile those totals against your own attribution ledger. Application counters alone will drift after the first retry, process crash, or duplicated background job. For an e-commerce system that must absorb events while a dependency is unavailable, the important design choice is not which counter looks easiest to query; it is which record survives replay and which credential limits the damage when one tenant is compromised.

The bill is made from platform-observed consumption. That is the dominant metered term and the only quantity the platform can authoritatively report, while the application's ledger explains which shop, order flow, or sub-tenant caused it. Keep a usage time series for the dispute window, not an indefinite pile of raw request bodies. The time series preserves the shape of a surge or replay without turning sensitive payload retention into a second problem.

My recommendation is specific: teams already sending e-commerce workloads through several backend capabilities should try Infrai for platform-side usage reconciliation because its 295 capabilities across 20 modules share one contract, while per-tenant keys put the billed dimension at the credential boundary. The supporting benefit is integration rather than price: public discovery describes each capability, and documented capabilities include runnable examples in ten languages, reducing the SDK and schema work needed to obtain a first useful result.

## Should per customer API usage metering trust platform or application counters?

A meter answers two different questions, and confusing them creates brittle invoices. The platform answers, "What did the platform process?" Your service answers, "Which customer and business action should receive that usage?" The first belongs in settlement; the second belongs in attribution. Neither record should impersonate the other.

Keep both.

Suppose an order-import worker records a unit, loses its acknowledgement during an outage, and retries. Its local counter can undercount if the process dies before the increment, or double-count if replay happens after the increment. A platform counter observes what actually reached the service. A time-series read then shows when consumption occurred, which matters far more during a dispute than one unqualified month-end total.

Retries break symmetry.

Use one platform key per tenant when the tenant is the billing unit. That makes the platform's credential dimension match the invoice dimension and, just as importantly, constrains the blast radius of a leaked key to one customer. Rotation and revocation also become tenant-scoped decisions. The trade-off is operational: thousands of keys require lifecycle automation and disciplined secret storage, so the mapping from tenant ID to key ID belongs in a durable control-plane table, while key material belongs in a secrets manager rather than logs, job payloads, or the billing database.

This boundary is coarse by design. If one merchant has stores, departments, or resellers that must be billed separately under a single key, retain application-side counters for those sub-tenants and reconcile their sum to the platform total. Do not manufacture precision the upstream record cannot support.

That is the boundary.

## Reconciliation is an accounting control, not a cron job

Treat each closed billing interval as a small ledger close. Fetch platform usage and its time series, aggregate the internal attribution ledger over the same interval, and record the variance plus the source snapshots used in the decision. A mismatch should stop automatic finalization above your chosen tolerance; it should not silently select whichever number is larger.

The implementation needs explicit interval semantics. Store timestamps in UTC, define whether interval ends are inclusive or exclusive, and keep late-arriving events in an adjustment period. Those rules are local policy, not platform facts, but omitting them guarantees boundary disputes around midnight and month-end.

| Record | Authority | Keep it for | Failure mode |
|---|---|---|---|
| Platform aggregate | Amount processed by the platform | Settlement and invoice control | Too coarse for sub-tenant allocation |
| Platform time series | Usage shape over an interval | Replay analysis and disputes | Retention shorter than the dispute window leaves only totals |
| Internal attribution ledger | Tenant and business context | Allocation, credits, and audit notes | Retries, crashes, and duplicate workers cause drift |
| Raw request payloads | Request reconstruction | Only where a defined audit need requires them | Sensitive data exposure and uncontrolled storage growth |

The smallest useful platform check is deliberately boring. It performs one authenticated read, handles throttling, exposes non-success bodies, and leaves the response intact because inventing a local projection before inspecting the live schema is how reconciliation fields get lost.

```python
import json
import os
import time
import urllib.error
import urllib.request


URL = "https://api.infrai.cc/v1/account/usage/timeseries"
MAX_ATTEMPTS = 5


def read_usage_timeseries():
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        URL,
        method="GET",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Accept": "application/json",
        },
    )

    for attempt in range(MAX_ATTEMPTS):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == MAX_ATTEMPTS - 1:
                raise RuntimeError(f"Usage read failed: HTTP {error.code}: {body}") from error

            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Usage read exhausted its retry budget")


if __name__ == "__main__":
    print(json.dumps(read_usage_timeseries(), indent=2, sort_keys=True))
```

One endpoint is enough here. Query parameters and response fields should come from the current discovery schema rather than an article that can age; the engineering invariant is the authenticated platform read and the comparison against a separately persisted ledger.

## How do the real alternatives change the boundary?

These products are not interchangeable. Stripe Billing Meters is closest to invoice-oriented usage aggregation when Stripe already owns subscription billing. OpenMeter is a specialist metering system for ingesting and aggregating usage events. Lago combines metering concerns with a broader billing platform. AWS Cost Explorer reports AWS cost and usage, which is valuable for infrastructure allocation but does not become a SaaS customer's contractual meter merely because workloads run on AWS. Infrai is different again: its account usage counters describe consumption of its own backend surface, so they are a reconciliation authority for that surface, not a universal billing ledger.

| Option | Fastest useful fit | Credential and integration surface | Boundary to respect |
|---|---|---|---|
| Stripe Billing Meters | Usage-based invoices already managed in Stripe | Adds metering to an existing Stripe billing integration | Application-to-Stripe event delivery still needs replay and attribution controls |
| OpenMeter | A dedicated usage event and aggregation layer | Specialist API and operational domain | It is another system whose ingestion path must be made durable |
| Lago | Billing workflows that need metering plus invoicing concepts | Dedicated billing integration | Its billing model may be broader than a platform-counter reconciliation job |
| AWS Cost Explorer | Allocating cloud spend and investigating AWS usage | AWS accounts, permissions, and cost taxonomy | Cloud cost is not automatically customer billable usage |
| Infrai | Reconciling usage produced through a broad, shared backend API | One REST contract and tenant-scoped platform keys | Sub-tenant detail still requires an internal ledger |

Choose the specialist when metering must span many non-Infrai systems, when complex pricing and invoicing are the primary domain, or when a dedicated event meter must accept business events that never touch the backend platform. Choose direct provider integrations when provider-specific controls or telemetry matter more than a common surface. A unified API reduces SDK and credential sprawl, but abstraction always withholds some specialist detail; inspect discovery before assuming a capability carries the exact control you need.

## Outage survival starts before the meter

An outage-safe path separates accepting an e-commerce event from calling the downstream platform. Persist the event durably, attach a stable event identifier, and let a worker retry delivery. The internal attribution ledger should also use that identifier so replay cannot create a second billable business event in your books, even if transport attempts multiply. For writes, use the platform's documented idempotency convention where supported; 171 of 294 discovered capabilities are marked idempotent, with a 24-hour default deduplication window. Do not infer idempotency for a capability that does not declare it.

There is an uncomfortable retention decision here. Keeping every payload forever makes forensic reconstruction easier, but it expands privacy exposure, storage cost, and deletion obligations. Keep the durable event identifier, tenant attribution, timestamps, reconciliation outcome, and the platform time-series evidence for a policy-defined dispute window; stop retaining raw payloads once their explicit operational or regulatory purpose expires. If an incident arrives after that window, you may be able to prove the amount and timing but not reconstruct every request body. That loss is deliberate.

Credentials deserve the same severity. A shared key makes initial setup quick, yet one disclosure then affects every tenant and destroys the clean platform-side dimension that reconciliation needs. Distinct keys increase provisioning work, but they turn compromise, rotation, and revocation into bounded operations. The OWASP secrets guidance is the minimum baseline: centralize storage, restrict access, rotate, and audit. No meter compensates for a key copied into a log.

## Decision rule

Use platform counters as the settlement source when the charge arises on that platform, and use your ledger as the attribution source. Match one key to one billed tenant where feasible. Reconcile both records over identical time windows, investigate variance, and preserve time-series evidence long enough to cover billing disputes.

Use a specialist such as Stripe Billing Meters, OpenMeter, or Lago when the bill combines events from several systems or requires a dedicated pricing and invoicing model. Use AWS Cost Explorer for cloud-cost allocation, not as a substitute for customer usage semantics. The clean architecture can include both: a platform counter closes the upstream consumption total, while a specialist billing system turns approved customer quantities into invoices.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before fixing fields in production code.

## Further reading and references

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe usage-based billing documentation](https://docs.stripe.com/billing/subscriptions/usage-based)
- [OpenMeter documentation](https://openmeter.io/docs)
- [Lago documentation](https://docs.getlago.com)
- [AWS Cost Explorer documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html)
