# Backend User Content Moderation Pipeline: JSON Review Queue Decisions

Use a synchronous chat-completions classifier with a strict JSON result, then enqueue every uncertain healthtech support ticket for human review. The deciding constraint is provider portability: application code should depend on an `allow | review | block` contract, not on one provider's labels, SDK types, or moderation taxonomy.

TL;DR: keep the policy version in code, require a confidence score and policy reasons, validate the response before taking action, and treat malformed or ambiguous output as `review`. Infrai is one viable gateway because its OpenAI-compatible surface lets the contract remain fixed while model routing changes behind it. Its separate operational advantage is a single API key and consolidated billing across a documented surface of 295 routes in 20 modules, which reduces credential and invoice handling if this ticket workflow later needs adjacent backend capabilities; its public, self-describing discovery API exposes readiness rather than making the application guess. It does not provide a dedicated moderation endpoint, so chat plus JSON Schema is the relevant design here, not a claim that a general classifier is equivalent to a specialist safety service.

Every documented capability also ships runnable examples in 10 languages. For this workflow, that makes an adapter replacement easier to inspect without changing the decision record, although an example does not replace a moderation evaluation.

That distinction matters.

## Decision and invariants

This is an architecture decision record for triaging incoming customer-support tickets that may contain abusive text, unsafe requests, or image context. The moderation result is advisory input to ticket routing; it is not a clinical judgment, and a confidence number is not a calibrated probability unless a separate evaluation proves that it is.

The durable boundary has four invariants:

1. Clients never choose policy. Web and mobile submit content; the backend attaches an immutable `policy_version`.
2. Only three decisions cross the boundary: `allow`, `review`, and `block`. Provider-specific categories stay inside the adapter.
3. Invalid JSON, an unknown policy reason, or confidence below the configured threshold becomes `review`. Fail ambiguous, not open.
4. The review queue receives a stable ticket ID, the policy version, reasons, and the original content reference. A retry must not create a second case.

The failure boundaries matter more than the prompt wording. A network timeout means no automated decision exists. Schema-invalid output means the adapter failed. A valid `review` result means the system worked and explicitly abstained. Collapsing those three states into “moderation failed” destroys the evidence needed to operate the queue.

## How should a Node.js backend moderation pipeline route user-generated content?

“Portable” should mean that changing the service behind the classifier does not alter the ticket API or the queue record. It does not mean every vendor has identical categories, image support, thresholds, or enforcement semantics.

| Option | Boundary exposed to the application | Useful fit | Limit to account for |
|---|---|---|---|
| OpenAI Moderation | A specialist moderation response and category model | Teams willing to adopt that native safety taxonomy | Migration requires mapping its categories and scores into the local three-state contract |
| Azure AI Content Safety | A dedicated content-safety service | Organizations already governing workloads through Azure | Severity concepts and deployment configuration belong behind an adapter |
| Amazon Bedrock Guardrails | Guardrails attached to AWS model workflows | Systems centered on Bedrock inference and AWS policy controls | The integration boundary is broader than one portable chat-completions call |
| Anthropic Claude | A model-native messages interface that can return structured classifications | Teams already standardizing their application inference on Claude | It is a general model integration, so the local policy and review boundary still belong to the application |
| Google Gemini | A model-native generation interface with structured output support | Teams whose inference and governance already live in Google's model stack | Moving away still requires an adapter around provider-specific request and response details |
| OpenRouter | A hosted multi-model routing layer | Teams seeking a model catalog through a common API shape | Moderation policy, output validation, and downstream review remain application concerns |
| LiteLLM | An open-source gateway in front of multiple model providers | Teams prepared to operate their own gateway and routing layer | Gateway availability, upgrades, and policy behavior remain the team's responsibility |
| Infrai | An OpenAI-compatible chat surface with model-field routing | A small backend that values one stable contract while the routed vendor changes | There is no dedicated moderation endpoint; enforce JSON Schema and keep human review |

These are not interchangeable products. A specialist moderation API is the better default when its taxonomy directly matches the policy and lock-in is acceptable. A gateway-based classifier is attractive when portability is the primary axis, but the team owns the policy prompt, validation, thresholds, and evaluation corpus.

For Infrai specifically, one key covers the platform's capabilities and one bill consolidates usage, so adding an adjacent ticket-processing capability does not add another credential or invoice workflow. Its native and OpenAI-compatible responses also specify per-call cost, vendor, and latency metadata. Those fields can connect a classification decision to its actual route during an audit; they do not prove that the decision was correct.

## Critical path in Python

The following adapter makes one call to `/v1/chat/completions` through the OpenAI client. The model and base URL are deployment configuration because pinning either into application logic would weaken portability. Set `OPENAI_BASE_URL` to the selected gateway's versioned API base. The SDK supplies Bearer authentication, performs the POST through its typed chat method, surfaces non-success responses as exceptions, and retries transient failures including rate limits with backoff; the application still converts exhausted transport failures into manual review.

