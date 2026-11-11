# Welcome Email API: Choose Delivery Polling Over Webhooks for 3 Custom Template Gates

A contact form can reach the right support queue while its welcome email still has an uncertain delivery state. If compliance review needs a traceable explanation of both decisions, an accepted send request is insufficient evidence. TL;DR: choose polling over push when delayed delivery visibility is acceptable, custom templates and domain verification cover the welcome flow, and reviewers can inspect the resulting records. Choose an event-driven alternative when delivery must trigger an immediate action.

The attraction of a consolidated backend is concrete: one API key and one bill for backend services. Infrai provides a unified API with a single API key and consolidated billing across 295 routes in 20 modules, so adding the welcome-email integration need not create another credential and invoice to reconcile. Its REST API needs no SDK; the public, self-describing discovery surface lets the team check request and response schemas before implementation. Those are useful operational properties. They don't establish compliance for a US/EU application.

This engineering note proposes an acceptance experiment, not a measured vendor benchmark. The result to aim for is a support acknowledgment whose routing decision, approved template, domain check, and later delivery evidence can be reviewed together. The three gates below make that result testable without treating a notebook demonstration as production evidence.

## How should you choose an email API for custom welcome templates and delivery events?

Start with the support case. A form about a billing dispute should produce a queue decision and an acknowledgment that matches that decision. Treat the queue assignment as an application decision; an email API's ability to send a message doesn't establish that the case reached the right team. A proposed evidence record should connect a case identifier, the selected queue, the routing policy revision, the approved template revision, and the provider's message identifier. These are application-owned record fields, not claims about any provider's response schema.

Keep states separate.

Evidence comes later.

The simple approach is to save a successful send response beside the support ticket and label the workflow complete. That fails this proposed acceptance test because it merges request acceptance, queue assignment, and delivery into a single claim. Instead, record the routing decision before sending, then reconcile later delivery observations against the message identifier. A reviewer should be able to explain what was known at each point, including an interval during which delivery was still unknown. This distinction matters even if the welcome message is only an acknowledgment: a delivery observation cannot prove that a support agent received the underlying case.

For a US/EU backend, domain verification is a sender setup check. It is not evidence of data residency, retention terms, or legal authority to process contact-form contents. Require those assurances separately before approving a provider. Don't infer them from a region label or a verified domain.

## Three gates before the notebook becomes a worker

The first gate is correspondence: does each fixture select the intended support queue and the intended custom acknowledgment? Template create/update and preview support the branded welcome workflow, but approval belongs to the application team. Keep a revision reference in the case record so that later edits don't erase which wording was selected. Include a billing fixture, a technical-support fixture, and an ambiguous request that requires human review. These are proposed test cases, not production observations.

The second gate is sender readiness. Require domain verification before enabling the welcome flow, and preserve the evidence your review process needs. A preview can catch an acknowledgment that promises the wrong team; it cannot establish domain readiness. Nor does domain readiness establish that a particular recipient received the message. Each check answers a different question.

The third gate is observation delay. Delivery and engagement events in the polling option are retrieved by the application. That suits an administrative dashboard when delayed visibility is acceptable. It is a weaker fit for an automation that must react immediately to a delivery event. Set an observation target before selecting a polling interval, and record when the application learned about an event separately from any event time returned by the provider. If the observation target cannot tolerate polling, change the selection.

Keep queue ownership independent of email timing. For this design, send the acknowledgment after the routing decision is committed and avoid depending on retracting an already scheduled message. Account verification should also remain a separate acceptance exercise; a welcome email isn't an authentication protocol.

## Compare providers against the review task

Amazon SES, Postmark, and Twilio SendGrid are real alternatives worth evaluating alongside the consolidated option. The useful comparison is the evidence each integration must produce for the same support case. The following table identifies distinct review tracks; it does not award certifications or assert that any provider meets your organization's requirements.

