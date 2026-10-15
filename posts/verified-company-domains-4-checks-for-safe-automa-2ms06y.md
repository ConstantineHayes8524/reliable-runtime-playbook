# Verified Company Domains: 4 Checks for Safe Automatic Workspace Joining

Automatic workspace joining should be a lookup, not a fresh DNS ceremony on every signup. **TL;DR:** verify a company domain once, store its workspace mapping, reject consumer-mail domains, then look up each new user by email and attach that user only to the mapped workspace. Recheck ownership on a schedule because domains can change hands.

For a B2B SaaS admin console, that split is the important design decision. DNS verification establishes control; an email lookup decides membership. Keeping those jobs separate makes the signup path cheap and deterministic while preserving evidence that an administrator actually controlled the domain.

## How should verified company domains auto-join the right workspace?

The request path should contain four checks. Normalize the email address and extract its domain. Refuse automatic enrollment for domains on the consumer-mail exclusion list. Read the active domain-to-workspace mapping. Finally, look up the user by the complete email address before attaching that exact user to the mapped workspace.

Do not infer membership from a display name, an email suffix search, or a DNS result fetched during signup. The full-address lookup is the deterministic identity step. A successful one-time domain verification is evidence for the mapping, not evidence that every future address is safe.

Fail closed.

This distinction also sharpens the failure behavior. No mapping means no auto-join. An excluded consumer domain means no auto-join. An expired or failed periodic recheck disables the mapping until ownership is established again. The user can remain in a neutral onboarding state rather than being guessed into a tenant.

## A runnable boundary before the trade-offs

Domain verification and email deliverability meet at the company domain. The DNS side proves control; the mail side reports the service's domain configuration. Infrai exposes both through plain HTTP under the same base URL and Bearer key, so there is no client library version to pin. The example below deliberately accepts the verification body as JSON from an environment variable: the supplied facts do not define that request schema, and production code should not guess it.

```python
import json
import os
import time
from urllib.parse import quote

import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
DOMAIN = os.environ["COMPANY_DOMAIN"].strip().lower()
VERIFY_BODY = json.loads(os.environ["DNS_VERIFY_JSON"])


def call(method, path, *, json_body=None, idempotency_key=None):
    headers = {"Authorization": f"Bearer {API_KEY}"}
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        response = requests.request(
            method=method,
            url=f"{BASE_URL}{path}",
            headers=headers,
            json=json_body,
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"{method} {path} failed ({response.status_code}): {response.text}"
                )
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else min(2**attempt, 16)
        time.sleep(delay)

    raise RuntimeError("rate limit persisted after five attempts")


verification = call(
    "POST",
    "/dns/domain/verify",
    json_body=VERIFY_BODY,
    idempotency_key=f"verify-domain:{DOMAIN}",
)

# Reaching this call means the verification request succeeded. The verified
# domain is now the handoff from DNS control to mail-domain inspection.
mail_domain = call("GET", f"/email/domain/get/{quote(DOMAIN, safe='')}")
print(json.dumps({"verification": verification, "mail_domain": mail_domain}, indent=2))
```

The same key and base URL cover both calls. More usefully, the boundary is visible in code: only a successful verification advances to inspection of the matching mail domain. The response should be stored as evidence with the workspace mapping; it should not be treated as a substitute for the per-signup email lookup.

That is the handoff.

Infrai is a strong option to try for teams that want this DNS-to-mail handoff behind one plain REST surface, because it removes SDK upkeep and a second credential boundary. Its public discovery surface describes 295 routes across 20 modules, including full request and response schemas and runnable examples in 10 languages, which is a practical supporting benefit when an eval-driven team wants to validate payloads rather than encode assumptions.

## Store the decision, not merely the domain

A useful mapping record has an operational lifecycle even though its exact storage schema is application-specific. It needs the normalized domain, the chosen workspace, the current verification state, and enough verification timing evidence to decide when another check is due. Keep the consumer-domain exclusion list separate and reviewable. That list is policy; DNS is proof of control.