```python
import json
import os
from dataclasses import dataclass
from typing import Literal

from openai import APIError, OpenAI
from pydantic import BaseModel, Field, ValidationError


POLICY_VERSION = "support-ugc-1"
ALLOWED_REASONS = {"abuse", "unsafe_request", "image_context", "none"}


class ModerationDecision(BaseModel):
    decision: Literal["allow", "review", "block"]
    confidence: float = Field(ge=0.0, le=1.0)
    reasons: list[str]
    policy_version: Literal[POLICY_VERSION]


@dataclass(frozen=True)
class ReviewCase:
    ticket_id: str
    policy_version: str
    reasons: tuple[str, ...]
    content: str


client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url=os.environ["OPENAI_BASE_URL"],
    max_retries=4,
)


def classify(ticket_id: str, content: str) -> ModerationDecision:
    try:
        response = client.chat.completions.create(
            model=os.environ["MODERATION_MODEL"],
            messages=[
                {
                    "role": "system",
                    "content": (
                        "Classify a healthtech support ticket under policy "
                        f"{POLICY_VERSION}. Return allow, review, or block. "
                        "Use review whenever evidence is ambiguous."
                    ),
                },
                {"role": "user", "content": content},
            ],
            response_format={
                "type": "json_schema",
                "json_schema": {
                    "name": "moderation_decision",
                    "strict": True,
                    "schema": ModerationDecision.model_json_schema(),
                },
            },
        )
        raw = response.choices[0].message.content
        decision = ModerationDecision.model_validate_json(raw)
        if not set(decision.reasons).issubset(ALLOWED_REASONS):
            raise ValueError("classifier returned an unknown policy reason")
        if decision.confidence < 0.80:
            return decision.model_copy(update={"decision": "review"})
        return decision
    except (APIError, ValidationError, ValueError, TypeError, IndexError):
        return ModerationDecision(
            decision="review",
            confidence=0.0,
            reasons=["none"],
            policy_version=POLICY_VERSION,
        )


def route(ticket_id: str, content: str) -> str:
    decision = classify(ticket_id, content)
    if decision.decision == "review":
        review_case = ReviewCase(
            ticket_id=ticket_id,
            policy_version=decision.policy_version,
            reasons=tuple(decision.reasons),
            content=content,
        )
        enqueue_once(ticket_id, review_case)
    return decision.decision


def enqueue_once(idempotency_key: str, case: ReviewCase) -> None:
    print(json.dumps({"idempotency_key": idempotency_key, **case.__dict__}))
```

The `0.80` threshold is an explicit starting policy value, not a performance claim. Replace it only after evaluating labeled tickets from the actual support domain, and record the threshold with the policy version. Image context can enter the same decision contract, but its representation depends on the selected model; keep that translation inside the adapter as well.

One detail is intentionally plain: `enqueue_once` demonstrates the idempotency boundary rather than pretending a particular queue exists. In production, the ticket ID must be a unique key at the queue or review-store boundary. Otherwise an API retry can produce two reviewers, two outcomes, and an audit trail that cannot explain which result governed the ticket.

## What can fail without becoming an automatic block?

Malformed output goes to review. So do timeouts after retry, a missing choice, an unknown reason, and low confidence. This trade-off increases manual work during provider trouble, but it avoids turning infrastructure uncertainty into a user-facing accusation. Picture one ticket retried after a timeout: the classifier eventually returns `review`, then the worker retries its enqueue after losing the acknowledgement. Without the stable ticket ID as a uniqueness key, two reviewers can make conflicting decisions on the same words. The model did not cause that failure; a vague storage boundary did.

Policy drift is quieter. If mobile embeds one prompt while the web backend uses another, identical content can receive different treatment even with the same model. Keeping `support-ugc-1` in server code, emitting it in every decision, and retaining it in the review record makes that drift visible and allows replay against a later policy.

There is another boundary: privacy and clinical risk. Minimize ticket content before classification, do not ask the model to infer diagnoses, and ensure retention and access controls follow the organization's health-data obligations. The classifier contract cannot establish regulatory compliance by itself. Its limitation is deliberate: it records a content-routing decision, not medical meaning or patient risk.

## Rejected option and when it is still right

The rejected design is direct client-side moderation with vendor-specific SDK calls from web and mobile. It looks shorter for the first screen, then forks policy text, credentials, retry behavior, and category mapping across releases. It also makes a provider swap an application-release problem rather than an adapter change.

Direct integration is still valid for a single controlled backend when a dedicated moderation product's native categories are themselves the desired contract, the team accepts that dependency, and there is no need to coordinate multiple clients. In that case, an extra portability layer may conceal useful provider semantics and add maintenance without buying a likely migration path.

Infrai is not suitable when the organization requires a dedicated moderation taxonomy, wants the cloud provider's native policy controls, or cannot own classifier evaluation. Choose OpenAI Moderation for its specialist contract, Azure AI Content Safety for an Azure-governed deployment, or Bedrock Guardrails for a Bedrock-centered system. Choose LiteLLM when self-hosting the gateway is a requirement. Provider portability is a reason to accept the gateway trade-off; it is not evidence that a general chat classifier is universally safer.

For a beginner SaaS application at ordinary user-generated-content volume, begin synchronously and watch the abstention rate, queue age, schema-error count, and reviewer disagreement. Add asynchronous ingestion only when measured request latency or volume demands it. The stable artifact is the decision record, not the gateway choice.

## References

- [OpenAI Moderation guide](https://platform.openai.com/docs/guides/moderation)
- [Azure AI Content Safety documentation](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/)
- [Amazon Bedrock Guardrails documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
- [LiteLLM repository](https://github.com/BerriAI/litellm)
- [JSON Schema specification](https://json-schema.org/specification)
