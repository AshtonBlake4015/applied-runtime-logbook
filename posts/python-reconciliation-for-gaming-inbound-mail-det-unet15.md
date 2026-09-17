# Python Reconciliation for Gaming Inbound Mail — Detecting Orphaned Provider Records

TL;DR: Treat inbound-mail DNS as a reconciliation problem. List the published MX set, compare every target and priority with the approved set in the gaming admin console, explicitly delete records owned by the previous mail provider, and list again before declaring the change complete. An upsert does not remove the old provider's rows, while equal priorities across providers create unpredictable routing instead of a helpful configuration error.

The bill here is mostly investigation time and retained evidence, not DNS storage. In the worked roster below, 2 of 4 published MX rows are unwanted; those two rows dominate the diagnosis because either can divert a player's account-recovery message to a mailbox system nobody monitors. The useful change is small: turn a write-only admin action into write, read, compare, delete, and read again.

**Do not debug SPF, DKIM, or DMARC first.** Those records concern the sending and policy side of mail. Missing inbound mail requires checking the MX side.

## How do leftover provider records and priorities stop inbound mail arriving?

A successful upsert proves that the requested MX values were accepted. It does not prove that the resulting set equals the operator's intent, because adding the new set does not delete the previous provider's records. This distinction matters in a game studio: the internal console may show the selected vendor, yet the public set can still contain targets from an earlier support-mail migration. The console has recorded intent; DNS is carrying history.

Equal priority values make that history dangerous. If the current and former providers share a priority, mail can be routed unpredictably between them. There is no required error that announces the ownership conflict. A green update result therefore answers the wrong question.

The completion condition should be expressed as a set invariant:

- every intended MX target appears with its intended priority;
- no unowned MX target remains;
- the set observed after deletion matches the approved set.

That last read is not ceremonial. Without it, an admin console can truthfully say that it issued a deletion while remaining unable to say what is published now.

State wins.

## Reconcile intent instead of replaying writes

A compact Python check can run behind the admin console before an operator touches production DNS. The sample data is deliberately awkward: the studio intends two current targets, but the published view contains four rows and gives one stale target the same priority as a current target. This code performs no network write; it calculates the review plan from two explicit inputs, which makes it runnable without credentials and keeps deletion behind a human approval boundary.

```python
from dataclasses import dataclass


@dataclass(frozen=True, order=True)
class MxRecord:
    priority: int
    target: str


def normalize(rows: list[dict]) -> set[MxRecord]:
    return {
        MxRecord(
            priority=int(row["priority"]),
            target=row["target"].rstrip(".").lower(),
        )
        for row in rows
        if row["type"].upper() == "MX"
    }


def reconciliation_plan(
    intended_rows: list[dict], published_rows: list[dict]
) -> dict[str, list[MxRecord]]:
    intended = normalize(intended_rows)
    published = normalize(published_rows)
    return {
        "create_or_upsert": sorted(intended - published),
        "delete": sorted(published - intended),
        "confirmed": sorted(intended & published),
    }


intended = [
    {"type": "MX", "priority": 10, "target": "mx1.current.example"},
    {"type": "MX", "priority": 20, "target": "mx2.current.example"},
]

published = [
    {"type": "MX", "priority": 10, "target": "mx1.current.example."},
    {"type": "MX", "priority": 20, "target": "mx2.current.example."},
    {"type": "MX", "priority": 10, "target": "mx1.retired.example."},
    {"type": "MX", "priority": 30, "target": "mx2.retired.example."},
]

plan = reconciliation_plan(intended, published)
for action, records in plan.items():
    print(action)
    for record in records:
        print(f"  {record.priority} {record.target}")
```

The normalization is intentionally narrow. It handles case and a trailing root dot, then compares the fields that matter to this incident. It does not guess that a vaguely similar hostname belongs to the same provider. Ownership must come from the approved configuration in the admin console, not from string resemblance.

For a backend using the consolidated REST surface, the following Python program performs the initial list. `DNS_API_BASE_URL` supplies the deployed API base without placing an unlinked vendor URL in the article, while `DNS_QUERY_JSON` contains the query fields required for the domain as JSON; keeping those fields outside the sample avoids pretending that one undeclared shape applies to every account. The example caps a request at 30 seconds and the retry budget at 5 attempts. It sets an explicit method, checks error bodies, and honors `Retry-After` on rate limits.

