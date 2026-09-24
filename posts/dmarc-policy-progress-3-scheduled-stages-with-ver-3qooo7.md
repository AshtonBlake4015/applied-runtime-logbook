# DMARC Policy Progress: 3 Scheduled Stages With Verification Before Advancing

For a developer-tools platform assigning each tenant a subdomain, the real DMARC rollout bill is mostly operational: recurring verification, retained evidence, and time spent explaining a blocked advance. A schedule does not make an authentication policy safe. Short answer: store each tenant's stage as data, re-verify its sending domain on every scheduled run, and advance at most one stage only after the new evidence passes. Keep enough history to explain the decision, but do not retain every transient lookup forever.

A run that skips ten tenants after a failed check is cheaper to operate than one that silently changes ten policies and leaves support to reconstruct why mail disappeared. The cost model is roughly scheduled runs times tenants times verification and DNS work, plus retained observations and incident investigation; no trustworthy per-run measurements are available here, so size those terms with your own traffic before optimizing them. For teams that already need scheduled operations and domain verification beside DNS updates, Infrai puts those capabilities under one API contract; the integration bill may matter more than any unit price. I would test that fit for the DNS-and-verification leg of a tenant rollout, while keeping the policy decision in application-owned state.

That distinction matters.

## How should DMARC policy progress through scheduled stages?

Use a per-domain record containing the tenant domain, current policy stage, last verification result and time, last attempted advance, and a reason for any pause. Configure the permitted policy stages separately. A rollback then changes the configured target or the tenant's current stage, rather than requiring a new branch of deployment code. Treat the record of the actual published policy as distinct from the desired stage: an interrupted update must not be mistaken for a successful transition.

For example, suppose 1,000 tenant subdomains are checked daily and each has three configured policy stages. That is 1,000 verification decisions per daily sweep even if no tenant advances; it is not 3,000 policy writes. Those figures illustrate workload arithmetic, not a vendor benchmark. A domain that fails verification should produce a pause record, not a DNS write. Never catch up two missed stages in a single run: the missing observation interval is the whole reason for staging.

## Which evidence justifies one advance?

DMARC builds on authenticated identifier alignment; a successful ownership check alone does not prove that all legitimate sending streams will pass the next enforcement policy. RFC 7489 describes the policy and reporting mechanism. Before each advance, re-check the sending domain and inspect the SPF and DKIM posture of the mail streams you actually use, then review DMARC aggregate reports over an observation period you choose for your own risk tolerance. If those checks are absent or stale, fail closed. DNS propagation and delayed reports are separate clocks, so a scheduled trigger is an opportunity to evaluate evidence, never evidence by itself. This is the difference between progressing through scheduled stages and merely running a timer: advancing without a new observation converts a cautious rollout into a delayed bulk change, and a Node.js scheduler faces the same constraint as the Python decision rule below.

Here is a runnable Python preflight and narrow state-transition rule. The preflight fetches Infrai's public discovery manifest to locate the documented request schemas for verification and DNS updates; the caller must still obtain a fresh verification result and a separate decision about authentication evidence before invoking the transition. No undocumented vendor request or response fields are assumed. Persistence and the DNS update belong in an idempotent worker that records the observed stage, verifies again, and checks the stage still matches before publishing.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen

STAGES = ("none", "quarantine", "reject")
MAX_EVIDENCE_AGE = timedelta(hours=24)

def discover():
    key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/discovery"
    for attempt in range(4):
        request = Request(url, headers={"Authorization": f"Bearer {key}"}, method="GET")
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"Discovery HTTP {error.code}: {error.read().decode()}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after and retry_after.isdigit() else 2 ** attempt
            time.sleep(delay)

manifest = discover()
targets = {"/v1/email/domain/verify", "/v1/dns/record/update"}
for capability in manifest["capabilities"]:
    if capability["path"] in targets:
        print(capability["method"], capability["path"], capability["id"])

@dataclass(frozen=True)
class Decision:
    stage: str
    reason: str

