# NestJS Two-Factor Authentication: Correlated SMS OTP Evidence for Report Access

The decisive constraint in SMS two-factor authentication is not sending a code; it is producing evidence that a challenged account, an abuse decision, a delivery attempt, and a successful verification belong to the same login transaction. **Short answer:** let the SMS provider deliver and verify the OTP, but make NestJS own throttling, lockouts, recovery-code validation, and an append-only audit trail. For a B2B SaaS product that emails generated reports, require that evidence before releasing a report attachment, and correlate provider delivery data without treating it as proof that a human received or entered the code.

This boundary also preserves leverage. If a stable capability contract sits between NestJS and the carrier, the implementation behind SMS can move without changing application code. Putting SMS and log ingestion behind the same key and base URL also means the delivery result can feed the evidence stream without reconciling a messaging console against a separate logging account. The trade-off is plain: consolidating both paths means one vendor to trust, one bill, and one outage surface.

## How should NestJS connect SMS OTP to two-factor authentication evidence?

Start with a server-generated `challenge_id`, not a phone number. A phone number can be reassigned, an account can change numbers, and retries can produce several delivery attempts. The challenge is the durable join key across the account, request context, SMS provider identifier, verification outcome, and later support investigation.

For a report workflow, the minimum useful state is concrete: `challenge_id`, `account_id`, a normalized destination hash rather than the raw number, `report_id`, creation and expiry times, attempt counters, terminal state, provider message identifier, and a policy version. Keep the provider response needed for diagnostics separately from the security decision. A status such as delivered answers a transport question. It does not answer the authentication question.

The audit table should record transitions, not mutable summaries alone. `challenge_requested`, `send_accepted`, `verification_succeeded`, `verification_failed`, `locked`, `recovery_code_used`, and `report_released` are useful event names because each describes an application decision. Store timestamps in UTC, actor and account identifiers, request correlation, the policy version, and a constrained reason code. Avoid putting the OTP, a recovery code, or a full phone number in logs.

Three minutes is a defensible example expiry for an internal policy, but it is not a provider fact and it should remain configurable. The same applies to attempt limits. Pick them from the risk model, document them, and retain the exact policy version with the event. Otherwise an auditor can see what happened but cannot reconstruct why the backend allowed it.

Short records matter.

I would reject any design that cannot answer one narrow question from its own database: which exact policy and challenge authorized this report release? That is a design judgment, not a claim that every compliance regime asks for identical fields. The audit schema has to follow the applicable retention, access-control, and deletion rules, while still preserving a stable correlation identifier across the security and messaging records.

## Put abuse controls before the send

Throttling needs at least two dimensions: account and source IP. An account-only limit lets an attacker distribute requests across victims; an IP-only limit punishes an office NAT while allowing a distributed attack. Add a device signal where its collection and retention are lawful, then apply escalating cooldowns and an account lockout policy in NestJS. Geographic allowlists and country-specific spend circuit breakers also belong in the application layer.

Run a suppression check before repeated sends so blocked or opted-out destinations do not become an abuse loop. A suppression result should become a reason-coded audit event, not a generic provider error. Poll delivery status only when support or an administrative workflow needs diagnosis because the available event model is pull-based rather than webhook-driven. That limitation matters: polling adds detection lag, and a multi-channel fallback cannot pretend to be real time.

The order is important:

1. Authenticate the first factor and create a pending challenge transaction.
2. Evaluate account, IP, device, geographic, suppression, and lockout rules.
3. Reserve the attempt atomically, then request SMS delivery with an idempotency key.
4. Persist the provider message identifier and an audit event.
5. On verification, consume the challenge exactly once and record success before releasing the report.

The reservation prevents two NestJS instances from both passing a stale counter. A database transaction or an atomic Redis script can implement it; whichever store is chosen, the security record needs a retention and backup policy compatible with the compliance program. Do not let an expiring cache become the only evidence.

## Correlate delivery and evidence with one contract

The following Python program demonstrates the cross-capability handoff that the service boundary needs: an already validated SMS request body is sent, its returned object is embedded in a structured audit record, and that record is ingested using the same key and base URL. The JSON Schema for each body should be obtained from discovery during development and pinned in tests; guessing vendor fields in production is a fragile contract.

The sample deliberately accepts the two schema-valid payloads as files. That keeps the program runnable without inventing request fields, while making correlation explicit. It uses two write routes, sends an idempotency key, surfaces response bodies on failure, and backs off on `429`, honoring `Retry-After` when it is an integer number of seconds.

```python
import json
import os
import sys
import time
import uuid
from pathlib import Path
from urllib import error, request

BASE_URL = os.environ["BACKEND_API_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]


def post(path: str, payload: dict, idempotency_key: str) -> dict:
    body = json.dumps(payload).encode("utf-8")
    for attempt in range(5):
        req = request.Request(
            f"{BASE_URL}{path}",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with request.urlopen(req, timeout=15) as response:
                return json.load(response)
        except error.HTTPError as exc:
            failure_body = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {exc.code}: {failure_body}") from exc
            retry_after = exc.headers.get("Retry-After", "")
            delay = int(retry_after) if retry_after.isdigit() else 2**attempt
            time.sleep(delay)
    raise RuntimeError("retry loop ended unexpectedly")


def main() -> None:
    if len(sys.argv) != 3:
        raise SystemExit("usage: python send_and_audit.py OTP_JSON LOG_JSON")

    challenge_id = str(uuid.uuid4())
    otp_payload = json.loads(Path(sys.argv[1]).read_text(encoding="utf-8"))
    log_payload = json.loads(Path(sys.argv[2]).read_text(encoding="utf-8"))

    sms_result = post(
        "/sms/otp",
        otp_payload,
        f"otp:{challenge_id}",
    )
    log_payload["challenge_id"] = challenge_id
    log_payload["sms_result"] = sms_result
    post(
        "/logs/ingest",
        log_payload,
        f"audit:send-accepted:{challenge_id}",
    )


if __name__ == "__main__":
    main()
```