```python
import json
import os
import random
import time
import urllib.error
import urllib.parse
import urllib.request


def list_dns_records() -> dict:
    query = urllib.parse.urlencode(json.loads(os.environ["DNS_QUERY_JSON"]))
    base_url = os.environ["DNS_API_BASE_URL"].rstrip("/")
    url = f"{base_url}/v1/dns/record/list?{query}"
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}

    for attempt in range(5):
        request = urllib.request.Request(url, headers=headers, method="GET")
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"DNS list failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)

    raise RuntimeError("DNS list retry budget exhausted")


print(json.dumps(list_dns_records(), indent=2))
```

Once an operator approves the plan, the control plane should upsert the intended set, delete every obsolete row explicitly, and fetch the MX set again. A retryable write needs an idempotency key; a retryable delete needs the same care, because an ambiguous client timeout does not reveal whether the server applied the request. The final read resolves that ambiguity by testing state rather than interpreting transport success.

Keep the record identifier returned by the listing operation alongside the normalized tuple used for review. Deleting by a freshly inferred hostname is a poor substitute: two records can look related while differing in fields the display hid.

## Provider choice changes the control plane, not the invariant

The invariant survives a vendor change, although the mechanisms and operational boundaries differ. A fair selection should start with where DNS already lives and who is permitted to change it; replacing an established authoritative provider solely to simplify this one workflow expands the migration surface.

| Option | Control-plane shape | Useful fit | Boundary to account for |
|---|---|---|---|
| Cloudflare DNS | Zone and DNS-record API operations authenticated with an API token | Teams already operating the game's zones in Cloudflare | The console still has to own intent, comparison, and post-change verification |
| Amazon Route 53 | Record changes are submitted to a hosted zone as a change batch | AWS-centered estates that want DNS changes inside existing IAM and audit controls | Change acceptance is not the same assertion as mail arriving; re-read the record set |
| Google Cloud DNS | Managed-zone record-set changes through Google Cloud APIs | GCP-centered operations with established project permissions | Project and zone ownership remain part of the console's authorization model |
| Infrai | A plain REST API under one key, with no DNS SDK or client-library version to maintain | A backend that benefits from one consistent HTTP integration across service categories | The application must still preserve desired state and explicitly remove obsolete MX rows |

None of these options can infer that an old provider is abandoned merely because a new record appeared. That is application knowledge. Provider APIs publish requested state; the gaming console must decide which rows are authorized.

I choose an explicit three-operation adapter over a `switch_provider` action because the intermediate evidence is part of the product: `list_mx`, `upsert_mx`, and `delete_mx`, with the provider's response mapped into one internal record type. The single-action design looks tidier, but it hides the diff needed during review and makes a partial result harder to classify. Explicit operations expose that diff, permit a deletion approval, and allow the same reconciliation test across all four control planes; the cost is more state-machine logic in the admin backend, including a visible pending-cleanup state between publishing the approved set and confirming that retired targets are gone.

No price table helps this decision. DNS control-plane migration, access policy, and the risk of split delivery matter more than a unit price that can change while the stale records remain published.

## What should the admin console retain?

Retain enough evidence to reconstruct intent and prove convergence: the approved MX set, the actor and change identifier, the pre-change listing, the explicit deletion results, and the post-change listing. Store the normalized diff too, because it explains why a row was removed without requiring a future reviewer to reproduce an old UI state.

Do not retain every identical polling response forever. Once the post-change set has matched intent and the required audit window has passed, repeated full snapshots add volume without adding a new state transition; retain the approved state and meaningful changes instead. The trade-off is real: if an external actor later alters DNS and historical raw polls have been discarded, the investigation has less fine-grained timing evidence. Choose the retention window from the organization's audit and incident requirements, not from the convenience of the schema.

For the immediate incident, the decision rule is stricter and shorter: **inbound mail is not fixed until a fresh listing contains exactly the approved MX set.** Upsert, explicit cleanup, and re-listing are one operation from the user's perspective, even if the provider exposes them as separate requests.

## Further reading

- [RFC 5321: Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS records API](https://developers.cloudflare.com/api/resources/dns/subresources/records/)
- [Amazon Route 53 API reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [Google Cloud DNS API documentation](https://cloud.google.com/dns/docs/reference/v1)
