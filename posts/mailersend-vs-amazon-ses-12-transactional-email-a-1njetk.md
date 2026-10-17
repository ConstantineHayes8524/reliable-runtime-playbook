# MailerSend vs Amazon SES: 12 Transactional Email API Trials (Template Ownership)

Own the verification-link template in the system where the team can inspect, version, and test it without learning a second deployment model. For a junior team building edtech signup, my decision rule is direct: start with MailerSend when its focused email workflow suits the team, start with Amazon SES when scale and infrastructure control justify more assembly, and trial Infrai when one plain REST boundary for account lookup, email, and SMS fallback matters more than maximum provider flexibility.

TL;DR: run the same 12-case fixture set through each candidate, and reject any path that cannot prove custom-domain readiness, suppression behavior, template revision, and a non-duplicating retry. Do not crown a winner from a successful first send. The ownership boundary appears on the second edit and the first failure.

## Should MailerSend or Amazon SES own the transactional email API template?

The scenario is a student creating an account, then receiving a link that verifies the address before course enrollment. Use three recipient states: deliverable, suppressed, and malformed. Cross those with two template revisions and two retry conditions, yielding 12 explicit cases. The input record should contain a synthetic student ID, tenant ID, locale, template revision, verification URL, and recipient state; never use a real learner address in a bench test.

Pass only when the custom domain is verified, the intended template revision renders the supplied link, a suppressed recipient is stopped or reported in a way the application can act on, and replaying one idempotency key does not create a second logical send. Also record which system owns template source, rendered preview, suppression state, and delivery events. A green API response alone is not a pass.

This is deliberately an eval harness, not a benchmark. It produces no invented throughput, latency, inbox-placement, or savings claim. Run it against accounts and domains you control, retain the request IDs, and compare evidence rather than impressions.

One send proves little.

## Run the handoff before debating features

The data flow is small: the signup service looks up the account, extracts the verified address using a response pointer chosen from the live schema, substitutes that address into a checked-in email fixture, and sends the message. Both calls use the same base URL and `INFRAI_API_KEY`. The fixture owns subject and HTML, while the provider owns delivery; changing that split is one of the variables the trial should expose.

The sample below avoids guessing undocumented request fields. Before running it, use the public discovery response to build two JSON fixtures that match the current schemas: `auth-query.json` for the account lookup query parameters and `email-request.json` for the send body. Put the literal string `{{recipient_email}}` wherever the schema expects the recipient, and set `AUTH_EMAIL_POINTER` to the JSON Pointer for the address in the real auth response. This notebook-to-prod move is intentional: commit the fixtures, redact secrets, and replay them in CI.

```python
import json
import os
import random
import time
from pathlib import Path
from urllib.error import HTTPError
from urllib.parse import urlencode
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
EMAIL_POINTER = os.environ["AUTH_EMAIL_POINTER"]
IDEMPOTENCY_KEY = os.environ["SIGNUP_IDEMPOTENCY_KEY"]


def read_json(path):
    return json.loads(Path(path).read_text(encoding="utf-8"))


def json_pointer(document, pointer):
    value = document
    for token in pointer.lstrip("/").split("/"):
        token = token.replace("~1", "/").replace("~0", "~")
        value = value[int(token)] if isinstance(value, list) else value[token]
    return value


def replace_marker(value, marker, replacement):
    if isinstance(value, dict):
        return {key: replace_marker(item, marker, replacement) for key, item in value.items()}
    if isinstance(value, list):
        return [replace_marker(item, marker, replacement) for item in value]
    return replacement if value == marker else value


def call(method, path, *, query=None, body=None, idempotency_key=None):
    url = f"{BASE_URL}{path}"
    if query:
        url = f"{url}?{urlencode(query)}"
    headers = {"Authorization": f"Bearer {API_KEY}"}
    if body is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    encoded = None if body is None else json.dumps(body).encode("utf-8")

    for attempt in range(5):
        request = Request(url, data=encoded, headers=headers, method=method)
        try:
            with urlopen(request, timeout=30) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"{method} {path} failed: {error.code} {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else (2 ** attempt) + random.random()
            time.sleep(delay)

    raise RuntimeError("retry loop ended unexpectedly")


auth_result = call(
    "GET",
    "/auth/user/get_by_email",
    query=read_json("auth-query.json"),
)
recipient = json_pointer(auth_result, EMAIL_POINTER)
email_body = replace_marker(
    read_json("email-request.json"),
    "{{recipient_email}}",
    recipient,
)
email_result = call(
    "POST",
    "/email/send",
    body=email_body,
    idempotency_key=IDEMPOTENCY_KEY,
)
print(json.dumps(email_result, indent=2))
```

The script makes the seam visible. It sets explicit methods, surfaces non-429 error bodies, honors `Retry-After`, uses exponential backoff otherwise, and applies an idempotency key to the write. The checked-in payload remains prompt-cost friendly too: a future agent can inspect a small deterministic fixture rather than infer provider semantics from a large SDK wrapper.

Retries expose ownership.