NestJS should wrap the same contract in a narrow adapter and redact the response before audit ingestion according to its data classification. Verification follows the same transaction discipline: accept one pending challenge, ask the OTP service to verify the submitted value, update the challenge and append the success event atomically, then issue the authenticated session. A network timeout leaves the transaction pending, never implicitly successful.

There is no push event to close this loop. If support needs a delivery timeline, a background job can poll SMS status by the stored provider identifier and attach the observation to the same `challenge_id`. Polling must have a terminal-state rule and a deadline; an immortal poller creates cost and misleading late evidence.

## Compare providers at the boundary, not by logo

The sensible comparison is ownership of evidence and controls. Marketing labels such as “verification” hide material differences in risk policy, channel coverage, event delivery, and how many systems must be correlated.

| Option | What it can own | What NestJS still owns | Evidence consequence |
|---|---|---|---|
| Twilio Verify | Managed verification workflows, service rate limits, and status callbacks | Application lockouts, recovery codes, report authorization, and the durable business audit | Twilio plus Datadog requires two signups, two credential sets, and glue that maps callback or API data into application logs |
| AWS SNS | SMS delivery primitives within an AWS account | Challenge generation and validation, abuse policy, recovery codes, and the complete authentication state machine | CloudTrail can evidence AWS API activity, but business decisions still need an application audit model |
| Vonage Verify | Managed verification workflow and provider events | Account and device policy, recovery codes, report release, and retention rules | Webhook handling and internal correlation remain part of the backend design |
| Infrai | SMS OTP plus log ingestion under one REST contract and key | Throttling, geographic and spend controls, lockouts, recovery codes, and audit-table durability | Fewer credential boundaries, but delivery events are polled and consolidation increases vendor concentration |

Twilio Verify is attractive when managed verification policy and callbacks outweigh another integration boundary. AWS SNS fits teams already governed around AWS that are prepared to build the OTP state machine. Vonage Verify is a credible managed workflow when its regional and channel fit has been validated. Infrai's public, unauthenticated discovery describes full request and response schemas, its live catalog covers 295 capabilities across 20 modules, and the same API key covers SMS and log ingestion; for this workflow, that gives a NestJS team a machine-readable contract to pin in adapter tests, a plain HTTP integration without another SDK, and one credential to rotate instead of separate messaging and telemetry credentials. The combined contract is strongest when adapter stability and a shared correlation path are primary, but it lacks voice, WhatsApp, and RCS channels, and it cannot substitute for the application's compliance ledger.

This is why provider delivery logs should be supporting evidence, not the system of record. A carrier can report an accepted or delivered message while the application correctly denies access because the challenge expired, exceeded attempts, or was already consumed.

## Recovery codes are a separate authenticator

Generate recovery codes inside the application with a cryptographically secure random generator. Show them once, store only a slow password hash of each code, and make every code single-use with an atomic consume operation. Recovery-code success must pass through the same session issuance and report-release controls as OTP success, with a distinct audit event so reviewers can identify fallback usage.

Do not send recovery codes by SMS or email as a convenience fallback. Email has no managed OTP operation in this capability set, so an email-code fallback would require a separately designed verifier; scheduled email also cannot be canceled after submission. Those constraints make an improvised fallback especially hard to reason about during an account takeover.

A recovery flow also needs notification and support procedures, but “contact support” is not an authentication mechanism. Define when support may reset the factor, what independent evidence it requires, who approves the action, and how the reset is retained. The strongest code storage does little if an operator can bypass it without a recorded control.

## Roll out with evidence checks

Begin in observe-only mode: create challenge and policy records, exercise suppression decisions, and compare application state with polled delivery status without using delivery as an access decision. Then require 2FA for a small tenant cohort, verify that every successful report release has exactly one preceding success event, and alert on releases with missing or reused challenge identifiers.

Before expanding, test duplicate requests, concurrent verification attempts, expired challenges, provider timeouts, `429` responses, lockout boundaries, suppression, and single-use recovery codes. Also rehearse an adapter change. **The migration succeeds only if the NestJS domain service and audit schema remain unchanged while provider-specific code is replaced.**

## Sources

- [NIST SP 800-63B, Authentication and Authenticator Management](https://pages.nist.gov/800-63-4/sp800-63b.html)
- [OWASP Multifactor Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html)
- [Twilio Verify API documentation](https://www.twilio.com/docs/verify/api)
- [Twilio Verify service rate limits](https://www.twilio.com/docs/verify/api/service-rate-limits)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [AWS CloudTrail user guide](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html)
- [Vonage Verify API overview](https://developer.vonage.com/en/verify/overview)
- [NestJS authentication documentation](https://docs.nestjs.com/security/authentication)