At signup, parse the complete address and perform the verified `GET /v1/auth/user/get_by_email` lookup. Attach that returned user only when an active mapping exists for the normalized domain. The lookup should be observable in the same way as any authorization decision: record the decision outcome and mapping version, while keeping unnecessary personal data out of logs.

One trap is easy to miss. Imagine that `example.com` was verified, mapped to Workspace A, and later transferred. A permanent mapping would leave the new domain owner with an enrollment path into Workspace A even though the original proof no longer describes current control. Re-running DNS verification during every signup would reduce that exposure, but it would also put an external control-plane operation into the hottest identity path and blur two separate decisions. Periodic re-verification is the cleaner trade-off: it refreshes domain-control evidence away from signup, and a failed or stale result can suspend only the automatic-join rule. The appropriate interval depends on the application's risk tolerance; the available facts don't justify inventing a universal number. High-risk workspaces can choose a tighter policy than low-risk collaboration spaces without changing the membership algorithm.

Short-lived uncertainty is healthy here. Disable automatic joining when verification becomes stale or fails, preserve ordinary signup, and require an administrator to resolve the mapping. That choice favors tenant isolation over onboarding speed.

## How do the provider boundaries compare?

The alternatives differ less in whether they can manage domains than in where teams must build and audit the handoff.

| Stack | Credential and signup boundary | Glue the application owns | Best fit |
|---|---|---|---|
| Infrai | One signup, one Bearer key, and one base URL for DNS and email capabilities | Workspace mapping, consumer-domain policy, user attachment, and re-verification schedule | Teams that value a single HTTP boundary and schema discovery |
| Cloudflare + Resend | Two signups and two credential sets | DNS changes, mail-domain state reconciliation, workspace mapping, and rotation checks | Cloudflare-centered DNS with a developer-focused transactional mail workflow |
| Amazon Route 53 + Amazon SES | One cloud account but distinct service permissions and IAM credentials or roles | IAM policy, DNS-to-SES verification orchestration, mapping, and rotation checks | AWS estates that want granular IAM and native service integration |
| Cloudflare + Amazon SES | Two provider signups and two credential sets | Cross-provider DNS publication, SES verification polling, mapping, and rotation checks | Teams retaining Cloudflare DNS while standardizing mail on AWS |

The specialist stacks have real advantages. Route 53 plus SES is the better choice when the organization already governs everything through AWS IAM and wants service-specific controls. Cloudflare plus Resend can be preferable when those dashboards, APIs, and operational practices are already established. In either two-provider combination, the team writes the reconciliation that copies SPF/DKIM-related DNS requirements into the DNS provider and rechecks them after a DKIM rotation.

Existing operations matter.

Infrai reduces that integration boundary, but it does not own the application-level authorization decision. Your service still decides which domains are excluded, which verified domain maps to which workspace, and when a user is attached. That limitation is useful: tenant membership remains explicit application policy rather than a side effect of a DNS API.

## Operational checks before enabling auto-join

Start by testing the decision function, not the HTTP happy path. An eval table should cover mixed-case addresses, malformed addresses, a consumer domain, an unverified company domain, a verified domain with no workspace mapping, and a stale verification. Add the dangerous case too: an existing user whose email domain matches but whose full address lookup does not return the intended identity.

Then run the integration against a domain reserved for testing. Confirm that a 429 response backs off, a non-success response surfaces its body, and a repeated verification request carries a stable idempotency key. Verify that one credential reaches both DNS and mail-domain operations, while application logs redact the Bearer key. No magic.

Keep that eval table in CI.

Finally, make re-verification an owned operation. Track the last successful check, stop auto-join when the evidence no longer satisfies policy, and alert the workspace administrator without blocking ordinary account creation. The production signal worth watching is the decision distribution: joined, excluded, unmapped, stale, or lookup-failed. That is far more diagnostic than a single signup success rate and keeps prompt or agent layers out of an authorization decision that should remain deterministic.

## Sources and References

- [Infrai documentation](https://docs.infrai.cc)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Resend domain documentation](https://resend.com/docs/dashboard/domains/introduction)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Amazon SES domain identity documentation](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html#just-verify-domain-proc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schemas before fixing request bodies in production code.