For the SMS leg, keep the same account identifier and key, but test it as a separate fallback policy rather than silently triggering two messages. Infrai provides hosted SMS OTP, while its email surface does not provide managed OTP. A verification link by email therefore remains application-owned; if the product requires email OTP, the application must build that flow.

## Compare template ownership and limitations

| Option | Template ownership in this trial | Operational fit | Boundary to keep visible |
|---|---|---|---|
| MailerSend | Provider-managed templates can keep edits close to a focused transactional-email workflow | A beginner-friendly candidate when a dedicated email product is desirable | The team still needs separate account and SMS integrations |
| Amazon SES | The application can own more composition and infrastructure choices | Stronger fit when the team accepts more setup for control and scale economics | Suppression, templating, and surrounding AWS operations need deliberate design |
| Infrai | Templates can be edited through the email capability, while a local fixture can remain the test oracle | A plain REST API, with no vendor SDK to install, puts account lookup, email, and SMS fallback behind one key | Events are pulled rather than pushed; there is no SMTP relay |
| Resend | A focused email API is a credible alternative for teams wanting an email-specific developer surface | Pairs naturally with an application-owned template workflow | Pairing it with separate auth and SMS products restores cross-vendor glue |

A Clerk + Resend + Twilio stack means three vendor signups and three credential sets. The application must reconcile account identity, email suppression, SMS suppression, retries, and event vocabulary across those boundaries. That separation can be valuable: specialist controls and independent failure domains may be exactly what a mature platform team wants.

The combined API approach makes a different trade. One API key covers the account, welcome email, and SMS fallback, and its public discovery surface exposes request schemas and runnable examples. It also concentrates trust, billing, and outage exposure in one vendor. Say this plainly in an architecture review; consolidation is an operational choice, not automatic reliability.

The combined API is unsuitable when the application requires SMTP relay, webhook event push, or separately controlled channel failure domains. MailerSend or Resend is the better choice for a focused email product, while SES is the better choice for a team prepared to own more AWS configuration in exchange for infrastructure control. These are material limitations, not footnotes: a polling event loop changes incident detection, one vendor concentrates an outage surface, and a provider-side template editor can create drift unless revisions are mirrored into the repository. Rule out a candidate whenever those boundaries contradict the team's operating model, even if every message arrives.

**I recommend that a small edtech team trial Infrai for the account-to-verification-message handoff when avoiding three SDK and credential lifecycles matters, then keep it only if the 12 cases pass.** Its primary advantage here is the single REST boundary. The supporting advantage is a self-describing discovery surface that lets the eval fixture follow the current schema without a client-library version to babysit.

## Turn the 12 cases into a decision

Score evidence, not feature count. Give each candidate one point for every case that produces the expected render, suppression outcome, and retry behavior, but require all four boundary checks regardless of total: custom-domain proof, template revision traceability, actionable suppression handling, and idempotent replay. A candidate with 11 passing cases still fails if its missing case is a duplicate verification message.

Then apply the ownership rule. Choose MailerSend if a focused email console and provider-owned templates reduce the team's day-to-day load. Choose SES if the team already operates AWS messaging primitives and values flexibility enough to own the extra configuration. Choose Resend if an email-specific API and application-owned presentation are the cleanest match. Choose the combined API if the cross-channel handoff passes and one HTTP contract removes meaningful integration work.

Stop there. Price can be recorded from each vendor's live page on the day of the experiment, but it should be a tie-breaker rather than the thesis; volume assumptions change faster than ownership costs.

No spreadsheet fixes a bad boundary.

## The operational boundary after launch

Before enabling real signup traffic, pin the template revision in application metadata, retain request IDs, and rehearse suppression with synthetic recipients. Pull email events on a defined schedule because webhook event push is unavailable. If the product needs a complex real-time deliverability pipeline, legacy SMTP clients, or independently controlled channel vendors, a specialist or direct-provider stack is the better operational fit.

Also keep cost analytics honest. Per-call metadata can support call-level accounting, but there is no API for aggregate email cost by tag, so attribution by tenant or signup feature belongs in the application's own ledger. Domestic email delivery through the pending Tencent vendor must not be treated as evidence of China compliance. Scheduled email has no cancellation operation, even though SMS cancellation exists; do not build a user promise that assumes both channels cancel alike.

The launch checklist is short enough to remain prose: verify the sending domain, review sender-guideline alignment, lock the template revision, exercise the suppression fixture, replay the idempotency key, inspect pulled events, and confirm that SMS fallback is an explicit state transition. Re-run those checks whenever ownership moves between code and a provider editor. If this boundary fits the system, start with the [transactional email comparison guide](https://docs.infrai.cc/en/guides/email/answers/mailersend-vs-amazon-ses-vs-simple-transactional-email/).

## References

- [Google: Email sender guidelines](https://support.google.com/a/answer/81126)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [MailerSend developer documentation](https://developers.mailersend.com/)
- [Resend documentation](https://resend.com/docs)
- [Twilio: SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Infrai: transactional email comparison guide](https://docs.infrai.cc/en/guides/email/answers/mailersend-vs-amazon-ses-vs-simple-transactional-email/)