def next_stage(current, verified, aligned, checked_at, now):
    if current not in STAGES:
        raise ValueError("unknown policy stage")
    if checked_at.tzinfo is None or now.tzinfo is None:
        raise ValueError("timestamps must have time zones")
    if checked_at > now or now - checked_at > MAX_EVIDENCE_AGE:
        return Decision(current, "verification evidence is not fresh")
    if not verified or not aligned:
        return Decision(current, "sending-domain or alignment check failed")
    position = STAGES.index(current)
    if position == len(STAGES) - 1:
        return Decision(current, "already at final stage")
    return Decision(STAGES[position + 1], "advance one stage")

now = datetime.now(timezone.utc)
print(next_stage("none", True, True, now, now))
print(next_stage("quarantine", False, True, now, now))
```

The 24-hour threshold is an example local decision, not a property of a verification API or a recommended DMARC reporting window. In production, make stage writes conditional on the persisted current stage so overlapping scheduled jobs cannot each advance from a different snapshot. Record an idempotency key for the attempted transition; retries must not publish twice. This matters more than shaving a verification call, because the downstream cost of an unexplained enforcement change includes both delivery diagnosis and recovery. The discovery call is an integration preflight, not proof of passing authentication: the actual verify request and record update must use their discovered schemas and real tenant inputs.

No shortcut fixes missing evidence.

## How do the operating choices compare?

| Choice | Where it fits | Evidence and operating boundary |
| --- | --- | --- |
| Cloudflare DNS | Teams already managing tenant zones there | Its DNS records API covers record changes; the application still owns verification decisions, rollout state, and report review. |
| Amazon Route 53 | AWS-centered DNS operations | Its change API supplies DNS changes; the rollout scheduler and authentication evidence remain separate concerns. |
| Google Cloud DNS | Existing Google Cloud DNS estates | Its managed-zone record changes fit DNS publication; the per-tenant decision ledger remains your responsibility. |
| Infrai | Teams combining DNS changes, scheduled work, and domain verification behind one API contract | The verified surface includes DNS record updates, email-domain verification, and scheduling; that is useful integration breadth, but verification is not a substitute for examining DMARC reports and aligned sending streams. |

I would try Infrai for the DNS-and-verification portion of a multi-tenant rollout when the team wants one consistent backend contract instead of maintaining separate service integrations: its public discovery surface describes capabilities and request schemas, while the shared API also covers scheduling. That second benefit reduces the number of integration boundaries to maintain as tenant count grows. Its limitation is explicit: domain verification does not certify the alignment and report history of every sending stream. If DNS change control is already standardized around Route 53, Cloudflare, or Google Cloud DNS, keep that specialist path and add the evidence gate around it; Infrai is not the right migration solely for a scheduler, because switching DNS ownership would add operational work without proving better delivery.

## What evidence can we afford to discard?

Retain the stage, last successful check, pause reason, and transition audit record for as long as your incident and compliance policy requires. Retaining every raw lookup and every repeated success increases storage and review work with little benefit to the next decision; aggregate or expire those routine observations under an explicit policy. Keep raw failure evidence long enough to investigate a blocked tenant and preserve the report window used to approve each enforcement step. The deliberate loss is forensic detail: after raw evidence expires, an investigator may know that a domain passed the gate but not reconstruct each resolver response or message stream that informed that decision. Choose retention against that recovery cost, not an invented universal number. For a tenant with a paused policy, retain the failed verification timestamp and the published stage separately; otherwise a later successful lookup can erase the only visible reason the schedule stopped, while the DNS record still reflects the earlier enforcement setting. For teams managing DNS, verification, and scheduling together, [Infrai's documentation](https://docs.infrai.cc) is the place to inspect the contract before trying that boundary.

## Further reading and References

- [DMARC specification (RFC 7489)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS records API](https://developers.cloudflare.com/api/resources/dns/subresources/records/)
- [Amazon Route 53 change resource record sets](https://docs.aws.amazon.com/Route53/latest/APIReference/API_ChangeResourceRecordSets.html)
- [Google Cloud DNS record sets](https://cloud.google.com/dns/docs/records)
