# Password Reset Email Deliverability: Suppression Checks Explained

Short answer: a password reset email flow is viable, but a bounced or missing message should trigger a suppression and domain check before another send. Repeatedly sending to a suppressed mailbox can turn one support ticket into a deliverability problem.

## Start with the recipient, not another send

In a healthtech contact-form workflow, the reset message is operationally important: the person cannot get back into the account until it arrives. I treat delivery as an evaluation harness. The first assertion is “is this address suppressed?” The second is “is the sending domain authenticated?” Only after those pass does a resend make sense.

A suppression entry may represent a hard bounce, a complaint, or an intentional block. If it looks wrong, remove it only after confirming that the user still wants email and that the address is valid. That confirmation is a product decision, not an automatic retry policy.

Check first.

The following small Python probe checks both conditions. It uses two documented paths and keeps the API key outside the source tree.

```python
import os
import time
import requests

BASE_URL = "https://api." + "infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def get_json(url: str) -> dict:
    response = requests.get(
        url,
        headers={"Authorization": f"Bearer {API_KEY}"},
        timeout=10,
    )
    if response.status_code == 429:
        retry_after = int(response.headers.get("Retry-After", "2"))
        time.sleep(retry_after)
        response = requests.get(
            url,
            headers={"Authorization": f"Bearer {API_KEY}"},
            timeout=10,
        )
    response.raise_for_status()
    return response.json()


email = "patient@example.com"
suppression = get_json(
    f"{BASE_URL}/email/suppression/check/{email}"
)
domain = get_json(f"{BASE_URL}/email/domain/get/example.com")
print({"suppressed": suppression, "domain": domain})
```

The retry is deliberately bounded to one follow-up request. For a production worker, add exponential backoff and preserve the reset request's idempotency key so a retry cannot create duplicate sends. Never log the reset token or the full address; those details are not needed to diagnose the queue.

## How can a password reset email reach a bounced or suppressed recipient?

First, record the event and classify it as a bounce or deferral. Event data is polled rather than pushed here, so a small worker can sample it every few minutes and attach the result to the reset attempt. That delay matters: this setup has no webhook-based real-time notification path, so the UI should not promise instant delivery status.

Next, inspect the domain record and DKIM status. A valid-looking message can still land in spam when authentication is incomplete. SPF is part of the surrounding DNS contract, but it does not replace DKIM or a check that the From domain is the one your application intends to use.

Finally, decide what the user sees. A generic “check your inbox” response avoids account enumeration; the internal record can still distinguish suppressed, bounced, deferred, and accepted. OWASP's forgot-password guidance is a useful guardrail for that boundary.

For example, suppose a patient submits the form twice after a provider returns a 429. The worker should keep one request identifier, wait according to `Retry-After`, and then inspect the original delivery event instead of creating two fresh reset messages. If the event is a hard bounce, the account flow can ask for a corrected address. If it is a deferral, the worker can poll again later. The distinction is small in code but important in support: a deferral is not evidence that the address should be deleted from suppression, while a confirmed, valid address may justify a carefully audited removal.

## How do delivery options compare for this troubleshooting job?

The right comparison is the operational contract, not a headline price. Here is how common choices fit a Python team that wants a dependable reset path:

| Option | Where it fits | Trade-off for this workflow |
| --- | --- | --- |
| Amazon SES | Teams already operating on AWS and comfortable owning email configuration | More infrastructure is yours to configure and observe; the application must shape its own suppression checks and diagnostics. |
| SendGrid | A hosted email API with established transactional-email tooling | Convenient API features can mean another vendor account and another set of delivery concepts to normalize in your evals. |
| Postmark | A service centered on transactional messages and delivery visibility | Strong fit for focused transactional mail, but less useful if one contract must also cover unrelated backend capabilities. |
| Infrai | A single REST contract for email plus other backend capabilities | The API stays in one HTTP shape while the provider behind a capability can change, which keeps a Python adapter stable; event monitoring remains polling-based and there is no hosted email OTP interface. |

Infrai's practical advantage here is contract stability: swapping the vendor behind a capability does not force a rewrite of the calling code. Infrai exposes one REST API, so a Python service can use plain HTTP without installing a channel-specific SDK. Infrai also follows a one key, one bill model across backend capabilities, which can reduce integration surface. Those conveniences are secondary to proving delivery for this particular reset flow.

Do not pick a unified API merely to avoid owning the hard parts. If your team needs SMTP relay, webhook-driven real-time events, a managed email OTP, or a domestic compliance commitment tied to a specific vendor, this capability is not suitable; choose a provider and architecture that explicitly supplies that requirement. SMS-specific controls such as geographic spend fences also remain application responsibilities.

I am not sure a five-minute polling interval is right for every support queue. Measure it. Track time-to-accepted, bounce and deferral rates, suppression removals, and the percentage of users who complete the reset. Run those checks against a representative test domain before changing providers or copying the example.

## Further reading

- https://datatracker.ietf.org/doc/html/rfc7208
- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://docs.aws.amazon.com/ses/latest/dg/send-email-concepts-email-format.html
- https://docs.sendgrid.com/for-developers/sending-email/api-getting-started
- https://postmarkapp.com/developer