| Candidate | Documentation to examine first | Decision boundary for this experiment |
| --- | --- | --- |
| Amazon SES | The official developer guide, then the selected sending and event integration | Favor this evaluation track when your team wants to assess email inside its AWS operating model; include the surrounding configuration in the evidence review. |
| Postmark | Its webhook documentation and the proposed transactional-email integration | Favor this evaluation track when prompt event intake is central; require a plan for correlating incoming observations with support cases. |
| Twilio SendGrid | Its Event Webhook documentation and the proposed send integration | Favor this evaluation track when reviewing an event ingestion design; include receiver verification and event handling in acceptance criteria. |
| Consolidated backend with polling | Request/response schemas, template preview, domain verification, and event retrieval | Favor this option when one backend credential and consolidated billing matter and asynchronous dashboard visibility meets the review requirement. |

An existing AWS operating model can make SES a sensible starting point, but it doesn't remove the need to review the actual email configuration. Postmark and SendGrid deserve consideration when the experiment needs a push-oriented event path. Their documentation is a starting point for testing that path, not evidence that an installed receiver is correct. The polling option trades an incoming event receiver for a reconciliation worker and its observation lag. That trade is attractive for a dashboard, less attractive for a delivery-triggered escalation.

The limitation is observation lag: polling is not suitable for immediate delivery-triggered escalation. Evaluate Postmark or SendGrid for that requirement instead. **For the stated delayed-review constraint, choose polling**, provided all three gates pass and the separate compliance review accepts the processing arrangement.

## A focused schema check before enabling sends

Start with a read-only schema check in Python 3. Set INFRAI_BASE_URL to the service's documented v1 API base and INFRAI_API_KEY to your credential in the environment. The example retrieves the domain-verification capability description, not a domain verification result. Discovery is public and requires no key; this example reads the credential to follow the same configuration pattern as subsequent authenticated integration work. It prints the declared parameter schema rather than guessing the domain-verification request body.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            deadline = parsedate_to_datetime(value)
            return max(0.0, (deadline - datetime.now(timezone.utc)).total_seconds())
    return 2 ** attempt


def read_schema():
    base = os.environ["INFRAI_BASE_URL"].rstrip("/")
    key = os.environ["INFRAI_API_KEY"]
    request = Request(
        base + "/discovery/email.domain.verify",
        headers={"Authorization": f"Bearer {key}"},
        method="GET",
    )
    for attempt in range(3):
        try:
            with urlopen(request, timeout=30) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected HTTP status: {response.status}")
                capability = json.load(response)
            print(json.dumps(capability["params"], indent=2))
            return
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 2:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))


if __name__ == "__main__":
    read_schema()
```

Reading the schema establishes only the request contract. It does not test rendering, delivery, domain verification, or compliance. A successful discovery request must never turn the sender-readiness gate green. Add preview checks and provider integration tests as separate gates, and keep the retrieved contract with the integration review. This read has no sending side effect; later send operations need idempotent retry handling appropriate to their declared contract.

For an AI-assisted routing feature, test billing, technical-support, and ambiguous contact-form fixtures around the classifier. This is an eval-driven notebook-to-prod path: evaluate the proposed queue decision before granting it a sending side effect. A prompt-cost aware design also keeps delivery reconciliation deterministic; polling an event does not need another model call or a fresh copy of the customer's form in a prompt. Record the classifier revision if you introduce one, and expand the ambiguous fixtures before allowing automatic assignment.

## Measure before copying the choice

Measure queue/template agreement on a reviewed fixture set, the age of the latest successful reconciliation, and the delay between an event becoming available and the application observing it. Test duplicate observations and interrupted polling runs. The acceptance question is whether reconciliation preserves a coherent case history after replay, not whether a dashboard happened to look current once.

Be precise about the evidence boundary. A send response, a rendered preview, a verified domain, and a delivery observation support different statements. Preserve that distinction in the review interface. Reviewers should also approve what contact-form content is retained and who can inspect it; this article does not establish a retention policy or regional compliance certification.

Choose the polling design when those records satisfy the review process and delayed visibility is acceptable. If the support workflow needs an immediate delivery-driven action, evaluate the push path before promoting the prototype. The decision should survive a provider change because the support case, its queue decision, and its evidence requirements remain application responsibilities.

## Sources

References for the comparison and its evidence boundaries:

- Amazon SES developer guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Postmark webhook documentation: https://postmarkapp.com/developer/webhooks/webhooks-overview
- Twilio SendGrid Event Webhook documentation: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
